# Hermes Agent 中文技术文档

> Nous Research 开源自进化 AI Agent 框架 -- 架构设计与实现详解

---

## 文档索引

| 编号 | 文档 | 说明 |
|------|------|------|
| 1 | [架构概述](architecture-overview.md) | 系统定位、核心理念、技术栈、目录结构 |
| 2 | [核心 Agent 循环](core-agent-loop.md) | AIAgent 类、chat() 方法、迭代预算、会话管理 |
| 3 | [工具系统](tool-system.md) | 注册机制、调度流程、工具集、各工具介绍 |
| 4 | [网关系统](gateway-system.md) | 多平台路由、平台适配器、会话持久化、消息投递 |
| 5 | [记忆系统](memory-system.md) | MemoryManager、Provider 接口、内置/插件记忆、预取流程 |
| 6 | [上下文压缩](context-compression.md) | 触发条件、压缩策略、结构化摘要、会话链接 |
| 7 | [记忆自我进化](memory-self-evolution.md) | 会话→长期记忆固化、Memory Flush、迭代摘要、行为漂移回路 |
| 8 | [委派系统](delegation-system.md) | delegate_task、子代理构建、并行执行、深度限制与隔离机制 |
| 9 | [多业务工具场景适配指南](business-tools-adaptation.md) | 多业务域工具组织、Toolset/MCP/Delegate 4 种模式、改造步骤 |
| 10 | [子代理与用户交互模式](subagent-user-interaction.md) | 子代理无法直接问用户的 5 种应对方案与推荐组合 |
| 11 | [技能系统](skill-system.md) | 技能发现、YAML 格式、Skills Hub、技能创建流程 |
| 12 | [CLI 系统](cli-system.md) | 命令架构、交互式 TUI、配置系统、认证体系 |
| 13 | [ACP 编辑器集成](acp-integration.md) | 协议实现、会话管理、权限控制 |
| 14 | [定时任务系统](cron-system.md) | 调度器、作业生命周期、投递目标 |
| 15 | [RL 训练环境](rl-training.md) | Gym 环境、基准测试、轨迹生成 |
| 16 | [数据流图](data-flow.md) | 用户消息流、工具执行流、记忆流、压缩流 |
| 17 | [模块依赖关系](module-dependency.md) | 导入图、关键依赖、设计决策 |
| 18 | [记忆体系代码级深度分析](memory-architecture-deep-dive.md) | 短期记忆处理、压缩算法代码流程、短期→长期记忆转化机制 |
| 19 | [clarify 工具设计](clarify-design.md) | agent 向用户提问的原语、回调注入、同步/异步桥接、子代理禁用与降级 |
| 20 | [消息存储与会话持久化](message-persistence.md) | state.db 两层架构、长会话格式、sessions 账本、压缩链与消息归属、FTS5 检索 |
| 21 | [上下文管理架构演进](context-management-evolution.md) | 压缩三阶段流水线、反震荡机制、压缩锁与原地压缩、TTFT 优化、工具输出预算、@引用与上下文分解 |

---

## 阅读建议

### 快速了解

如果你是第一次接触 Hermes Agent，建议按以下顺序阅读：

1. **[架构概述](architecture-overview.md)** -- 建立全局认识
2. **[核心 Agent 循环](core-agent-loop.md)** -- 理解核心执行流程
3. **[数据流图](data-flow.md)** -- 通过图解掌握数据走向

### 按角色阅读

**后端开发者：**
- [核心 Agent 循环](core-agent-loop.md) -> [工具系统](tool-system.md) -> [委派系统](delegation-system.md) -> [消息存储与会话持久化](message-persistence.md) -> [模块依赖关系](module-dependency.md)

**平台集成开发者：**
- [网关系统](gateway-system.md) -> [定时任务系统](cron-system.md) -> [ACP 编辑器集成](acp-integration.md)

**AI/ML 工程师：**
- [记忆系统](memory-system.md) -> [上下文压缩](context-compression.md) -> [记忆自我进化](memory-self-evolution.md) -> [记忆体系深度分析](memory-architecture-deep-dive.md) -> [RL 训练环境](rl-training.md)

**技能/插件作者：**
- [技能系统](skill-system.md) -> [工具系统](tool-system.md) -> [记忆系统](memory-system.md)

**运维/部署：**
- [CLI 系统](cli-system.md) -> [网关系统](gateway-system.md) -> [定时任务系统](cron-system.md)

**业务集成开发者：**
- [工具系统](tool-system.md) -> [委派系统](delegation-system.md) -> [多业务工具场景适配指南](business-tools-adaptation.md) -> [子代理与用户交互模式](subagent-user-interaction.md) -> [clarify 工具设计](clarify-design.md)

---

## 项目概况

| 属性 | 值 |
|------|-----|
| 项目名称 | Hermes Agent |
| 开发组织 | Nous Research |
| 许可证 | MIT |
| 主要语言 | Python 3.10+ |
| 核心框架 | OpenAI API (兼容层) + prompt_toolkit + asyncio |
| 持久化 | SQLite (WAL 模式, FTS5 全文检索) |
| 支持平台 | 19 个消息平台 + 3 个编辑器 (VS Code, Zed, JetBrains) |
| 工具数量 | 40+ 个自注册工具 |
| 技能分类 | 30+ 个类别 |
| 记忆插件 | 8 个 (honcho, hindsight, holographic, mem0, supermemory 等) |

---

## 架构一览

```
                          +------------------+
                          |    用户入口       |
                          +------------------+
                          |  CLI  | Gateway  | ACP
                          +--+----+----+-----+--+
                               |         |        |
                               v         v        v
                          +------------------+
                          |    AIAgent       |
                          | (run_agent.py)   |
                          +--------+---------+
                                   |
                    +--------------+--------------+
                    |              |              |
                    v              v              v
              +---------+   +-----------+   +----------+
              | 工具系统 |   | 记忆系统   |   | 上下文    |
              | tools/  |   | memory    |   | 压缩器   |
              +---------+   +-----------+   +----------+
                    |
         +----+----+----+----+----+
         |    |    |    |    |    |
         v    v    v    v    v    v
       终端 浏览器 文件 MCP 委托 搜索 ...
```

---

## 贡献

如发现文档错误或希望补充内容，欢迎提交 Pull Request。
文档源文件位于 `docs/zh/` 目录。
