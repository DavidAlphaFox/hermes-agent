# 定时任务系统

> 调度器架构、作业存储、生命周期管理、投递目标、文件锁防并发

---

## 目录

- [概述](#概述)
- [架构](#架构)
- [调度器主循环](#调度器主循环)
- [作业存储](#作业存储)
- [调度解析](#调度解析)
- [作业生命周期](#作业生命周期)
- [作业执行](#作业执行)
- [投递系统](#投递系统)
- [安全机制](#安全机制)
- [脚本系统](#脚本系统)

---

## 概述

Hermes Agent 的定时任务系统允许 Agent 自调度任务，实现自动化的定期执行。系统由两个核心模块组成：

| 模块 | 文件 | 大小 | 职责 |
|------|------|------|------|
| 作业存储 | `cron/jobs.py` | 25KB | 作业 CRUD、调度解析、到期检测 |
| 调度器 | `cron/scheduler.py` | 35KB | tick 主循环、执行引擎、结果投递 |

```
网关 (gateway/run.py)
  │
  └─ 每 60 秒 ──► tick()
                    │
                    ├─ get_due_jobs()        ← jobs.json
                    │
                    ├─ advance_next_run()     ← 预推进（防崩溃重复执行）
                    │
                    ├─ run_job(job)           ← AIAgent 执行
                    │   ├─ _build_job_prompt()
                    │   └─ agent.run_conversation()
                    │
                    ├─ save_job_output()      ← 保存到文件
                    │
                    ├─ _deliver_result()      ← 投递到平台
                    │
                    └─ mark_job_run()         ← 更新状态
```

---

## 架构

### 存储位置

```
~/.hermes/cron/
  ├─ jobs.json           # 作业定义（JSON）
  ├─ .tick.lock          # 文件锁（防并发）
  └─ output/
       └─ <job_id>/
            └─ 2026-04-16_14-30-00.md   # 执行输出
```

所有目录权限设为 0700（仅所有者可访问），文件权限设为 0600。

### 触发方式

定时任务由网关守护进程驱动：

```bash
# 安装为系统服务（推荐）
hermes gateway install

# 手动前台运行
hermes gateway

# 手动触发一次 tick
python -m cron.scheduler
```

---

## 调度器主循环

### tick() 函数

`scheduler.py` 的核心是 `tick()` 函数，每 60 秒由网关调用一次：

```python
def tick(verbose=True, adapters=None, loop=None) -> int:
    """检查并运行所有到期作业。

    参数:
        verbose:  是否打印状态消息
        adapters: Platform → 活跃适配器映射（来自网关）
        loop:     asyncio 事件循环（来自网关）

    返回:
        执行的作业数量（0 = 另一实例持有锁）
    """
```

### 文件锁机制

使用文件锁防止多个进程同时执行 tick：

```python
# Unix: fcntl.flock (LOCK_EX | LOCK_NB)
# Windows: msvcrt.locking (LK_NBLCK)

lock_fd = open(_LOCK_FILE, "w")
fcntl.flock(lock_fd, fcntl.LOCK_EX | fcntl.LOCK_NB)
# 如果获取失败 → 跳过本次 tick
```

锁文件路径：`~/.hermes/cron/.tick.lock`

### 执行流程

```
tick()
  │
  ├─ 获取文件锁（非阻塞，失败则跳过）
  │
  ├─ get_due_jobs() → 到期作业列表
  │
  ├─ 对每个到期作业:
  │   │
  │   ├─ advance_next_run()          ← 预推进 next_run_at（at-most-once 语义）
  │   │
  │   ├─ run_job(job)                ← 执行 AIAgent
  │   │   返回: (success, output, final_response, error)
  │   │
  │   ├─ save_job_output(job_id, output)  ← 保存到 output/<job_id>/
  │   │
  │   ├─ 判断是否投递:
  │   │   ├─ [SILENT] 响应 → 跳过投递
  │   │   ├─ 失败 → 投递错误消息
  │   │   └─ 成功 → 投递最终响应
  │   │
  │   ├─ _deliver_result()           ← 投递到目标平台
  │   │
  │   └─ mark_job_run()              ← 更新状态 + 计数
  │
  └─ 释放文件锁
```

---

## 作业存储

### jobs.json 格式

```json
{
  "jobs": [
    {
      "id": "a1b2c3d4e5f6",
      "name": "每日代码审查",
      "prompt": "检查最近的提交...",
      "skills": ["code-reviewer"],
      "skill": "code-reviewer",
      "model": null,
      "provider": null,
      "base_url": null,
      "script": "check_repos.py",
      "schedule": {
        "kind": "cron",
        "expr": "0 9 * * *",
        "display": "0 9 * * *"
      },
      "schedule_display": "0 9 * * *",
      "repeat": {
        "times": null,
        "completed": 5
      },
      "enabled": true,
      "state": "scheduled",
      "created_at": "2026-04-10T08:00:00",
      "next_run_at": "2026-04-17T09:00:00",
      "last_run_at": "2026-04-16T09:00:00",
      "last_status": "ok",
      "last_error": null,
      "deliver": "origin",
      "origin": {
        "platform": "telegram",
        "chat_id": "123456789",
        "chat_name": "Dev Channel"
      }
    }
  ],
  "updated_at": "2026-04-16T09:01:00"
}
```

### 原子写入

作业文件使用原子写入保护数据完整性：

```python
def save_jobs(jobs):
    # 1. 创建临时文件 (mkstemp)
    # 2. 写入 JSON + fsync
    # 3. os.replace() 原子替换
    # 4. 设置文件权限 0600
```

### CRUD 操作

| 函数 | 说明 |
|------|------|
| `create_job()` | 创建新作业，解析调度，计算首次运行时间 |
| `get_job(job_id)` | 按 ID 获取作业 |
| `list_jobs()` | 列出所有作业（可选包含禁用的） |
| `update_job(job_id, updates)` | 更新作业字段，自动刷新调度 |
| `remove_job(job_id)` | 删除作业 |
| `pause_job(job_id)` | 暂停作业（保留定义） |
| `resume_job(job_id)` | 恢复暂停的作业，计算下次运行时间 |
| `trigger_job(job_id)` | 手动触发（下次 tick 立即执行） |

---

## 调度解析

### parse_schedule() 支持的格式

| 格式 | 示例 | 类型 |
|------|------|------|
| 持续时间 | `30m`, `2h`, `1d` | 一次性（从现在起） |
| 循环间隔 | `every 30m`, `every 2h` | 周期性 |
| Cron 表达式 | `0 9 * * *` | Cron 表达式（需 croniter） |
| ISO 时间戳 | `2026-04-17T14:00` | 一次性（指定时间） |

### 解析结果

```python
# 一次性
{"kind": "once", "run_at": "2026-04-16T15:00:00+08:00", "display": "once in 30m"}

# 循环间隔
{"kind": "interval", "minutes": 120, "display": "every 120m"}

# Cron
{"kind": "cron", "expr": "0 9 * * *", "display": "0 9 * * *"}
```

### 下次运行计算

```python
def compute_next_run(schedule, last_run_at=None):
    # once  → 尚未运行且在宽限期内 → run_at; 已运行 → None
    # interval → last_run + minutes; 首次 → now + minutes
    # cron → croniter.get_next(datetime)
```

---

## 作业生命周期

### 状态机

```
创建 ─────► scheduled ─────► (执行中) ─────► scheduled
                │                               │
                │                         ┌─────┘
                ▼                         │
             paused ◄─── pause_job()      │
                │                         │
                ▼                         │
             scheduled ◄── resume_job()   │
                                          │
                               repeat.completed >= repeat.times
                                          │
                                          ▼
                                      completed (或删除)
```

### 作业状态字段

| 字段 | 值 | 说明 |
|------|-----|------|
| `state` | `scheduled` | 正常调度中 |
| `state` | `paused` | 已暂停 |
| `state` | `completed` | 一次性作业已完成 |
| `enabled` | `true/false` | 是否启用 |
| `repeat.times` | `null/N` | 总执行次数（null=无限） |
| `repeat.completed` | `N` | 已完成次数 |

### 到期检测

`get_due_jobs()` 的检测逻辑：

```
对每个启用的作业:
  1. 检查 next_run_at 是否 <= 当前时间
  2. 对于周期性作业 (cron/interval):
     - 计算宽限期 = min(周期/2, 2小时)
     - 超过宽限期的过期作业 → 快进到下次运行（防止网关重启后爆发执行）
  3. 对于一次性作业 (once):
     - 120 秒宽限期（防止刚好错过）
     - 已运行过 → 永不再运行
```

### 崩溃安全

为防止进程崩溃导致重复执行，采用 **at-most-once** 语义：

```python
# 执行前：预推进 next_run_at
advance_next_run(job_id)
# 如果崩溃 → 作业不会在下次重启时重新触发

# 执行后：mark_job_run() 更新状态
```

一次性作业不做预推进，以便崩溃后可重试。

---

## 作业执行

### run_job() 流程

```python
def run_job(job) -> tuple[bool, str, str, Optional[str]]:
    # 1. 初始化 SQLite 会话存储
    # 2. 构建 prompt (_build_job_prompt)
    # 3. 注入环境变量（origin 平台信息）
    # 4. 重新加载 .env 和 config.yaml
    # 5. 解析供应商路由
    # 6. 创建 AIAgent 实例
    # 7. 在独立线程中运行，带不活跃超时监控
    # 8. 返回 (success, output_doc, final_response, error)
```

### prompt 构建

`_build_job_prompt()` 按以下顺序组装提示词：

```
1. [SYSTEM: cron 执行指南]
   - 告知 Agent 以定时任务身份运行
   - 说明自动投递机制（不需要手动发送）
   - [SILENT] 标记说明

2. [脚本输出]（如有 script 配置）
   - 执行数据收集脚本
   - 注入输出作为上下文

3. [技能加载]（如有 skills 配置）
   - 按顺序加载每个技能内容

4. 用户 prompt
```

### 不活跃超时

作业使用基于不活跃的超时机制（默认 600 秒）：

```python
# 每 5 秒轮询 agent.get_activity_summary()
# 如果 seconds_since_activity >= 限制 → 中断
# 可通过 HERMES_CRON_TIMEOUT 环境变量覆盖（0=无限）
```

作业可以运行数小时（活跃调用工具/接收流式 token），但 hung 的 API 调用或卡住的工具会被检测并中断。

### AIAgent 配置

定时任务使用特定的 Agent 配置：

```python
agent = AIAgent(
    disabled_toolsets=["cronjob", "messaging", "clarify"],  # 禁用自调度和消息发送
    quiet_mode=True,        # 静默模式
    skip_memory=True,       # 跳过记忆（避免污染用户画像）
    platform="cron",        # 平台标记
    session_id=f"cron_{job_id}_{timestamp}",
)
```

---

## 投递系统

### 投递目标解析

`_resolve_delivery_target()` 解析作业的投递配置：

| deliver 值 | 行为 |
|------------|------|
| `"local"` | 仅保存到本地文件 |
| `"origin"` | 投递到创建作业的原始聊天 |
| `"telegram"` | 投递到 Telegram 主频道 |
| `"discord:channel_id"` | 投递到 Discord 指定频道 |
| `"slack:channel_name"` | 投递到 Slack 指定频道 |
| `"webhook"` | 投递到 Webhook |

### origin 回退链

当 `deliver="origin"` 但无原始聊天信息时：

```
origin.platform + origin.chat_id → 直接投递
  │ (不存在)
  ▼
依次检查 HOME_CHANNEL 环境变量:
  MATRIX_HOME_CHANNEL → TELEGRAM_HOME_CHANNEL
  → DISCORD_HOME_CHANNEL → SLACK_HOME_CHANNEL
```

### 投递路径

```
_deliver_result()
  │
  ├─ 优先: 使用网关活跃适配器 (支持端到端加密)
  │   └─ asyncio.run_coroutine_threadsafe(adapter.send(...))
  │
  └─ 回退: 独立 HTTP 发送
      └─ _send_to_platform() (来自 send_message_tool)
```

### 媒体文件投递

输出中的 `MEDIA:` 标签会被提取为原生平台附件：

```python
# 按扩展名路由:
# .ogg/.mp3/.wav  → send_voice()
# .mp4/.mov       → send_video()
# .jpg/.png/.webp → send_image_file()
# 其他            → send_document()
```

### [SILENT] 标记

Agent 可通过返回 `[SILENT]` 抑制投递（仍保存到本地）：

```
[SYSTEM: ...如果确实没有新内容需要报告，
请只回复 "[SILENT]"（不含其他内容）以抑制投递...]
```

### 投递包装

默认情况下，投递内容会添加头尾信息：

```
Cronjob Response: 每日代码审查
-------------

(Agent 输出内容)

Note: The agent cannot see this message, and therefore cannot respond to it.
```

可通过 `cron.wrap_response: false` 关闭包装。

---

## 安全机制

### 文件权限

```python
_secure_dir(path)   # 目录: 0700 (仅所有者)
_secure_file(path)  # 文件: 0600 (仅所有者读写)
```

### 脚本沙箱

数据收集脚本必须位于 `~/.hermes/scripts/` 目录内：

```python
# 路径遍历防护
path.relative_to(scripts_dir_resolved)
# 相对路径 → 解析为 scripts/ 下
# 绝对路径 → 验证在 scripts/ 内
# 符号链接 → resolve() 后验证

# 执行限制
subprocess.run(
    [sys.executable, str(path)],
    timeout=120,        # 120 秒超时
    capture_output=True  # 隔离 stdout/stderr
)

# 输出脱敏
redact_sensitive_text(stdout)  # 移除 API 密钥等敏感信息
```

### 环境变量隔离

每个作业执行期间注入的环境变量在 finally 块中清理，防止泄露到其他作业：

```python
finally:
    for key in (
        "HERMES_SESSION_PLATFORM",
        "HERMES_SESSION_CHAT_ID",
        "HERMES_CRON_AUTO_DELIVER_PLATFORM",
        ...
    ):
        os.environ.pop(key, None)
```

### 投递平台白名单

投递目标的平台名使用白名单验证，防止通过构造平台名枚举环境变量：

```python
_KNOWN_DELIVERY_PLATFORMS = frozenset({
    "telegram", "discord", "slack", "whatsapp", "signal",
    "matrix", "mattermost", "homeassistant", ...
})
```
