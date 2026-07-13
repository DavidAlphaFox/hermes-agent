# 数据流图

> 用户消息流、工具执行流、记忆流、上下文压缩流、网关消息路由流

---

## 目录

- [概述](#概述)
- [用户消息流](#用户消息流)
- [工具执行流](#工具执行流)
- [记忆流](#记忆流)
- [上下文压缩流](#上下文压缩流)
- [网关消息路由流](#网关消息路由流)
- [ACP 编辑器数据流](#acp-编辑器数据流)
- [定时任务数据流](#定时任务数据流)

---

## 概述

Hermes Agent 的数据流可以从五个维度理解：

1. **用户消息流** -- 从输入到 LLM 回复的完整路径
2. **工具执行流** -- LLM 决策到工具执行再到结果反馈
3. **记忆流** -- 跨会话知识的加载、注入和同步
4. **上下文压缩流** -- token 预算管理和对话摘要
5. **网关路由流** -- 多平台消息的接收、处理和回复

---

## 用户消息流

### CLI 路径

```
用户输入 (prompt_toolkit TextArea)
  |
  v
cli.py: 斜杠命令拦截
  |-- /help, /model, /tools -> 本地处理
  +-- 普通消息
  |
  v
cli.py: 构建 user_message
  |
  v
AIAgent.chat(user_message, conversation_history)
  |
  |-- 1. 预取记忆 (memory_manager.prefetch_context)
  |      +-- 异步: 供应商 recall() -> 记忆片段
  |
  |-- 2. 构建系统提示 (_build_system_prompt)
  |      |-- SOUL.md 基础人格
  |      |-- 记忆注入（等待预取完成）
  |      |-- 工具定义
  |      +-- 会话上下文
  |
  |-- 3. 检查 token 预算
  |      +-- 超限 -> 触发上下文压缩
  |
  |-- 4. API 调用 (client.chat.completions.create)
  |      |-- messages: [system, ...history, user_message]
  |      |-- tools: [...tool_definitions]
  |      +-- stream: True
  |
  |-- 5. 流式响应处理
  |      |-- 文本 -> message_callback -> 显示
  |      |-- tool_calls -> 进入工具执行流
  |      +-- reasoning -> thinking_callback -> 显示
  |
  |-- 6. 工具循环 (如有 tool_calls)
  |      +-- 见 "工具执行流"
  |
  +-- 7. 最终响应
         |-- 更新 conversation_history
         |-- 记忆同步 (memory_manager.on_turn_complete)
         +-- 会话持久化 (SessionDB)
```

### Gateway 路径

```
平台消息 (Telegram/Discord/Slack/...)
  |
  v
gateway/run.py: 消息接收
  |
  v
BasePlatformAdapter._process_message_background()
  |-- 提取: 文本 / 图片 / 语音 / 文件
  |-- 查找或创建会话 (session_id = platform:chat_id)
  +-- 转换为统一格式
  |
  v
AIAgent.chat(user_message, conversation_history)
  | (与 CLI 路径相同)
  |
  v
最终响应
  |-- 分块发送 (长消息拆分)
  |-- 媒体附件投递
  +-- 线程/话题管理
```

### ACP 路径

```
编辑器 (VS Code/Zed/JetBrains)
  | JSON-RPC (stdio)
  v
HermesACPAgent.prompt(content_blocks, session_id)
  |
  |-- 提取文本 (_extract_text)
  |-- 斜杠命令拦截 -> 本地处理
  +-- 普通消息
  |
  v
ThreadPoolExecutor: agent.run_conversation(user_text, history)
  |
  |-- 回调桥接:
  |    |-- tool_progress -> ACP ToolCallStart
  |    |-- thinking      -> ACP ThoughtText
  |    |-- step          -> ACP ToolCallComplete
  |    +-- message       -> ACP MessageText
  |
  +-- PromptResponse(stop_reason, usage)
```

---

## 工具执行流

### 完整路径

```
LLM 返回 tool_calls
  |
  v
AIAgent._process_tool_calls(tool_calls)
  |
  |-- 对每个 tool_call:
  |    |
  |    |-- tool_progress_callback("tool.started", name, args)
  |    |
  |    |-- registry.dispatch(name, args, task_id)
  |    |    |
  |    |    |-- 查找注册的 handler
  |    |    |-- 可用性检查 (availability_fn)
  |    |    +-- 执行 handler(args)
  |    |         |
  |    |         |-- 终端工具 -> subprocess / Modal / Docker
  |    |         |-- 文件工具 -> 本地文件系统
  |    |         |-- 浏览器工具 -> Playwright
  |    |         |-- Web 工具 -> httpx / firecrawl
  |    |         |-- MCP 工具 -> MCP 服务器 RPC
  |    |         +-- 委托工具 -> 子 AIAgent
  |    |
  |    |-- 构建 tool_result 消息
  |    |    {"role": "tool", "tool_call_id": ..., "content": result}
  |    |
  |    +-- step_callback(api_call_count, prev_tools)
  |
  |-- 将所有 tool_result 追加到 messages
  |
  |-- 检查 token 预算 -> 可能触发压缩
  |
  +-- 继续下一轮 API 调用 (携带 tool_results)
       +-- LLM 决定: 继续调用工具 / 生成最终回复
```

### 审批流程

```
终端工具执行前:
  |
  |-- CLI 模式:
  |    +-- approval_callback -> 终端提示用户确认
  |
  |-- ACP 模式:
  |    +-- approval_callback -> 编辑器弹窗
  |        +-- conn.request_permission() -> 用户响应
  |
  |-- Gateway 模式:
  |    +-- 自动批准（无交互用户）
  |
  +-- --yolo 模式:
       +-- 全部自动批准
```

---

## 记忆流

### 预取流程

```
chat() 入口
  |
  |-- [异步] memory_manager.prefetch_context(user_message)
  |    |
  |    |-- 内置记忆 (memory.py):
  |    |    +-- SQLite FTS5 搜索 -> 相关记忆条目
  |    |
  |    +-- 插件记忆 (MemoryProvider):
  |         |-- honcho: recall() -> 用户画像 + 语义搜索
  |         |-- holographic: recall() -> HRR 向量检索
  |         |-- supermemory: recall() -> 语义搜索 + 实体上下文
  |         |-- mem0: recall() -> 平台 API 搜索
  |         +-- ...
  |
  |-- [并行] 继续构建系统提示...
  |
  +-- _build_system_prompt():
       |-- 等待预取完成 (future.result(timeout=5))
       +-- 注入记忆到系统提示
            +-- <memory>...</memory> 标签
```

### 写入流程

```
对话结束 / 每 N 轮:
  |
  v
memory_manager.on_turn_complete(messages)
  |
  |-- 内置记忆:
  |    |-- 自动提取: LLM 分析对话 -> 提取关键事实
  |    +-- 手动: 用户 /remember 命令
  |
  +-- 插件记忆:
       |-- honcho: ingest() -> 服务端对话分析 + 结论
       |-- holographic: ingest() -> 实体解析 + HRR 编码 + SQLite
       |-- supermemory: ingest() -> API 存储 + 去重
       +-- ...
```

---

## 上下文压缩流

### 触发条件

```
每轮 API 调用前:
  |
  |-- estimate_messages_tokens_rough(messages) -> 估算 token 数
  |
  |-- 当前 token 数 > model_context_window * threshold
  |    |
  |    +-- 触发压缩
  |
  +-- 未超限 -> 正常调用
```

### 压缩流程

```
_compress_context(messages, system_prompt)
  |
  |-- 1. 分类消息:
  |      |-- 保护区: system + 最近 N 条消息
  |      +-- 可压缩区: 中间的历史消息
  |
  |-- 2. 选择策略:
  |      |-- prune     -- 删除旧工具调用结果（保留摘要）
  |      |-- summarize -- LLM 生成结构化摘要
  |      +-- hybrid    -- 先裁剪再摘要
  |
  |-- 3. 生成摘要:
  |      +-- LLM 调用: "将以下对话压缩为结构化摘要..."
  |           |-- 保留: 关键决策、文件修改、用户偏好
  |           +-- 丢弃: 重复的工具输出、中间推理
  |
  |-- 4. 替换:
  |      |-- 删除被压缩的消息
  |      +-- 插入摘要消息: {"role": "system", "content": summary}
  |
  +-- 5. 会话链接:
         |-- 创建新 session_id (parent -> child)
         +-- 保留完整历史的引用链
```

### 压缩前后对比

```
压缩前 (150 条消息, ~80K tokens):
  [system] [user] [assistant] [tool] [tool] [assistant]
  [user] [assistant] [tool] [tool] [tool] [assistant]
  ... (大量中间工具调用)
  [user] [assistant]  <-- 最近消息

压缩后 (20 条消息, ~15K tokens):
  [system]
  [system: 对话摘要]  <-- 替换了中间 130 条消息
  [user] [assistant]  <-- 保留的最近消息
```

---

## 网关消息路由流

### 入站消息流

```
              平台 SDK / Webhook
                   |
                   v
         BasePlatformAdapter
          on_message()
                   |
              消息预处理
                   |
      +------------+------------+
      |            |            |
   文本消息     语音消息     图片消息
               STT转录     视觉描述
      |            |            |
      +------------+------------+
                   |
              会话查找/创建
                   |
            AIAgent.chat()
                   |
              响应处理
                   |
      +------------+------------+
      |            |            |
   文本回复     语音合成     图片生成
   分块发送      TTS       直接发送
```

### 出站消息流

```
AIAgent 最终响应
  |
  v
BasePlatformAdapter._process_message_background()
  |
  |-- 提取 MEDIA: 标签 -> 独立媒体文件
  |
  |-- 长消息分块:
  |    |-- 按代码块边界分割
  |    |-- 按段落边界分割
  |    +-- 硬分割（最后手段）
  |
  |-- 发送文本块:
  |    +-- adapter.send(chat_id, chunk, metadata)
  |
  +-- 发送媒体文件:
       |-- send_voice() / send_image_file()
       |-- send_video() / send_document()
       +-- 按扩展名自动路由
```

### 会话持久化

```
网关会话管理:
  |
  |-- session_id = f"{platform}:{chat_id}"
  |
  |-- SessionDB (SQLite):
  |    |-- sessions 表: 会话元数据
  |    +-- messages 表: 对话历史
  |
  +-- 内存缓存:
       +-- conversation_histories[session_id] = [...]
```

---

## ACP 编辑器数据流

### 请求-响应流

```
编辑器                    HermesACPAgent              AIAgent
  |                            |                        |
  |-- prompt(text) ----------> |                        |
  |                            |-- 创建回调 -----------> |
  |                            |-- run_in_executor ----> |
  |                            |                        |-- API 调用
  | <-- thought_text --------- | <-- thinking_cb ------ |
  | <-- tool_call_start ------ | <-- tool_progress_cb -- |
  |                            |                        |-- 工具执行
  | <-- tool_call_complete --- | <-- step_cb ---------- |
  | <-- message_text --------- | <-- message_cb -------- |
  |                            |                        |
  | <-- PromptResponse ------ | <-- 完成 -------------- |
  |                            |                        |
  |                            |-- 持久化到 SQLite       |
  |                            +-- 更新 session history  |
```

### 权限请求流

```
AIAgent (工作线程)         HermesACPAgent (主线程)        编辑器
  |                            |                          |
  |-- approval_callback -----> |                          |
  |   (command, description)   |-- request_permission --> |
  |                            |                          |-- 用户确认弹窗
  |                            | <-- response ----------- |
  | <-- "once"/"always"/"deny" |                          |
```

---

## 定时任务数据流

### 执行数据流

```
网关 tick (每 60 秒)
  |
  |-- 获取文件锁
  |
  |-- get_due_jobs() <-- 读取 jobs.json
  |
  |-- 对每个到期作业:
  |    |
  |    |-- advance_next_run() -> 写入 jobs.json (预推进)
  |    |
  |    |-- _build_job_prompt():
  |    |    |-- cron 系统提示
  |    |    |-- [可选] 脚本输出 (_run_job_script)
  |    |    |-- [可选] 技能加载 (skill_view)
  |    |    +-- 用户 prompt
  |    |
  |    |-- run_job():
  |    |    |-- 加载 .env + config.yaml
  |    |    |-- resolve_runtime_provider()
  |    |    |-- AIAgent(skip_memory=True, platform="cron")
  |    |    |-- agent.run_conversation(prompt) <-- 独立线程 + 不活跃超时
  |    |    +-- 返回 (success, output, response, error)
  |    |
  |    |-- save_job_output() -> output/<job_id>/timestamp.md
  |    |
  |    |-- _deliver_result():
  |    |    |-- _resolve_delivery_target() -> 平台 + chat_id
  |    |    |-- 优先: 活跃适配器 (支持 E2EE)
  |    |    +-- 回退: 独立 HTTP 发送
  |    |
  |    +-- mark_job_run() -> 更新 jobs.json (状态 + 计数)
  |
  +-- 释放文件锁
```
