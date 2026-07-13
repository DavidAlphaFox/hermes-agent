# 多业务工具场景适配指南

> 当业务工具数量多、按业务域分组、且跨域工作流频繁时，如何用 Hermes 现有能力适配

---

## 目录

- [问题场景](#问题场景)
- [Hermes 已具备的能力](#hermes-已具备的能力)
- [4 种适配模式](#4-种适配模式)
- [推荐方案：C + D 混合](#推荐方案c--d-混合)
- [改造步骤](#改造步骤)
- [改造点对照表](#改造点对照表)
- [容易踩的坑](#容易踩的坑)
- [参考](#参考)

---

## 问题场景

典型企业 Agent 场景：

- 工具数量多（几十到几百）
- 工具按**业务域**自然分组：项目管理、订单管理、配送管理、客户管理...
- 业务域之间功能差异**很大**，几乎没有共享工具
- 部分工作流**跨域**：例如"修改订单 → 重新规划配送路径"

直接把所有工具都加载到一个 Agent 里会面临：

- **系统提示词膨胀**：所有工具描述都被注入提示词，token 成本高
- **工具选择准确率下降**：模型在 100+ 工具里挑选时容易用错
- **维护协作困难**：业务团队多人维护一个工具仓库容易冲突
- **热更新麻烦**：改一个业务工具要重启整个 Agent

---

## Hermes 已具备的能力

**好消息**：Hermes 的基础设施已经能覆盖绝大部分需求，**不需要大改造**：

| 能力 | 实现位置 | 用途 |
|------|---------|------|
| **Toolset 注册** | `tools/registry.py:72` | 工具按 `toolset` 分组注册，支持运行时启用/禁用 |
| **Toolset 组合** | `toolsets.py` 的 `includes` / `resolve_toolset` | 一个 toolset 可以引用其他 toolsets，递归解析 |
| **运行时过滤** | `cli-config.yaml` 的 `enabled_toolsets` | 启动时决定加载哪些 toolset |
| **MCP 客户端** | `tools/mcp_tool.py`（约 1050 行） | 完整 MCP 协议支持，可挂任意外部 MCP server |
| **委派工具集过滤** | `tools/delegate_tool.py:_strip_blocked_tools` | `delegate_task(toolsets=[...])` 给子代理指定工具子集 |
| **Skill 系统** | `skills/` 目录 | 描述跨域工作流的 Markdown 文件 |

详见 [工具系统](tool-system.md) / [委派系统](delegation-system.md) / [技能系统](skill-system.md)。

---

## 4 种适配模式

### 模式 A：纯 Toolset 分组，全部加载（最轻）

把每个业务域注册成独立 toolset，全部启用：

```python
# tools/business/order_tools.py
from tools.registry import registry

registry.register(
    name="create_order",
    toolset="order",                   # 业务域分组
    schema=CREATE_ORDER_SCHEMA,
    handler=lambda args, **kw: create_order_handler(args),
    check_fn=lambda: bool(os.getenv("ORDER_API_KEY")),
    emoji="📦",
)
registry.register(name="modify_order", toolset="order", ...)

# tools/business/delivery_tools.py
registry.register(name="update_delivery_path", toolset="delivery", ...)

# tools/business/project_tools.py
registry.register(name="create_project", toolset="project", ...)
```

`cli-config.yaml`：

```yaml
enabled_toolsets:
  - order
  - delivery
  - project
  - terminal      # 保留基础
  - file
```

**适用**：工具总数 < 30。
**问题**：工具一多，系统提示词膨胀，模型选错率上升。

### 模式 B：组合 toolset + 按场景加载

利用 Hermes 已有的 `includes` 组合机制（`toolsets.py`），按业务场景预定义组合：

```python
# 在 toolsets.py 中追加
_BUSINESS_TOOLSETS = {
    "order-management": {
        "description": "订单管理场景：创建/修改/查询订单，含配送联动",
        "includes": ["order", "delivery"],     # 跨域组合
    },
    "project-management": {
        "description": "项目管理场景",
        "includes": ["project", "file"],
    },
    "full-business": {
        "description": "全业务能力",
        "includes": ["order-management", "project-management"],
    },
}
```

不同入口加载不同组合：

- 订单系统 webhook → `enabled_toolsets: [order-management]`
- 客服 IM 接入 → `enabled_toolsets: [full-business]`

**适用**：业务场景边界清晰、能在会话启动前就判定。
**问题**：会话期间临时跨域比较僵硬，需要重启会话或追加 toolset。

### 模式 C：Router + Delegate_task 分发

主代理只装最小 toolset（`delegation` + `clarify` + `terminal` + `file`），**用 `delegate_task(toolsets=[...])` 按需派子代理**带特定工具集去做。

```python
# 用户："帮我把订单 #123 改成下周二配送，地址改到上海"

delegate_task(
    goal="修改订单 #123：配送日期改为 2026-05-19，配送地址改为上海市XX区XX路，"
         "并重新规划该订单的配送路径",
    context="""
    订单系统：通过 modify_order 工具修改订单字段
    配送系统：通过 update_delivery_path 工具重算路径
    流程：先 query_delivery(order_id=123) 拿现有路径 →
         modify_order 改字段 → update_delivery_path 重新计算
    """,
    toolsets=["order", "delivery"],   # 只给这次任务相关的工具
)
```

并行场景（最多 3 并发）：

```python
delegate_task(
    tasks=[
        {"goal": "创建项目 X，结构按...", "toolsets": ["project"]},
        {"goal": "订单 #100 改加急配送", "toolsets": ["order", "delivery"]},
        {"goal": "订单 #101 改加急配送", "toolsets": ["order", "delivery"]},
    ]
)
```

**适用**：域多、跨域常见、单任务边界清晰。
**优势**：
- 主代理上下文极干净，永远不被一堆业务工具描述污染
- 子代理只看到相关工具，决策准确率高
- 多个独立任务可并行
**限制**：
- `MAX_DEPTH=2`，子代理不能再嵌套委派 → 跨 3+ 域的复杂工作流要靠主代理多轮派发
- 子代理对主代理对话零记忆，必须完整传 `context`
- 每次委派有 3-10 秒启动开销

### 模式 D：MCP 服务化

把每组 tools 包装成独立 MCP server（订单服务、配送服务、项目服务各一个进程），Hermes 通过 MCP 协议接入。

```yaml
# cli-config.yaml
mcp_servers:
  order-service:
    command: "python"
    args: ["-m", "myorg.order_mcp_server"]
    env:
      ORDER_API_KEY: "${ORDER_API_KEY}"
  delivery-service:
    command: "python"
    args: ["-m", "myorg.delivery_mcp_server"]
  project-service:
    transport: "http"
    url: "https://project-svc.internal/mcp"
```

每个 server 独立写（任何语言，符合 MCP 协议即可），独立部署，独立升级。

**适用**：
- 业务团队多，各自独立开发
- 工具会频繁热更新（不想重启 Hermes）
- 想跨 agent 框架复用（Claude Desktop / Cursor / 其他 agent 也能接）

**问题**：
- 多一层进程通信，延迟略高（毫秒级）
- 复杂事务（跨 server 一致性）需要在主代理逻辑里编排
- 调试比纯 Python 注册麻烦

---

## 推荐方案：C + D 混合

针对"多业务域 + 跨域工作流"场景，**推荐 C + D 组合**（这是主流企业 agent 的事实标准）：

```
+--------------------------------------------------+
| Hermes 主代理（极简 toolset）                     |
| - delegation, clarify, terminal, file             |
+----------------+---------------------------------+
                 | delegate_task(toolsets=[...])
                 v
+--------------------------------------------------+
| 子代理（按需装载特定 toolset）                     |
+----------------+---------------------------------+
                 | 工具调用走 MCP
                 v
+--------------------------------------------------+
| MCP servers（按业务域独立部署）                    |
| +----------+  +----------+  +----------+          |
| | order    |  | delivery |  | project  |          |
| | -mcp     |  | -mcp     |  | -mcp     |          |
| +----------+  +----------+  +----------+          |
+--------------------------------------------------+
```

**好处叠加**：

- C 提供"按需加载工具集 + 上下文隔离"
- D 提供"独立开发部署 + 跨框架复用"
- 主代理 token 成本低
- 业务团队各自维护各自的 MCP server，不互相阻塞

---

## 改造步骤

### 第 1 步：把每个业务域包成 MCP server

每个域单独建 Python 模块（也可以用 Node/Go 等其他语言）：

```python
# myorg/order_mcp_server/__main__.py
from mcp.server import Server, stdio_server
from mcp.types import Tool, TextContent

server = Server("order-service")

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="create_order",
            description="创建订单。参数：customer_id, items, ...",
            inputSchema={...},
        ),
        Tool(name="modify_order", ...),
        Tool(name="query_order", ...),
    ]

@server.call_tool()
async def call_tool(name, arguments):
    if name == "create_order":
        result = await order_api.create(**arguments)
        return [TextContent(type="text", text=json.dumps(result))]
    # ...

if __name__ == "__main__":
    asyncio.run(stdio_server(server))
```

类似地建 `delivery_mcp_server`、`project_mcp_server`。

### 第 2 步：在 Hermes 配置里挂上 MCP servers

`cli-config.yaml`（或 profile 配置）：

```yaml
mcp_servers:
  order:
    command: "python"
    args: ["-m", "myorg.order_mcp_server"]
    env:
      ORDER_API_BASE: "https://order.internal/api"
      ORDER_API_KEY: "${ORDER_API_KEY}"

  delivery:
    command: "python"
    args: ["-m", "myorg.delivery_mcp_server"]

  project:
    command: "python"
    args: ["-m", "myorg.project_mcp_server"]
```

启动 Hermes 时 `tools/mcp_tool.py` 会自动连接这些 server，把工具注册成 toolset（toolset 名通常用 server 名前缀）。

### 第 3 步：主代理只装最小 toolset

主代理配置（用户面对的入口，如 telegram / cli / web）：

```yaml
enabled_toolsets:
  - delegation       # delegate_task
  - clarify          # 跟用户确认
  - terminal         # 主代理偶尔自己跑命令
  - file             # 读项目文档/skill
  # 注意：不直接装 order / delivery / project
```

### 第 4 步：写业务工作流 skill

用 Hermes 的 skill 系统（`skills/`）描述跨域工作流：

```markdown
<!-- skills/business/modify-order-with-delivery/SKILL.md -->
---
name: modify-order-with-delivery
description: 当用户要修改订单且涉及配送路径变更时使用此工作流
---

# 修改订单（含配送路径）

## 何时使用
用户请求修改订单的配送日期、配送地址、收货人时

## 工作流

1. 用 delegate_task 派一个子代理：
   - toolsets: ["order", "delivery"]
   - context 中说明先查现有路径再改

2. 子代理流程：
   - query_order(order_id) → 拿现有信息
   - modify_order(order_id, ...) → 改字段
   - update_delivery_path(order_id) → 重算路径
   - 返回摘要

## 错误处理
- 配送路径无解 → 回主代理用 clarify 问用户
```

类似地建 `skills/business/create-order/`、`skills/business/create-project/`。

主代理在系统提示里看到 skill 描述，就知道什么时候该用什么模式派活。

### 第 5 步：（可选）建立意图分流前置

如果业务请求来自不同入口（订单系统 webhook、客服 IM、内部工具），可以在 gateway 层做一次轻量识别，决定加载哪些 MCP servers：

- 订单 webhook → 启动时只挂 `order` + `delivery` MCP
- 客服 IM → 挂全部
- 内部 PM 工具 → 只挂 `project`

这样不同会话的子代理候选 toolset 自然不同。

---

## 改造点对照表

| 改造项 | 改 / 不改 | 说明 |
|--------|----------|------|
| `tools/registry.py` | **不改** | toolset 注册机制已支持 |
| `toolsets.py` | **可选追加** | 加自定义业务组合（模式 B） |
| `tools/mcp_tool.py` | **不改** | 完整 MCP 客户端已实现 |
| `tools/delegate_tool.py` | **不改** | 已支持 `toolsets=[...]` 过滤 |
| `cli-config.yaml` | **改** | 加 `mcp_servers` 配置和最小化 `enabled_toolsets` |
| 业务 MCP servers | **新增** | 每个业务域一个独立项目 |
| `skills/business/` | **新增** | 跨域工作流的 skill 描述 |

---

## 容易踩的坑

### 1. `MAX_DEPTH=2` 的硬限制

子代理不能再 `delegate_task`。如果工作流跨 3+ 域且有依赖，必须由主代理在自己的对话循环里多轮派发。

详见 [委派系统 - 深度限制](delegation-system.md#深度限制禁止多级委派)。

### 2. 子代理零记忆

`context` 字段必须自包含 -- ID、API 地址、字段约束都要传。子代理不知道父对话历史。

### 3. MCP server 长连接稳定性

`mcp_serve.py` 默认 stdio transport，每个进程长驻；崩了 Hermes 会断开该 toolset。生产环境建议带健康检查和自动重连。

### 4. 工具命名冲突

不同 MCP server 里同名工具会冲突。建议用 `order.create` / `delivery.create` 之类**带前缀的命名**。

### 5. 认证传递

MCP server 拿到的请求里没有用户身份。需要在 MCP server 启动 env 里注入 service-to-service 凭据，业务侧自己做权限。如需用户级权限隔离，可以在主代理用环境变量或 profile 切换 MCP server 实例。

### 6. 跨域事务一致性

modify_order + update_delivery_path 失败时没有自动回滚。要么：
- 业务侧实现幂等接口 + 显式补偿（在 skill 里描述失败处理）
- 或者在 MCP server 层暴露事务接口（一次调用做两件事）

### 7. 子代理需要跟用户交互

子代理被剥夺 `clarify` 工具，无法直接问用户。这是设计意图，不是 bug。详见 [子代理与用户交互模式](subagent-user-interaction.md)。

---

## 参考

- [工具系统](tool-system.md) -- 工具注册与调度
- [委派系统](delegation-system.md) -- delegate_task 工具与子代理隔离
- [技能系统](skill-system.md) -- skill 发现与加载
- [子代理与用户交互模式](subagent-user-interaction.md) -- 处理子代理需要用户输入的场景
