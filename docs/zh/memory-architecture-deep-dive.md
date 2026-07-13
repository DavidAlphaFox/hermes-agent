# 记忆体系代码级深度分析

> 短期记忆处理、上下文压缩算法、短期→长期记忆转化的完整代码流程分析

---

## 目录

- [整体架构概览](#整体架构概览)
- [短期记忆：messages 的生命周期](#短期记忆messages-的生命周期)
  - [数据结构](#数据结构)
  - [轮次流程](#轮次流程)
  - [SQLite 持久化](#sqlite-持久化)
- [短期记忆压缩：ContextCompressor 核心算法](#短期记忆压缩contextcompressor-核心算法)
  - [触发时机](#触发时机)
  - [压缩算法五步走](#压缩算法五步走)
  - [结构化摘要模板](#结构化摘要模板)
  - [摘要前缀注入](#摘要前缀注入)
  - [会话分裂](#会话分裂)
- [从短期到长期：学习机制](#从短期到长期学习机制)
  - [内置文件记忆](#内置文件记忆)
  - [后台审查线程](#后台审查线程)
  - [外部记忆提供者](#外部记忆提供者)
  - [技能策展人](#技能策展人)
- [完整数据流总结](#完整数据流总结)
- [关键设计巧妙之处](#关键设计巧妙之处)
- [相关文档](#相关文档)

---

## 整体架构概览

Hermes Agent 的记忆系统分为三个层次，类比人类认知：

```
┌─────────────────────────────────────────────────────────┐
│                    长期记忆 (Long-term)                   │
│  MEMORY.md / USER.md + 外部记忆插件 (Honcho/Hindsight等)  │
│  Skill 技能库 (SKILL.md + references/)                    │
│  跨会话持久化，文件/数据库存储                               │
├─────────────────────────────────────────────────────────┤
│                    短期记忆 (Short-term)                   │
│  messages[] 对话历史列表                                    │
│  SQLite session_db (hermes_state.py)                     │
│  单会话内有效，上下文窗口承载                                 │
├─────────────────────────────────────────────────────────┤
│                    工作记忆 (Working)                      │
│  当前轮次的 system prompt + 用户消息 + 工具调用              │
│  prompt_caching 缓存的前缀                                │
│  单轮有效                                                 │
└─────────────────────────────────────────────────────────┘
```

**关键源文件**：

| 文件 | 职责 |
|------|------|
| `agent/context_engine.py` | 上下文引擎抽象基类 |
| `agent/context_compressor.py` | 默认压缩引擎实现（~1700 行） |
| `agent/conversation_compression.py` | 压缩编排（会话分裂、插件通知） |
| `agent/memory_provider.py` | 记忆提供者抽象基类 |
| `agent/memory_manager.py` | 记忆管理器（编排层） |
| `agent/background_review.py` | 后台学习线程 |
| `agent/conversation_loop.py` | 主对话循环（~3900 行） |
| `tools/memory_tool.py` | 内置记忆工具（MEMORY.md/USER.md） |
| `hermes_state.py` | SQLite 会话持久化 |
| `agent/prompt_caching.py` | Prompt cache 标记 |
| `agent/curator.py` | 技能生命周期管理 |

---

## 短期记忆：messages 的生命周期

### 数据结构

短期记忆就是 **OpenAI 格式的消息列表** `messages: List[Dict]`，在 `conversation_loop.py` 中贯穿整个对话。每条消息的结构：

```python
{
    "role": "user" | "assistant" | "tool" | "system",
    "content": "...",              # 文本或多模态内容列表
    "tool_calls": [...],           # assistant 消息中的工具调用
    "tool_call_id": "..."          # tool 消息中的关联 ID
}
```

### 轮次流程

每个用户消息触发一个完整的轮次循环。以下标注了 `conversation_loop.py` 中的关键行号：

```
用户输入
  │
  ├─ 1. on_turn_start()     通知记忆提供者新轮次开始     (L630-638)
  │     memory_manager.on_turn_start(turn_count, message)
  │
  ├─ 2. prefetch_all()      从外部记忆预取相关上下文     (L646-651)
  │     返回 <memory-context>...</memory-context> 注入到消息中
  │
  ├─ 3. 构建 API 请求
  │     ├─ system prompt（含 MEMORY.md/USER.md 冻结快照）
  │     ├─ messages[]（历史 + 记忆上下文 + 新用户消息）
  │     └─ prompt caching 标记（system + 最近 3 条非系统消息）
  │
  ├─ 4. 调用 LLM            流式响应
  │
  ├─ 5. 处理工具调用         工具结果追加到 messages[]
  │
  ├─ 6. Token 检查           should_compress() 判断是否需要压缩
  │     if prompt_tokens > threshold_tokens → 触发压缩  (L3586-3596)
  │
  └─ 7. 继续循环直到无工具调用 → 返回最终响应
  
轮次结束
  │
  ├─ 8. sync_all()           将 (user, assistant) 轮次同步到外部记忆 (L4253-4257)
  │     memory_manager.sync_all(user_content, assistant_content)
  │
  └─ 9. 后台审查             派生 background_review 线程学习   (L4261-4269)
        _spawn_background_review(messages_snapshot, review_memory, review_skills)
```

### SQLite 持久化

每轮的 `messages[]` 增量写入 SQLite（`hermes_state.py`，WAL 模式）：

| 表 | 内容 |
|---|------|
| `sessions` | session_id、platform、model、parent_session_id（压缩分裂链） |
| `messages` | 完整消息内容，含 JSON 序列化的 tool_calls |
| FTS5 虚表 | 全文搜索历史消息（`search_messages()`） |

关键特性：
- **Schema 版本 13**，含自动迁移
- **WAL 模式**支持网关场景下的并发读写
- **parent_session_id 链**：压缩时新旧会话通过此字段建立父子关系

---

## 短期记忆压缩：ContextCompressor 核心算法

### 触发时机

代码位于 `agent/context_compressor.py:604-624`：

```python
def should_compress(self, prompt_tokens=None) -> bool:
    tokens = prompt_tokens or self.last_prompt_tokens
    if tokens < self.threshold_tokens:       # 默认: context_length * 0.50
        return False
    # 防抖：连续两次压缩效果 <10% 则跳过，避免无限循环
    if self._ineffective_compression_count >= 2:
        return False
    return True
```

在主循环中的调用点（`conversation_loop.py:3569-3592`）：

```python
# 优先使用 API 返回的真实 prompt_tokens
# 仅使用 prompt_tokens，不含 completion/reasoning tokens
# 思考模型（GLM-5.1、QwQ、DeepSeek R1）会膨胀 completion_tokens，导致过早压缩
if _compressor.last_prompt_tokens > 0:
    _real_tokens = _compressor.last_prompt_tokens
else:
    # 回退：含工具 schema 的粗略估算（50+ 工具可增加 20-30K tokens）
    _real_tokens = estimate_request_tokens_rough(messages, tools=agent.tools)

if agent.compression_enabled and _compressor.should_compress(_real_tokens):
    messages, active_system_prompt = agent._compress_context(...)
```

**关键参数**：

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `threshold_percent` | 0.50 | 上下文使用率达到 50% 时触发 |
| `protect_first_n` | 3 | 保护头部 3 条非系统消息 |
| `protect_last_n` | 20 | 尾部保护的最小消息数 |
| `summary_target_ratio` | 0.20 | 摘要预算占阈值的 20% |
| `_SUMMARY_TOKENS_CEILING` | 12,000 | 摘要 Token 绝对上限 |
| `_MIN_SUMMARY_TOKENS` | 2,000 | 摘要 Token 最低保底 |

### 压缩算法五步走

代码位于 `agent/context_compressor.py` 的 `compress()` 方法：

```
原始消息列表:
[system][user1][asst1][tool1][user2][asst2][tool2]...[userN][asstN]
   ↑                                                        ↑
  头部保护区                                              尾部保护区
```

#### 步骤 1：工具输出预裁剪（`_prune_old_tool_results`，L630-796）

无需 LLM 调用的廉价预处理，三遍扫描：

| 遍次 | 操作 | 示例 |
|------|------|------|
| Pass 1: 去重 | MD5 哈希相同的工具结果只保留最新 | 同一文件读取 5 次 → 保留最后一次 |
| Pass 2: 摘要 | 保护区外的大型工具输出替换为 1 行摘要 | `[terminal] ran 'npm test' -> exit 0, 47 lines` |
| Pass 3: 截断 | 裁剪 >500 字符的 tool_call arguments | write_file 50KB 内容 → 200 字符预览 |

工具摘要生成（`_summarize_tool_result`，L322-441）针对不同工具定制格式：

```python
# terminal: [terminal] ran `npm test` -> exit 0, 47 lines output
# read_file: [read_file] read config.py from line 1 (1,200 chars)
# search_files: [search_files] content search for 'compress' in agent/ -> 12 matches
# delegate_task: [delegate_task] 'Fix auth bug' (3,400 chars result)
```

JSON 参数截断使用结构化方式（`_truncate_tool_call_args_json`，L168-211），解析 JSON 后仅缩短字符串值，保证输出仍是合法 JSON，避免下游提供商 400 错误。

#### 步骤 2：确定保护区边界

```python
# 头部保护（_protect_head_size，L1299-1317）
head = 0
if messages[0].get("role") == "system":
    head = 1                      # system prompt 始终保护
head += self.protect_first_n      # + 3 条非系统消息

# 尾部保护（_find_tail_cut_by_tokens）
# 从后往前累加 Token，直到达到 tail_token_budget
# 确保最近 ~20K tokens 完整保留

# 关键锚定（_ensure_last_user_message_in_tail，L1356-1397）
# 保证最后一条 user 消息在尾部保护区内
# 否则用户最新请求会被压缩进摘要，导致任务丢失
```

#### 步骤 3：摘要生成（`_generate_summary`，L904-1182）

使用辅助模型（可配置，默认用主模型）生成结构化摘要：

- **首次压缩**：从零开始构建完整摘要
- **迭代压缩**：基于 `_previous_summary` 增量更新
  - 指令："PRESERVE all existing information, ADD new completed actions"
  - 将已完成项从 "In Progress" 移到 "Completed Actions"
  - 更新 "Active Task" 为最新未完成请求

摘要内容先经过敏感信息脱敏（`redact_sensitive_text`），输出也再次脱敏，防止 API 密钥泄漏。

**摘要预算计算**（`_compute_summary_budget`，L802-811）：

```python
budget = int(content_tokens * _SUMMARY_RATIO)   # 压缩内容的 20%
budget = max(_MIN_SUMMARY_TOKENS, min(budget, max_summary_tokens))
# max_summary_tokens = min(context_length * 5%, 12000)
```

**辅助模型容错**：当配置的摘要模型失败时，自动回退到主模型重试：

```python
# 模型不存在 (404/503)、超时 (408/429/504)、JSON 解析错误、流断开
# → _fallback_to_main_for_compression() → 立即重试
# 完全无提供者 → 600 秒冷却
# 临时错误 → 30-60 秒冷却
```

#### 步骤 4：消息列表重组

```
压缩后:
[system] + [保护头部 3 条] + [摘要消息] + [保护尾部 ~20K tokens]
```

#### 步骤 5：修复工具调用对完整性（`_sanitize_tool_pairs`，L1229-1287）

压缩可能破坏 tool_call/tool_result 的配对关系：

```python
# 1. 删除孤立的 tool result（对应的 assistant tool_call 被压缩掉了）
orphaned_results = result_call_ids - surviving_call_ids
messages = [m for m in messages if m.tool_call_id not in orphaned_results]

# 2. 为缺失结果的 tool_call 插入占位 stub
missing_results = surviving_call_ids - result_call_ids
# → 插入 {"role": "tool", "content": "[Result from earlier conversation]"}
```

### 结构化摘要模板

摘要使用固定结构（L951-1007），确保关键信息不丢失：

```markdown
## Active Task          ← 最关键：用户最新未完成请求的原文
## Goal                 ← 用户整体目标
## Constraints & Preferences  ← 用户偏好、编码风格、约束
## Completed Actions    ← 编号列表，含工具名、目标、结果
## Active State         ← 工作目录、分支、修改文件、测试状态
## In Progress          ← 压缩触发时正在进行的工作
## Blocked              ← 阻塞项，含精确错误信息
## Key Decisions        ← 技术决策及原因
## Resolved Questions   ← 已回答的问题（避免重复回答）
## Pending User Asks    ← 未回答的用户请求
## Relevant Files       ← 读取/修改/创建的文件
## Remaining Work       ← 剩余工作（作为上下文，非指令）
## Critical Context     ← 关键值、错误信息等不能丢失的细节
```

### 摘要前缀注入

压缩后的摘要被包裹在特殊前缀中（`SUMMARY_PREFIX`，L27-41）：

```
[CONTEXT COMPACTION — REFERENCE ONLY] Earlier turns were compacted
into the summary below. This is a handoff from a previous context
window — treat it as background reference, NOT as active instructions.
Do NOT answer questions or fulfill requests mentioned in this summary;
they were already addressed.
Your current task is identified in the '## Active Task' section —
resume exactly from there.
IMPORTANT: Your persistent memory (MEMORY.md, USER.md) is ALWAYS
authoritative — never deprioritize memory content due to this compaction.
Respond ONLY to the latest user message that appears AFTER this summary.
```

这确保模型不会把历史摘要中的问题当作新指令执行。

### 会话分裂

压缩时同步完成 SQLite 会话分裂（`conversation_compression.py:355-390`）：

```python
# 结束旧会话
agent._session_db.end_session(agent.session_id, "compression")
old_session_id = agent.session_id

# 创建新会话，建立父子链
agent.session_id = f"{datetime.now()...}_{uuid.uuid4().hex[:6]}"
agent._session_db.create_session(
    session_id=agent.session_id,
    parent_session_id=old_session_id,    # ← 链式关系
)

# 传播标题（自动编号）
new_title = agent._session_db.get_next_title_in_lineage(old_title)

# 通知上下文引擎（LCM 等插件可保留 DAG 谱系）
agent.context_compressor.on_session_start(
    agent.session_id, boundary_reason="compression", old_session_id=...
)

# 通知记忆提供者（reset=False，逻辑对话继续，仅 ID 轮换）
agent._memory_manager.on_session_switch(
    agent.session_id, parent_session_id=old_session_id, reset=False
)
```

---

## 从短期到长期：学习机制

### 内置文件记忆

代码位于 `tools/memory_tool.py`。

#### 存储结构

```
~/.hermes/memories/
  ├── MEMORY.md    ← Agent 的知识（环境事实、项目信息、学到的东西）
  └── USER.md      ← 用户画像（偏好、风格、期望）
```

条目以 `§` 分隔符分割，有字符上限（MEMORY: 2200 字符, USER: 1375 字符）。

#### 双态机制：冻结快照 vs 实时状态

```python
class MemoryStore:
    def __init__(self):
        self.memory_entries: List[str] = []        # 实时状态
        self.user_entries: List[str] = []           # 实时状态
        self._system_prompt_snapshot = {"memory": "", "user": ""}  # 冻结快照

    def load_from_disk(self):
        # 1. 从文件加载条目到 memory_entries / user_entries
        # 2. 拍摄冻结快照 → _system_prompt_snapshot
        #    该快照在整个会话期间不变，保证 prefix cache 稳定
        # 3. 对快照内容进行注入扫描（threat_patterns 检测）
        #    任何命中替换为 [BLOCKED: ...]
```

**双态设计的意义**：

- **系统提示注入的是冻结快照**：整个会话 prefix cache 稳定（~75% 输入 token 成本节省）
- **工具调用操作的是实时状态**：`memory(action=add)` 立即写入磁盘
- **下次会话启动时**：重新加载，快照刷新

#### 安全防护

- 写入时扫描注入/渗透模式（`_scan_memory_content` → `threat_patterns.py`）
- 快照构建时脱毒（命中模式 → `[BLOCKED: ...]`）
- 外部漂移检测（`_drift_error`）：文件被外部修改时拒绝覆盖写入

### 后台审查线程

代码位于 `agent/background_review.py`。这是 **短期记忆→长期记忆转化** 的核心路径。

#### 触发条件

在 `conversation_loop.py:4246-4269`：

```python
# 技能审查：每 N 轮工具调用触发一次
if (agent._skill_nudge_interval > 0
        and agent._iters_since_skill >= agent._skill_nudge_interval):
    _should_review_skills = True

# 记忆审查：每 N 轮用户对话触发一次
if (agent._memory_nudge_interval > 0
        and agent._turns_since_memory >= agent._memory_nudge_interval):
    _should_review_memory = True

# 在主响应交付后异步启动（不阻塞用户）
if _should_review_memory or _should_review_skills:
    agent._spawn_background_review(
        messages_snapshot=list(messages),   # 传入当前对话历史快照
        review_memory=_should_review_memory,
        review_skills=_should_review_skills,
    )
```

#### 执行流程

`_run_review_in_thread`（L317-545）的详细流程：

```
主对话完成响应
    │
    └─ 后台守护线程启动
        │
        ├─ 1. 复制 AIAgent
        │     ├─ 继承父级 provider/model/credentials/api_key
        │     ├─ skip_memory=True（不触发外部记忆插件，防止 harness prompt 泄漏）
        │     ├─ 继承父级 _cached_system_prompt（命中 prompt cache，节约 ~26% 成本）
        │     ├─ 继承父级 _memory_store（内置记忆写入仍生效）
        │     └─ suppress_status_output=True（静默运行）
        │
        ├─ 2. 设置工具白名单
        │     set_thread_tool_whitelist({"memory", "skill_manage", ...})
        │     其他工具调用会被拒绝（返回 deny 而非阻塞 input()）
        │
        ├─ 3. 运行审查 prompt
        │     ├─ 记忆审查:
        │     │   "Has the user revealed things about themselves?"
        │     │   "Has the user expressed expectations about behavior?"
        │     │   → memory(action=add, target="user", content="...")
        │     │
        │     ├─ 技能审查:
        │     │   "User corrected your style? → 更新相关 skill"
        │     │   "Non-trivial technique emerged? → 捕获到技能"
        │     │   "A loaded skill turned out wrong? → 立即修补"
        │     │   → skill_manage(action=create/update/write_file)
        │     │
        │     └─ 组合审查: 同时覆盖记忆 + 技能
        │
        ├─ 4. 收集成功操作，输出简短摘要
        │     "💾 Self-improvement review: Memory updated · Skill 'X' updated"
        │
        └─ 5. 清理
              关闭 review agent，清除线程工具白名单，清除审批回调
```

#### 学习信号分类

审查提示词（`_COMBINED_REVIEW_PROMPT`）明确定义了学习策略：

**应该学习的信号**：

| 信号类型 | 存储目标 | 示例 |
|---------|---------|------|
| 用户身份/偏好 | USER.md（记忆） | "用户是数据科学家" |
| 行为纠正 | SKILL.md（技能） | "不要用 mock 数据库" |
| 工作流修正 | SKILL.md（技能） | "先运行测试再提交" |
| 新技术/方法 | SKILL.md（技能） | "使用 uv 替代 pip" |
| 风格/格式偏好 | SKILL.md + USER.md | "不要加 emoji" |

**不应学习的内容**（避免形成自我限制性约束）：

- 环境依赖失败（"command not found"、未安装的包）
- 对工具/功能的否定断言（"browser tools don't work"）
- 已自行恢复的临时错误
- 一次性任务叙述

**技能更新优先级**：

1. 更新**当前已加载的技能**（本次会话中通过 `/skill-name` 或 `skill_view` 使用过的）
2. 更新**已有的伞式技能**（通过 `skills_list` + `skill_view` 查找）
3. 在已有技能下**添加支持文件**（`references/`、`templates/`、`scripts/`）
4. 仅在无现有技能覆盖时**创建新的类级伞式技能**

### 外部记忆提供者

通过插件系统（`plugins/memory/`）支持更高级的长期记忆后端：

```
MemoryManager (编排层，agent/memory_manager.py)
  ├── BuiltinMemoryProvider (MEMORY.md/USER.md)  ← 始终存在
  └── ExternalProvider (最多一个)
       ├── Honcho    ← 会话管理 + 用户画像
       ├── Hindsight ← 基于嵌入的语义检索
       ├── Mem0      ← 结构化记忆图谱
       ├── Holographic ← 全息记忆存储+检索
       ├── OpenViking  ← 开放检索增强
       ├── RetainDB    ← 数据库持久化
       ├── SuperMemory  ← 超级记忆
       └── ByteRover   ← 字节检索
```

生命周期钩子确保外部提供者参与所有关键时刻：

| 钩子 | 触发时机 | 作用 | 代码位置 |
|------|---------|------|---------|
| `on_turn_start()` | 每轮开始 | 轮次计数、节奏管理 | `conversation_loop.py:636` |
| `prefetch()` | API 调用前 | 召回相关上下文 | `conversation_loop.py:649` |
| `sync_turn()` | 轮次结束 | 持久化 (user, assistant) 对 | `conversation_loop.py:4253` |
| `on_pre_compress()` | 压缩前 | 从即将被丢弃的消息中提取洞察 | `conversation_compression.py:289` |
| `on_session_end()` | 会话结束 | 终局事实提取和摘要 | `memory_manager.py:429` |
| `on_memory_write()` | 内置记忆写入 | 镜像 MEMORY.md 变更到外部后端 | `memory_provider.py:247` |
| `on_delegation()` | 子代理完成 | 观察委派任务的结果 | `memory_provider.py:199` |
| `on_session_switch()` | session_id 轮换 | 更新缓存的会话状态 | `memory_manager.py:440` |

**记忆上下文注入格式**（`memory_manager.py:210-224`）：

```xml
<memory-context>
[System note: The following is recalled memory context,
NOT new user input. Treat as authoritative reference data —
this is the agent's persistent memory and should inform all responses.]

{prefetched context from providers}
</memory-context>
```

`StreamingContextScrubber`（L45-208）确保流式输出中的 `<memory-context>` 标签不泄漏到用户界面，即使标签跨越多个流式 chunk。

### 技能策展人

代码位于 `agent/curator.py`（~600 行）。后台定期审查 Agent 自建的技能：

```
draft (草稿)
  ├─→ frozen (稳定)     ← 质量达标，停止自动更新
  ├─→ archived (归档)   ← 被更好的技能替代
  ├─→ merged (合并)     ← 与相似技能整合
  └─→ patched (修补)    ← 修复缺陷
```

- **Pinned 技能**跳过所有自动转换（但允许内容修补）
- **Bundled/Hub-installed 技能**受保护，不可编辑
- 策展人仅操作 Agent 自建技能

---

## 完整数据流总结

```
用户消息
  │
  ▼
┌─────────────────────────────────────────┐
│ 工作记忆（单轮）                          │
│ system_prompt + memory_snapshot + prefetch │
│ + messages[-tail:] + new_user_msg         │
└──────────────────┬──────────────────────┘
                   │
                   ▼ LLM 推理 + 工具调用循环
                   │
┌──────────────────┴──────────────────────┐
│ 短期记忆（单会话）                        │
│ messages[] 全量对话历史                    │
│  ├─ 增量写入 SQLite (hermes_state.py)    │
│  └─ Token > 50% → ContextCompressor 压缩 │
│       ├─ 工具输出裁剪（无 LLM 调用）      │
│       ├─ 结构化摘要（辅助模型生成）        │
│       ├─ 会话分裂（新 session_id）         │
│       └─ 迭代摘要（保留上次摘要，增量更新） │
└──────────────────┬──────────────────────┘
                   │
                   ▼ 轮次结束后异步触发
                   │
┌──────────────────┴──────────────────────┐
│ 长期记忆（跨会话）                        │
│  ├─ 后台审查线程:                         │
│  │   ├─ 记忆审查 → MEMORY.md / USER.md   │
│  │   └─ 技能审查 → SKILL.md              │
│  ├─ 外部记忆插件:                         │
│  │   sync_turn() → Honcho/Hindsight/...  │
│  └─ 技能策展人:                           │
│      curator → freeze/archive/merge/patch │
└─────────────────────────────────────────┘
```

---

## 关键设计巧妙之处

1. **冻结快照保证 prefix cache**：长期记忆注入系统提示后不再变更，确保整个会话的 API 调用都能命中 prompt cache（节省 ~75% 输入 token 成本）

2. **压缩时的记忆挽救**：`on_pre_compress()` 钩子让外部记忆提供者在消息被丢弃前提取有价值信息，防止知识流失

3. **后台学习不阻塞主对话**：review 线程完全异步，复用父级 prompt cache（PR #17276 测得 ~26% 成本降低），用户无感知

4. **迭代式摘要**：每次压缩基于上次摘要增量更新（`_previous_summary`），避免信息在多次压缩中退化

5. **结构化摘要保护任务连续性**：`## Active Task` 强制保留用户最新未完成请求原文，`_ensure_last_user_message_in_tail` 锚定最后用户消息到保护区

6. **工具输出智能摘要**：不使用通用占位符，而是生成针对每种工具的结构化 1 行摘要，保留关键信息（命令、退出码、匹配数等）

7. **JSON 参数安全截断**：解析后缩短字符串值而非粗暴切割 JSON 文本，避免生成非法 JSON 导致后续 API 调用全部 400 报错

8. **学习信号的正负清单**：明确区分应学习和不应学习的内容类型，防止 Agent 将临时环境故障固化为永久性自我限制

---

## 相关文档

- [记忆系统设计](memory-system.md) — MemoryManager / Provider 接口设计
- [上下文压缩机制](context-compression.md) — 压缩算法与会话链接细节
- [记忆自我进化与会话固化](memory-self-evolution.md) — 会话如何总结并写入长期记忆
- [核心 Agent 循环](core-agent-loop.md) — 主对话循环架构
- [技能系统](skill-system.md) — 技能生命周期与管理
- [数据流](data-flow.md) — 系统整体数据流向
