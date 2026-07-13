# 模块依赖关系

> 导入图、关键依赖说明、核心设计决策

---

## 目录

- [概述](#概述)
- [模块导入图](#模块导入图)
- [核心模块依赖](#核心模块依赖)
- [依赖层次](#依赖层次)
- [关键设计决策](#关键设计决策)
- [循环依赖规避](#循环依赖规避)
- [外部依赖](#外部依赖)

---

## 概述

Hermes Agent 的模块依赖遵循清晰的分层原则：**基础设施层 -> 核心层 -> 应用层 -> 入口层**。禁止跨层反向依赖，循环依赖通过延迟导入和注册表模式解决。

---

## 模块导入图

### 自上而下视图

```
入口层 (Entry Points)
+----------------------------------------------------------+
|  hermes_cli/main.py --> cli.py --> run_agent.py          |
|  acp_adapter/entry.py --> acp_adapter/server.py          |
|  gateway/run.py --> cron/scheduler.py                     |
|  environments/hermes_base_env.py                          |
|  batch_runner.py / trajectory_compressor.py               |
+----------------------------------------------------------+
                          | 依赖
应用层 (Application)
+----------------------------------------------------------+
|  run_agent.py (AIAgent)                                   |
|  model_tools.py (工具发现与编排)                            |
|  hermes_cli/config.py (配置管理)                           |
|  hermes_cli/auth.py (认证体系)                             |
|  acp_adapter/session.py (ACP 会话)                         |
|  cron/jobs.py (作业存储)                                   |
+----------------------------------------------------------+
                          | 依赖
核心层 (Core)
+----------------------------------------------------------+
|  agent/                                                   |
|    +-- memory_manager.py (记忆编排)                        |
|    +-- memory_provider.py (ABC)                           |
|    +-- model_metadata.py (模型元数据)                      |
|    +-- context_compressor.py (上下文压缩)                  |
|    +-- smart_model_routing.py (智能路由)                   |
|  tools/                                                   |
|    +-- registry.py (工具注册表)                            |
|    +-- *_tool.py (各工具实现)                              |
|  plugins/memory/ (记忆插件)                                |
|  gateway/platforms/ (平台适配器)                            |
+----------------------------------------------------------+
                          | 依赖
基础设施层 (Infrastructure)
+----------------------------------------------------------+
|  hermes_constants.py (全局常量、路径)                       |
|  hermes_time.py (时区感知时间)                              |
|  hermes_state.py (SessionDB SQLite)                       |
|  hermes_logging.py (日志配置)                              |
+----------------------------------------------------------+
```

### 详细导入关系

```
hermes_cli/main.py
  +-- hermes_cli.config
  +-- hermes_cli.env_loader
  +-- hermes_cli.auth
  +-- hermes_constants
  +-- hermes_logging
  +-- hermes_state
  +-- cli (cli.py)
  +-- plugins.memory
  +-- tools.*
  +-- agent.*
  +-- acp_adapter (跳转)

cli.py
  +-- run_agent (AIAgent)
  +-- model_tools
  +-- hermes_cli.config
  +-- hermes_constants
  +-- hermes_state
  +-- hermes_logging
  +-- agent.*
  +-- tools.*
  +-- cron (定时任务工具)
  +-- gateway (网关状态查询)

run_agent.py (AIAgent)
  +-- model_tools
  +-- agent.memory_manager
  +-- agent.model_metadata
  +-- agent.context_compressor
  +-- hermes_cli.config
  +-- hermes_constants
  +-- hermes_time
  +-- hermes_logging
  +-- plugins.memory
  +-- tools.registry

acp_adapter/server.py
  +-- acp_adapter.auth
  +-- acp_adapter.events
  +-- acp_adapter.permissions
  +-- acp_adapter.session
  +-- hermes_cli (版本)
  +-- model_tools
  +-- agent.*
  +-- tools.*

acp_adapter/session.py
  +-- run_agent (AIAgent)
  +-- hermes_cli.config
  +-- hermes_cli.runtime_provider
  +-- hermes_constants
  +-- hermes_state (SessionDB)
  +-- tools.terminal_tool

cron/scheduler.py
  +-- cron.jobs
  +-- run_agent (AIAgent)
  +-- hermes_cli.config
  +-- hermes_cli.runtime_provider
  +-- hermes_constants
  +-- hermes_state (SessionDB)
  +-- hermes_time
  +-- agent.smart_model_routing
  +-- tools.send_message_tool
  +-- gateway.config

cron/jobs.py
  +-- hermes_constants
  +-- hermes_time
  +-- croniter (可选)

gateway/run.py
  +-- run_agent (AIAgent)
  +-- gateway.config
  +-- gateway.platforms.*
  +-- cron.scheduler
  +-- hermes_cli.config
  +-- hermes_constants
  +-- hermes_state
  +-- hermes_logging
  +-- agent.*
  +-- tools.*

model_tools.py
  +-- tools.registry
  +-- tools.*_tool (动态发现)
  +-- hermes_cli.config
```

---

## 核心模块依赖

### run_agent.py -- 核心消费者

`run_agent.py` 中的 `AIAgent` 类是系统的核心，消费几乎所有核心层模块：

| 依赖 | 用途 |
|------|------|
| `model_tools` | 获取工具定义、调度工具执行 |
| `agent.memory_manager` | 记忆预取、注入、同步 |
| `agent.model_metadata` | 模型上下文窗口、token 估算 |
| `agent.context_compressor` | 上下文压缩策略 |
| `hermes_cli.config` | 读取用户配置 |
| `plugins.memory` | 加载记忆插件 |
| `tools.registry` | 工具注册表（tool_error/tool_result 辅助函数） |

### tools/registry.py -- 零依赖注册表

工具注册表是唯一不导入任何工具文件的模块：

```python
# registry.py 只定义:
# - register_tool()  注册工具 schema 和 handler
# - get_tools()      获取注册的工具列表
# - dispatch()       分发工具调用

# 工具文件在导入时自动注册:
# tools/terminal_tool.py  -> import 时调用 register_tool()
# tools/read_file_tool.py -> import 时调用 register_tool()
```

### hermes_constants.py -- 最底层

纯常量模块，无项目内依赖：

```python
# 提供:
# - get_hermes_home() -> Path  (HERMES_HOME 路径)
# - OPENROUTER_BASE_URL
# - parse_reasoning_effort()
# - 其他全局常量
```

---

## 依赖层次

### 分层规则

| 层 | 可依赖的层 | 示例 |
|----|-----------|------|
| 基础设施层 | 无 (或仅标准库) | hermes_constants, hermes_time |
| 核心层 | 基础设施层 | agent/, tools/, plugins/ |
| 应用层 | 核心层 + 基础设施层 | run_agent, model_tools, hermes_cli/ |
| 入口层 | 全部 | main.py, cli.py, gateway/run.py |

### 跨层依赖特例

少数模块需要跨层访问，通过延迟导入解决：

```python
# cron/scheduler.py 需要 run_agent.py (应用层 -> 应用层)
# 在函数内部导入:
def run_job(job):
    from run_agent import AIAgent  # 延迟导入避免启动时循环

# acp_adapter/session.py 同理:
def _make_agent(self, ...):
    from run_agent import AIAgent
    from hermes_cli.config import load_config
```

---

## 关键设计决策

| 决策 | 原因 | 实现 |
|------|------|------|
| 工具自注册 | 添加工具无需修改中心文件 | `tools/*.py` 导入时调用 `register_tool()` |
| 记忆插件化 | 支持 8+ 种记忆后端 | `MemoryProvider` ABC + `plugins/memory/` |
| 配置优先于代码 | 用户可调整而不需修改代码 | config.yaml + .env + 命令行参数 |
| Profile 隔离 | 多实例互不干扰 | HERMES_HOME 环境变量 + Profile 系统 |
| SessionDB 统一 | CLI/Gateway/ACP 共享会话 | SQLite WAL + 线程安全 + source 标记 |
| 延迟导入 | 避免循环依赖和启动性能 | 函数内 `from X import Y` |
| stdio 保护 | ACP JSON-RPC 传输安全 | _print_fn 重定向到 stderr |
| 原子写入 | 防止作业文件损坏 | mkstemp -> write -> fsync -> replace |
| 文件锁 | 防止并发 tick 重复执行 | fcntl.flock / msvcrt.locking |
| at-most-once | 崩溃不重复执行定时任务 | advance_next_run() 预推进 |

---

## 循环依赖规避

### 问题场景

```
# 潜在循环:
# run_agent.py -> model_tools.py -> tools/*.py -> run_agent.py (委托工具)
#
# 解决: tools/*.py 在函数内部延迟导入 run_agent
```

### 常用策略

1. **延迟导入 (Lazy Import)**
   ```python
   def handler(args):
       from run_agent import AIAgent  # 仅在执行时导入
   ```

2. **注册表模式 (Registry Pattern)**
   ```python
   # registry.py 不导入任何工具文件
   # 工具文件导入 registry.py 并注册自己
   ```

3. **ABC 接口 (Abstract Base Class)**
   ```python
   # memory_provider.py 定义 MemoryProvider ABC
   # 具体插件导入并实现 ABC
   # memory_manager.py 只依赖 ABC
   ```

4. **TYPE_CHECKING 守卫**
   ```python
   from typing import TYPE_CHECKING
   if TYPE_CHECKING:
       from honcho import Honcho  # 仅类型检查时导入
   ```

---

## 外部依赖

### 核心依赖

| 包 | 用途 | 必需 |
|----|------|------|
| `openai` | LLM API 客户端 | 是 |
| `prompt_toolkit` | 交互式 TUI | 是 (CLI) |
| `yaml` (PyYAML) | 配置文件解析 | 是 |
| `httpx` | HTTP 客户端 | 是 |
| `python-dotenv` | .env 文件加载 | 是 |
| `rich` | 富文本终端输出 | 是 |

### 可选依赖

| 包 | 用途 | 必需 |
|----|------|------|
| `croniter` | Cron 表达式解析 | 否 (cron 系统) |
| `acp` | ACP 协议库 | 否 (编辑器集成) |
| `atroposlib` | RL 训练框架 | 否 (RL 环境) |
| `numpy` | HRR 向量运算 | 否 (holographic 记忆) |
| `honcho` | Honcho SDK | 否 (honcho 记忆插件) |
| `playwright` | 浏览器自动化 | 否 (浏览器工具) |
| `fire` | CLI 参数解析 | 否 (batch_runner) |

### 降级策略

所有可选依赖都有优雅降级：

```python
# croniter 不可用 -> cron 表达式不可用，但间隔/一次性仍工作
try:
    from croniter import croniter
    HAS_CRONITER = True
except ImportError:
    HAS_CRONITER = False

# numpy 不可用 -> HRR 权重归零，使用 FTS+Jaccard
if not hrr._HAS_NUMPY:
    fts_weight = 0.6
    jaccard_weight = 0.4
    hrr_weight = 0.0
```
