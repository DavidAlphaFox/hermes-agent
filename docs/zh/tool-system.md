# 工具系统设计

> 自注册工具架构、调度流程、工具集组合、各工具详解

---

## 目录

- [概述](#概述)
- [注册机制](#注册机制)
- [调度流程](#调度流程)
- [工具集系统](#工具集系统)
- [核心工具详解](#核心工具详解)
- [工具审批机制](#工具审批机制)
- [异步桥接](#异步桥接)
- [MCP 工具集成](#mcp-工具集成)
- [添加新工具](#添加新工具)

---

## 概述

Hermes Agent 的工具系统采用**自注册架构**：每个工具文件在导入时自动向中央注册表声明自己的 schema、handler 和元数据，无需在中心文件中手动维护工具列表。

```
tools/registry.py   <--  tools/*.py   <--  model_tools.py   <--  run_agent.py
  (注册表定义)          (工具注册)        (发现与编排)          (消费者)
```

### 设计优势

| 优势 | 说明 |
|------|------|
| 去中心化 | 添加工具只需创建文件，无需修改中心注册表 |
| 循环依赖安全 | 注册表不导入任何工具文件 |
| 运行时可发现 | 支持动态工具（MCP 服务器运行时注册/注销） |
| 条件可用 | 每个工具可以声明可用性检查函数 |

---

## 注册机制

### ToolEntry 结构

```python
class ToolEntry:
    """单个注册工具的元数据。"""
    __slots__ = (
        "name",         # 工具名称 (唯一标识)
        "toolset",      # 所属工具集
        "schema",       # OpenAI function schema
        "handler",      # 处理函数 (sync 或 async)
        "check_fn",     # 可用性检查函数
        "requires_env", # 需要的环境变量
        "is_async",     # 是否为异步处理器
        "description",  # 人类可读描述
        "emoji",        # 显示用 emoji
    )
```

### ToolRegistry 类

```python
class ToolRegistry:
    """单例注册表，收集所有工具文件的 schema 和 handler。"""
    
    def __init__(self):
        self._tools: Dict[str, ToolEntry] = {}
        self._toolset_checks: Dict[str, Callable] = {}
    
    def register(self, name, toolset, schema, handler,
                 check_fn=None, requires_env=None,
                 is_async=False, description="", emoji=""):
        """注册工具。在各工具模块导入时调用。"""
        self._tools[name] = ToolEntry(...)
    
    def deregister(self, name):
        """注销工具。MCP 动态工具重建时使用。"""
        self._tools.pop(name, None)
```

### 注册示例

```python
# tools/web_tools.py
from tools.registry import registry

def _handle_web_search(function_args, task_id=None, user_task=None):
    query = function_args.get("query", "")
    # ... 执行搜索 ...
    return json.dumps(results)

registry.register(
    name="web_search",
    toolset="web",
    schema={
        "type": "function",
        "function": {
            "name": "web_search",
            "description": "Search the web for information",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "Search query"},
                },
                "required": ["query"],
            },
        },
    },
    handler=_handle_web_search,
    emoji="🔍",
)
```

---

## 调度流程

### model_tools.py -- 编排层

`model_tools.py` 是工具系统的公共 API 层，连接注册表与 AIAgent：

```
get_tool_definitions(enabled_toolsets, disabled_toolsets)
    |
    |-- 1. 触发工具发现 (导入所有 tools/*.py)
    |
    |-- 2. 从 registry 获取所有工具
    |
    |-- 3. 按工具集过滤
    |       - 仅保留 enabled_toolsets 中的工具
    |       - 排除 disabled_toolsets 中的工具
    |
    |-- 4. 检查可用性
    |       - 运行每个工具的 check_fn
    |       - 检查 requires_env 环境变量
    |
    +-- 5. 返回 OpenAI function schema 列表
```

### 工具调用分发

```
handle_function_call(function_name, function_args, task_id, user_task)
    |
    |-- 1. 从 registry 查找 ToolEntry
    |
    |-- 2. 检查工具是否可用
    |
    |-- 3. 执行 handler
    |       |
    |       |-- 同步工具: 直接调用 handler(function_args, ...)
    |       |
    |       +-- 异步工具: 通过持久事件循环执行
    |               loop = _get_tool_loop()
    |               loop.run_until_complete(handler(function_args, ...))
    |
    |-- 4. 格式化结果
    |       - 成功: 返回 tool_result 字符串
    |       - 失败: 返回 tool_error 字符串
    |
    +-- 5. 返回结果
```

---

## 工具集系统

工具集（Toolset）允许将工具分组，支持嵌套组合和场景预设。

### 定义在 toolsets.py

```python
# 核心工具列表 (CLI 和所有平台共享)
_HERMES_CORE_TOOLS = [
    "web_search", "web_extract",
    "terminal", "process",
    "read_file", "write_file", "patch", "search_files",
    "vision_analyze", "image_generate",
    "skills_list", "skill_view", "skill_manage",
    "browser_navigate", "browser_snapshot", "browser_click",
    "browser_type", "browser_scroll", "browser_back",
    "todo", "memory", "session_search", "clarify",
    "execute_code", "delegate_task", "cronjob", "send_message",
    ...
]

# 工具集定义 (支持嵌套组合)
TOOLSETS = {
    "web": {
        "description": "Web research and content extraction",
        "tools": ["web_search", "web_extract"],
        "includes": [],
    },
    "terminal": {
        "description": "Terminal command execution",
        "tools": ["terminal", "process"],
        "includes": [],
    },
    "research": {
        "description": "Research toolkit",
        "tools": [],
        "includes": ["web", "vision"],  # 组合其他工具集
    },
    "development": {
        "description": "Software development toolkit",
        "tools": [],
        "includes": ["terminal", "file", "code_execution"],
    },
    "full_stack": {
        "description": "All tools",
        "tools": [],
        "includes": ["development", "research", "browser"],
    },
}
```

### 工具集解析

```python
def resolve_toolset(name: str) -> Set[str]:
    """递归解析工具集，展开所有 includes。"""
    toolset = TOOLSETS.get(name)
    tools = set(toolset["tools"])
    for include in toolset["includes"]:
        tools |= resolve_toolset(include)  # 递归展开
    return tools
```

### 工具集层次

```
full_stack
    |-- development
    |       |-- terminal: [terminal, process]
    |       |-- file: [read_file, write_file, patch, search_files]
    |       +-- code_execution: [execute_code]
    |
    |-- research
    |       |-- web: [web_search, web_extract]
    |       +-- vision: [vision_analyze]
    |
    +-- browser
            [browser_navigate, browser_snapshot, ...]
```

---

## 核心工具详解

### 终端工具 (terminal_tool.py, 69KB)

支持 6 种执行后端：

| 后端 | 说明 | 使用场景 |
|------|------|----------|
| `local` | 本地 subprocess | 默认 CLI 模式 |
| `docker` | Docker 容器内执行 | 沙箱隔离 |
| `ssh` | SSH 远程执行 | 远程服务器 |
| `modal` | Modal.com 云执行 | 云端沙箱 |
| `daytona` | Daytona 工作空间 | 开发环境 |
| `singularity` | Singularity 容器 | HPC 环境 |

```python
# 终端工具注册
registry.register(
    name="terminal",
    toolset="terminal",
    schema={...},
    handler=_handle_terminal,
    check_fn=_check_terminal_available,
)
```

### 浏览器工具 (browser_tool.py, 83KB)

基于 Playwright 的浏览器自动化，注册多个细粒度工具：

| 工具名 | 功能 |
|--------|------|
| `browser_navigate` | 导航到 URL |
| `browser_snapshot` | 获取页面快照 (无障碍树) |
| `browser_click` | 点击元素 |
| `browser_type` | 输入文本 |
| `browser_scroll` | 滚动页面 |
| `browser_back` | 后退导航 |
| `browser_press` | 按键操作 |
| `browser_get_images` | 获取页面图片 |
| `browser_vision` | 页面截图分析 |
| `browser_console` | 查看控制台日志 |

### 文件操作工具 (file_operations.py, 41KB + file_tools.py, 37KB)

| 工具名 | 功能 |
|--------|------|
| `read_file` | 读取文件内容 (支持行范围) |
| `write_file` | 写入文件 (原子写入) |
| `patch` | 精确补丁 (搜索+替换) |
| `search_files` | 文件内容搜索 (ripgrep) |

### MCP 工具 (mcp_tool.py, 81KB)

Model Context Protocol 桥接，支持三种传输方式：

| 传输方式 | 说明 |
|----------|------|
| `stdio` | 标准输入输出 (本地进程) |
| `sse` | Server-Sent Events (HTTP) |
| `streamable-http` | 可流式 HTTP |

MCP 工具在运行时动态注册/注销：

```python
# MCP 服务器发送 tools/list_changed 通知时
registry.deregister(old_tool_name)    # 清除旧工具
registry.register(new_tool_name, ...) # 注册新工具
```

### 委托工具 (delegate_tool.py, 40KB)

创建子 Agent 执行独立任务：

```python
registry.register(
    name="delegate_task",
    toolset="delegation",
    schema={
        "function": {
            "name": "delegate_task",
            "parameters": {
                "properties": {
                    "task": {"type": "string"},
                    "context": {"type": "string"},
                    "tools": {"type": "array"},
                },
            },
        },
    },
    handler=_handle_delegate_task,
)
```

### 其他重要工具

| 工具文件 | 工具名 | 功能 |
|----------|--------|------|
| `web_tools.py` (85KB) | `web_search`, `web_extract` | 网页搜索与内容提取 |
| `memory_tool.py` (22KB) | `memory` | 读写持久化记忆 |
| `code_execution_tool.py` (51KB) | `execute_code` | 沙箱代码执行 |
| `skills_tool.py` (49KB) | `skills_list`, `skill_view` | 技能管理 |
| `session_search_tool.py` (20KB) | `session_search` | 会话历史搜索 |
| `send_message_tool.py` (39KB) | `send_message` | 跨平台消息发送 |
| `clarify_tool.py` (5KB) | `clarify` | 向用户请求澄清 |
| `todo_tool.py` (9KB) | `todo` | 任务管理 |
| `vision_tools.py` (22KB) | `vision_analyze` | 图像分析 |
| `image_generation_tool.py` (27KB) | `image_generate` | AI 图像生成 |
| `transcription_tools.py` (24KB) | 语音转文字 | 音频/视频转录 |
| `tts_tool.py` (36KB) | `text_to_speech` | 文字转语音 |
| `cronjob_tools.py` (20KB) | `cronjob` | 定时任务管理 |
| `homeassistant_tool.py` (16KB) | `ha_*` | Home Assistant 控制 |
| `rl_training_tool.py` (56KB) | RL 训练相关 | 强化学习训练 |

---

## 工具审批机制

`tools/approval.py` (34KB) 实现了危险操作的审批机制：

```
工具调用
    |
    |-- 是否需要审批？
    |       - 检查命令内容 (rm -rf, sudo, etc.)
    |       - 检查工具配置中的审批规则
    |
    |-- 需要审批:
    |       |-- CLI 模式: 弹出确认提示
    |       |-- Gateway 模式: 设置 HERMES_EXEC_ASK=1
    |       |-- ACP 模式: 通过权限回调请求
    |       |
    |       |-- 用户批准 -> 执行
    |       +-- 用户拒绝 -> 返回拒绝消息给 LLM
    |
    +-- 不需要审批: 直接执行
```

---

## 异步桥接

许多工具的底层实现是异步的（如浏览器、MCP、网络请求），但 AIAgent 的工具调用循环运行在同步上下文中。`model_tools.py` 提供了异步桥接机制：

```python
# 主线程使用持久事件循环
_tool_loop = None

def _get_tool_loop():
    """获取持久事件循环，避免循环创建/关闭导致的问题。"""
    global _tool_loop
    if _tool_loop is None or _tool_loop.is_closed():
        _tool_loop = asyncio.new_event_loop()
    return _tool_loop

# 工作线程 (delegate_task) 使用线程本地事件循环
_worker_thread_local = threading.local()

def _get_worker_loop():
    """每个工作线程拥有独立的持久事件循环。"""
    loop = getattr(_worker_thread_local, 'loop', None)
    if loop is None or loop.is_closed():
        loop = asyncio.new_event_loop()
        asyncio.set_event_loop(loop)
        _worker_thread_local.loop = loop
    return loop
```

### 为什么不用 asyncio.run()？

`asyncio.run()` 每次调用会创建新的事件循环并在结束时关闭它。但 httpx/AsyncOpenAI 等客户端会缓存到事件循环中，循环关闭后客户端变为无效，导致 "Event loop is closed" 错误。持久事件循环避免了这个问题。

---

## MCP 工具集成

MCP (Model Context Protocol) 工具支持动态发现和注册外部工具服务器：

```
MCP 服务器配置 (config.yaml)
    |
    |-- 启动时连接各 MCP 服务器
    |
    |-- 调用 tools/list 获取工具列表
    |
    |-- 为每个工具调用 registry.register()
    |       name = "mcp_{server}_{tool}"
    |       toolset = "mcp"
    |       handler = mcp_proxy_handler
    |
    |-- 运行时: 收到 tools/list_changed 通知
    |       -> deregister 旧工具
    |       -> register 新工具
    |
    +-- 工具调用时:
            -> 序列化参数
            -> 通过 MCP 传输发送到服务器
            -> 反序列化结果
```

### MCP OAuth 认证

`tools/mcp_oauth.py` (17KB) 实现了 MCP 服务器的 OAuth 认证流程，支持在 CLI 和 Gateway 模式下获取和刷新 OAuth 令牌。

---

## 添加新工具

添加新工具的步骤：

1. 创建 `tools/my_tool.py`
2. 导入注册表并注册工具
3. 完成 -- 无需修改其他文件

```python
# tools/my_tool.py
from tools.registry import registry

def _handle_my_tool(function_args, task_id=None, user_task=None):
    """工具处理函数。"""
    param = function_args.get("param", "")
    # ... 实现逻辑 ...
    return json.dumps({"result": "success"})

def _check_available():
    """可选：检查工具是否可用。"""
    return True, ""

registry.register(
    name="my_tool",
    toolset="my_toolset",
    schema={
        "type": "function",
        "function": {
            "name": "my_tool",
            "description": "My custom tool",
            "parameters": {
                "type": "object",
                "properties": {
                    "param": {
                        "type": "string",
                        "description": "Tool parameter",
                    },
                },
                "required": ["param"],
            },
        },
    },
    handler=_handle_my_tool,
    check_fn=_check_available,
    emoji="🔧",
)
```

如果工具是异步的：

```python
async def _handle_my_async_tool(function_args, task_id=None, user_task=None):
    result = await some_async_operation()
    return json.dumps(result)

registry.register(
    name="my_async_tool",
    ...,
    handler=_handle_my_async_tool,
    is_async=True,
)
```

---

## 相关文档

- [核心 Agent 循环](core-agent-loop.md) -- 工具调用在 Agent 循环中的位置
- [MCP 工具集成](#mcp-工具集成) -- MCP 协议详解
- [模块依赖关系](module-dependency.md) -- 工具系统的导入链
- [数据流图](data-flow.md) -- 工具执行数据流
