# clarify 工具设计

> agent 向用户提问的唯一原语 —— 工具是空壳，交互靠回调，跨同步/异步线程架桥

---

## 目录

- [核心思想：工具 = 纯逻辑，交互 = 回调](#核心思想工具--纯逻辑交互--回调)
- [工具 schema](#工具-schema)
- [返回数据结构](#返回数据结构)
- [一个工具，五种运行环境](#一个工具五种运行环境)
- [CLI/TUI 交互](#clitui-交互)
- [Gateway 异步交互：同步 agent ↔ 异步事件循环](#gateway-异步交互同步-agent--异步事件循环)
- [clarify_gateway 注册表 API](#clarify_gateway-注册表-api)
- [权力隔离：子代理禁用 clarify](#权力隔离子代理禁用-clarify)
- [优雅降级：每条失败路径都有兜底](#优雅降级每条失败路径都有兜底)
- [关键文件速查](#关键文件速查)
- [设计哲学](#设计哲学)
- [参考](#参考)

---

## 核心思想：工具 = 纯逻辑，交互 = 回调

`clarify` 是 Hermes 里**唯一让 agent 主动向用户提问并阻塞等待回答**的结构化交互原语。它的设计支点是：**工具本身不知道用户在哪、用什么终端，"怎么问用户"完全由运行环境注入的回调决定**。

`tools/clarify_tool.py:23` 的 `clarify_tool()` 只做三件事：

```python
def clarify_tool(question, choices=None, callback=None) -> str:
    # 1. 校验 question 非空、清洗 choices(去重、空串过滤、截断到 MAX_CHOICES=4)
    # 2. user_response = callback(question, choices)   ← 真正的交互全在这
    # 3. return JSON: {question, choices_offered, user_response}
```

`callback` 签名固定为 `(question, choices) -> str`，在 `agent/agent_init.py:386` 注入到 `agent.clarify_callback`。工具执行入口 `agent/tool_executor.py:929` 把它透传进去：

```python
function_result = _clarify_tool(
    question=function_args.get("question", ""),
    choices=function_args.get("choices"),
    callback=agent.clarify_callback,   # 平台实现由此注入
)
```

**换运行环境 = 换一个 callback 实现，工具和 schema 一字不改。** 这是整个设计的关键。

---

## 工具 schema

`tools/clarify_tool.py:87` `CLARIFY_SCHEMA`：

```json
{
  "name": "clarify",
  "parameters": {
    "type": "object",
    "properties": {
      "question": { "type": "string" },
      "choices": {
        "type": "array",
        "items": { "type": "string" },
        "maxItems": 4
      }
    },
    "required": ["question"]
  }
}
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `question` | 是 | 要向用户呈现的问题文本 |
| `choices` | 否 | 最多 **4** 个预设选项（`MAX_CHOICES=4`）；省略或为空 → **开放式问答** |

**自动追加 "Other"**：当提供了 `choices`，各环境的 UI 都会**自动追加一项 "Other (type your answer)"**，保证用户永远能自由作答——模型不必穷举所有可能选项。注册见 `tools/clarify_tool.py:131`（toolset=`clarify`，emoji=`❓`，`check_fn` 恒为 True）。

---

## 返回数据结构

成功时返回 JSON 字符串（`tools/clarify_tool.py:71`）：

```json
{
  "question": "用户需要选择什么？",
  "choices_offered": ["选项1", "选项2"],   // null 表示开放式
  "user_response": "用户的选择或自由文本"
}
```

`user_response` 已 `.strip()`。各种失败路径见 [优雅降级](#优雅降级每条失败路径都有兜底)。

---

## 一个工具，五种运行环境

同一个 `clarify_tool`，靠不同的 `callback` 适配五种"用户在哪"的场景：

| 环境 | callback 实现 | 交互形态 | 超时 |
|------|--------------|---------|------|
| **CLI/TUI** | `cli.py:11689` `_clarify_callback` | prompt_toolkit 弹框，↑↓ 选 / 数字键快选 / Enter 确认，选 Other 进自由输入 | 120s |
| **Gateway/Telegram** | `gateway/run.py:17908` `_clarify_callback_sync` | 内联按钮 `cl:{id}:{idx}`，点按钮或回文字 | 600s |
| **其他网关平台** | `gateway/platforms/base.py` `send_clarify`（默认实现） | 纯文本编号列表，回数字/选项文本/自由文本 | 600s |
| **Oneshot（无人值守）** | `hermes_cli/oneshot.py:377` `_oneshot_clarify_callback` | 无用户可问，返回"自行判断"提示 | — |
| **子代理** | `None`（显式禁用） | 工具直接返回 error，不能提问 | — |

---

## CLI/TUI 交互

CLI 下 callback（`cli.py:11689`）是**同步阻塞**实现，直接在 agent 线程里等键盘：

1. 把问题/选项写入 `self._clarify_state`，设 `_clarify_deadline`（`CLI_CONFIG.clarify.timeout`，默认 **120s**），调 `_invalidate()` 触发 TUI 重绘。
2. 在一个 `queue.Queue` 上轮询等待，每 1 秒检查一次超时。
3. 渲染由 `_get_clarify_display()`（`cli.py:14437`）完成，自适应终端高度：优先保证选项可见，问题过长则截断为 `… (question truncated)`。

**键盘交互**（filter 仅在 `_clarify_state` 存在时生效）：

| 按键 | 行为 |
|------|------|
| `↑` / `↓` | 在选项间移动（最后一项是 "Other"） |
| `1-9` / `0` | 数字快选；选中 choices 范围外 → 进入 Other 文本输入 |
| `Enter` | 确认当前选项；若选中 "Other" → 切到自由文本模式 |
| 文本模式下 `Enter` | 提交自由文本 |

UI 形态：

```
╭─ Hermes needs your input ─────╮
│ ❓ 问题文本                    │
│ ❯ 1. 选项 A（已选中）         │
│   2. 选项 B                   │
│   3. Other (type your answer) │
│ ⏱ 剩余: 45s                  │
╰───────────────────────────────╯
```

---

## Gateway 异步交互：同步 agent ↔ 异步事件循环

CLI 直接阻塞等键盘即可，但 **gateway 跑在 asyncio 事件循环上，agent 在 worker 线程上**——Python 不允许在普通线程里 `await`。Hermes 用 `tools/clarify_gateway.py` 的一个 `threading.Event` 注册表跨越这两个上下文：

```
agent 线程                          gateway 事件循环           用户
  register(clarify_id) ──记到 _entries
  schedule send_clarify ──────────► 发按钮消息 ───────────► 看到问题
       │ (safe_schedule_threadsafe)
  wait_for_response()  ◄═ 阻塞在 entry.event ═
       │ (每 1s 醒一次打心跳)                                点按钮 / 回文字
       │                            resolve_gateway_clarify ◄┘
       │                            entry.response=…; event.set()
  被唤醒，返回 user_response ◄═
```

`gateway/run.py:17908` `_clarify_callback_sync` 的流程：

1. `register(clarify_id, session_key, question, choices)` 在注册表登记。
2. `safe_schedule_threadsafe(adapter.send_clarify(...))` 把异步发送任务调度到 gateway 事件循环，用 `concurrent.futures.Future.result(timeout=15)` 同步等它发出去（避免在线程里 `await`）。
3. `wait_for_response(clarify_id, timeout=600)` 同步阻塞等用户。

**两个关键设计点**：

1. **1 秒切片轮询而非一次性等满 600 秒**（`clarify_gateway.py:99`）。`entry.event.wait(timeout=min(1.0, remaining))` 每醒一次就 `touch_activity_if_due()` 打一次心跳——**防止 gateway 的 inactivity watchdog 把"正在等用户回答"的会话误判为死亡而杀掉**。

2. **文本拦截 resume**。用户没点按钮而是直接打字时，`gateway/run.py` 的消息处理会先查 `get_pending_for_session()`——若该 session 有 `awaiting_text` 的 pending clarify，就把这条消息当答案 `resolve` 掉、**不再派发给 agent**（斜杠命令 `/` 开头除外，留给命令处理）。Telegram 点 "Other" 按钮会 `mark_awaiting_text()` 切到这条路径。

---

## clarify_gateway 注册表 API

`tools/clarify_gateway.py` 用模块级 `RLock` 保护的字典实现线程安全的跨线程交接。核心数据结构 `_ClarifyEntry`（`:44`）：

```python
@dataclass
class _ClarifyEntry:
    clarify_id: str
    session_key: str
    question: str
    choices: Optional[List[str]]
    event: threading.Event          # agent 线程阻塞在此
    response: Optional[str] = None  # 适配器设置
    awaiting_text: bool = False     # 选了 Other / 开放式 → 等文本输入
```

| API | 行号 | 调用方 | 作用 |
|------|------|--------|------|
| `register()` | `:74` | agent 线程 | 登记 pending；开放式问题 `awaiting_text=True` |
| `wait_for_response()` | `:99` | agent 线程 | 1s 切片阻塞等待 + 心跳；返回响应或 None |
| `resolve_gateway_clarify()` | `:146` | gateway | 设 response、`event.set()` 唤醒 agent |
| `get_pending_for_session()` | `:161` | gateway | FIFO 取该 session 待文本回答的 pending |
| `mark_awaiting_text()` | `:179` | gateway | 用户点 Other → 切文本捕获模式 |
| `clear_session()` | `:199` | gateway | 会话结束清理所有 pending，防 agent 线程永挂 |
| `get_clarify_timeout()` | `:227` | agent 线程 | 读 `agent.clarify_timeout`，默认 600s |
| `register_notify()` | `:257` | gateway | 为 session 注册通知回调 |

`get_clarify_timeout` 的取值权衡：**足够长**让用户从容作答，又**足够短**避免触碰 gateway inactivity watchdog（通常 10–30 分钟）的红线。

---

## 权力隔离：子代理禁用 clarify

子代理被**双重防护**地剥夺了 clarify 能力：

1. `tools/delegate_tool.py:1124` 构造子代理时显式 `clarify_callback=None`——即使工具被加上也无回调可用，`clarify_tool` 直接返回 `{"error": "Clarify tool is not available in this execution context."}`。
2. `clarify` 同时在 `DELEGATE_BLOCKED_TOOLS` 黑名单里（见 [委派系统](delegation-system.md#子代理工具剥夺)），工具本身就不会出现在子代理的工具集中。

设计意图——**主代理是唯一与用户对话的实体，子代理只能"汇报"不能"打断"**：

- 多个并行子代理同时向用户发问 → 用户困惑"谁在跟我说话"，对话历史混乱；
- 子代理的澄清响应跨线程无法可靠路由回正确实例；
- 子代理需要用户决策时，应**结构化返回**给父代理，由父代理用自己的 clarify 询问后再派新子代理继续。完整方案见 [子代理与用户交互模式](subagent-user-interaction.md)。

> clarify 与 delegate 是一对镜像设计：**delegate 让 agent 向下派活，clarify 让 agent 向上问人，而子代理两者皆不可——既不能再派，也不能越级对话。**

---

## 优雅降级：每条失败路径都有兜底

clarify **从不卡死**——所有异常都返回一段告诉模型"接下来该怎么办"的文字，让对话继续：

| 情况 | 返回 |
|------|------|
| 子代理调用（callback=None） | `{"error": "Clarify tool is not available in this execution context."}` |
| callback 抛异常 | `{"error": "Failed to get user input: <exc>"}` |
| CLI 120s 超时 | `"...did not provide a response within the time limit. Use your best judgement..."` |
| Gateway 超时 | `"[user did not respond within 10m]"` |
| Gateway 派送失败 | `"[clarify prompt could not be delivered]"` |
| 会话被 `/new`/关闭清理 | `clear_session` 设空串 + `event.set()`，唤醒并放行 agent 线程 |
| Oneshot 无人 | `"[oneshot mode: no user available. Make the most reasonable assumption...]"` |

---

## 关键文件速查

| 功能 | 文件:行 | 符号 |
|------|---------|------|
| 工具逻辑 | `tools/clarify_tool.py:23` | `clarify_tool(question, choices, callback)` |
| schema / MAX_CHOICES | `tools/clarify_tool.py:87` / `:20` | `CLARIFY_SCHEMA` / `MAX_CHOICES=4` |
| 工具注册 | `tools/clarify_tool.py:131` | `registry.register(...)` |
| 工具执行入口 | `agent/tool_executor.py:929` | 透传 `agent.clarify_callback` |
| callback 注入 | `agent/agent_init.py:386` | `agent.clarify_callback = ...` |
| CLI callback | `cli.py:11689` | `_clarify_callback` |
| CLI 渲染 | `cli.py:14437` | `_get_clarify_display` |
| Gateway callback | `gateway/run.py:17908` | `_clarify_callback_sync` |
| Telegram 发送 | `gateway/platforms/telegram.py` | `send_clarify`（内联按钮 `cl:{id}:{idx}`） |
| 平台默认实现 | `gateway/platforms/base.py` | `send_clarify`（文本编号列表回退） |
| 跨线程注册表 | `tools/clarify_gateway.py` | `register / wait_for_response / resolve_gateway_clarify ...` |
| Oneshot | `hermes_cli/oneshot.py:377` | `_oneshot_clarify_callback` |
| 子代理禁用 | `tools/delegate_tool.py:1124` | `clarify_callback=None` |

---

## 设计哲学

clarify 集中体现了 Hermes 的四条原则：

- **环境无关的工具 + 环境相关的回调** —— 一套 agent 逻辑适配 CLI/Telegram/oneshot/子代理。
- **线程安全 + 同步/异步桥接** —— `threading.Event` 注册表让阻塞的 agent 线程与异步 gateway 事件循环安全交接。
- **心跳保活** —— 1 秒切片轮询，等用户期间持续打活动心跳，不被看门狗误杀。
- **权力隔离 + 全路径兜底** —— 对话权单一（仅主代理），且每条失败路径都有可继续的返回，永不卡死。

---

## 参考

- [委派系统](delegation-system.md) —— delegate_task 与子代理隔离；clarify 在黑名单中
- [委派系统 - 子代理如何与用户交互（间接）](delegation-system.md#子代理如何与用户交互间接) —— 子代理的输出/控制通道
- [子代理与用户交互模式](subagent-user-interaction.md) —— 子代理"需要问用户"时的 5 种应对方案
