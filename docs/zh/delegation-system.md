# 委派系统设计

> delegate_task 工具、子代理构建、并行执行、深度限制与隔离机制

---

## 目录

- [概述](#概述)
- [核心设计原则](#核心设计原则)
- [工具 schema 与两种模式](#工具-schema-与两种模式)
- [深度限制：禁止多级委派](#深度限制禁止多级委派)
- [子代理工具剥夺](#子代理工具剥夺)
- [子代理构建流程](#子代理构建流程)
- [父代理与子代理：同一个类，不同配置](#父代理与子代理同一个类不同配置)
- [子代理如何与用户交互（间接）](#子代理如何与用户交互间接)
- [并行执行机制](#并行执行机制)
- [子代理执行与结果回传](#子代理执行与结果回传)
- [父子上下文流动：单向、精选、隔离](#父子上下文流动单向精选隔离)
- [凭据与模型路由](#凭据与模型路由)
- [中断与取消传播](#中断与取消传播)
- [与记忆系统的集成](#与记忆系统的集成)
- [配置参数](#配置参数)
- [何时委派任务给子代理](#何时委派任务给子代理)
- [设计取舍与替代方案](#设计取舍与替代方案)
- [关键文件速查](#关键文件速查)

---

## 概述

Hermes Agent 通过 `delegate_task` 工具支持**单层并行委派**：父代理可以同时派发最多 3 个子代理到隔离上下文中并行工作，子代理完成后只把最终摘要回传给父代理 -- 中间工具结果不污染父代理的上下文窗口。

**核心限制**：**不支持多级嵌套委派**。父代理（depth=0）→ 子代理（depth=1）→ 第三层会被硬性拒绝。这是为防止递归爆炸刻意做出的设计取舍。

实现位于 `tools/delegate_tool.py`（约 970 行），通过 `ThreadPoolExecutor` 在父代理进程内启动子 `AIAgent` 实例。

---

## 核心设计原则

| 原则 | 实现 |
|------|------|
| 上下文隔离 | 每个子代理拥有独立对话历史，父代理只看到最终摘要 |
| 终端隔离 | 每个子代理拥有独立终端会话（独立 cwd 与状态） |
| 并行有上限 | 单次委派最多 3 个子任务（`MAX_CONCURRENT_CHILDREN=3`） |
| 深度有上限 | 委派最多 1 级嵌套（`MAX_DEPTH=2`） |
| 工具有限制 | 子代理被剥夺敏感工具（递归委派、用户交互、记忆写入、跨平台消息） |
| 凭据可路由 | 子代理可以走与父代理不同的 provider:model（如父用 Opus、子用 Gemini Flash） |
| 中断可传播 | 父代理被中断时，所有活跃子代理同步中断 |

---

## 工具 schema 与两种模式

`tools/delegate_tool.py:834-949` 定义 `DELEGATE_TASK_SCHEMA`：

| 模式 | 触发参数 | 行为 |
|------|---------|------|
| **单任务模式** | `goal` (+ 可选 `context`, `toolsets`) | 直接同步运行单个子代理（无线程池开销） |
| **批量并行模式** | `tasks` 数组（最多 3 项） | `ThreadPoolExecutor` 并发运行，全部完成后汇总 |

**主要参数：**

```jsonc
{
  "goal": "字符串 — 子代理要完成什么（必须自包含，子代理不知道父对话历史）",
  "context": "字符串 — 文件路径、错误信息、约束等背景信息",
  "toolsets": ["terminal", "file", "web"],  // 启用工具集，默认继承父代理
  "tasks": [                                  // 批量模式（最多 3）
    {"goal": "...", "context": "...", "toolsets": [...]}
  ],
  "max_iterations": 50,                       // 每个子代理的工具调用轮数上限
  "acp_command": "claude",                    // 可选：用 ACP 协议生成外部代理（如 Claude Code）
  "acp_args": ["--acp", "--stdio"]
}
```

**注册位于** `tools/delegate_tool.py:955-970`，工具集名称为 `"delegation"`，emoji 🔀。

---

## 深度限制：禁止多级委派

**这是委派系统最重要的硬性限制**，用两道防护线实现：

### 防护线 ①：执行入口的深度检查

`tools/delegate_tool.py:36`：

```python
MAX_DEPTH = 2  # parent (0) -> child (1) -> grandchild rejected (2)
```

`tools/delegate_tool.py:528-536`：

```python
depth = getattr(parent_agent, '_delegate_depth', 0)
if depth >= MAX_DEPTH:
    return json.dumps({
        "error": (
            f"Delegation depth limit reached ({MAX_DEPTH}). "
            "Subagents cannot spawn further subagents."
        )
    })
```

每构建一个子代理就累加深度（`tools/delegate_tool.py:315`）：

```python
child._delegate_depth = getattr(parent_agent, '_delegate_depth', 0) + 1
```

### 防护线 ②：子代理工具集剥夺

即便有人绕过深度检查，子代理本身也拿不到 `delegate_task` 工具（见下节）。

### 形态对比

| 形态 | 支持？ | 说明 |
|------|--------|------|
| A → 单个 B | ✅ | 单任务模式 |
| A → 并行 B / C / D | ✅ | 批量模式（最多 3 个） |
| **A → B → C / D**（嵌套） | ❌ | 被 `MAX_DEPTH` 拒绝 |
| 子代理读 `MEMORY.md` | ❌ | `skip_memory=True`，系统提示**不注入** MEMORY.md/USER.md（`delegate_tool.py:1123`） |
| 子代理写 `MEMORY.md` | ❌ | `memory` 在黑名单 |
| 子代理调 `clarify` | ❌ | 在黑名单 |
| 子代理调 `send_message` | ❌ | 在黑名单 |

---

## 子代理工具剥夺

`tools/delegate_tool.py:27-33` 定义工具黑名单：

```python
DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task",   # 禁止递归委托
    "clarify",         # 禁止用户交互（中间代理不应僭越父代理的对话权）
    "memory",          # 禁止写入共享的 MEMORY.md
    "send_message",    # 禁止跨平台副作用
    "execute_code",    # 子代理应逐步推理，而非编写脚本
])
```

`tools/delegate_tool.py:105-110` 的 `_strip_blocked_tools()` 同时把对应**工具集**移除：

```python
def _strip_blocked_tools(toolsets: List[str]) -> List[str]:
    blocked_toolset_names = {
        "delegation", "clarify", "memory", "code_execution",
    }
    return [t for t in toolsets if t not in blocked_toolset_names]
```

### 剥夺时机

`_build_child_agent()`（`tools/delegate_tool.py:237-245`）在选择子代理工具集时分三种情况，每种都过 `_strip_blocked_tools()`：

1. 调用方显式指定 `toolsets` → 与父代理工具集求交集，再剥离敏感项
2. 父代理有 `enabled_toolsets` → 直接剥离
3. 都没有 → 用 `DEFAULT_TOOLSETS = ["terminal", "file", "web"]` 剥离

**关键不变量**：子代理的工具集**永远不会超出父代理已加载的工具范围**（取交集），且必然不包含黑名单项。

---

## 子代理构建流程

`_build_child_agent()`（`tools/delegate_tool.py:193-332`）在主线程同步构造子 `AIAgent`，**返回前不运行**：

```
1. 推导父代理工具集（enabled_toolsets / 已加载工具反推 / DEFAULT_TOOLSETS）
2. 与传入 toolsets 求交集 → _strip_blocked_tools 剥夺敏感工具集
3. 解析 workspace_hint（环境变量 / 子目录提示 / 父代理 cwd）
4. 构造子代理系统提示（_build_child_system_prompt）：
   - "You are a focused subagent working on a specific delegated task"
   - YOUR TASK / CONTEXT / WORKSPACE PATH 三段
   - 末尾要求"清晰简洁的总结：做了什么/发现什么/创建或修改的文件/遇到的问题"
5. 解析有效凭据（配置覆盖 > 父代理继承）
6. 构造子 AIAgent，关键参数：
   - quiet_mode=True                    # 子代理不打印
   - ephemeral_system_prompt=child_prompt
   - skip_context_files=True            # 不加载 AGENTS.md / CLAUDE.md
   - skip_memory=True                   # 不加载 MEMORY.md / USER.md（虽然没记忆工具）
   - clarify_callback=None              # 关闭用户交互
   - iteration_budget=None              # 子代理拥有独立预算
   - log_prefix=f"[subagent-{i}]"
   - parent_session_id=父 session_id    # SQLite 血缘追踪
7. child._delegate_depth = parent._delegate_depth + 1   # 累加深度
8. 解析子代理凭据池（如同 provider 则共享，便于轮换冷却同步）
9. 注册到 parent._active_children（用于中断传播）
```

**线程安全**：构造在主线程完成（避免 `model_tools._last_resolved_tool_names` 全局变量竞态），运行才在工作线程。每个子代理构造后保存父代理工具名快照（`tools/delegate_tool.py:599`）：`child._delegate_saved_tool_names = _parent_tool_names`，子代理结束后用于恢复全局变量。

---

## 父代理与子代理：同一个类，不同配置

**结论：父代理和子代理不是两种类型，而是同一个 `AIAgent` 类的实例**（`run_agent.py:327`，全项目只有这一个类定义、无子类）。两者走**完全相同**的构造路径 `AIAgent.__init__()` → `agent/agent_init.py:init_agent()`，区别只来自构造参数与构造后打上的几个运行时属性。这是"组合优于继承"——避免子类爆炸，用参数灵活配置行为。

### 子代理身份字段

子代理由 `_build_child_agent()`（`tools/delegate_tool.py:868`）构造**普通 `AIAgent` 之后**，额外设置以下私有属性来表达"我是子代理"：

| 字段 | 设置位置 | 父代理值 | 子代理值 | 用途 |
|------|---------|---------|---------|------|
| `_delegate_depth` | agent_init.py:428 / delegate_tool.py:1138 | `0` | `父+1`（子=1,孙=2） | 深度限制、TUI 嵌套可视化 |
| `_subagent_id` | delegate_tool.py:1144 | *无此字段* | `"sa-{index}-{uuid8}"` | 唯一标识、中断/暂停定位 |
| `_parent_subagent_id` | delegate_tool.py:1145 | *无此字段* | 父的 `_subagent_id`（可为 None） | 构建家族树 |
| `_delegate_role` | delegate_tool.py:1141 | *无此字段* | `"leaf"` / `"orchestrator"` | 控制能否再委派 |
| `_subagent_goal` | delegate_tool.py:1146 | *无此字段* | 任务目标字符串 | 展示与日志 |
| `_parent_turn_id` | delegate_tool.py:1147 | *无此字段* | 父代理当前 turn id | 血缘追踪 |

**判定方式靠属性而非 `isinstance`**：代码里区分父子，是检查 `_subagent_id` 是否存在、`_delegate_depth` 取值，而不是类型判断。顶层父代理 `_delegate_depth` 恒为 `0` 且不带 `_subagent_id`——这是唯一、可编程判定的边界。

### 行为差异及其来源

| 维度 | 父代理 | 子代理 | 实现手段 |
|------|--------|--------|---------|
| system prompt | 配置/默认 | `_build_child_system_prompt()`（聚焦单任务） | `ephemeral_system_prompt` 参数 |
| 工具集 | 全量 | 与父交集后剥除黑名单（见上文工具剥夺） | `_strip_blocked_tools()` |
| 最大迭代 | 90（run_agent.py:361） | 50（`delegation.max_iterations`） | `max_iterations` 参数 |
| 上下文/记忆 | 完整历史+记忆 | `skip_context_files=True` / `skip_memory=True` | 构造参数 |
| 用户交互 | 有 clarify 回调 | `clarify_callback=None`，审批走自动 allow/deny（见 [clarify 设计](clarify-design.md)） | 构造参数 |
| 输出 | 直接展示 | 经 `tool_progress_callback` 上报给父代理 | 回调参数 |

---

## 子代理如何与用户交互（间接）

子代理**不直接和用户对话**（`clarify` / `send_message` 已被黑名单剥夺）。它通过统一的 `tool_progress_callback` 把事件单向冒泡给父代理，父代理再按当前运行环境选择传输通道；用户的控制（中断/暂停）走反向 RPC 通道回来。

### 输出通道（单向，父代理代为转发）

子代理事件（`tool.started` / `_thinking` / `subagent.start|complete`）经 `_build_child_progress_callback()`（`tools/delegate_tool.py:681`）上报，每个事件携带 `_identity_kwargs`（`subagent_id`/`parent_id`/`depth`/`model`）供客户端渲染家族树：

| 客户端 | 传输方式 | 关键位置 |
|------|---------|---------|
| CLI | stdout + KawaiiSpinner（`print_above`，树形视图） | `agent/tool_executor.py` |
| 网页 / REST | HTTP **SSE** 流（`text/event-stream`） | `gateway/platforms/api_server.py` |
| 编辑器（Zed 等） | **ACP** JSON-RPC `session_update` | `acp_adapter/events.py` |
| TUI | **WebSocket** JSON 事件（`subagent.spawn/start/tool/thinking/complete`） | `tui_gateway/server.py` |
| 插件/钩子 | `subagent_start` / `subagent_stop` 回调 | `hermes_cli/plugins.py` |

### 反向控制通道（用户 → 子代理）

| 动作 | 入口 | 实现 |
|------|------|------|
| 中断单个子代理 | TUI Stop / `subagent.interrupt` RPC | `interrupt_subagent(subagent_id)`（`tools/delegate_tool.py:186`），查 `_active_subagents` 注册表后调 `agent.interrupt()` |
| 暂停新生成 | `delegation.pause` RPC | `set_spawn_paused()`（`tools/delegate_tool.py:156`），运行中子代理不受影响 |
| 父代理整体中断传播 | Ctrl+C / 网关停止 | 遍历 `_active_children`，见 [中断与取消传播](#中断与取消传播) |

> 子代理"需要问用户"时的应对（协议返回、按决策点拆分、`request_clarification` 工具等），见 [子代理与用户交互模式](subagent-user-interaction.md)。

---

## 并行执行机制

`delegate_task()`（`tools/delegate_tool.py:506-687`）的执行分支：

### 单任务路径（`n_tasks == 1`）

`tools/delegate_tool.py:605-609`：

```python
_i, _t, child = children[0]
result = _run_single_child(0, _t["goal"], child, parent_agent)
results.append(result)
```

无线程池开销，主线程同步运行。

### 批量并行路径（`n_tasks > 1`）

`tools/delegate_tool.py:610-667`：

```python
with ThreadPoolExecutor(max_workers=MAX_CONCURRENT_CHILDREN) as executor:
    futures = {executor.submit(_run_single_child, ...): i for i, t, child in children}

    for future in as_completed(futures):
        entry = future.result()
        results.append(entry)
        # 在父代理 spinner 上方打印逐任务完成行：
        #   ✓ [1/3] 调研 X    (12.3s)
        #   ✗ [2/3] 调研 Y    (45.1s)
```

完成后按 `task_index` 排序，保证返回顺序与输入一致。

### 进度展示

- **CLI 模式**：`_build_child_progress_callback`（`tools/delegate_tool.py:113-191`）把子代理工具调用打印到父代理 spinner 上方的树形视图
- **网关模式**：批量收集工具名（`_BATCH_SIZE=5`），通过父代理的 `tool_progress_callback` 转发到目标平台

---

## 子代理执行与结果回传

`_run_single_child()`（`tools/delegate_tool.py:334-505`）在工作线程内：

```
1. （如有）从凭据池租用一个凭据并热替换到子代理
2. 调 child.run_conversation(user_message=goal)
   ↑ 这里是同步阻塞调用，子代理跑完整个对话循环
3. 抽取关键字段：
   - final_response → summary
   - completed / interrupted → status (completed / interrupted / failed)
   - api_calls / token 计数 / model
4. 从对话消息构建 tool_trace：
   - 遍历 messages，按 role=assistant 拿 tool_calls
   - 按 role=tool 用 tool_call_id 配对工具结果
   - 每条 trace：{tool, args_bytes, result_bytes, status}
5. 决定 exit_reason: interrupted / completed / max_iterations
6. 返回结构化 entry
7. finally：释放凭据租约 + 恢复 _last_resolved_tool_names + 从 _active_children 移除
```

### 返回给父代理的 JSON 结构

`tools/delegate_tool.py:684-687`：

```jsonc
{
  "results": [
    {
      "task_index": 0,
      "status": "completed",            // completed / interrupted / failed / error
      "summary": "<final_response 文本>",
      "api_calls": 12,
      "duration_seconds": 23.4,
      "model": "google/gemini-3-flash-preview",
      "exit_reason": "completed",       // completed / max_iterations / interrupted
      "tokens": {"input": 4523, "output": 891},
      "tool_trace": [
        {"tool": "read_file", "args_bytes": 124, "result_bytes": 5421, "status": "ok"}
      ]
    }
  ],
  "total_duration_seconds": 25.1
}
```

**关键属性**：父代理只看到 summary + 元数据，**不会看到子代理读取的文件内容、工具响应原文** -- 这是上下文隔离的核心价值。

---

## 父子上下文流动：单向、精选、隔离

委派的上下文流动**两个方向都被刻意收窄**：父→子只给一份手写的"任务简报"，子→父只回一份"成果摘要"。子代理对父代理而言是个**黑盒**——父喂它什么它才知道什么，它干了什么父只看结果。目的一致：**保护父代理的上下文窗口不被污染，同时不泄露父代理的私密记忆与推理过程**。

### 方向一：父代理上下文会记录子代理的什么？

子代理跑完后，`delegate_task` 把 [返回 JSON](#返回给父代理的-json-结构) 作为**一条 `role=tool` 消息**写进父代理对话历史。父代理"读到"的就只有这条：

| 父代理上下文里**有** | 父代理上下文里**没有** |
|------|------|
| 最终摘要 `summary`（来自子代理 `final_response`） | 子代理的完整 `messages`（几十轮对话） |
| `tokens` / `api_calls` / `duration_seconds` / `status` | 每个工具调用的**实际参数与返回原文** |
| `tool_trace`：仅 `{tool, args_bytes, result_bytes, status}` | 子代理的思考 / 推理 / 中间步骤 |

**`tool_trace` 只是元数据**：调了哪个工具、参数多少字节、结果多少字节、成没成功——看不到任何实际内容。子代理那份完整 `messages` 在结束后**直接丢弃**：因 `skip_memory=True` 连 SQLite session 都不落盘，父代理无从访问（见 `delegate_tool.py` 顶部模块注释：*"父级上下文仅看到委派调用和摘要结果，不会看到子级的中间工具调用或推理过程"*）。

### 方向二：子代理能拿到父代理的多少上下文？

**子代理启动时是一张空白对话历史。** `child.run_conversation(user_message=goal)`（`delegate_tool.py:1521` 一带）**不传 `conversation_history`**，子代理的 `messages` 从零开始，第一条就是 `goal`——它**完全看不到父代理的对话线程**。

它能拿到的"父上下文"全部经由 **system prompt 注射**（而非历史继承），由 `_build_child_system_prompt()`（`delegate_tool.py:868` 内构造）拼装：

- **`goal`** —— 任务目标（同时作为第一条 user message）
- **`context`** —— 父代理调用 `delegate_task` 时**显式传入**的那段文本，拼进系统提示的 `CONTEXT:` 块
- **workspace 路径** —— 从父代理的 `cwd` / `terminal_cwd` 解析继承

三道隔离开关切断其余一切：

| 开关 | 位置 | 效果 |
|------|------|------|
| `skip_context_files=True` | `delegate_tool.py:1122` | **不**加载 AGENTS.md / CLAUDE.md / .cursorrules |
| `skip_memory=True` | `delegate_tool.py:1123` | **不**加载、**不**注入 MEMORY.md / USER.md（`_memory_store` 保持 None） |
| 空白对话历史 | `delegate_tool.py:1521` | 看不到父代理任何对话 / 思考 |

> **注意**：子代理**不读** MEMORY.md。"上下文通过系统提示注射而非消息历史继承"指的是 `goal`/`context`/workspace，**不含** MEMORY.md——参见 [形态对比](#形态对比) 表与 [与记忆系统的集成](#与记忆系统的集成)。

### 子代理的执行自主性：很高

尽管上下文被收窄，子代理的**执行**是高度自主的：

| 维度 | 自主程度 |
|------|---------|
| 工具集 | 父集合的受限子集（剔除 `DELEGATE_BLOCKED_TOOLS`，见 [子代理工具剥夺](#子代理工具剥夺)） |
| 迭代预算 | **独立** `IterationBudget`，默认 50 轮，与父的 90 轮互不影响 |
| task_id | **独立**，隔离终端会话 / 文件状态缓存 |
| 模型 / 凭据 | 默认继承父，可被 `delegation.provider/model` **覆盖**为完全不同的 provider（见下节） |

**一句话**：父→子是"任务简报"不是"共享上下文"，子→父是"成果摘要"不是"过程回灌"——两个方向都隔离，唯独执行自主。

---

## 凭据与模型路由

`_resolve_delegation_credentials()`（在 `delegate_task` 入口处调用）支持把子代理路由到完全不同的 provider:model：

### 配置示例（`cli-config.yaml`）

```yaml
delegation:
  max_iterations: 50
  default_toolsets: ["terminal", "file", "web"]
  model: "google/gemini-3-flash-preview"  # 子代理用便宜的快速模型
  provider: "openrouter"                  # 自动解析 base_url / api_key / api_mode
```

### 支持的 provider

`openrouter`、`nous`、`zai`、`kimi-coding`、`minimax`（详见配置文件示例的注释）。

### 凭据池共享

`_resolve_child_credential_pool()`（`tools/delegate_tool.py:690-718`）规则：

- 子代理与父代理用**同一 provider** → 共享父代理的凭据池（保持冷却状态和密钥轮换同步）
- 子代理用**不同 provider** → 独立凭据池

### ACP 子代理（外部代理）

通过 `acp_command` / `acp_args` 参数，可以让父代理通过 ACP 协议派发外部代理子进程（如 Claude Code）。这使得任意 Hermes 父代理（包括 Discord/Telegram/CLI）都能生成 `claude --acp --stdio` 之类的 ACP-capable 子代理。

---

## 中断与取消传播

父代理在 `_build_child_agent()` 注册子代理（`tools/delegate_tool.py:323-330`）：

```python
if hasattr(parent_agent, '_active_children'):
    lock = getattr(parent_agent, '_active_children_lock', None)
    if lock:
        with lock:
            parent_agent._active_children.append(child)
    else:
        parent_agent._active_children.append(child)
```

当父代理被中断（Ctrl+C / 网关停止），它遍历 `_active_children` 把中断信号传给每个子代理。子代理 `run_conversation` 检测到中断后退出循环，返回 `interrupted=True`。

`_run_single_child` 的 `finally` 块确保不论是否异常都会从 `_active_children` 移除（`tools/delegate_tool.py:495-503`）。

---

## 与记忆系统的集成

委派完成后，父代理通知记忆系统（`tools/delegate_tool.py:670-680`）：

```python
if parent_agent and hasattr(parent_agent, '_memory_manager') and parent_agent._memory_manager:
    for entry in results:
        parent_agent._memory_manager.on_delegation(
            task=_task_goal,
            result=entry.get("summary", "") or "",
            child_session_id=getattr(children[entry["task_index"]][2], "session_id", "") ...,
        )
```

外部记忆 Provider（如 Honcho、Hindsight）可以借此把"任务 → 摘要 → 子 session_id"三元组写入自己的后端，用于跨会话的委派模式学习。

**子代理本身** `skip_memory=True`（`tools/delegate_tool.py:301`）+ `memory` 工具被剥夺，所以子代理既不读也不写共享 MEMORY.md，避免污染长期记忆。

详见 [记忆自我进化](memory-self-evolution.md) 的"路径 C：生命周期钩子"章节。

---

## 配置参数

`cli-config.yaml` 中的 `delegation:` 段：

```yaml
delegation:
  max_iterations: 50                            # 每个子代理的工具调用轮数上限
  default_toolsets: ["terminal", "file", "web"] # 子代理默认工具集
  # model: "google/gemini-3-flash-preview"      # 覆盖子代理模型（空=继承父代理）
  # provider: "openrouter"                      # 覆盖子代理 provider
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `max_iterations` | 50 | 单个子代理可执行的工具调用轮数上限 |
| `default_toolsets` | `["terminal", "file", "web"]` | 调用方未指定 toolsets 时的回退 |
| `model` | 空（继承父） | 强制子代理使用的模型 |
| `provider` | 空（继承父） | 强制子代理使用的 provider；自动解析凭据 |

**硬编码常量**（不可通过配置修改）：

- `MAX_DEPTH = 2` -- 委派深度上限
- `MAX_CONCURRENT_CHILDREN = 3` -- 单次委派最大并发子代理数

---

## 何时委派任务给子代理

委派决策由**两层信号**驱动：**工具可用性**（系统侧 -- 决定能不能委派）+ **工具描述与技能引导**（模型侧 -- 决定该不该委派）。

### 第一层：能不能委派 -- 4 个前提

| 条件 | 判定来源 | 不满足时 |
|------|---------|---------|
| `delegation` 工具集已启用 | `cli-config.yaml` 的 `enabled_toolsets` 或运行时 `--toolsets` | 模型根本看不到 `delegate_task` |
| 当前不是子代理（`_delegate_depth < 2`） | `tools/delegate_tool.py:528-536` | 工具调用直接返回 "depth limit reached" |
| 当前不是 ACP 受限子代理 | 子代理被 `_strip_blocked_tools()` 剥夺了 `delegation` 工具集 | 工具不在 schema 中 |
| 父代理类型不是受限会话 | gateway 在某些受限平台可能禁用 | 工具被过滤 |

### 第二层：该不该委派 -- 模型的决策规则

模型唯一的判断依据是 `delegate_task` 的工具 schema 描述（`tools/delegate_tool.py:836-860`）。schema 明确给出**正反两面**：

#### ✅ 应该委派（WHEN TO USE）

| 触发场景 | 解读 |
|---------|------|
| **推理密集型子任务** | 调试、代码评审、研究综合 -- 需要多步推理但与主线任务相对独立 |
| **会污染父上下文的中间数据** | 子代理读 30 个文件后只回 200 字摘要，主代理上下文不被淹没 |
| **可并行的独立工作流** | 同时调研 A 和 B、同时跑两个独立的修复路径 |

#### ❌ 不应该委派（WHEN NOT TO USE）

| 反模式 | 替代方案 |
|--------|---------|
| 机械的多步操作（无需推理） | 用 `execute_code` 写脚本一次跑完 |
| 单个工具调用 | 直接调那个工具 |
| 需要跟用户互动的任务 | 子代理无 `clarify` 工具，根本不能问 |

#### 隐性禁忌（schema "IMPORTANT" 段）

- 子代理对父对话**零记忆** -- 必须在 `context` 字段里把所有相关信息（文件路径、错误信息、约束）打包传过去；否则子代理两眼一抹黑
- 子代理**不能调** `delegate_task / clarify / memory / send_message / execute_code`
- 每个子代理获得**独立终端会话**（独立 cwd 和状态） -- 不能假定父代理的 shell 状态

> 子代理需要询问用户怎么办？详见 [子代理与用户交互模式](subagent-user-interaction.md)。

### Hermes 推荐的两种"高质量委派模式"

#### 模式 A：subagent-driven-development（按计划执行）

`skills/software-development/subagent-driven-development/SKILL.md` 给出标准三步循环：

```
对计划里的每个任务：
  1. 派发 implementer 子代理执行 → 拿摘要
  2. 派发 spec compliance reviewer 子代理（核对是否符合规格）
  3. 派发 code quality reviewer 子代理（核对代码质量）
  → 通过则下一任务；不通过派 fix 子代理修
全部完成后：派发 final integration reviewer
```

**触发条件**（`SKILL.md:23-27`）：

- 已有实现计划
- 任务大致独立
- 在意质量与规格符合性
- 想要任务间自动评审

**任务粒度**：每个委派 ≈ 2-5 分钟聚焦工作。"实现整套用户认证系统"太大；"创建 User model 含 email 和 password 字段"才合适。

#### 模式 B：spawning hermes 子进程（重度长任务）

`skills/autonomous-ai-agents/hermes-agent/SKILL.md:414-422` 区分两种"派活"方式：

| 维度 | `delegate_task` | `terminal(command="hermes chat -q ...")` |
|------|-----------------|------------------------------------------|
| 隔离级别 | 独立对话，共享进程 | 完全独立子进程 |
| 时长 | 分钟级（受父循环约束） | 数小时/数天 |
| 工具访问 | 父代理工具的子集 | 完整工具访问 |
| 可交互 | 否 | 是（PTY 模式） |
| 适用场景 | 快速并行子任务 | 长时间自主任务 |

**默认偏好**（`SKILL.md:482`）："**Prefer `delegate_task` for quick subtasks** -- less overhead than spawning a full process"。

### 影响委派决策的隐性因素

#### 1. 智能模型路由把"委派"识别为复杂任务

`agent/smart_model_routing.py:54-55` 把 `"delegate"` 和 `"subagent"` 列为复杂关键词 -- 一旦用户消息里出现这两个词，**强制路由到主力模型**。这意味着委派决策本身就被认为是"复杂任务"，需要强模型来判断。

#### 2. 系统提示中没有委派决策指令

`agent/prompt_builder.py` 里**没有**任何关于委派的内置指令。委派决策**完全靠 schema 描述驱动**，没有藏在系统提示里。模型必须自己读懂 schema 来决定。

#### 3. 工具集继承约束

委派时如果不显式指定 `toolsets`，子代理继承父代理工具集（去掉黑名单）。**父代理没装的工具，子代理也没法用** -- 比如父代理没启用 `web` 工具集，就不能委派"上网研究 X"任务（`tools/delegate_tool.py:237-245` 的工具集求交集逻辑）。

### 决策树

```
有可独立完成、推理密集、会大量读文件/上网的子任务？
├─ 是 → 会跟用户互动吗？
│       ├─ 是 → 不委派（用主代理处理）
│       └─ 否 → 是单纯的机械步骤吗？
│              ├─ 是 → 用 execute_code
│              └─ 否 → ✅ 委派
│                       ├─ 多个独立 → 批量 tasks（最多 3 并行）
│                       └─ 单一 → goal 单任务
└─ 否 → 直接在主循环里调单个工具或对话回应
```

### 核心心法

> **委派 = 用 token 换上下文整洁度 + 并行度**

代价是子代理拿不到父对话历史（必须在 `context` 里完整传递）+ 多花一些 API 调用做总结。当**节省的上下文窗口空间** + **并行获得的时间收益** > 这些代价时，才值得委派。

---

## 设计取舍与替代方案

### 为什么禁止多级委派？

| 问题 | 后果 |
|------|------|
| 递归爆炸 | 二级委派 = 最多 3 × 3 = 9 个并发子代理；三级 = 27 个；token 与线程指数级膨胀 |
| 摘要失真 | 多层摘要的损失会累积，最终回到顶层的信息可能严重失真 |
| 调试困难 | 多级嵌套使错误归因和性能分析极其复杂 |
| 凭据冲突 | 多层并发对同一密钥发起请求，速率限制治理变得棘手 |

### 想要"多级"效果的变通方案

**把分层逻辑上提到父代理来编排**：

```
父代理 A：
  1. 第一轮 delegate_task 并行 B / C / D，拿到摘要
  2. 自己分析摘要，决定下一步
  3. 第二轮 delegate_task 派发新任务 E / F / G
  4. 整合所有结果
```

这本质上是把"B 内部的二次委派"上提到 A 的对话循环里，A 始终在 depth=0，每轮派发都是 depth=1，不违反 `MAX_DEPTH`。

### 替代委派的工具

委派工具的 schema 描述（`tools/delegate_tool.py:849-852`）明确建议**何时不要用** `delegate_task`：

| 场景 | 替代方案 |
|------|---------|
| 机械的多步操作（无需推理） | `execute_code`（写脚本） |
| 单个工具调用 | 直接调那个工具 |
| 需要用户交互的任务 | 不能委派 -- 子代理无法用 `clarify` |

---

## 关键文件速查

| 文件 / 位置 | 职责 |
|------------|------|
| `tools/delegate_tool.py:27-33` | `DELEGATE_BLOCKED_TOOLS` 工具黑名单 |
| `tools/delegate_tool.py:35-38` | 关键常量（`MAX_CONCURRENT_CHILDREN`, `MAX_DEPTH`, `DEFAULT_MAX_ITERATIONS`, `DEFAULT_TOOLSETS`） |
| `tools/delegate_tool.py:46-78` | `_build_child_system_prompt()` -- 子代理系统提示模板 |
| `tools/delegate_tool.py:105-110` | `_strip_blocked_tools()` -- 工具集剥夺 |
| `tools/delegate_tool.py:113-191` | `_build_child_progress_callback()` -- 进度转发回调 |
| `tools/delegate_tool.py:193-332` | `_build_child_agent()` -- 子代理构造（含深度累加 / `_active_children` 注册 / 凭据池绑定） |
| `tools/delegate_tool.py:334-505` | `_run_single_child()` -- 工作线程内执行子代理并构建结构化结果 |
| `tools/delegate_tool.py:506-687` | `delegate_task()` -- 主入口（深度检查 / 任务规范化 / 并行调度 / 记忆通知） |
| `tools/delegate_tool.py:528-536` | 深度限制检查 |
| `tools/delegate_tool.py:670-680` | `on_delegation` 钩子调用 |
| `tools/delegate_tool.py:690-718` | `_resolve_child_credential_pool()` -- 凭据池共享逻辑 |
| `tools/delegate_tool.py:834-949` | `DELEGATE_TASK_SCHEMA` -- 工具 schema 定义 |
| `tools/delegate_tool.py:955-970` | 工具注册（toolset=`"delegation"`） |

---

## 总结

Hermes 的委派系统是一个**精心约束的并行执行框架**而非通用的 Agent 编排引擎：

1. **单层并行**：`MAX_DEPTH=2` + 工具黑名单双重保护，杜绝嵌套委派
2. **上下文隔离**：子代理跑在独立对话 / 终端 / 工具集 / 凭据池中，父代理只看到摘要
3. **零信任**：子代理被剥夺所有可能产生外部副作用的工具（用户交互、跨平台消息、记忆写入、代码执行）
4. **可观测性**：tool_trace + 进度回调 + spinner 提供完整可观察性
5. **可路由**：子代理可以走完全不同的 provider:model，方便父用强模型 / 子用便宜快速模型

这种设计在**确定性、可控性、成本可预测**之间做出明确取舍，放弃了多级嵌套带来的灵活性。需要分层编排时，由父代理在自己的对话循环里多轮派发是唯一受支持的模式。
