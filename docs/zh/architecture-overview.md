# 架构概述

> Hermes Agent 的系统定位、核心理念、技术栈与目录结构

---

## 目录

- [系统定位](#系统定位)
- [核心设计理念](#核心设计理念)
- [技术栈](#技术栈)
- [目录结构](#目录结构)
- [系统架构总览](#系统架构总览)
- [核心组件关系](#核心组件关系)
- [部署形态](#部署形态)
- [设计决策](#设计决策)

---

## 系统定位

Hermes Agent 是由 Nous Research 开发的开源、自进化 AI Agent 框架。其核心目标是提供一个**通用的、可扩展的 AI 代理运行时**，能够：

- 连接任意 OpenAI 兼容 API（本地模型、OpenRouter、各大云服务商）
- 通过工具调用与真实世界交互（终端、浏览器、文件系统、MCP 服务器等）
- 支持 19 个消息平台的网关接入（Telegram、Discord、Slack、微信企业号等）
- 集成编辑器（VS Code、Zed、JetBrains）作为 AI 编程助手
- 支持强化学习训练环境，实现 Agent 自我进化

```
+-------------------------------------------------------------------+
|                      Hermes Agent 系统边界                          |
|                                                                   |
|  +-----------+  +----------+  +---------+  +------------------+   |
|  |  CLI/TUI  |  | Gateway  |  |   ACP   |  | RL 训练环境       |   |
|  +-----------+  +----------+  +---------+  +------------------+   |
|        |              |             |              |              |
|        +------+-------+------+------+------+-------+              |
|               |              |             |                      |
|        +------v------+ +----v----+ +------v------+                |
|        |  AIAgent    | | 工具系统 | | 记忆系统     |                |
|        | (核心循环)   | | (40+)   | | (内置+插件)  |                |
|        +------+------+ +----+----+ +------+------+                |
|               |              |             |                      |
|        +------v--------------v-------------v------+               |
|        |          SQLite 持久化层 (state.db)        |               |
|        +--------------------------------------------+               |
+-------------------------------------------------------------------+
```

---

## 核心设计理念

### 1. 模型无关性

Hermes Agent 不绑定特定 LLM 提供商。通过 OpenAI 兼容 API 层，支持：

| 类别 | 示例 |
|------|------|
| 云端服务 | OpenRouter, OpenAI, Anthropic, Google, DeepSeek |
| 本地推理 | vLLM, llama.cpp, Ollama, LM Studio |
| 聚合路由 | OpenRouter (多后端自动切换) |

关键实现：`agent/model_metadata.py` 维护模型元数据，`agent/anthropic_adapter.py` 处理 Anthropic 原生 API 适配。

### 2. 自注册工具架构

工具在导入时自动注册到全局注册表，无需在中心文件中手动维护列表：

```python
# tools/registry.py - 中央注册表
class ToolRegistry:
    def register(self, name, toolset, schema, handler, check_fn=None, ...):
        self._tools[name] = ToolEntry(...)

# tools/some_tool.py - 工具文件
from tools.registry import registry
registry.register(
    name="my_tool",
    toolset="my_toolset",
    schema={...},
    handler=my_handler_fn,
)
```

### 3. 单会话单 Agent

每个会话（CLI 交互、Gateway 聊天、ACP 编辑器会话）对应一个独立的 `AIAgent` 实例。Agent 实例持有完整的会话状态，包括消息历史、上下文压缩器、记忆管理器等。

### 4. 插件化记忆

记忆系统采用 Provider 模式，内置记忆（MEMORY.md）始终激活，同时最多支持一个外部记忆插件（Honcho、Mem0、Supermemory 等）。

### 5. 可组合工具集

工具通过 `toolsets.py` 分组为工具集（toolset），支持嵌套组合和场景预设：

```
research = web + vision
development = terminal + file + code_execution
full_stack = development + research + browser
```

---

## 技术栈

### 语言与运行时

| 组件 | 技术 |
|------|------|
| 主语言 | Python 3.10+ |
| 异步框架 | asyncio (Gateway/ACP), ThreadPoolExecutor (工具执行) |
| CLI 框架 | prompt_toolkit (交互式 TUI) |
| 参数解析 | argparse (CLI), fire (直接运行) |

### 存储

| 组件 | 技术 |
|------|------|
| 会话存储 | SQLite3 (WAL 模式, 支持并发读) |
| 全文检索 | SQLite FTS5 |
| 配置存储 | YAML (config.yaml) |
| 环境变量 | dotenv (.env) |
| 记忆存储 | Markdown 文件 (MEMORY.md, USER.md) |

### 网络与通信

| 组件 | 技术 |
|------|------|
| HTTP 客户端 | openai SDK, httpx, aiohttp |
| WebSocket | 各平台 SDK (discord.py, python-telegram-bot 等) |
| MCP 通信 | stdio / SSE / StreamableHTTP |
| ACP 协议 | agent-client-protocol SDK |

### AI/ML

| 组件 | 技术 |
|------|------|
| LLM 接口 | OpenAI Chat Completions API |
| RL 训练 | Atropos (基于 Gymnasium) |
| 嵌入/搜索 | 各记忆插件自带 |

---

## 目录结构

```
hermes-agent/
|
|-- run_agent.py            # AIAgent 核心类 -- 会话循环、工具调用、上下文管理
|-- cli.py                  # 交互式 CLI/TUI 入口
|-- model_tools.py          # 工具编排层 -- 连接 registry 与 AIAgent
|-- hermes_state.py         # SQLite 会话存储 (FTS5)
|-- toolsets.py             # 工具集定义与组合
|-- hermes_constants.py     # 全局常量 (HERMES_HOME 等)
|-- mcp_serve.py            # MCP 服务器 -- 对外暴露会话
|-- batch_runner.py         # 批量任务运行器
|-- rl_cli.py               # RL 训练 CLI
|
|-- agent/                  # Agent 内部模块
|   |-- prompt_builder.py       # 系统提示词构建
|   |-- context_compressor.py   # 上下文自动压缩
|   |-- memory_manager.py       # 记忆管理器
|   |-- memory_provider.py      # 记忆 Provider 抽象基类
|   |-- builtin_memory_provider.py # 内置记忆实现
|   |-- model_metadata.py       # 模型元数据 (上下文长度、能力)
|   |-- auxiliary_client.py     # 辅助 LLM 调用 (摘要、标题生成)
|   |-- credential_pool.py      # 多 API Key 凭证池
|   |-- display.py              # 终端显示 (spinner、工具预览)
|   |-- usage_pricing.py        # 用量计费估算
|   |-- anthropic_adapter.py    # Anthropic API 适配器
|   |-- prompt_caching.py       # 提示缓存 (Anthropic)
|   |-- skill_utils.py          # 技能工具函数
|   |-- insights.py             # 会话洞察分析
|   +-- ...
|
|-- tools/                  # 自注册工具集
|   |-- registry.py             # 中央注册表 (ToolRegistry)
|   |-- terminal_tool.py        # 终端工具 (6 个后端)
|   |-- browser_tool.py         # 浏览器自动化工具
|   |-- file_operations.py      # 文件操作 (read/write/patch)
|   |-- mcp_tool.py             # MCP 协议工具
|   |-- delegate_tool.py        # 子 Agent 委托
|   |-- web_tools.py            # 网页搜索与提取
|   |-- skills_tool.py          # 技能管理工具
|   |-- memory_tool.py          # 记忆读写工具
|   |-- code_execution_tool.py  # 代码执行工具
|   |-- approval.py             # 工具审批机制
|   +-- ...
|
|-- gateway/                # 多平台消息网关
|   |-- run.py                  # GatewayRunner 主控
|   |-- config.py               # 网关配置
|   |-- session.py              # 会话管理
|   |-- delivery.py             # 消息投递路由
|   |-- platforms/              # 平台适配器
|   |   |-- base.py                 # BasePlatformAdapter 抽象基类
|   |   |-- telegram.py             # Telegram
|   |   |-- discord.py              # Discord
|   |   |-- slack.py                # Slack
|   |   |-- whatsapp.py             # WhatsApp
|   |   |-- feishu.py               # 飞书
|   |   |-- wecom.py                # 企业微信
|   |   |-- matrix.py               # Matrix
|   |   +-- ...                     # 更多平台
|   +-- ...
|
|-- hermes_cli/             # CLI 命令实现
|   |-- main.py                 # 入口与命令路由
|   |-- config.py               # 配置管理
|   |-- auth.py                 # 认证体系
|   |-- gateway.py              # 网关管理命令
|   |-- setup.py                # 安装向导
|   |-- models.py               # 模型管理
|   |-- doctor.py               # 诊断工具
|   +-- ...
|
|-- acp_adapter/            # ACP 编辑器集成
|   |-- server.py               # ACP 服务器
|   |-- session.py              # 会话管理
|   |-- permissions.py          # 权限控制
|   |-- auth.py                 # 认证
|   +-- events.py               # 事件推送
|
|-- cron/                   # 定时任务
|   |-- scheduler.py            # 调度器
|   +-- jobs.py                 # 作业管理
|
|-- environments/           # RL 训练环境
|   |-- hermes_base_env.py      # 基础环境抽象
|   |-- agent_loop.py           # Agent 循环适配
|   |-- agentic_opd_env.py      # OPD 环境
|   |-- web_research_env.py     # 网页研究环境
|   +-- hermes_swe_env/         # SWE-Bench 环境
|
|-- plugins/memory/         # 记忆插件
|   |-- honcho/                 # Honcho AI
|   |-- mem0/                   # Mem0
|   |-- supermemory/            # Supermemory
|   |-- holographic/            # Holographic
|   +-- ...
|
|-- skills/                 # 技能知识库 (YAML + Markdown)
|   |-- software-development/
|   |-- research/
|   |-- devops/
|   |-- creative/
|   +-- ... (30+ 分类)
|
+-- tests/                  # 测试套件
```

---

## 系统架构总览

```
+--------------------------------------------------------------+
|                       用户交互层                               |
|  +--------+  +-----------+  +-------+  +------------------+  |
|  |  CLI   |  |  Gateway  |  |  ACP  |  |  批量/RL/MCP     |  |
|  | (TUI)  |  | (19平台)   |  | (IDE) |  |  运行器          |  |
|  +---+----+  +-----+-----+  +---+---+  +--------+---------+  |
|      |             |            |               |             |
+------+-------------+------------+---------------+-------------+
       |             |            |               |
       v             v            v               v
+--------------------------------------------------------------+
|                     AIAgent 核心层                              |
|                                                              |
|  +------------------+  +-----------------+  +-------------+  |
|  | 系统提示词构建    |  | 上下文压缩器     |  | 迭代预算    |  |
|  | prompt_builder   |  | context_compressor| | IterationBudget|
|  +------------------+  +-----------------+  +-------------+  |
|                                                              |
|  +------------------+  +-----------------+  +-------------+  |
|  | 模型元数据       |  | 凭证池           |  | 用量计费    |  |
|  | model_metadata   |  | credential_pool  | | usage_pricing|  |
|  +------------------+  +-----------------+  +-------------+  |
+------+-------------------------------------------------------+
       |
       v
+--------------------------------------------------------------+
|                      工具编排层                                |
|  +------------------+  +-----------------+                   |
|  | model_tools.py   |  | toolsets.py     |                   |
|  | (分发与桥接)      |  | (工具集组合)     |                   |
|  +--------+---------+  +-----------------+                   |
|           |                                                  |
+--------------------------------------------------------------+
       |
       v
+--------------------------------------------------------------+
|                      工具执行层                                |
|  +--------+ +--------+ +------+ +-----+ +--------+ +------+ |
|  | 终端   | | 浏览器  | | 文件  | | MCP | | 委托   | | 搜索  | |
|  | 6后端  | | 自动化  | | 操作  | | 桥接 | | 子Agent| | 网页  | |
|  +--------+ +--------+ +------+ +-----+ +--------+ +------+ |
+--------------------------------------------------------------+
       |
       v
+--------------------------------------------------------------+
|                      持久化层                                  |
|  +------------------+  +-----------------+  +-------------+  |
|  | SQLite state.db  |  | MEMORY.md       |  | config.yaml |  |
|  | (会话/消息/FTS5) |  | USER.md         |  | .env        |  |
|  +------------------+  +-----------------+  +-------------+  |
+--------------------------------------------------------------+
```

---

## 核心组件关系

| 组件 | 职责 | 关键文件 | 大小 |
|------|------|----------|------|
| AIAgent | 会话循环、工具调用、上下文管理 | `run_agent.py` | 463 KB |
| CLI/TUI | 交互式终端界面 | `cli.py` | 376 KB |
| 工具注册表 | 工具发现与注册 | `tools/registry.py` | 12 KB |
| 工具编排 | 工具分发与异步桥接 | `model_tools.py` | 22 KB |
| 网关主控 | 多平台生命周期管理 | `gateway/run.py` | 341 KB |
| 会话存储 | SQLite 持久化 + FTS5 | `hermes_state.py` | 51 KB |
| 记忆管理 | 内置 + 插件记忆编排 | `agent/memory_manager.py` | 14 KB |
| 上下文压缩 | 自动摘要与上下文裁剪 | `agent/context_compressor.py` | 30 KB |
| 提示词构建 | 系统提示词组装 | `agent/prompt_builder.py` | 40 KB |

---

## 部署形态

Hermes Agent 支持多种部署方式：

### 1. 本地 CLI 模式

```
用户 <-> CLI (prompt_toolkit TUI) <-> AIAgent <-> LLM API
```

最常见的使用方式，直接在终端中交互。

### 2. 网关服务模式

```
消息平台 (Telegram/Discord/...) <-> GatewayRunner <-> AIAgent <-> LLM API
```

作为后台服务运行，同时服务多个消息平台的多个用户。

### 3. 编辑器集成模式

```
编辑器 (VS Code/Zed/JetBrains) <-> ACP 协议 <-> ACP Server <-> AIAgent <-> LLM API
```

作为 stdio 进程运行，通过 Agent Client Protocol 与编辑器通信。

### 4. RL 训练模式

```
Atropos 训练器 <-> HermesBaseEnv <-> AIAgent <-> vLLM/训练模型
```

在强化学习循环中运行，生成轨迹用于模型训练。

---

## 设计决策

### 为什么使用 SQLite 而非文件系统？

早期版本使用 JSONL 文件存储会话，但在网关多平台并发场景下出现竞争问题。SQLite 的 WAL 模式支持并发读和单写，FTS5 提供高效的全文检索，同时保持了零依赖部署的优势。

### 为什么工具自注册而非中心配置？

工具自注册（模块导入时调用 `registry.register()`）消除了添加新工具时需要修改多个文件的问题。导入链严格单向，避免循环依赖：

```
tools/registry.py  <-  tools/*.py  <-  model_tools.py  <-  run_agent.py
```

### 为什么限制只有一个外部记忆插件？

多个记忆插件同时运行会导致工具 schema 膨胀（每个插件注册自己的工具），且不同后端可能产生冲突的上下文信息。单插件约束保持了简洁性和可预测性。

### 为什么上下文压缩使用辅助模型？

主模型可能很贵（如 GPT-4、Claude Opus），用一个便宜的辅助模型（如 GPT-4o-mini）来做摘要可以显著降低成本，同时不影响摘要质量。

---

## 相关文档

- [核心 Agent 循环](core-agent-loop.md) -- 深入了解 AIAgent 的工作原理
- [数据流图](data-flow.md) -- 可视化理解系统数据流向
- [模块依赖关系](module-dependency.md) -- 模块间的导入与依赖
