# 消息存储与会话持久化

> 主 agent 的消息如何落盘 -- 内存/SQLite 两层架构、长会话格式、sessions 账本、消息归属判定

---

## 目录

- [整体架构：两层 + 批量 flush](#整体架构两层--批量-flush)
- [内存层：谁持有 messages](#内存层谁持有-messages)
- [持久化层：state.db 表结构](#持久化层statedb-表结构)
- [写入时机与调用链](#写入时机与调用链)
- [长 session 的消息格式](#长-session-的消息格式)
- [sessions 的一行长什么样](#sessions-的一行长什么样)
- [消息归属：物理归属 vs 逻辑归属](#消息归属物理归属-vs-逻辑归属)
- [会话恢复（resume）](#会话恢复resume)
- [各运行环境的差异](#各运行环境的差异)
- [FTS5 全文检索](#fts5-全文检索)
- [关键文件速查](#关键文件速查)
- [参考](#参考)

---

## 整体架构：两层 + 批量 flush

主 agent 的消息存储是**两层架构**：

1. **内存层**：`messages` 列表，由**调用方持有**（CLI / gateway / ACP），每 turn 传入传出
2. **持久化层**：SQLite 单库 `~/.hermes/state.db`（WAL 模式 + FTS5），turn 结束**批量增量** flush

写入不是逐条实时的——`run_conversation()` 的各个出口统一走 `_persist_session()`，用 `_last_flushed_db_idx` 索引去重，只写真正的新消息。

与子代理形成鲜明对照：**主 agent 全量落盘可恢复；子代理（`skip_memory=True`）一律不落盘、阅后即焚**——见 [委派系统 - 父子上下文流动](delegation-system.md#父子上下文流动单向精选隔离)。

---

## 内存层：谁持有 messages

`run_conversation()`（`agent/conversation_loop.py:343`）**不拥有**对话历史，它在 `:491` 做本地副本：

```python
messages = list(conversation_history) if conversation_history else []
```

循环中追加、结束后返回。所有权在外层：

| 环境 | 持有者 | turn 间延续 |
|------|--------|------------|
| CLI | `HermesCLI.conversation_history` 实例变量 | 跨多次 `run_conversation()` 累积 |
| Gateway | 每条用户消息**新建 Agent** | 从 SessionDB 加载历史传入 |
| ACP | `SessionState.agent_history` | SessionManager 维护 |

**Prefill / few-shot 消息只在 API 调用时注入，从不进 messages 列表**，因此不落盘、不进轨迹（`conversation_loop.py:521` 附近注释）。

---

## 持久化层：state.db 表结构

| 项 | 值 |
|----|----|
| 路径 | `~/.hermes/state.db`（Windows：`%LOCALAPPDATA%\hermes\state.db`） |
| 并发 | WAL 模式（网络文件系统自动降级回滚） |
| 外键 | `PRAGMA foreign_keys=ON`（`hermes_state.py:442`，SQLite 默认关闭、这里显式打开） |
| 全文索引 | FTS5 ×2（标准 unicode61 + trigram 专供 CJK 子串） |

核心两张表（`hermes_state.py:234` / `:271`）：

**`sessions`**（会话账本，36 列）——身份（`id/source/cwd`）、血缘（`parent_session_id`）、生命周期（`started_at/ended_at/end_reason`）、计量（`message_count`、token 五项、成本六项、`api_call_count`）、运营标记（`title/handoff_*/rewind_count/archived`）。

**`messages`**（消息本体）：

```sql
CREATE TABLE messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL REFERENCES sessions(id),  -- 归属，非空 + 外键
    role TEXT NOT NULL,            -- user / assistant / tool
    content TEXT,                  -- 文本或 JSON 编码的多模态部分
    tool_call_id TEXT,             -- tool 结果配对
    tool_calls TEXT,               -- JSON：[{name, arguments}]
    tool_name TEXT,
    timestamp REAL NOT NULL,
    token_count INTEGER,
    finish_reason TEXT,            -- stop / tool_calls / length
    reasoning TEXT,                -- thinking 内容（仅 assistant 行）
    reasoning_content TEXT, reasoning_details TEXT,
    codex_reasoning_items TEXT, codex_message_items TEXT,
    platform_message_id TEXT,      -- 外部平台消息 ID（Telegram update_id 等）
    observed INTEGER DEFAULT 0,    -- UI 是否已读
    active INTEGER NOT NULL DEFAULT 1  -- 0 = 已 rewind（软删除）
);
```

注意：**system prompt 不在 messages 表里**——存在 `sessions.system_prompt` 列（session 级一份，经 `redact_sensitive_text` 脱敏）。

---

## 写入时机与调用链

```
run_conversation() 各出口（成功 / 重试耗尽 / 上下文超限…共 9+ 处）
  → agent._persist_session()                  (run_agent.py:1446)
      ├ 清理失败重试的脚手架消息
      ├ _save_session_log()                   （可选 JSON 日志）
      └ _flush_messages_to_session_db()       (run_agent.py:1515)
          → 只写 messages[_last_flushed_db_idx:] 的增量
          → 逐条 SessionDB.append_message()   (hermes_state.py:1929)
              INSERT INTO messages
              UPDATE sessions 计数器（message_count/tool_call_count，:2013）
```

三个关键设计：

1. **`_last_flushed_db_idx` 去重**：多个出口都可能调 `_persist_session()`，靠这个索引保证不重复写（修复过的 #860 重复写入 bug）。resume 时把它设为 `len(conversation_history)`，防止旧消息重写。
2. **多模态降级**：base64 图片**不落盘**，替换为文本摘要或 `"[screenshot]"` 占位——防 DB 膨胀，跨会话回放也用不上。
3. **token/成本另走一条线**：`update_session_usage()`（`hermes_state.py:1180`）——CLI 按 API 调用**增量累加**，gateway 传 `absolute=True` **直接覆盖**（缓存 agent 自己持有累计值）。

---

## 长 session 的消息格式

一个典型的"一轮（turn）"在 messages 表里：

| id | role | content | tool_calls | tool_call_id | tool_name | finish_reason | reasoning |
|----|------|---------|-----------|--------------|-----------|---------------|-----------|
| 101 | `user` | `"帮我查下测试覆盖率"` | | | | | |
| 102 | `assistant` | `NULL` | `[{"name":"terminal","arguments":"{\"command\":\"pytest --cov\"}"}]` | | | `tool_calls` | `"先跑 pytest..."` |
| 103 | `tool` | `"coverage: 73%"` | | `call_abc123` | `terminal` | | |
| 104 | `assistant` | `"覆盖率 73%，缺口在..."` | | | | `stop` | |

长 session 就是 `user → (assistant+tool_calls → tool)×N → assistant(stop)` 模式的大量重复。各 role 的列使用规则（`_flush_messages_to_session_db`）：

- **assistant 发起工具调用**：`content=NULL`，`tool_calls` 只存 `{name, arguments}`（不存 OpenAI 原始 id/type 包装）
- **tool 结果**：`content` 存输出原文，`tool_call_id` + `tool_name` 配对
- **思考内容只在 assistant 行**：`reasoning` / `reasoning_content` / `reasoning_details` / `codex_*`，按 provider 填不同列
- **结构化字段全部 JSON 序列化成 TEXT**

### 长 session 特有的两个形态

**1. `active=0` 软删除（rewind）**：用户回退后，之后的行不删除、只置 `active=0`，同时 `sessions.rewind_count +1`（`hermes_state.py:2603`）。物理行数 ≥ 逻辑行数；读回默认 `WHERE active=1`。

**2. 压缩链——超长会话实际是"一串 session"**：上下文逼近上限触发压缩后，当前 session 终结（`end_reason='compression'`），新建 session 用 `parent_session_id` 链回去（`agent/conversation_compression.py:497`）：

```
20260601_091500_a1b2c3   ended, end_reason='compression'   ← 原始消息完整保留
     ▲ parent_session_id
20260601_143000_d4e5f6   ended, end_reason='compression'   ← 开头是压缩摘要
     ▲ parent_session_id
20260602_100000_g7h8i9   active                            ← 当前正在写
```

每个新段的**第一条消息就是上一段的压缩摘要**（普通 `role=assistant` 文本，无特殊标记）。原始消息**不删**，留在旧段。

---

## sessions 的一行长什么样

以一个已被压缩终结的 CLI 长会话为例（`.mode line` 视图，节选）：

```
                 id = 20260605_093012_e4f7a2     ← agent_init.py:956：时间戳+6位uuid
             source = cli                         ← cli / telegram / acp / unknown
              model = anthropic/claude-opus-4-8
      system_prompt = You are Hermes...           ← session 级一份，脱敏后
  parent_session_id = 20260604_181455_b3c9d1      ← 压缩链；首段为 NULL
         started_at = 1781055012.48
           ended_at = 1781069410.29               ← NULL = 活跃
         end_reason = compression                 ← user / compression / error…
      message_count = 412
    tool_call_count = 187
       input_tokens = 8431205    （另有 output/cache_read/cache_write/reasoning 四项）
 estimated_cost_usd = 14.73
              title = hermes-docs-deep-dive       ← 全库唯一索引
     api_call_count = 203
       rewind_count = 2
           archived = 0
```

**这一行不是一次写成的**，四个阶段渐进填充：

1. **创建**（`_insert_session_row`，`hermes_state.py:940`）：只写 9 列骨架（id/source/user_id/model/model_config/system_prompt/parent_session_id/cwd/started_at），`INSERT OR IGNORE` 抗并发。
2. **运行中累加**：每条消息 → 计数器 +1；每次 API 调用 → token/成本/api_call_count。
3. **终结**（`end_session`，`:975`）：填 `ended_at/end_reason`，**先到先得**——已终结的行不被改写，保护压缩链上的 `end_reason='compression'` 不被 stale 调用覆盖。
4. **复活**（`reopen_session`，`:993`）：resume 时清回 NULL。

---

## 消息归属：物理归属 vs 逻辑归属

**单条消息属于哪个 session，由 `messages.session_id` 决定**——三层保证：

1. **Schema**：`session_id TEXT NOT NULL REFERENCES sessions(id)` + `PRAGMA foreign_keys=ON`（`:442`），不存在孤儿消息。
2. **写入**：flush 循环用 `session_id=self.session_id` 盖章——归属时点是 **flush 时刻 agent 身上的 session_id**。压缩切换正是靠这个机制：换 id + `_last_flushed_db_idx` 归零，之后的消息自然归新段，旧消息无需迁移。
3. **查询**：`idx_messages_session(session_id, timestamp)` 索引加速。

但注意 **"属于这个 session" ≠ "属于这场对话"**：压缩链使一场逻辑对话横跨多个 session 段，每条消息只物理归属其中一段。按整场对话取数要沿链聚合：

```python
db.get_messages_as_conversation(session_id, include_ancestors=True)
# 内部 _session_lineage_root_to_tip()（hermes_state.py:2498）
# 沿 parent_session_id 从根到梢，WHERE session_id IN (整条链)
```

核对账本的实用 SQL：

```sql
SELECT s.id, s.message_count, COUNT(m.id) AS actual
FROM sessions s LEFT JOIN messages m ON m.session_id = s.id
WHERE s.id = '20260605_093012_e4f7a2';
```

---

## 会话恢复（resume）

`hermes --resume <id>`（`cli.py:5089` 一带）：

```
1. get_session(id) 取元数据
2. resolve_resume_session_id(id)        ← 解决压缩链：若该段已被压缩，跳到链上有数据的段
3. get_messages_as_conversation(id)     ← 加载回 conversation_history
4. agent._last_flushed_db_idx = len(history)   ← 防旧消息重写
5. reopen_session(id)                   ← 清 ended_at，会话复活
```

读回是无损往返（图片除外）：`tool_calls` JSON 反序列化、content 经 `sanitize_context` 清洗，产出标准 OpenAI 格式字典列表，直接作下一轮 `conversation_history`。

---

## 各运行环境的差异

三种环境共用同一个 `state.db`，靠 `source` 字段区分：

| 环境 | source | session 对应 | 额外持久化 |
|------|--------|-------------|-----------|
| CLI | `cli` | 一次启动（可 resume） | 无 |
| Gateway | `telegram` / `discord`… | 一个用户/平台组合 | **JSONL transcript 备份**（仅当 agent 未持久化成功，`gateway/run.py:9787`，避免双写） |
| ACP | `acp` | 一个编辑器工作区 | SessionManager 另存 JSON |

---

## FTS5 全文检索

服务于 **`session_search_tool`** 历史搜索工具（`tools/session_search_tool.py`）：

| 模式 | 用法 | FTS5 |
|------|------|------|
| DISCOVERY | 跨会话关键词搜索，BM25 排名 + snippet 高亮 | ✅ |
| SCROLL | 会话内按消息翻页 | 直接 SELECT |
| BROWSE | 列最近会话 | 按时间排序 |

- 索引内容 = 消息文本 + 工具名 + 工具调用 JSON，触发器自动同步（`hermes_state.py:320-370`）
- **CJK 处理**：查询含 ≥3 个 CJK 字符走 trigram 表（子串匹配），更短降级 LIKE
- 查询语法支持 `"exact phrase"` / `OR` / `NOT` / `prefix*`（`search_messages()`，`hermes_state.py:2791`）

---

## 关键文件速查

| 功能 | 文件:行 | 符号 |
|------|---------|------|
| 对话循环 / messages 副本 | `agent/conversation_loop.py:343` / `:491` | `run_conversation` |
| 持久化入口 | `run_agent.py:1446` | `_persist_session` |
| 增量 flush + 去重 | `run_agent.py:1515` | `_flush_messages_to_session_db` / `_last_flushed_db_idx` |
| 表结构 | `hermes_state.py:234` / `:271` | sessions / messages schema |
| 外键开关 | `hermes_state.py:442` | `PRAGMA foreign_keys=ON` |
| 会话生命周期 | `hermes_state.py:940/971/975/993` | `_insert_session_row / create_session / end_session / reopen_session` |
| 用量计费 | `hermes_state.py:1180` | `update_session_usage`（增量 / absolute 两模式） |
| 消息写入 | `hermes_state.py:1929` | `append_message` |
| 消息读回 | `hermes_state.py:2410` | `get_messages_as_conversation` |
| 压缩链遍历 | `hermes_state.py:2498` | `_session_lineage_root_to_tip` |
| 全文搜索 | `hermes_state.py:2791` | `search_messages` |
| session_id 生成 | `agent/agent_init.py:956` | 时间戳 + 6 位 uuid |
| 压缩分段 | `agent/conversation_compression.py:497` | `parent_session_id` 设置 |
| Gateway JSONL 备份 | `gateway/run.py:9787` | `append_to_transcript` |

---

## 参考

- [上下文压缩](context-compression.md) -- 压缩触发条件与会话链接
- [数据流图](data-flow.md) -- 用户消息流与工具执行流
- [委派系统 - 父子上下文流动](delegation-system.md#父子上下文流动单向精选隔离) -- 子代理为何不落盘
- [记忆体系代码级深度分析](memory-architecture-deep-dive.md) -- 短期→长期记忆转化
