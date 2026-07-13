# 核心 Agent 循环

> AIAgent 类的设计与实现 -- 会话管理、工具调用循环、迭代预算

---

## 目录

- [概述](#概述)
- [AIAgent 类](#aiagent-类)
- [初始化流程](#初始化流程)
- [chat 方法与 run_conversation](#chat-方法与-run_conversation)
- [工具调用循环](#工具调用循环)
- [迭代预算机制](#迭代预算机制)
- [系统提示词构建](#系统提示词构建)
- [会话持久化](#会话持久化)
- [错误处理与恢复](#错误处理与恢复)
- [流式输出](#流式输出)
- [子 Agent 与委托](#子-agent-与委托)

---

## 概述

`AIAgent` 是 Hermes Agent 的核心类，定义在 `run_agent.py` 中（约 463KB，9000+ 行）。
它管理完整的会话生命周期：

```
用户消息 -> 构建系统提示词 -> LLM 请求 -> 工具调用循环 -> 后处理 -> 返回响应
```

每个 AIAgent 实例代表一个独立会话，持有自己的：

- 消息历史 (`messages: List[Dict]`)
- 上下文压缩器 (`ContextCompressor`)
- 记忆管理器 (`MemoryManager`)
- 工具配置与状态
- 会话元数据

---

## AIAgent 类

### 构造函数

AIAgent 的构造函数接受大量参数，覆盖模型连接、工具配置、回调、会话管理等方面：

```python
class AIAgent:
    def __init__(
        self,
        # -- 模型连接 --
        base_url: str = None,           # API 端点
        api_key: str = None,            # API 密钥
        provider: str = None,           # 提供商标识
        model: str = "",                # 模型名称
        
        # -- 工具配置 --
        max_iterations: int = 90,       # 最大工具调用迭代数
        tool_delay: float = 1.0,        # 工具调用间隔（秒）
        enabled_toolsets: List[str] = None,   # 启用的工具集
        disabled_toolsets: List[str] = None,  # 禁用的工具集
        
        # -- 回调 --
        tool_progress_callback: callable = None,  # 工具进度回调
        stream_delta_callback: callable = None,   # 流式输出回调
        thinking_callback: callable = None,       # 思考过程回调
        clarify_callback: callable = None,        # 澄清请求回调
        
        # -- 会话管理 --
        session_id: str = None,          # 会话 ID
        session_db = None,               # SQLite 会话数据库
        parent_session_id: str = None,   # 父会话 ID（子 Agent）
        persist_session: bool = True,    # 是否持久化会话
        
        # -- 高级配置 --
        iteration_budget: "IterationBudget" = None,  # 迭代预算
        credential_pool = None,          # 凭证池（多 Key 轮换）
        fallback_model: Dict = None,     # 备用模型配置
        reasoning_config: Dict = None,   # 推理配置
        platform: str = None,            # 运行平台标识
        ...
    ):
```

### 核心属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `messages` | `List[Dict]` | 完整的消息历史 |
| `model` | `str` | 当前使用的模型 |
| `_context_compressor` | `ContextCompressor` | 上下文压缩器 |
| `_memory_manager` | `MemoryManager` | 记忆管理器 |
| `_tool_definitions` | `List[Dict]` | 当前可用的工具 schema |
| `_session_db` | `SessionDB` | SQLite 会话数据库 |
| `session_id` | `str` | 当前会话 ID |
| `total_input_tokens` | `int` | 累计输入 token 数 |
| `total_output_tokens` | `int` | 累计输出 token 数 |

---

## 初始化流程

AIAgent 初始化包含以下关键步骤：

```
AIAgent.__init__()
    |
    |-- 1. 解析模型连接参数 (base_url, api_key, provider)
    |
    |-- 2. 创建 OpenAI 客户端
    |       OpenAI(base_url=..., api_key=...)
    |
    |-- 3. 获取模型元数据
    |       fetch_model_metadata(model) -> 上下文长度、能力标识
    |
    |-- 4. 初始化上下文压缩器
    |       ContextCompressor(model, threshold_percent=0.50)
    |
    |-- 5. 初始化记忆管理器
    |       MemoryManager()
    |       add_provider(BuiltinMemoryProvider)
    |       add_provider(外部插件 Provider)  # 可选
    |
    |-- 6. 加载工具定义
    |       get_tool_definitions(enabled_toolsets, disabled_toolsets)
    |
    |-- 7. 初始化会话存储
    |       SessionDB (SQLite)
    |       创建或恢复会话
    |
    +-- 8. 构建初始系统提示词
            _build_system_prompt()
```

---

## chat 方法与 run_conversation

### chat() -- 简单接口

```python
def chat(self, message: str, stream_callback=None) -> str:
    """简单聊天接口，返回最终响应文本。"""
    result = self.run_conversation(message, stream_callback=stream_callback)
    return result["final_response"]
```

### run_conversation() -- 核心方法

`run_conversation()` 是 Agent 循环的核心入口，处理完整的一轮对话：

```
run_conversation(message)
    |
    |-- 1. 预处理用户消息
    |       - 斜杠命令检测
    |       - 图片/文件附件处理
    |
    |-- 2. 记忆预取
    |       memory_manager.prefetch_all(message)
    |       -> 将记忆上下文注入系统提示词
    |
    |-- 3. 构建完整的消息列表
    |       [system_prompt, ...history, user_message]
    |
    |-- 4. 上下文压缩检查（预飞行）
    |       if compressor.should_compress_preflight(messages):
    |           messages = compressor.compress(messages)
    |
    |-- 5. 调用 LLM API
    |       client.chat.completions.create(
    |           model=self.model,
    |           messages=messages,
    |           tools=self._tool_definitions,
    |           stream=True,
    |       )
    |
    |-- 6. 处理响应
    |       if 有 tool_calls:
    |           -> 进入工具调用循环
    |       else:
    |           -> 返回文本响应
    |
    |-- 7. 后处理
    |       - 保存到 SessionDB
    |       - 同步记忆 (memory_manager.sync_all)
    |       - 更新 token 计数
    |       - 触发标题生成（首轮对话）
    |
    +-- 8. 返回结果
            {
                "final_response": str,
                "messages": List[Dict],
                "usage": Dict,
            }
```

---

## 工具调用循环

当 LLM 返回 `tool_calls` 时，Agent 进入工具调用循环：

```
工具调用循环 (最多 max_iterations 次)
    |
    |-- while iteration < max_iterations:
    |       |
    |       |-- 1. 解析 tool_calls
    |       |       [{name: "terminal", arguments: {...}}, ...]
    |       |
    |       |-- 2. 逐个执行工具
    |       |       for tool_call in tool_calls:
    |       |           result = handle_function_call(
    |       |               name, args, task_id, user_task
    |       |           )
    |       |           -> 追加 tool result 到 messages
    |       |
    |       |-- 3. 上下文压缩检查
    |       |       if compressor.should_compress(prompt_tokens):
    |       |           messages = compressor.compress(messages)
    |       |
    |       |-- 4. 再次调用 LLM
    |       |       response = client.chat.completions.create(...)
    |       |
    |       |-- 5. 检查响应
    |       |       if 有新的 tool_calls:
    |       |           -> 继续循环
    |       |       elif 有文本响应:
    |       |           -> 跳出循环
    |       |
    |       +-- 6. 递增迭代计数
    |
    +-- 达到 max_iterations 时强制停止
```

### 工具执行的关键细节

| 细节 | 说明 |
|------|------|
| 异步桥接 | 异步工具通过持久事件循环执行，避免循环创建/关闭 |
| 工具延迟 | 每次工具调用后等待 `tool_delay` 秒（可配置） |
| 进度回调 | 通过 `tool_progress_callback` 通知 UI 层 |
| 错误捕获 | 工具异常被捕获并作为 `tool_error` 返回给 LLM |
| 审批机制 | 危险操作（rm、sudo 等）通过 `approval.py` 请求用户确认 |

---

## 迭代预算机制

`IterationBudget` 控制 Agent 在单次对话中可以执行的最大工具调用次数：

```python
class IterationBudget:
    """管理工具调用迭代预算。
    
    支持分级预算：
    - 主 Agent: 默认 90 次迭代
    - 子 Agent: 从父 Agent 分配预算
    - 网关 Agent: 可通过配置调整
    """
    max_iterations: int = 90
    used_iterations: int = 0
    
    def consume(self, n: int = 1) -> bool:
        """消耗迭代预算，返回是否仍有剩余。"""
        self.used_iterations += n
        return self.used_iterations < self.max_iterations
    
    def remaining(self) -> int:
        return max(0, self.max_iterations - self.used_iterations)
```

### 预算分配策略

```
主 Agent (budget=90)
    |
    |-- 委托子 Agent 1 (budget=从父级分配)
    |       消耗主 Agent 的预算
    |
    +-- 委托子 Agent 2 (budget=从父级分配)
            消耗主 Agent 的预算
```

---

## 系统提示词构建

系统提示词由 `agent/prompt_builder.py` 中的多个函数组装：

```
_build_system_prompt()
    |
    |-- 1. Agent 身份
    |       DEFAULT_AGENT_IDENTITY (默认身份)
    |       或 SOUL.md (自定义身份)
    |
    |-- 2. 平台提示
    |       PLATFORM_HINTS[platform]
    |       CLI / Telegram / Discord / 编辑器 等
    |
    |-- 3. 工具使用指导
    |       TOOL_USE_ENFORCEMENT_GUIDANCE (特定模型)
    |
    |-- 4. 记忆指导
    |       MEMORY_GUIDANCE
    |
    |-- 5. 技能索引
    |       build_skills_system_prompt()
    |       -> 扫描 skills/ 目录生成可用技能列表
    |
    |-- 6. 上下文文件
    |       build_context_files_prompt()
    |       -> 读取 AGENTS.md, .cursorrules 等
    |       -> 注入安全扫描 (_scan_context_content)
    |
    |-- 7. 记忆上下文
    |       memory_manager.build_system_prompt()
    |       + build_memory_context_block(prefetched)
    |
    |-- 8. 临时系统提示词
    |       ephemeral_system_prompt (网关/ACP 注入)
    |
    +-- 9. Nous 订阅提示词
            build_nous_subscription_prompt()
```

### 提示词安全

上下文文件（AGENTS.md 等）在注入前经过安全扫描：

```python
_CONTEXT_THREAT_PATTERNS = [
    (r'ignore\s+(previous|all|above)\s+instructions', "prompt_injection"),
    (r'do\s+not\s+tell\s+the\s+user', "deception_hide"),
    (r'curl\s+[^\n]*\$\{?\w*(KEY|TOKEN|SECRET)', "exfil_curl"),
    ...
]
```

检测到威胁时，内容会被替换为阻止消息，不会注入到提示词中。

---

## 会话持久化

### SQLite 会话存储

每次对话轮次完成后，Agent 将数据持久化到 `state.db`：

```python
# hermes_state.py 核心表结构
sessions:
    id, source, user_id, model, system_prompt,
    parent_session_id, started_at, ended_at,
    message_count, tool_call_count,
    input_tokens, output_tokens,
    estimated_cost_usd, title

messages:
    session_id, role, content, tool_call_id,
    tool_calls, tool_name, timestamp, token_count

messages_fts:  # FTS5 虚拟表
    content  -- 全文索引，支持快速搜索
```

### 会话链接

上下文压缩时会创建新的子会话，通过 `parent_session_id` 链接：

```
session_001 (原始会话)
    -> 上下文压缩触发
session_002 (parent=session_001, 包含摘要)
    -> 再次压缩
session_003 (parent=session_002, 包含更新摘要)
```

---

## 错误处理与恢复

### API 错误重试

```
API 调用
    |
    |-- 成功 -> 继续处理
    |
    |-- 速率限制 (429) -> 指数退避重试
    |
    |-- 上下文超长 (context_length_exceeded)
    |       -> 触发上下文压缩
    |       -> 重试
    |
    |-- 认证错误 (401/403)
    |       -> 凭证池切换下一个 Key
    |       -> 重试
    |
    |-- 模型不可用 (404)
    |       -> 尝试 fallback_model
    |
    +-- 其他错误 -> 记录日志，返回错误信息
```

### 上下文长度探测

当模型报告上下文超长时，Agent 会自动降级上下文长度并记录：

```python
# 从错误消息中解析实际上下文长度
limit = parse_context_limit_from_error(error_message)
if limit:
    save_context_length(model, limit)
    compressor.context_length = limit
```

---

## 流式输出

Agent 支持流式输出，通过回调机制通知上层：

```python
response = client.chat.completions.create(
    model=self.model,
    messages=messages,
    tools=self._tool_definitions,
    stream=True,
)

for chunk in response:
    delta = chunk.choices[0].delta
    
    if delta.content:
        # 文本内容
        stream_delta_callback(delta.content)
    
    if delta.tool_calls:
        # 工具调用（增量拼接）
        accumulate_tool_call(delta.tool_calls)
```

### 回调层次

| 回调 | 触发时机 | 消费者 |
|------|----------|--------|
| `stream_delta_callback` | 每个文本 token | CLI 终端输出 |
| `tool_start_callback` | 工具开始执行 | 进度显示 |
| `tool_progress_callback` | 工具执行中 | 状态更新 |
| `tool_complete_callback` | 工具完成 | 结果展示 |
| `thinking_callback` | 模型思考过程 | 思考链展示 |
| `step_callback` | 每个推理步骤 | ACP 事件推送 |

---

## 子 Agent 与委托

通过 `delegate_tool.py`，主 Agent 可以创建子 Agent 来处理独立任务：

```
主 Agent (session_001)
    |
    |-- delegate_task(task="审查这段代码", ...)
    |       |
    |       |-- 创建子 AIAgent
    |       |       session_id = new_uuid
    |       |       parent_session_id = session_001
    |       |       iteration_budget = 从父级分配
    |       |
    |       |-- 子 Agent.run_conversation(task)
    |       |
    |       +-- 返回子 Agent 的最终响应
    |
    +-- 将子 Agent 结果作为 tool_result 追加到消息历史
```

### 子 Agent 特性

| 特性 | 说明 |
|------|------|
| 独立会话 | 子 Agent 拥有独立的消息历史和 session_id |
| 预算共享 | 子 Agent 消耗父 Agent 的迭代预算 |
| 工具隔离 | 子 Agent 可以配置不同的工具集 |
| 线程执行 | 在 ThreadPoolExecutor 中运行，不阻塞主 Agent |
| 记忆通知 | 完成后通过 `on_delegation()` 通知记忆插件 |

---

## 相关文档

- [工具系统](tool-system.md) -- 工具注册与调度的详细设计
- [上下文压缩](context-compression.md) -- 压缩算法的深入分析
- [记忆系统](memory-system.md) -- 记忆的预取与同步机制
- [数据流图](data-flow.md) -- 完整的数据流可视化
