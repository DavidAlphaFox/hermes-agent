# CLI 系统设计

> 命令架构、交互式 TUI、配置系统、认证体系、子命令分发

---

## 目录

- [概述](#概述)
- [入口与分发](#入口与分发)
- [子命令架构](#子命令架构)
- [交互式 TUI](#交互式-tui)
- [配置系统](#配置系统)
- [环境变量加载](#环境变量加载)
- [认证体系](#认证体系)
- [会话管理](#会话管理)
- [Profile 多实例隔离](#profile-多实例隔离)

---

## 概述

Hermes Agent 的 CLI 系统由两个主要模块组成：

| 模块 | 文件 | 职责 |
|------|------|------|
| 命令入口 | `hermes_cli/main.py` (~220KB) | argparse 定义、子命令分发、会话恢复 |
| 交互式 TUI | `cli.py` (~376KB) | 基于 prompt_toolkit 的富交互终端界面 |
| 配置管理 | `hermes_cli/config.py` (~104KB) | YAML 配置读写、默认值、config 子命令 |
| 环境变量 | `hermes_cli/env_loader.py` | .env 加载、编码回退、优先级控制 |
| 认证系统 | `hermes_cli/auth.py` (~110KB) | OAuth 设备码流、API key 管理、多供应商 |

```
hermes (命令)
  │
  ├─ hermes_cli/main.py     ← argparse 入口、子命令路由
  │    └─ _apply_profile_override()  ← 最先执行，设置 HERMES_HOME
  │    └─ load_hermes_dotenv()       ← 加载 .env
  │    └─ main()                      ← argparse → func 分发
  │
  └─ cli.py                  ← 交互式 TUI (prompt_toolkit)
       └─ cmd_chat() 调用
```

---

## 入口与分发

### main.py 启动流程

`hermes_cli/main.py` 是整个 CLI 的入口点。启动时按严格顺序执行：

```python
# 1. Profile 覆盖 — 在任何模块导入前执行
_apply_profile_override()
#    解析 --profile/-p 参数 → 设置 HERMES_HOME 环境变量
#    回退: ~/.hermes/active_profile 文件（粘性默认配置）

# 2. 加载环境变量
load_hermes_dotenv(project_env=PROJECT_ROOT / '.env')
#    ~/.hermes/.env (override=True) → 项目 .env (补充)

# 3. 初始化日志
setup_logging(mode="cli")
#    agent.log + errors.log 集中式文件日志

# 4. argparse 解析 → 子命令分发
main()
```

### argparse 分发架构

```python
parser = argparse.ArgumentParser(prog="hermes")

# 顶级参数（所有子命令共享）
parser.add_argument("--version", "-V")
parser.add_argument("--resume", "-r")        # 恢复会话
parser.add_argument("--continue", "-c")      # 继续最近会话
parser.add_argument("--worktree", "-w")      # Git worktree 隔离
parser.add_argument("--skills", "-s")        # 预加载技能
parser.add_argument("--yolo")                # 跳过审批

subparsers = parser.add_subparsers(dest="command")
# chat, model, gateway, setup, config, cron, skills, ...
```

当无子命令时默认执行 `cmd_chat()`，即进入交互模式。

### 供应商自动检测

`_has_any_provider_configured()` 在启动时检测是否有可用的推理供应商：

```
检查链:
  1. 环境变量 (OPENROUTER_API_KEY, OPENAI_API_KEY, ANTHROPIC_API_KEY 等)
  2. ~/.hermes/.env 文件中的密钥
  3. 供应商特定认证回退 (如 Copilot 的 gh auth)
  4. Nous Portal OAuth 凭证 (~/.hermes/auth.json)
  5. config.yaml 中的 model.provider / model.base_url
  6. Claude Code OAuth 凭证 (~/.claude/.credentials.json)
```

若无供应商可用，自动引导用户进入 `hermes setup`。

---

## 子命令架构

| 子命令 | 处理函数 | 说明 |
|--------|----------|------|
| `chat` | `cmd_chat()` | 交互式对话（默认命令） |
| `model` | `cmd_model()` | 选择供应商和模型 |
| `gateway` | `cmd_gateway()` | 网关管理（run/start/stop/install/status） |
| `setup` | `cmd_setup()` | 交互式设置向导 |
| `config` | `cmd_config()` | 配置查看/编辑/设置 |
| `cron` | `cmd_cron()` | 定时任务管理 |
| `skills` | `cmd_skills()` | 技能浏览/安装/创建 |
| `tools` | `cmd_tools()` | 工具配置 |
| `sessions` | `cmd_sessions()` | 会话列表/浏览/重命名 |
| `doctor` | `cmd_doctor()` | 诊断检查 |
| `logs` | `cmd_logs()` | 日志查看 |
| `update` | `cmd_update()` | 自动更新 |
| `uninstall` | `cmd_uninstall()` | 卸载 |
| `acp` | (入口跳转) | 启动 ACP 编辑器服务器 |
| `auth` | `cmd_auth()` | 凭证池管理 |

### gateway 子命令嵌套

```
hermes gateway
  ├─ run         # 前台运行
  ├─ start       # 启动后台服务
  ├─ stop        # 停止服务 (--all 停止所有 profile)
  ├─ restart     # 重启服务
  ├─ status      # 状态查看 (--deep 深度检查)
  ├─ install     # 安装为系统服务 (--system 系统级)
  ├─ uninstall   # 卸载服务
  └─ setup       # 配置消息平台
```

---

## 交互式 TUI

### prompt_toolkit 架构

`cli.py` 基于 `prompt_toolkit` 构建了一个完整的终端用户界面：

```
┌──────────────────────────────────────────────────┐
│  Banner (ASCII 艺术字)                            │
│  Model: anthropic/claude-sonnet-4 @ openrouter   │
├──────────────────────────────────────────────────┤
│                                                  │
│  对话历史区域 (滚动)                               │
│  ├─ 用户消息                                      │
│  ├─ 工具调用进度                                   │
│  └─ 助手回复 (Markdown 渲染)                       │
│                                                  │
├──────────────────────────────────────────────────┤
│  > 输入区域 (TextArea + 自动补全)                  │
│    Ctrl+J 换行 | Enter 发送 | /help 命令          │
└──────────────────────────────────────────────────┘
```

### 核心组件

| 组件 | 功能 |
|------|------|
| `TextArea` | 多行输入框，支持 `FileHistory` 持久化 |
| `FormattedTextControl` | 状态栏、模型信息显示 |
| `CompletionsMenu` | 斜杠命令自动补全 |
| `KeyBindings` | Enter 发送、Ctrl+J 换行、Ctrl+C 中断 |
| `patch_stdout` | 保护 TUI 输出不被工具 stdout 干扰 |

### 斜杠命令

TUI 内置的斜杠命令（不经过 LLM）：

```
/help           显示可用命令
/model          切换模型
/tools          列出工具
/context        查看上下文统计
/reset          清空对话历史
/compact        压缩上下文
/undo           撤销上一条消息
/rollback       回滚文件系统变更（需 --checkpoints）
/session        查看当前会话信息
/quit           退出
```

### 工具执行反馈

TUI 通过回调实时展示工具执行状态：

```python
# 工具进度回调
tool_progress_callback(event_type, name, preview, args)
#   event_type: "tool.started" → 显示工具名和参数预览
#   event_type: "tool.completed" → 更新结果状态

# 思考回调
thinking_callback(text)
#   实时流式显示推理过程

# 消息回调
message_callback(text)
#   增量渲染助手回复文本
```

---

## 配置系统

### 配置文件布局

```
~/.hermes/                   # HERMES_HOME
  ├─ config.yaml             # 主配置文件
  ├─ .env                    # API 密钥和敏感配置
  ├─ auth.json               # OAuth 凭证存储
  ├─ SOUL.md                 # 自定义系统人格
  └─ profiles/               # 多 Profile 实例
       └─ work/
            ├─ config.yaml
            ├─ .env
            └─ auth.json
```

### config.yaml 结构

```yaml
model:
  default: "anthropic/claude-sonnet-4"    # 默认模型
  provider: "openrouter"                  # 推理供应商
  base_url: null                          # 自定义端点

agent:
  max_turns: 90                           # 最大工具调用迭代
  reasoning_effort: "medium"              # 推理力度
  
terminal:
  backend: "local"                        # local/ssh/docker/modal/daytona
  allowed_commands: ["git", "npm", ...]   # 白名单命令

memory:
  provider: "honcho"                      # 记忆插件

toolsets:
  enabled: ["core", "web", "terminal"]    # 启用的工具集
  disabled: []                            # 禁用的工具集

smart_model_routing:                       # 智能模型路由
  rules: [...]
```

### 配置操作

```python
# 加载配置（带缓存）
from hermes_cli.config import load_config
config = load_config()

# 设置配置值
# hermes config set model.default "gpt-4o"
# hermes config set agent.max_turns 120
```

### 受管模式

当 `HERMES_MANAGED=true` 或存在 `~/.hermes/.managed` 文件时，部分配置操作被锁定（NixOS/Homebrew 声明式管理）。

---

## 环境变量加载

### env_loader.py

`load_hermes_dotenv()` 统一了所有入口点的 .env 加载行为：

```
加载顺序:
  1. ~/.hermes/.env   (override=True, 覆盖已有环境变量)
  2. 项目 .env        (仅在用户 .env 存在时为补充模式)

编码回退:
  UTF-8 → Latin-1 (兼容所有单字节字符)
```

### 关键环境变量

| 变量 | 说明 |
|------|------|
| `HERMES_HOME` | 主目录路径（默认 ~/.hermes） |
| `HERMES_MODEL` | 模型覆盖 |
| `HERMES_MANAGED` | 受管模式标记 |
| `OPENROUTER_API_KEY` | OpenRouter API 密钥 |
| `OPENAI_API_KEY` | OpenAI API 密钥 |
| `ANTHROPIC_API_KEY` | Anthropic API 密钥 |
| `OPENAI_BASE_URL` | 自定义 API 端点 |
| `HERMES_REASONING_EFFORT` | 推理力度覆盖 |

---

## 认证体系

### 多供应商架构

`hermes_cli/auth.py` 实现了统一的多供应商认证系统：

```
                    ┌───────────────────────┐
                    │   ProviderRegistry    │
                    │  (供应商配置注册表)     │
                    └───────────┬───────────┘
                                │
            ┌───────────┬───────┴───────┬──────────┐
            │           │               │          │
      ┌─────v─────┐ ┌──v──┐  ┌────────v────┐ ┌───v───┐
      │  Nous     │ │ OR  │  │ Copilot/    │ │ Gemini│
      │  Portal   │ │     │  │ Claude Code │ │       │
      │ (OAuth)   │ │(Key)│  │ (OAuth)     │ │ (Key) │
      └───────────┘ └─────┘  └─────────────┘ └───────┘
```

### 支持的供应商

| 供应商 | 认证类型 | 说明 |
|--------|----------|------|
| Nous Portal | OAuth 设备码流 | 默认推理服务商 |
| OpenRouter | API Key | 模型路由聚合 |
| OpenAI | API Key | 直接 API 访问 |
| Anthropic | API Key | Claude 系列 |
| Copilot | OAuth (gh auth) | GitHub Copilot 凭证 |
| Claude Code | OAuth | ~/.claude/.credentials.json |
| Gemini | API Key | Google AI |
| HuggingFace | API Key | HF Inference |
| 自定义端点 | API Key + base_url | vLLM、llama.cpp 等 |

### 凭证存储

```json
// ~/.hermes/auth.json
{
  "version": 1,
  "active_provider": "nous",
  "providers": {
    "nous": {
      "access_token": "...",
      "refresh_token": "...",
      "expires_at": "2026-04-17T12:00:00Z",
      "agent_key": "...",
      "agent_key_expires_at": "..."
    }
  }
}
```

凭证文件使用跨进程文件锁保护（fcntl/msvcrt），权限设为 0600。

### 凭证解析优先级

`resolve_runtime_provider()` 按以下优先级解析推理凭证：

```
1. 显式指定的 --provider 参数
2. config.yaml 中的 model.provider
3. 环境变量 (OPENROUTER_API_KEY 等)
4. ~/.hermes/auth.json 中的活跃供应商
5. 外部凭证回退 (gh auth, Claude Code OAuth)
```

---

## 会话管理

### 会话恢复

CLI 支持多种会话恢复方式：

```bash
hermes -c                    # 继续最近一次 CLI 会话
hermes -c "my project"       # 按名称查找会话（自动跟踪 lineage）
hermes --resume <session_id> # 按 ID 精确恢复
hermes sessions browse       # curses 交互式浏览器
```

### 会话浏览器

`_session_browse_picker()` 提供基于 curses 的交互式会话选择器：

- 上下箭头导航
- 实时打字过滤
- 显示标题、预览、来源、最后活跃时间
- Esc 退出或清除搜索
- 回退到编号列表（Windows 兼容）

---

## Profile 多实例隔离

### 工作原理

Profile 系统允许运行多个完全隔离的 Hermes 实例：

```bash
hermes --profile work        # 使用 work 配置
hermes -p personal           # 使用 personal 配置

# 每个 Profile 拥有独立的:
# ~/.hermes/profiles/<name>/config.yaml
# ~/.hermes/profiles/<name>/.env
# ~/.hermes/profiles/<name>/auth.json
# ~/.hermes/profiles/<name>/state.db
```

### 实现机制

1. `_apply_profile_override()` 在模块导入前解析 `--profile` 参数
2. 设置 `HERMES_HOME` 环境变量到 Profile 目录
3. 后续所有 `get_hermes_home()` 调用自动指向正确目录
4. `~/.hermes/active_profile` 文件提供粘性默认值

```
~/.hermes/
  ├─ active_profile          # 内容: "work" → 默认使用 work 配置
  └─ profiles/
       ├─ work/              # HERMES_HOME=~/.hermes/profiles/work
       └─ personal/          # HERMES_HOME=~/.hermes/profiles/personal
```
