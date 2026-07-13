# 上下文管理架构演进深度分析

> 分析基准：分叉点 `96cd37e2`（2026-06-04）→ main `aaf569126`（2026-07-12），约 38 天，4602 commits。

本文档记录 Hermes Agent 在这段时间里，Agent 上下文管理（context management）子系统的架构演进。这次升级把上下文管理从一个**单一压缩器**重做成了**完整的上下文生命周期子系统**——`context_compressor.py` 单文件增长 +1300 行，`conversation_compression.py` 增长 +791 行，外加两个全新模块。

涉及的核心文件：

| 文件 | 变更规模 | 角色 |
|------|---------|------|
| `agent/context_compressor.py` | +1300 / −134 | 压缩执行器（三阶段流水线） |
| `agent/conversation_compression.py` | +791 / −185 | 压缩编排层（会话生命周期粘合） |
| `agent/context_references.py` | +98（新文件） | `@` 引用内联展开 |
| `agent/context_breakdown.py` | +156（新文件） | 上下文窗口实时分解（UI） |
| `agent/context_engine.py` | +7 / −2 | 上下文引擎抽象基类（接口稳定） |
| `plugins/context_engine/` | 无新增 | 第三方引擎发现框架（已就绪） |

---

## 一、压缩引擎：从"越压越乱"到"三阶段流水线"

这是最核心的进步。旧的压缩器是"摘要中间轮次"的简单逻辑；新版本是清晰的三阶段处理链。

### 阶段 1：工具输出修剪（零 LLM 调用，省 90%+ 体积）

- **重复工具结果去重**——同一文件被读 5 次，只保留最新完整副本
- **长工具结果替换为信息性摘要**——`[terminal] ran npm test → exit 0, 47 lines output`
- **大型 tool_call JSON 参数安全截断**——避免 provider 400 错误

这一阶段不调用任何模型，纯结构化处理，能削减 90% 以上的工具输出体积。

### 阶段 2：Token 预算保护的尾区

- 旧版按**固定消息数**（`protect_last_n`）保护尾部
- 新版按 **token 预算**滚动保护最近 ~20K token，兼顾多轮工具调用
- 修复了一个严重的低估 bug：旧版只计 `arguments`，新版计入完整 tool_call envelope（id + type + function.name + JSON 结构）——一个 4-tool-call 轮次从低估的 ~73 token 修正为真实的 ~1090 token（issue #28053）
- 同时计入 `codex_reasoning_items` 等 provider replay 字段（issue #55572）

### 阶段 3：LLM 摘要（迭代增量更新）

- 后续压缩**不再从零摘要**，而是以前一次摘要 + 新中间轮次做增量更新，保留历史累积信息
- 支持辅助模型（auxiliary model）独立执行摘要（降低主模型成本），失败时回退主模型
- 摘要预算按被压缩内容比例缩放（20% ratio，上限 10K token）

### 反压缩震荡（Anti-Thrash）— 修复 #60989

这是个真实痛点：**sub-512K 上下文模型把大部分时间花在反复重新总结上**。报告场景：会话每 1-2 回合就重新压缩一次，因为 50% 触发阈值留下的 post-compaction headroom 被不可压缩的地板（system prompt、工具 schema、受保护尾区、滚动摘要）吃光了。

三个叠加缺陷被修复：

| 缺陷 | 旧版 | 新版 |
|------|------|------|
| 触发阈值 | 50%（headroom 被不可压缩地板吃光，每 1-2 回合重压） | 512K 以下模型 **≥75%** 才触发（raise-only——更高配置值或 per-model autoraise 如 Codex gpt-5.5 的 85% 永远胜出） |
| 摘要 max_tokens | wire cap 截断摘要（Anthropic/NVIDIA 上 thinking 模型把 cap 烧在推理上，产出截断或纯 thinking 摘要 → 压缩循环） | **不设 max_tokens**，摘要预算仅作 prompt guidance（"Target ~N tokens"） |
| 推理轨迹 | thinking 模型的 `<think>`/`<reasoning>` 被存入 `_previous_summary` 反复膨胀 | **端到端剥离**——序列化给总结器前 + 总结器输出前双重过滤 |

外加完整的 **anti-thrash verdict 机制**（一系列 `fix(compaction)` commit）：

- `_ineffective_compression_count`：连续两次压缩节省不足 10% 则触发退避
- 基于**真实 provider 报告 token 数**判断有效性，而非基于消息列表长度的估算
- 摘要 LLM 失败的 cooldown 期间不重复触发
- 成功边界后 arm verdict，stale verdict 会被清除

### 头部保护衰减

- `protect_first_n` 在**第一次压缩后衰减为 0**（issue #11996）
- 旧版前 N 条消息永远保留，导致长会话中早期用户消息"永生化"，头部无限膨胀
- 新版首次压缩后头部只保留 system prompt，之前的历史已由摘要承接

### 压缩边界对齐

新增 `_align_boundary_forward()` / `_align_boundary_backward()` 防止在 tool_call/tool_result 配对中间切开——否则会导致 provider API 拒绝 orphan tool call。

---

## 二、可靠性：并发安全 + 失败分级 + 流式断路

### 压缩锁（防孤儿 session）

全新 SQLite-backed 机制。问题场景：多个 AIAgent 实例共享同一 session_id（如 parent agent 与 background-review fork），可能同时触发压缩，各自旋转到不同 child session，产生"孤儿 session"。

新增接口：

- `try_acquire_compression_lock()` / `release_compression_lock()` / `refresh_compression_lock()`
- `_CompressionLockLeaseRefresher`：后台线程每 TTL/2 秒自动续期，断连后最多容忍 TTL 时长

### 原地压缩（消除 session 旋转副作用）

- 旧版压缩**旋转 session_id**（旧 session 结束，新 child session 继承），导致 URL、标题、`/goal` 全部需要迁移，日志 session id 漂移
- 新版（config `compression.in_place`）：相同 session_id 下软归档（`active=0`，磁盘仍可搜索）被压缩轮次，直接插入压缩后内容为新的 active 集合
- 消除了"session 旋转"的所有副作用（issue #38763）

### 孤儿 session 防护

- 压缩创建 child session 失败时（旧版静默继续，导致 phantom session），新版**回滚到 parent session id**（issue #33906/#33907）
- 保留 `/goal` 迁移（issue #33618）
- 标题自动继承编号（issue #34089）

### 外部组件通知

压缩边界触发对相关组件的通知：

- `context_compressor.on_session_start()`（插件 context engine 用 `boundary_reason="compression"` 保留 DAG lineage）
- `memory_manager.on_session_switch()` 通知内存提供者刷新缓冲区
- `event_callback("session:compress", {...})` 供钩子（如 MemPalace sync）提取即将消失的旧 session 信息

### 压缩质量警告

连续压缩 ≥2 次时通过 `status_callback` 向所有平台（TUI/Telegram/Discord）发送警告："Session compressed N times — accuracy may degrade"。

### 摘要失败分级处理

旧版失败就静默丢弃中间窗口；新版分级：

| 失败类型 | 处理策略 |
|---------|---------|
| 认证/权限错误（401/403） | **中止压缩**（preserve session unchanged） |
| 网络/流中断错误 | **中止压缩**，cooldown 30s 后重试 |
| 临时错误（超时、429、JSON 解析失败） | 回退到主模型一次，cooldown 60s |
| 辅助模型不可用 | 回退到主模型继续 |
| 所有方式均失败 | 插入**确定性后备摘要**（从被压缩内容中提取文件路径、工具调用、用户提问等结构化锚点） |

### 辅助模型可行性检查（Lazy 化）

- 旧版在 `AIAgent.__init__` 同步执行，~400ms 冷启动开销
- 新版 `check_compression_model_feasibility()` 延迟到**第一次实际压缩尝试时**才执行
- 支持自动调低 session threshold（当辅助模型 context window 小于主模型阈值时）

### 图片压缩恢复

新增 `try_shrink_image_parts_in_messages()`：当 provider 因图片过大（Anthropic 5MB）拒绝时，将 `data:image/...;base64,...` 部分重新编码为更小尺寸后重试。

### 流式 stale 跨回合断路器（#58962）

真实故障：一个 session 对无响应 provider 卡住，**连续 494 次失败跨 3+ 天**无限循环——每回合都触发 stale-stream 检测器，烧满 180s × retries，且从不通知用户。

新版加 per-session 连续 stale-stream 计数器：

- 在 outer poll loop 中每次 stale-stream kill 时递增
- 仅在 stream 真正完成时重置为 0
- 达到 `HERMES_STREAM_STALE_GIVEUP`（默认 5）时，下一回合立即 abort 并报清晰、可操作的 `RuntimeError`

这与已有的单流 stale 工作（local-provider hard ceiling #44938、backoff/parse-error #60031）互补：那些约束单个卡住的 stream，这个约束跨回合的重复 staleness。

---

## 三、性能：TTFT 降低 80% + 工具输出预算

### TTFT 从 4.3s 降到 0.9s（#59332，全平台）

cProfile 定位了首字延迟（submit → 第一个 streamed token）前的四个阻塞，全部消除：

| 阻塞 | 耗时 | 修复 |
|------|------|------|
| Discord 能力检测（每个有 DISCORD_BOT_TOKEN 的冷进程都阻塞 HTTPS 调用 `get_tool_definitions` → `_get_dynamic_schema`） | ~2.0s（最坏 5s） | 非阻塞：内存缓存 → 24h 磁盘缓存 → permissive 默认 + 后台检测。permissive 默认 per-process 固定，工具 schema 不会对话中途翻转（prompt-cache 安全） |
| Ollama `/api/show` 探测（KNOWN provider 仍 POST 且不缓存 miss，每次全 HTTP 往返） | ~0.3s | 已知非 Ollama provider 跳过探测；local/custom/unknown 端点保持原行为 |
| env_probe 子进程扫描（首轮系统提示构建时跑 4-8 个子进程） | ~0.5s | 移到 agent init 线程外预热；提示构建命中缓存（同锁，飞行中预热 join 而非重算） |
| `tools.mcp_tool` 导入（零 MCP 时 between-turns 刷新仍导入整个包） | ~0.4s | 按 `sys.modules` 成员门控（MCP 工具只在已导入时存在） |

测量结果（cold first turn，submit → request dispatched，openrouter，discord token set）：**4.3s → 0.9s（~80%）**；agent 侧非导入成本从 2.9s 降到 0.36s（init）+ 0.27s（turn prologue）。CLI 还在 idle banner 窗口线程外预导入 `run_agent` + `openai`，隐藏剩余 ~1.5s 模块导入。

修复 1-4 适用于每个交互层（CLI、gateway、TUI、desktop、cron）。

### 工具输出预算按窗口缩放（#23767）

旧版固定 100K chars/结果、200K chars/回合——一个 65K token 本地模型会被单个 279K char 搜索结果撑爆（provider 拒绝 "Prompt too long"）。

- `budget_config.budget_for_context_window()` 按模型窗口分数缩放 per-result/per-turn char cap，clamp 到历史 100K/200K 默认（大模型不变），地板保证小模型可用
- `resolve_threshold()` 现在 cap per-tool registry 值在 `default_result_size`，防止注册固定 100K cap 的工具（web/terminal/x_search）重新膨胀已缩放的预算
- `tool_executor` 把 agent 的 live context_length（model switch 时重算）接入全部四个 persist/turn-budget 调用点

E2E 验证：279K-char 结果在 65K 模型上**自动折叠为 ~1.6K 预览**；200K 模型字节一致。`read_file` 保持 inf-pinned（无 persist 循环）。

### compact_rows 性能优化

`perf(state)`：`compact_rows` 让 session list 查询跳过 `system_prompt` blob——长会话的 system_prompt 可能很大，列表查询不需要它。

### prompt caching 加固

- `_can_carry_marker` 对齐修复——list 最后元素非 dict 时通过 carrier gate 却没收到 marker，**浪费四个 breakpoint 之一**
- 空 assistant/tool 消息跳过无效 `cache_control`（OpenRouter：role:tool 不再设 top-level cache_control，空 assistant turn 跳过，非空 tool content 包装使 marker 落在 content part）
- Kimi/Moonshot 纳入 OpenRouter prompt cache 策略（#25970）
- MoA aggregator 的 one-shot synthesis call 应用 prompt-caching decoration

---

## 四、新能力：用户可感知的输入与可视化

### `@` 引用内联展开（`context_references.py`，全新）

用户可在消息里写引用标记，系统自动展开内容注入当前轮次：

| 引用类型 | 行为 |
|---------|------|
| `@file:path/to/file` | 读取文件内容，支持行号范围语法 `@file:foo.py:10-20` |
| `@folder:path` | 列出目录结构（200 条目上限） |
| `@diff` / `@staged` | 执行 `git diff` / `git diff --staged` 并内联结果 |
| `@git:N` | 执行 `git log -N -p` 并内联 |
| `@url:https://...` | 通过 `web_extract` 抓取网页内容转为 markdown |

**安全设计：**

- 引用展开**受限工作目录**（cwd 或 `allowed_root`），不可逃逸到 `~/.hermes/`、`~/.ssh/`、`.aws`、`.gnupg`、`.env` 等敏感路径
- 接入 `agent.file_safety.get_read_block_error()` 的凭据文件拒绝列表（authorized_keys、id_rsa、.bashrc、.netrc 等）
- token 注入上限：硬上限 50% context window，软上限 25%，超限拒绝并警告

**异步并行：** `asyncio.gather()` 并行展开所有引用，不串行等待。

### 上下文窗口实时分解（`context_breakdown.py`，全新）

为 UI（TUI、Desktop、Gateway status）提供下一次 API 请求的 token 构成实时拆解——不再只显示总量，而是分段着色显示各部分占用量。

`compute_session_context_breakdown(agent, messages)` 返回的分类：

```
system_prompt / tool_definitions / rules / skills / mcp
subagent_definitions / memory / conversation
```

外加 `context_max`、`context_percent`、`context_used`、`estimated_total`、`model`。使用与 `agent.model_metadata.estimate_request_tokens_rough` 一致的 char/4 启发式，确保数字与压缩阈值对齐。TUI 和 Desktop 的 `/usage` 浮窗就是用它。

### 长上下文模型适配

**gpt-5.6（#63249 系列）：** Codex OAuth 路径硬限 272K context，但默认 50% 压缩触发会在 ~136K 就总结，浪费一半可用窗口。`_compression_threshold_for_model` 的 `codex_gpt55_autoraise` chokepoint 扩展匹配 gpt-5.6*，使这些 session 获得 0.85 auto-raise。Direct-API/OpenRouter 路径（完整 1.05M 窗口）不受影响。

**Codex app-server：** 当 `api_mode == "codex_app_server"` 时，压缩委托给 Codex 自有的 `thread.compact()` 接口（issue #36801），而非重写本地 OpenAI 格式 transcript——关键决策是**保持 provider-portable**：encrypted compaction items 会把会话历史锁死到 chatgpt.com，破坏 `/model` 切换和 provider fallback，所以 Responses-API native compaction 路径被移除，只保留 app-server 的 `compact_thread()` 路径。

### Codex 原生压缩路由

重构后（`refactor(compression): scope Codex-native compaction to the app-server runtime`）：

- **移除：** `responses.compact()` 调用路径、`codex_compaction_items` replay/persistence plumbing、`codex_native_compaction` + `codex_responses_threshold` config keys、desktop settings 字段及其测试
- **保留：** 全部 app-server 相关（`compact_thread()`、压缩通知、簿记、docs、tests）+ 存活 knob 的 cache-busting keys
- 控制项：`compression.codex_app_server_auto`（native | hermes | off），无 umbrella flag

---

## 五、架构演进总览

### 子系统结构

```
96cd37e2 时期（单一压缩器）              HEAD（完整上下文子系统）

run_agent                                run_agent / conversation_compression（编排层）
  └─ context_compressor                      ├─ context_compressor（三阶段：修剪→预算→摘要）
      └─ 简单中间摘要                         ├─ context_references（@引用，输入侧扩展）「新」
                                             ├─ context_breakdown（实时 UI 监控）「新」
                                             ├─ 压缩锁 + 反震荡 + 失败分级
                                             └─ plugins/context_engine（引擎可插拔架构）
```

### 维度对比

| 维度 | 旧架构 | 新架构 |
|------|--------|--------|
| 压缩触发 | 固定 50% 阈值 | 动态阈值 + 小窗口地板 75% + max_tokens 输出预留 |
| 尾部保护 | 固定消息数 | Token 预算（含完整 tool_call envelope + replay 字段） |
| Session 管理 | 每次压缩旋转 session_id | 支持原地压缩（同 id 软归档） |
| 并发安全 | 无 | SQLite-backed 压缩锁 + TTL 续期线程 |
| 摘要失败 | 静默丢弃 | 分级处理（abort / cooldown / 回退 / 后备） |
| 工具输出 | 保留原始内容 | 信息性摘要（减少 90%+ 体积） |
| 图片处理 | 无 | 压缩时剥离旧截图 base64，resize 恢复 |
| 边界完整性 | 无 | tool_call/tool_result 配对保护 + 边界对齐 |
| 流式卡死 | 无限循环 | 跨回合断路器（5 次 abort） |
| 首字延迟 | 4.3s | 0.9s（−80%） |
| 输入扩展 | 无 | `@file` / `@folder` / `@git` / `@url` / `@diff` 引用 |
| UI 可见性 | 总量 | 分段着色实时分解 |

### 一句话总结

这次升级让上下文管理从"能用但会卡死、会越压越乱"变成了"自适应、可观测、并发安全、失败优雅降级"——尤其对小上下文模型和长会话场景，体验是质变。

---

## 关键 issue / commit 溯源

| 编号 | 内容 |
|------|------|
| #60989 | 压缩反震荡（75% 地板 + 摘要不设 max_tokens + 推理轨迹剥离） |
| #59332 | TTFT 降低 80%（四个请求前阻塞消除） |
| #58962 | 流式 stale 跨回合断路器 |
| #23767 | 工具输出预算按窗口缩放 |
| #38763 | 原地压缩消除 session 旋转副作用 |
| #28053 | tool_call envelope token 低估修正 |
| #36801 | Codex app-server `thread.compact()` 委托 |
| #11996 | 头部保护首次压缩后衰减 |
| #33906/#33907 | 孤儿 session 回滚 |
| #33618 | `/goal` 迁移保留 |
| #25970 | Kimi/Moonshot prompt cache 策略 |
