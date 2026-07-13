# 上下文压缩机制

> 触发条件、压缩算法、结构化摘要、会话链接与迭代更新

---

## 目录

- [概述](#概述)
- [设计目标](#设计目标)
- [触发条件](#触发条件)
- [压缩算法](#压缩算法)
- [工具输出裁剪](#工具输出裁剪)
- [结构化摘要](#结构化摘要)
- [迭代摘要更新](#迭代摘要更新)
- [会话链接](#会话链接)
- [Token 预算计算](#token-预算计算)
- [孤立工具调用清理](#孤立工具调用清理)
- [上下文长度探测](#上下文长度探测)
- [配置参数](#配置参数)

---

## 概述

`ContextCompressor` (`agent/context_compressor.py`, 30KB) 实现了自动上下文窗口压缩，当对话长度接近模型的上下文限制时自动触发。其核心思想是**保留头尾、摘要中间**：

```
压缩前:
+------------------------------------------------------------------+
| [系统提示词] [第1轮] [第2轮] ... [第N-5轮] ... [第N轮]              |
|              ^                    ^              ^                |
|              头部保护区            中间区域        尾部保护区        |
+------------------------------------------------------------------+

压缩后:
+------------------------------------------------------------------+
| [系统提示词] [第1轮] [结构化摘要] [第N-3轮] [第N-2轮] [第N轮]       |
|              ^        ^                       ^                  |
|              保留      LLM生成的摘要            保留最近的上下文    |
+------------------------------------------------------------------+
```

---

## 设计目标

| 目标 | 实现方式 |
|------|----------|
| 保留关键上下文 | 头部保护（系统提示词 + 首轮对话）+ 尾部 token 预算保护 |
| 最小化信息丢失 | 结构化摘要模板，保留目标/进度/决策/文件/下一步 |
| 迭代稳定性 | 多次压缩时更新摘要而非重写，保留历史压缩信息 |
| 低成本 | 使用辅助模型（便宜模型）执行摘要，非主模型 |
| 双层压缩 | 先裁剪工具输出（零 LLM 成本），再 LLM 摘要 |
| 容错 | 摘要失败时回退到纯裁剪，设置冷却期避免反复失败 |

---

## 触发条件

### 阈值检测

```python
class ContextCompressor:
    def __init__(self, model, threshold_percent=0.50, ...):
        self.context_length = get_model_context_length(model)
        self.threshold_tokens = int(self.context_length * threshold_percent)
```

默认阈值：**上下文长度的 50%**。

### 检测时机

上下文压缩在两个时机检测：

```
1. 预飞行检查 (API 调用前)
   should_compress_preflight(messages)
       -> 使用粗略 token 估算
       -> 避免发送注定失败的请求

2. 响应后检查 (API 返回后)
   should_compress(prompt_tokens)
       -> 使用 API 返回的实际 token 数
       -> 更精确
```

```python
def should_compress(self, prompt_tokens=None) -> bool:
    """检查上下文是否超过压缩阈值。"""
    tokens = prompt_tokens or self.last_prompt_tokens
    return tokens >= self.threshold_tokens

def should_compress_preflight(self, messages) -> bool:
    """使用粗略估算的预飞行检查。"""
    rough_estimate = estimate_messages_tokens_rough(messages)
    return rough_estimate >= self.threshold_tokens
```

### 上下文利用率示例

| 模型 | 上下文长度 | 50% 阈值 | 触发时 |
|------|-----------|----------|--------|
| Claude Sonnet | 200K | 100K tokens | ~250 页对话 |
| GPT-4o | 128K | 64K tokens | ~160 页对话 |
| Llama 70B | 128K | 64K tokens | ~160 页对话 |
| 本地模型 | 8K | 4K tokens | ~10 页对话 |

---

## 压缩算法

### 完整流程

```
compress(messages, current_tokens)
    |
    |-- 1. 消息数量检查
    |       if len(messages) <= protect_first + protect_last + 1:
    |           return messages  # 太少，不压缩
    |
    |-- 2. Phase 1: 工具输出裁剪 (无 LLM 调用)
    |       _prune_old_tool_results(messages, protect_tail_count)
    |       -> 将老旧的工具结果替换为占位符
    |       -> 仅裁剪 >200 字符的工具输出
    |
    |-- 3. 确定保护区域
    |       头部: messages[0:protect_first_n]  (默认前 3 条)
    |       尾部: 按 token 预算从末尾向前计算
    |
    |-- 4. 提取中间区域
    |       middle = messages[head_end:tail_start]
    |
    |-- 5. Phase 2: LLM 摘要 (使用辅助模型)
    |       summary = _summarize_turns(middle)
    |       -> 调用便宜模型生成结构化摘要
    |
    |-- 6. 组装压缩后的消息
    |       result = head + [summary_message] + tail
    |
    |-- 7. 清理孤立工具调用
    |       _cleanup_orphaned_tool_calls(result)
    |
    |-- 8. 更新压缩计数
    |       self.compression_count += 1
    |
    +-- 9. 返回压缩后的消息列表
```

### 区域划分示意

```
messages[0]  messages[1]  messages[2]  messages[3] ... messages[N-5] ... messages[N]
|<----------- 头部保护 ----------->|   |<--- 中间 (被摘要) --->|  |<-- 尾部保护 -->|
        protect_first_n=3                                       tail_token_budget
```

---

## 工具输出裁剪

Phase 1 是一个零成本的预处理步骤，在 LLM 摘要之前执行：

```python
def _prune_old_tool_results(self, messages, protect_tail_count):
    """将老旧工具结果替换为占位符。
    
    从后向前遍历，保护最近的 protect_tail_count 条消息。
    对更早的 tool 角色消息，若内容 >200 字符则替换。
    """
    result = [m.copy() for m in messages]
    prune_boundary = len(result) - protect_tail_count
    
    for i in range(prune_boundary):
        msg = result[i]
        if msg.get("role") != "tool":
            continue
        content = msg.get("content", "")
        if len(content) > 200:
            result[i] = {**msg, "content": _PRUNED_TOOL_PLACEHOLDER}
    
    return result, pruned_count
```

占位符文本:

```
[Old tool output cleared to save context space]
```

### 为什么先裁剪工具输出？

| 原因 | 说明 |
|------|------|
| 零成本 | 不需要 LLM 调用 |
| 高收益 | 工具输出通常占上下文的大部分（终端输出、网页内容等） |
| 保留语义 | 工具名称和调用参数保留，模型仍能理解做了什么 |
| 减少摘要输入 | 更少的内容 -> 更快更便宜的摘要 |

---

## 结构化摘要

### 摘要提示模板

当需要 LLM 摘要时，使用结构化模板指导摘要生成：

```
Goal:        当前对话的整体目标
Progress:    已完成的工作步骤
Decisions:   做出的关键决策和原因
Files:       涉及的文件路径和修改内容
Next Steps:  下一步计划
```

### 摘要前缀

生成的摘要会加上特殊前缀，告知模型这是压缩后的上下文：

```python
SUMMARY_PREFIX = (
    "[CONTEXT COMPACTION] Earlier turns in this conversation were compacted "
    "to save context space. The summary below describes work that was "
    "already completed, and the current session state may still reflect "
    "that work (for example, files may already be changed). Use the summary "
    "and the current state to continue from where things left off, and "
    "avoid repeating work:"
)
```

### 摘要 Token 预算

摘要的长度根据被压缩内容的量动态调整：

```python
_MIN_SUMMARY_TOKENS = 2000       # 最小摘要 token
_SUMMARY_RATIO = 0.20            # 压缩内容的 20%
_SUMMARY_TOKENS_CEILING = 12000  # 绝对上限

def _compute_summary_budget(self, turns_to_summarize):
    content_tokens = estimate_messages_tokens_rough(turns_to_summarize)
    budget = int(content_tokens * _SUMMARY_RATIO)
    return max(_MIN_SUMMARY_TOKENS, min(budget, self.max_summary_tokens))
```

| 压缩内容 | 摘要预算 | 说明 |
|----------|----------|------|
| 5,000 tokens | 2,000 tokens | 使用最小值 |
| 20,000 tokens | 4,000 tokens | 20% 比例 |
| 80,000 tokens | 12,000 tokens | 使用上限 |

---

## 迭代摘要更新

当对话再次触发压缩时，新摘要会基于上一次的摘要进行迭代更新，而非完全重写：

```
第一次压缩:
    input = 中间区域的原始对话
    output = 初始摘要 S1

第二次压缩:
    input = 上次摘要 S1 + 新的中间区域对话
    output = 更新后的摘要 S2 (包含 S1 的信息 + 新信息)

第三次压缩:
    input = 上次摘要 S2 + 新的中间区域对话
    output = 更新后的摘要 S3
```

```python
# 存储上一次的摘要用于迭代更新
self._previous_summary: Optional[str] = None

# 摘要生成后更新
self._previous_summary = new_summary
```

### 迭代更新的优势

| 优势 | 说明 |
|------|------|
| 信息保留 | 早期对话的关键信息通过摘要链传递 |
| 一致性 | 模型看到的是渐进式更新，而非突然的上下文切换 |
| 效率 | 每次只需要摘要新增的中间区域 |

---

## 会话链接

上下文压缩触发时，系统会创建一个新的子会话：

```
session_001 (原始会话, 消息 1-100)
    |
    |-- 上下文压缩触发 (第 50 条消息时)
    |
    v
session_002 (parent=session_001)
    包含: [摘要(消息 1-70)] + [消息 71-100]
    |
    |-- 再次压缩 (第 150 条消息时)
    |
    v
session_003 (parent=session_002)
    包含: [更新摘要(消息 1-130)] + [消息 131-150]
```

### 在 SQLite 中的表示

```sql
-- sessions 表
| id          | parent_session_id | started_at | message_count |
|-------------|-------------------|------------|---------------|
| session_001 | NULL              | 1700000000 | 100           |
| session_002 | session_001       | 1700001000 | 80            |
| session_003 | session_002       | 1700002000 | 50            |
```

通过 `parent_session_id` 链可以追溯完整的对话历史。

---

## Token 预算计算

### 尾部保护预算

尾部保护使用 token 预算而非固定消息数量：

```python
# 尾部 token 预算 = 阈值 * 摘要目标比例
self.tail_token_budget = int(self.threshold_tokens * self.summary_target_ratio)
```

这意味着在大上下文窗口的模型上，尾部保护区域更大：

| 模型 | 上下文 | 阈值 (50%) | 尾部预算 (20%) | 约等于 |
|------|--------|-----------|---------------|--------|
| Claude Sonnet | 200K | 100K | 20K tokens | ~50 页 |
| GPT-4o | 128K | 64K | 12.8K tokens | ~32 页 |
| 本地 8K | 8K | 4K | 800 tokens | ~2 页 |

### 摘要长度上限

```python
self.max_summary_tokens = min(
    int(self.context_length * 0.05),  # 上下文长度的 5%
    _SUMMARY_TOKENS_CEILING,          # 绝对上限 12K
)
```

---

## 孤立工具调用清理

压缩后可能出现孤立的 tool_call 或 tool_result（配对的另一半被删除）。清理逻辑确保 API 不会收到不匹配的 ID：

```
压缩前:
    assistant: tool_calls=[{id: "call_1"}, {id: "call_2"}]
    tool: tool_call_id="call_1", content="result_1"
    tool: tool_call_id="call_2", content="result_2"

如果 call_1 的 result 被摘要吞掉:
    assistant: tool_calls=[{id: "call_1"}, {id: "call_2"}]
    [摘要]
    tool: tool_call_id="call_2", content="result_2"

清理后:
    assistant: tool_calls=[{id: "call_2"}]  # 移除孤立的 call_1
    [摘要]
    tool: tool_call_id="call_2", content="result_2"
```

---

## 上下文长度探测

当 API 返回上下文超长错误时，压缩器会自动调整：

```python
# 从错误消息中解析实际上下文长度
limit = parse_context_limit_from_error(error_message)
if limit:
    save_context_length(model, limit)
    self.context_length = limit
    self.threshold_tokens = int(limit * self.threshold_percent)
```

### 渐进式探测

```python
def get_next_probe_tier(current_length):
    """根据当前长度返回下一个探测层级。
    
    从大到小逐步降低:
    200K -> 128K -> 64K -> 32K -> 16K -> 8K -> 4K
    """
```

---

## 配置参数

### ContextCompressor 构造参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `model` | (必填) | 模型名称，用于确定上下文长度 |
| `threshold_percent` | 0.50 | 触发压缩的阈值（上下文长度百分比） |
| `protect_first_n` | 3 | 保护的头部消息数量 |
| `protect_last_n` | 20 | 保护的尾部消息数量（回退值） |
| `summary_target_ratio` | 0.20 | 摘要目标比例 (0.10 - 0.80) |
| `quiet_mode` | False | 静默模式（抑制日志） |
| `summary_model_override` | None | 覆盖摘要使用的模型 |
| `config_context_length` | None | 手动指定上下文长度 |

### 运行时状态

```python
def get_status(self) -> Dict[str, Any]:
    """获取当前压缩状态。"""
    return {
        "last_prompt_tokens": self.last_prompt_tokens,
        "threshold_tokens": self.threshold_tokens,
        "context_length": self.context_length,
        "usage_percent": ...,
        "compression_count": self.compression_count,
    }
```

### 失败冷却

摘要失败后设置 10 分钟冷却期，避免反复调用失败的辅助模型：

```python
_SUMMARY_FAILURE_COOLDOWN_SECONDS = 600  # 10 分钟

# 摘要失败时
self._summary_failure_cooldown_until = time.time() + _SUMMARY_FAILURE_COOLDOWN_SECONDS

# 冷却期内跳过 LLM 摘要，只执行工具输出裁剪
```

---

## 相关文档

- [核心 Agent 循环](core-agent-loop.md) -- 压缩在对话循环中的触发位置
- [记忆系统](memory-system.md) -- 压缩前的 `on_pre_compress` 钩子
- [数据流图](data-flow.md) -- 压缩数据流
- [架构概述](architecture-overview.md) -- 辅助模型的设计决策
