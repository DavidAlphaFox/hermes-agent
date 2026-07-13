# 子代理与用户交互模式

> 子代理无法直接问用户 -- 5 种应对方案与推荐组合

---

## 目录

- [问题与设计意图](#问题与设计意图)
- [为什么子代理不能直接问用户](#为什么子代理不能直接问用户)
- [5 种应对方案](#5-种应对方案)
- [方案对比表](#方案对比表)
- [推荐组合：方案 4 + 方案 1 改进版](#推荐组合方案-4--方案-1-改进版)
- [完整实现：request_clarification 工具](#完整实现request_clarification-工具)
- [完整执行流程示例](#完整执行流程示例)
- [关键改造点清单](#关键改造点清单)
- [关键不变量与陷阱](#关键不变量与陷阱)
- [参考](#参考)

---

## 问题与设计意图

### 问题

子代理在执行过程中遇到不确定点 -- 多个匹配项无法决定、操作有破坏性需要确认、信息不足无法继续 -- 需要询问用户。但 Hermes 子代理被刻意剥夺了 `clarify` 工具。

### 这是设计意图，不是 bug

`tools/delegate_tool.py:27-33`：

```python
DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task",   # 禁止递归委托
    "clarify",         # 禁止用户交互
    "memory",
    "send_message",
    "execute_code",
])
```

`tools/delegate_tool.py:302`：

```python
child = AIAgent(
    ...
    clarify_callback=None,        # 显式置空，即使工具被加上也无回调可用
    ...
)
```

双重防护。

---

## 为什么子代理不能直接问用户

| 原因 | 后果 |
|------|------|
| **僭越对话权** | 用户面对的是主代理，子代理直接发问会让用户困惑"谁在跟我说话" |
| **并发干扰** | 多个并行子代理同时发问，用户被打断 N 次，对话历史混乱 |
| **上下文不一致** | 用户回答会进入主代理上下文，但子代理拿不到完整上下文 |
| **无法路由** | 在 gateway 模式下，clarify 回答需要路由回正确的子代理实例 -- 工程复杂度爆炸 |

设计原则：**主代理是唯一与用户对话的实体，子代理只能"汇报"不能"打断"**。

---

## 5 种应对方案

### 方案 1：协议返回 + 父代理多轮派发（最 Hermes-idiomatic）

让子代理在遇到不确定点时**结构化返回**，主代理接收后用 `clarify` 问用户，拿到答案后**再派一个新子代理继续**。

#### 工作模式

```
主代理 → delegate("修改订单 #123 配送地址")
         |
         v
        子代理 #1 跑了 8 步，发现 3 个匹配客户
         |
         v
        子代理返回 {
          "status": "needs_clarification",
          "checkpoint": "...内部状态...",
          "question": "找到 3 个客户，请选择",
          "choices": ["张三 13800...", "张三 13900...", "李四 13700..."]
        }
         |
         v
主代理 <- 收到结构化结果
        |
主代理调 clarify(question, choices) -> 用户选了"张三 13800..."
        |
        v
主代理 -> delegate("继续修改订单 #123，已选客户 张三 13800...，
                  从这个 checkpoint 继续：...")
```

#### 子代理侧 - 在 system prompt 里约定协议

派发时在 `context` 里加一段协议说明：

```python
delegate_task(
    goal="修改订单 #123 的配送地址为'上海市浦东新区...'",
    context="""
    协议：当遇到以下情况时，停止执行并返回结构化 JSON：
    - 找到多个匹配项无法决定（用户/订单/地址歧义）
    - 操作有破坏性需要确认（取消订单/删除项目/批量修改）
    - 信息不足无法继续（必填字段缺失）

    返回格式（作为 final_response 的最后一段，用 ```clarify 围栏）：
    ```clarify
    {
      "question": "找到 3 个客户匹配'张三'，请选择",
      "choices": ["张三 138...", "张三 139...", "李四 137..."],
      "resume_context": "已完成步骤 1-3，订单查询返回 {...}，
                        待确认客户后执行步骤 4-6"
    }
    ```

    其他情况按正常流程完成。
    """,
    toolsets=["order", "delivery"],
)
```

#### 主代理侧 - 在 skill 里约定处理流程

```markdown
<!-- skills/business/delegate-with-clarify/SKILL.md -->
## 处理子代理的澄清请求

收到 delegate_task 结果后，扫描 final_response 末尾是否有 ```clarify 围栏：

1. 有 → 解析 JSON，调用 clarify(question=..., choices=...)
       拿到用户答案后，再次 delegate_task：
       - goal: "继续之前的任务，用户已选: {answer}"
       - context: "之前的 resume_context: {resume_context}, 用户选择: {answer}"

2. 无 → 任务完成，正常返回给用户
```

#### 优势

- **完全不改 Hermes 代码**
- 主代理保持唯一的对话权，用户体验一致
- 多个并行子代理的"澄清请求"由主代理排队询问，不会乱
- 子代理重启时通过 `resume_context` 继续

#### 代价

- **子代理重新启动有成本**（3-10 秒 + token）
- 子代理"重启后继续"靠 prompt 约束，不是真正的 checkpoint -- 模型可能重做某些步骤
- 协议约定靠 prompt，模型偶尔会忘记输出 `clarify` 围栏

#### 改进版：把协议固化成工具

为了让模型"忘不掉"协议，可以**新增一个 `request_clarification` 工具**给子代理（不是 `clarify` -- 它不直接问用户，而是把请求写入子代理的最终结果，由 `_run_single_child` 提升给主代理）。

完整实现见下文 [完整实现：request_clarification 工具](#完整实现request_clarification-工具) 章节。

---

### 方案 2：派发前预询问（最简单）

主代理在派发前**先用 clarify 把可能要问的都问清楚**，把答案塞到 context 里。

```python
# 主代理：检测到要修改订单，但用户没说改哪些字段
answer = clarify(
    question="要修改订单 #123 的什么？",
    choices=["配送地址", "配送日期", "商品数量", "取消订单"],
)

# 拿到答案后再派
delegate_task(
    goal=f"修改订单 #123 的{answer}",
    context=f"用户已确认要改: {answer}，新值: ...",
    toolsets=["order"],
)
```

#### 适用

- 不确定点**可预测**（订单系统就那几种修改类型）
- 业务流程相对固定

#### 不适用

- 不确定点出现在执行中途（如查询返回多个匹配项才知道要选）
- 需要根据中间结果动态决策

---

### 方案 3：放弃委派，改用主代理直接做

`delegate_task` 的 schema 描述（`tools/delegate_tool.py:849-852`）已经明确：

> **WHEN NOT TO USE: Tasks needing user interaction → subagents cannot use clarify**

如果任务**主线就需要频繁交互**（如"陪用户走完一个订单创建向导"），就**不要委派**，让主代理直接装 `order` toolset 自己做。

#### 适用

- 用户交互是任务核心（向导式流程、多步表单）
- 单域任务，工具加载量可控

#### 何时用 vs 用方案 1

| 特征 | 不委派（方案 3） | 委派 + 协议返回（方案 1） |
|------|------------------|--------------------------|
| 交互密度 | 每步都要问用户 | 偶尔问一次 |
| 推理负担 | 轻 | 重（调试、分析、综合） |
| 上下文压力 | 不会大量读文件 | 会大量读文件 / 调多个工具 |
| 并行需求 | 不需要 | 需要并行多个独立任务 |

---

### 方案 4：拆分任务边界 -- "不确定点"作为切分点

把原本一个跨域大任务，按"可能需要用户决定的点"拆成多个小任务：

```python
# 不要：一个大任务
# delegate("查找客户 → 创建订单 → 配置配送 → 发送通知")

# 而是：按决策点拆
result1 = delegate(goal="查找客户 张三", toolsets=["customer"])
# 主代理拿到匹配结果 → clarify 问用户选哪个

result2 = delegate(goal=f"为客户 {selected} 创建订单 ...", toolsets=["order"])
# 主代理拿到订单号 → clarify 问"是否同时安排配送？"

result3 = delegate(goal=f"为订单 {order_id} 安排配送 ...", toolsets=["delivery"])
```

主代理负责**编排**和**用户决策点**，子代理负责**单一域内的执行**。

#### 适用

- 业务流程能清晰拆出"决策点"
- 跨域工作流（典型企业场景）

---

### 方案 5：改用 hermes 子进程 + PTY（重场景才用）

`skills/autonomous-ai-agents/hermes-agent/SKILL.md:434-468` 描述的方案：用 `terminal(command="tmux ... 'hermes'")` 启动一个**完全独立的 Hermes 进程**，它有自己的 stdin/stdout，可以独立跟用户/操作员交互。

```bash
# 启动一个独立 hermes 实例
tmux new-session -d -s order-agent 'hermes'
tmux send-keys -t order-agent '帮我创建一个订单' Enter
```

#### 只在以下场景考虑

- 子任务持续数小时（远超 `delegate_task` 的分钟级预期）
- 子代理需要全套工具（不是 `delegate_task` 的工具子集）
- 真要"两个 agent 同时各自跟不同用户对话"

#### 绝不用于

普通业务场景（订单/配送）-- 杀鸡用牛刀。

---

## 方案对比表

| 方案 | 改 Hermes 代码 | 用户体验 | 复杂度 | 适用场景 |
|------|---------------|---------|--------|----------|
| 1. 协议返回 + 重派 | 不改（基础版） / 改一点（加 `request_clarification` 工具） | 好 | 中 | 不确定点偶尔出现 |
| 2. 派发前预询问 | 不改 | 好 | 低 | 不确定点可预测 |
| 3. 不委派，主代理做 | 不改 | 最好 | 低 | 交互密集型流程 |
| 4. 按决策点拆分 | 不改 | 好 | 中 | 跨域工作流 |
| 5. PTY 子进程 | 不改 | 复杂 | 高 | 长任务 / 多用户 |

---

## 推荐组合：方案 4 + 方案 1 改进版

针对典型企业业务场景（订单 / 配送 / 项目，跨域工作流，业务交互），用 **方案 4（按决策点拆分） + 方案 1 改进版（`request_clarification` 工具）** 组合：

- **方案 4** 负责"可预测的用户决策点" -- 主代理主动拆分
- **方案 1 改进版** 负责"中途意外出现的不确定点" -- 子代理上报

### 改造清单

1. **新增工具** `tools/request_clarification_tool.py`（约 50 行，参见上面方案 1 改进版）
2. **修改** `tools/delegate_tool.py:27-33`：确保 `request_clarification` 不在黑名单里（其工具集 `delegation_protocol` 也不在 `_strip_blocked_tools` 的 `blocked_toolset_names` 里）
3. **修改** `tools/delegate_tool.py` 的 `_run_single_child`（约 380-460 行）：在结果解析里识别 `_clarification_request` 标记，把它放到返回 entry 的顶层 `clarification` 字段
4. **新增 skill** `skills/business/delegate-with-clarify/SKILL.md`：描述主代理收到 `clarification` 后如何走流程
5. **业务流程拆分**：每个跨域工作流，画出"决策点"，按决策点切分 `delegate_task` 调用

---

## 完整实现：request_clarification 工具

本节给出可直接落地的完整代码，覆盖：

1. 新工具文件
2. `_run_single_child` 解析改动
3. `run_conversation` 终止检查（可选）
4. 主代理处理 skill

### 1. 新建 `tools/request_clarification_tool.py`

```python
#!/usr/bin/env python3
"""request_clarification 工具

允许子代理在执行过程中遇到不确定点时，把问题回传给主代理，
由主代理代理转问用户。本工具不会直接打断用户对话。

工作机制：
- 子代理调用 → 返回带 _clarification_request 标记的 JSON
- _run_single_child 识别标记，提升到结果 entry 的 clarification 字段
- 主代理收到结果，用自己的 clarify 工具问用户
- 主代理拿到答案后派新子代理，传入 resume_context + answer
"""

from __future__ import annotations

import json
from typing import Any, Dict, List, Optional


# 与 clarify 工具一致的最大选项数（UI 会自动追加 "Other" 选项）
MAX_CHOICES = 4


def request_clarification_tool(
    question: str,
    choices: Optional[List[str]] = None,
    resume_context: str = "",
    *,
    agent: Any = None,
) -> str:
    """子代理调用此工具上报澄清请求并准备结束本轮任务。

    Args:
        question: 要让用户回答的问题
        choices: 可选预设答案（最多 4 个）
        resume_context: 已完成步骤和中间状态描述，便于后续继续
        agent: 当前 Agent 实例（由 handler 通过 kwargs 注入），
               用于设置终止标记（可选优化）

    Returns:
        JSON 字符串。包含 _clarification_request: true 标记，
        _run_single_child 据此识别并提升到结果 entry。
    """
    if not question or not question.strip():
        return json.dumps(
            {"error": "question is required"}, ensure_ascii=False,
        )
    question = question.strip()

    if choices is not None:
        if not isinstance(choices, list):
            return json.dumps(
                {"error": "choices must be a list of strings"},
                ensure_ascii=False,
            )
        choices = [str(c).strip() for c in choices if str(c).strip()]
        if len(choices) > MAX_CHOICES:
            choices = choices[:MAX_CHOICES]
        if not choices:
            choices = None

    # 可选优化：在 agent 上设置标记，让对话循环立刻结束
    # 即使子代理"忘了"在调用后产出 final_response，
    # 这个标记也能让 run_conversation 提前 break。
    if agent is not None:
        try:
            agent._pending_clarification = {
                "question": question,
                "choices": choices,
                "resume_context": resume_context.strip(),
            }
        except Exception:
            pass

    # 返回给子代理对话循环的 tool result
    # 子代理看到 next_action="end_task" 后会按系统提示要求结束
    return json.dumps({
        "_clarification_request": True,
        "question": question,
        "choices": choices,
        "resume_context": resume_context.strip(),
        "next_action": "end_task",
        "instructions": (
            "Clarification request recorded. Now produce a brief final "
            "summary describing what you completed before this checkpoint, "
            "then stop. Do not call any more tools."
        ),
    }, ensure_ascii=False)


def check_request_clarification_requirements() -> bool:
    """无外部依赖，始终可用。"""
    return True


REQUEST_CLARIFICATION_SCHEMA = {
    "name": "request_clarification",
    "description": (
        "Use this tool when you are running as a subagent and encounter "
        "a situation that requires the user to decide. This does NOT "
        "interrupt the user directly -- the parent agent will receive "
        "your request and ask the user on your behalf, then start a new "
        "subagent with the user's answer to continue.\n\n"
        "WHEN TO USE:\n"
        "- Multiple matching results, can't decide which one\n"
        "- Destructive operation needs confirmation (cancel/delete/bulk-modify)\n"
        "- Required information is missing and cannot be inferred\n"
        "- Conflicting constraints discovered mid-task\n\n"
        "AFTER CALLING:\n"
        "- Produce a brief final summary of work done up to this checkpoint\n"
        "- Stop. Do not call any more tools.\n"
        "- The parent agent will resume with a fresh subagent.\n\n"
        "ALWAYS provide resume_context: a self-contained description of "
        "what was done, partial results, IDs, intermediate values -- "
        "so the next subagent can continue without redoing work."
    ),
    "parameters": {
        "type": "object",
        "properties": {
            "question": {
                "type": "string",
                "description": "The question to ask the user (parent will relay it).",
            },
            "choices": {
                "type": "array",
                "items": {"type": "string"},
                "maxItems": MAX_CHOICES,
                "description": (
                    "Up to 4 predefined answer choices. Omit for open-ended."
                ),
            },
            "resume_context": {
                "type": "string",
                "description": (
                    "Self-contained snapshot of progress: completed steps, "
                    "fetched IDs, partial results, what remains to do. "
                    "The next subagent will receive this in its context."
                ),
            },
        },
        "required": ["question", "resume_context"],
    },
}


# 注册为新 toolset，名字必须不在 _strip_blocked_tools 的黑名单里
from tools.registry import registry

registry.register(
    name="request_clarification",
    toolset="delegation_protocol",
    schema=REQUEST_CLARIFICATION_SCHEMA,
    handler=lambda args, **kw: request_clarification_tool(
        question=args.get("question", ""),
        choices=args.get("choices"),
        resume_context=args.get("resume_context", ""),
        agent=kw.get("agent"),  # 注入当前 agent 用于设置 pending 标记
    ),
    check_fn=check_request_clarification_requirements,
    emoji="❓",
)
```

**关键设计点**：

| 设计点 | 原因 |
|--------|------|
| 新 toolset 名 `delegation_protocol` | 不在 `_strip_blocked_tools` 的 `{"delegation", "clarify", "memory", "code_execution"}` 黑名单里，子代理可拿到 |
| 新工具名 `request_clarification` | 不在 `DELEGATE_BLOCKED_TOOLS` 里，不会被屏蔽 |
| handler 通过 `kw.get("agent")` 拿当前 agent | 设置 `_pending_clarification` 标记，让对话循环可提前终止 |
| 返回 `next_action: "end_task"` + `instructions` | 强化模型按规约结束 |

### 2. 修改 `tools/delegate_tool.py` 的 `_run_single_child`

在约 446 行（构造 entry 之前）插入扫描逻辑：

```python
# === 新增：扫描子代理消息里的 clarification 标记 ===
clarification = None
# 优先读 agent 上的 pending 标记（最可靠）
if hasattr(child, "_pending_clarification"):
    clarification = getattr(child, "_pending_clarification", None)

# 兜底：扫描 messages（如果 agent 没设标记，从 tool result 里找）
if clarification is None and isinstance(messages, list):
    for msg in reversed(messages):
        if not isinstance(msg, dict):
            continue
        if msg.get("role") != "tool":
            continue
        try:
            content_obj = json.loads(msg.get("content", "{}"))
        except (json.JSONDecodeError, TypeError):
            continue
        if isinstance(content_obj, dict) and content_obj.get("_clarification_request"):
            clarification = {
                "question": content_obj.get("question", ""),
                "choices": content_obj.get("choices"),
                "resume_context": content_obj.get("resume_context", ""),
            }
            break
# === 新增结束 ===

# 确定退出原因
if interrupted:
    exit_reason = "interrupted"
elif clarification is not None:           # ← 新增分支
    exit_reason = "needs_clarification"
elif completed:
    exit_reason = "completed"
else:
    exit_reason = "max_iterations"

# entry 构造
entry: Dict[str, Any] = {
    "task_index": task_index,
    "status": "needs_clarification" if clarification else status,  # ← 改
    "summary": summary,
    "api_calls": api_calls,
    "duration_seconds": duration,
    "model": _model if isinstance(_model, str) else None,
    "exit_reason": exit_reason,
    "tokens": {
        "input": _input_tokens if isinstance(_input_tokens, (int, float)) else 0,
        "output": _output_tokens if isinstance(_output_tokens, (int, float)) else 0,
    },
    "tool_trace": tool_trace,
}
if clarification:                          # ← 新增字段
    entry["clarification"] = clarification
if status == "failed":
    entry["error"] = result.get("error", "Subagent did not produce a response.")
```

**改动点对比**：

| 原行为 | 新行为 |
|--------|--------|
| `status` 取值：`completed / interrupted / failed` | 多一种 `needs_clarification` |
| `exit_reason` 取值：`completed / max_iterations / interrupted` | 多一种 `needs_clarification` |
| entry 字段：固定 9 个 | 多一个可选 `clarification: {question, choices, resume_context}` |

### 3. 让 agent 对话循环识别 `_pending_clarification` 标记（可选但推荐）

如果想让子代理调用 `request_clarification` 后**立刻**结束（不依赖模型自觉收尾），在 `run_agent.py` 的 `run_conversation` 主循环里，每次工具调用之后加一个检查：

```python
# 伪代码 -- 在工具调用执行完后，准备进入下一轮 LLM 调用之前
for tool_call in tool_calls:
    handler(tool_call.args, agent=self, ...)
    # ... 处理 tool result ...

# 新增：检查是否有 pending clarification
if getattr(self, "_pending_clarification", None) is not None:
    # 让模型最后产出一个简短摘要后退出
    messages.append({
        "role": "user",
        "content": "[System: A clarification request has been recorded. "
                   "Produce a 1-2 sentence summary of work completed so far, "
                   "then this conversation will end.]",
    })
    # 让模型再跑一轮产出 final_response
    response = self._call_llm(messages, tools=[])  # 不给工具，强制收尾
    final_response = response.choices[0].message.content
    completed = True
    break  # 退出主循环
```

**这一步可选** -- 如果不改，仍然能工作（依赖系统提示词 + 工具返回的 `instructions` 约束），但可能多 1-2 轮无效迭代。

### 4. 新建 skill `skills/business/delegate-with-clarify/SKILL.md`

```markdown
---
name: delegate-with-clarify
description: 派发子代理时遇到澄清请求的处理流程
metadata:
  hermes:
    tags: [delegation, clarification, workflow]
---

# 处理子代理的澄清请求

## 何时使用

任何使用 delegate_task 派发子任务的场景，子代理可能在执行中遇到
不确定点。在派发前必须把 `delegation_protocol` 工具集加入 toolsets，
让子代理可以调用 request_clarification。

## 派发模板

```python
delegate_task(
    goal="<具体任务>",
    context="""
    <业务上下文>

    PROTOCOL: 如果遇到以下情况，调用 request_clarification 上报：
    - 多个匹配项无法决定
    - 破坏性操作需要确认
    - 必填信息缺失或冲突

    上报后立刻产出一句话总结并结束，不要继续调工具。
    """,
    toolsets=["<业务工具集>", "delegation_protocol"],   # 关键
)
```

## 处理流程

收到 delegate_task 结果后：

1. 解析 results 数组，对每个 entry：
   - if entry.status == "completed" → 任务成功，整合结果
   - if entry.status == "needs_clarification" → 走澄清循环（见下）
   - if entry.status == "failed" → 错误处理

2. 澄清循环：
   ```python
   while entry.status == "needs_clarification":
       clar = entry["clarification"]

       # 用主代理的 clarify 工具问用户
       answer_json = clarify(
           question=clar["question"],
           choices=clar.get("choices"),
       )
       answer = json.loads(answer_json)["user_response"]

       # 派新子代理继续，把状态和答案完整传过去
       result = delegate_task(
           goal=f"继续之前的任务。用户对'{clar['question']}'的回答: {answer}",
           context=f"前一阶段已完成的状态:\n{clar['resume_context']}\n\n"
                   f"用户决定: {answer}\n\n"
                   f"请基于此继续完成原任务。",
           toolsets=<原 toolsets>,
       )
       entry = json.loads(result)["results"][0]
   ```

3. 循环退出后，按最终 entry.status 处理。

## 注意事项

- 同一任务最多走 3 轮澄清，避免无限循环
- 用户选 "Other" 自由输入时，answer 是用户原文
- 如果 resume_context 不足以让新子代理继续，可以在 context 里补充
- 并行 tasks 模式下，多个子代理可能同时上报澄清 -- 主代理逐个处理
```

---

## 完整执行流程示例

### 场景

> 用户："帮我修改张三的订单为下周二配送到上海"

### 第 1 轮派发

主代理调用：

```python
delegate_task(
    goal="查找名为'张三'的所有未完成订单，将配送日期改为 2026-05-19，"
         "配送地址改为'上海'，重新计算配送路径",
    context="""
    业务上下文：
    - 订单系统 API：通过 query_orders / modify_order 工具
    - 配送系统 API：通过 update_delivery_path 工具

    PROTOCOL: 如果遇到多个客户匹配 / 路径无解 / 字段冲突，
    调用 request_clarification 上报，附 resume_context 描述已完成步骤。
    上报后立刻产出一句话总结并结束。
    """,
    toolsets=["order", "delivery", "delegation_protocol"],
)
```

### 子代理执行轨迹

```
turn 1: query_orders(customer_name="张三")
  → 返回 [{id: 123, ...}, {id: 145, ...}, {id: 178, ...}]

turn 2: 模型推理 - 张三有 3 个订单，不确定要改哪些
  → 调用 request_clarification(
       question="找到张三的 3 个未完成订单，要修改哪些？",
       choices=["全部改", "#123", "#145", "#178"],
       resume_context="已查询客户'张三'，找到 3 个未完成订单：
                      #123 (当前配送日 2026-05-20，深圳)
                      #145 (当前配送日 2026-05-22，广州)
                      #178 (当前配送日 2026-05-25，杭州)
                      待确认要修改哪些订单。"
     )

handler 返回:
  {
    "_clarification_request": true,
    "question": "...",
    "choices": [...],
    "resume_context": "...",
    "next_action": "end_task",
    "instructions": "Now produce a brief final summary..."
  }
  同时 child._pending_clarification = {...}

turn 3: 按指令产出最终摘要
  → "已查询到张三的 3 个未完成订单，等待用户决定修改范围。"
  → run_conversation 检测到 _pending_clarification，结束
```

### `_run_single_child` 返回

```python
{
  "task_index": 0,
  "status": "needs_clarification",      # 新状态
  "summary": "已查询到张三的 3 个未完成订单，等待用户决定修改范围。",
  "exit_reason": "needs_clarification",
  "clarification": {                     # 新字段
    "question": "找到张三的 3 个未完成订单，要修改哪些？",
    "choices": ["全部改", "#123", "#145", "#178"],
    "resume_context": "已查询客户'张三'，找到 3 个未完成订单：..."
  },
  "tokens": {"input": 1234, "output": 156},
  "tool_trace": [...]
}
```

### 主代理处理

```
主代理收到 result，按 skill 处理流程：
  detect status == "needs_clarification"

主代理调 clarify(
  question="找到张三的 3 个未完成订单，要修改哪些？",
  choices=["全部改", "#123", "#145", "#178"],
)
  → CLI 弹出方向键选择 / Telegram 发编号列表
  → 用户选了 "#123"

主代理派新子代理：
delegate_task(
    goal="继续之前的任务。用户对'要修改哪些订单'的回答: #123",
    context="""
    前一阶段已完成的状态：
    已查询客户'张三'，找到 3 个未完成订单：
    #123 (当前配送日 2026-05-20，深圳)
    #145 (...)
    #178 (...)

    用户决定: 只改 #123

    请：
    1. modify_order(order_id=123, delivery_date='2026-05-19', address='上海')
    2. update_delivery_path(order_id=123)
    3. 返回完成摘要
    """,
    toolsets=["order", "delivery", "delegation_protocol"],
)
```

### 第 2 轮子代理

```
turn 1: modify_order(order_id=123, ...) → ok
turn 2: update_delivery_path(order_id=123) → 路径计算成功
turn 3: 模型产出 final_response
  → "已修改订单 #123：配送日期 2026-05-19，地址上海，
     新配送路径: 上海仓库 → 浦东配送站 → 收货地址，预计耗时 2.5 小时"

返回 status: "completed"
```

### 主代理整合返回给用户

> "已修改张三的订单 #123：配送日期 2026-05-19，地址上海，新路径预计耗时 2.5 小时。"

---

## 关键改造点清单

| 改造项 | 位置 | 改 / 不改 | 说明 |
|--------|------|----------|------|
| 新工具文件 | `tools/request_clarification_tool.py` | **新增** | 约 110 行（含 schema 与注册） |
| 注册新 toolset `delegation_protocol` | 同上 | 自动 | 由 `registry.register(toolset="delegation_protocol", ...)` 触发 |
| `DELEGATE_BLOCKED_TOOLS` | `tools/delegate_tool.py:27-33` | **不改** | 新工具名不在黑名单 |
| `_strip_blocked_tools` | `tools/delegate_tool.py:105-110` | **不改** | 新 toolset 名不在 `blocked_toolset_names` |
| `_run_single_child` 解析 | `tools/delegate_tool.py:~440-460` | **改** | 约 25 行新增（扫描标记 + entry 字段提升） |
| `run_conversation` 终止检查 | `run_agent.py` | **可选** | 加 `_pending_clarification` 检查，让子代理提前结束 |
| Skill 文件 | `skills/business/delegate-with-clarify/SKILL.md` | **新增** | 描述主代理处理 clarification 上报的流程 |
| 主代理 toolsets 配置 | `cli-config.yaml` | **改** | 加 `clarify` 工具集（如未有） |
| 派发调用方 context | 各业务 skill | **改** | 在 `context` 中加入协议说明，在 `toolsets` 加入 `delegation_protocol` |

---

## 关键不变量与陷阱

| 注意点 | 原因 |
|--------|------|
| 新 toolset 名**不要叫 `clarify`** | 会被 `_strip_blocked_tools` 干掉 |
| 新工具名**不要叫 `clarify`** | 会被 `DELEGATE_BLOCKED_TOOLS` 屏蔽 |
| handler 必须通过 `kw.get("agent")` 拿 agent | 而不是 `parent_agent` -- 那是 `delegate_task` 专用名 |
| `resume_context` 必须自包含 | 新子代理对前一个一无所知 |
| 主代理处理循环要加最大轮数 | 防止 LLM 反复要求澄清陷入死循环 |
| 并行 tasks 时多个 clarification 要排队问 | 主代理对话权是单通道 |
| 新工具 handler 不要做敏感操作 | 应该是纯函数 + agent 标记设置，不做 I/O |
| `_pending_clarification` 字段必须在新一轮派发前清空 | 否则子代理刚启动就被识别为"待澄清"状态 |

---

## 参考

- [委派系统](delegation-system.md) -- delegate_task 工具与子代理隔离机制
- [委派系统 - 子代理工具剥夺](delegation-system.md#子代理工具剥夺) -- 黑名单详情
- [多业务工具场景适配指南](business-tools-adaptation.md) -- 业务工具组织
- [技能系统](skill-system.md) -- skill 发现与加载
