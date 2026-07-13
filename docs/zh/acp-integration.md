# ACP 编辑器集成

> Agent Client Protocol 协议实现、会话管理、权限控制、工具转换

---

## 目录

- [概述](#概述)
- [协议架构](#协议架构)
- [服务器实现](#服务器实现)
- [会话管理](#会话管理)
- [权限控制](#权限控制)
- [工具 Schema 转换](#工具-schema-转换)
- [事件桥接](#事件桥接)
- [编辑器集成](#编辑器集成)
- [启动流程](#启动流程)

---

## 概述

Hermes Agent 通过 **Agent Client Protocol (ACP)** 与编辑器集成，将 AI Agent 能力直接嵌入 VS Code、Zed 和 JetBrains 等 IDE。ACP 适配器将 Hermes 的 AIAgent 封装为符合 ACP 规范的 JSON-RPC 服务，通过 stdio 传输协议与编辑器通信。

```
┌─────────────┐    JSON-RPC (stdio)    ┌─────────────────┐
│   编辑器     │ ◄──────────────────►  │  ACP 适配器      │
│  (VS Code/  │                        │  acp_adapter/   │
│   Zed/JB)   │                        │  server.py      │
└─────────────┘                        └───────┬─────────┘
                                               │
                                        ┌──────▼──────┐
                                        │   AIAgent   │
                                        │ run_agent.py│
                                        └─────────────┘
```

### 模块结构

| 文件 | 大小 | 职责 |
|------|------|------|
| `__init__.py` | 67B | 包声明 |
| `__main__.py` | 99B | `python -m acp_adapter` 入口 |
| `entry.py` | 2.4KB | 启动流程：加载 .env → 配置日志 → 启动服务 |
| `server.py` | 27.5KB | ACP Agent 主类，协议方法实现 |
| `session.py` | 17.6KB | 会话生命周期管理 + SQLite 持久化 |
| `permissions.py` | 2.5KB | ACP 权限请求到 Hermes 审批的桥接 |
| `tools.py` | 7.1KB | Hermes 工具到 ACP ToolKind 的映射 |
| `events.py` | 5.5KB | AIAgent 回调到 ACP 会话更新的桥接 |
| `auth.py` | 831B | 运行时供应商检测 |

---

## 协议架构

### ACP 协议生命周期

```
编辑器                                      HermesACPAgent
  │                                              │
  ├─── initialize ────────────────────────────► │
  │    (protocol_version, client_info)           │
  │ ◄─── InitializeResponse ──────────────────── │
  │    (agent_info, capabilities, auth_methods)  │
  │                                              │
  ├─── authenticate ──────────────────────────► │
  │ ◄─── AuthenticateResponse ────────────────── │
  │                                              │
  ├─── new_session ───────────────────────────► │
  │    (cwd, mcp_servers)                        │
  │ ◄─── NewSessionResponse ──────────────────── │
  │    (session_id)                              │
  │                                              │
  │    ┌──── prompt loop ────────────────────┐   │
  ├────┤ prompt(text, session_id) ──────────►│   │
  │    │ ◄── session_update (streaming) ──── │   │
  │    │ ◄── PromptResponse ─────────────── │   │
  │    └─────────────────────────────────────┘   │
  │                                              │
  ├─── cancel(session_id) ───────────────────► │
  ├─── fork_session ─────────────────────────► │
  ├─── list_sessions ────────────────────────► │
  └─── set_session_model ────────────────────► │
```

### 能力声明

```python
AgentCapabilities(
    session_capabilities=SessionCapabilities(
        fork=SessionForkCapabilities(),   # 支持会话分叉
        list=SessionListCapabilities(),   # 支持会话列表
    ),
)
```

### 认证方法

ACP 适配器通过 `detect_provider()` 检测当前配置的供应商，并将其作为认证方法声明给编辑器：

```python
auth_methods = [
    AuthMethodAgent(
        id=provider,
        name=f"{provider} runtime credentials",
        description=f"Authenticate Hermes using {provider} credentials",
    )
]
```

---

## 服务器实现

### HermesACPAgent 类

`server.py` 中的 `HermesACPAgent` 继承 `acp.Agent`，实现所有 ACP 协议方法：

```python
class HermesACPAgent(acp.Agent):
    """ACP Agent 实现，封装 Hermes AIAgent。"""

    def __init__(self, session_manager=None):
        self.session_manager = session_manager or SessionManager()
        self._conn: Optional[acp.Client] = None
```

### Prompt 处理流程

`prompt()` 是核心方法，处理用户输入并返回 Agent 回复：

```
1. 获取会话状态 (session_id → SessionState)
2. 提取纯文本 (_extract_text: TextContentBlock → str)
3. 拦截斜杠命令 (/help, /model, /tools, /reset, /compact, /version)
4. 创建回调:
   ├─ tool_progress_cb  → ACP ToolCallStart 事件
   ├─ thinking_cb       → ACP ThoughtText 更新
   ├─ step_cb           → ACP ToolCallComplete 事件
   ├─ message_cb        → ACP MessageText 更新
   └─ approval_cb       → ACP 权限请求桥接
5. 在 ThreadPoolExecutor 中运行 agent.run_conversation()
6. 更新会话历史 → 持久化到 SQLite
7. 返回 PromptResponse(stop_reason, usage)
```

### 斜杠命令

编辑器可发送斜杠命令，服务器本地处理而不调用 LLM：

| 命令 | 处理函数 | 功能 |
|------|----------|------|
| `/help` | `_cmd_help()` | 列出可用命令 |
| `/model [name]` | `_cmd_model()` | 查看或切换模型 |
| `/tools` | `_cmd_tools()` | 列出可用工具 |
| `/context` | `_cmd_context()` | 显示对话统计 |
| `/reset` | `_cmd_reset()` | 清空对话历史 |
| `/compact` | `_cmd_compact()` | 压缩上下文 |
| `/version` | `_cmd_version()` | 显示版本 |

命令通过 `AvailableCommandsUpdate` 向编辑器通告，编辑器可提供自动补全。

### 线程模型

```
主线程 (asyncio 事件循环)          工作线程 (ThreadPoolExecutor)
  │                                    │
  ├─ ACP JSON-RPC 收发                ├─ agent.run_conversation()
  ├─ session_update() 推送            ├─ 工具执行
  └─ 权限请求等待                      └─ 回调 → run_coroutine_threadsafe
```

AIAgent 的 `run_conversation()` 是同步方法，运行在 `ThreadPoolExecutor(max_workers=4)` 中。回调通过 `asyncio.run_coroutine_threadsafe()` 将 ACP 更新推送回主线程的事件循环。

### MCP 服务器注册

编辑器可在 `new_session`/`load_session` 时传入 MCP 服务器配置，ACP 适配器会自动注册这些服务器并刷新工具表面：

```python
async def _register_session_mcp_servers(self, state, mcp_servers):
    # 1. 将 ACP McpServer 配置转换为 Hermes 格式
    # 2. 调用 register_mcp_servers() 注册
    # 3. 调用 get_tool_definitions() 刷新工具列表
    # 4. 更新 agent.tools 和 agent.valid_tool_names
```

---

## 会话管理

### SessionManager 类

`session.py` 中的 `SessionManager` 管理 ACP 会话到 Hermes AIAgent 实例的映射：

```python
class SessionManager:
    """线程安全的 ACP 会话管理器。"""

    def __init__(self, agent_factory=None, db=None):
        self._sessions: Dict[str, SessionState] = {}  # 内存缓存
        self._lock = Lock()                             # 线程安全
        self._agent_factory = agent_factory             # 测试注入
        self._db_instance = db                          # 懒加载 SessionDB
```

### SessionState 数据结构

```python
@dataclass
class SessionState:
    session_id: str                      # UUID
    agent: Any                           # AIAgent 实例
    cwd: str = "."                       # 编辑器工作目录
    model: str = ""                      # 当前模型
    history: List[Dict[str, Any]] = []   # 对话历史
    cancel_event: threading.Event = None # 取消信号
```

### 会话生命周期

```
create_session(cwd)          → 创建新会话 + AIAgent + SQLite 记录
get_session(session_id)      → 内存查找 → 数据库恢复
update_cwd(session_id, cwd)  → 更新工作目录 + 工具环境
fork_session(session_id)     → 深拷贝历史到新会话
remove_session(session_id)   → 清理内存 + 数据库 + 工具环境
save_session(session_id)     → 持久化到 SQLite
cleanup()                    → 清理所有会话
```

### 持久化

会话通过 `SessionDB` (SQLite) 持久化到 `~/.hermes/state.db`：

- 会话记录：source="acp", model, model_config (包含 cwd, provider, base_url)
- 消息历史：所有对话消息按序存储
- 进程重启后自动恢复：`_restore()` 从数据库重建 AIAgent 实例

### 工作目录绑定

每个 ACP 会话的工作目录通过 `register_task_env_overrides()` 绑定到工具系统：

```python
# 创建/恢复会话时
register_task_env_overrides(session_id, {"cwd": cwd})

# 终端工具执行时，自动使用该 session 的 cwd
# 确保每个编辑器窗口的工具在正确的目录下运行
```

### stdout 保护

ACP 使用 stdio 传输 JSON-RPC，因此 AIAgent 的所有输出必须重定向到 stderr：

```python
agent._print_fn = _acp_stderr_print  # 所有 print 重定向到 stderr
```

---

## 权限控制

### 审批桥接

`permissions.py` 将 ACP 的权限请求机制桥接到 Hermes 的工具审批回调：

```
Hermes 工具执行
  └─ approval_callback(command, description)
       │
       ├─ 构建 ACP PermissionOption 列表:
       │    ├─ "Allow once"   (allow_once)
       │    ├─ "Allow always" (allow_always)
       │    └─ "Deny"         (reject_once)
       │
       ├─ 调用 conn.request_permission() → 编辑器弹窗
       │
       └─ 映射响应:
            allow_once  → "once"
            allow_always → "always"
            reject_*     → "deny"
```

### 超时处理

权限请求设有 60 秒超时，超时自动拒绝：

```python
future = asyncio.run_coroutine_threadsafe(coro, loop)
response = future.result(timeout=60)  # 60 秒等待编辑器用户响应
```

---

## 工具 Schema 转换

### ToolKind 映射

`tools.py` 将 Hermes 工具名映射到 ACP 的 ToolKind 枚举：

| Hermes 工具 | ACP ToolKind | 编辑器展示 |
|-------------|-------------|-----------|
| `read_file` | `read` | 文件读取图标 |
| `write_file`, `patch` | `edit` | 文件编辑图标 |
| `search_files` | `search` | 搜索图标 |
| `terminal`, `execute_code` | `execute` | 终端执行图标 |
| `web_search`, `web_extract` | `fetch` | 网络请求图标 |
| `browser_*` | `fetch`/`execute`/`read` | 浏览器操作图标 |
| `_thinking` | `think` | 思考图标 |
| 其他 | `other` | 默认图标 |

### 工具调用内容构建

不同工具生成不同的 ACP 内容类型：

```python
# patch 工具 → diff 内容
acp.tool_diff_content(path=path, new_text=new, old_text=old)

# write_file → diff 内容（新文件创建）
acp.tool_diff_content(path=path, new_text=content)

# terminal → 文本命令
acp.tool_content(acp.text_block(f"$ {command}"))

# 通用 → JSON 参数
acp.tool_content(acp.text_block(json.dumps(arguments)))
```

### 位置提取

工具调用可提取文件位置信息，编辑器可跳转到相关文件：

```python
def extract_locations(arguments):
    path = arguments.get("path")
    line = arguments.get("offset") or arguments.get("line")
    return [ToolCallLocation(path=path, line=line)]
```

---

## 事件桥接

### 回调工厂

`events.py` 提供四个回调工厂函数，将 AIAgent 的事件流桥接到 ACP 会话更新：

| 工厂函数 | AIAgent 回调 | ACP 事件 |
|----------|-------------|---------|
| `make_tool_progress_cb` | `tool_progress_callback` | `ToolCallStart` |
| `make_thinking_cb` | `thinking_callback` | `update_agent_thought_text` |
| `make_step_cb` | `step_callback` | `ToolCallProgress` (completed) |
| `make_message_cb` | `message_callback` | `update_agent_message_text` |

### 工具调用 ID 跟踪

工具调用使用 FIFO 队列跟踪 ID，支持并行同名工具调用的正确配对：

```python
tool_call_ids: Dict[str, Deque[str]]

# tool.started → 生成 ID, 入队
tc_id = make_tool_call_id()  # "tc-xxxxxxxxxxxx"
tool_call_ids[name].append(tc_id)

# step_callback → 出队, 发送完成事件
tc_id = tool_call_ids[tool_name].popleft()
build_tool_complete(tc_id, tool_name, result)
```

### 跨线程通信

所有回调使用 `asyncio.run_coroutine_threadsafe()` 从工作线程安全地推送到主事件循环：

```python
def _send_update(conn, session_id, loop, update):
    future = asyncio.run_coroutine_threadsafe(
        conn.session_update(session_id, update), loop
    )
    future.result(timeout=5)  # 5 秒超时
```

---

## 编辑器集成

### VS Code

通过 ACP 扩展连接，使用 stdio 传输：

```json
{
  "command": "hermes",
  "args": ["acp"],
  "transport": "stdio"
}
```

### Zed

Zed 编辑器原生支持 ACP 协议，配置方式类似。

### JetBrains

通过 JetBrains ACP 插件连接，支持 IntelliJ、PyCharm 等全系列 IDE。

---

## 启动流程

### entry.py 启动序列

```python
def main():
    # 1. 配置日志（全部输出到 stderr，保护 stdout 用于 JSON-RPC）
    _setup_logging()
    #    - 压制 httpx, httpcore, openai 的 WARNING 以下日志

    # 2. 加载环境变量
    _load_env()
    #    - ~/.hermes/.env → 系统环境变量

    # 3. 确保项目根目录在 sys.path
    sys.path.insert(0, project_root)

    # 4. 创建并启动 ACP Agent
    agent = HermesACPAgent()
    asyncio.run(acp.run_agent(agent, use_unstable_protocol=True))
```

### 启动命令

```bash
# 方式 1: hermes 子命令
hermes acp

# 方式 2: Python 模块
python -m acp_adapter

# 方式 3: 直接入口
python acp_adapter/entry.py
```
