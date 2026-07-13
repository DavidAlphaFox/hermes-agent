# 网关系统设计

> 多平台消息路由、平台适配器架构、会话持久化、消息投递

---

## 目录

- [概述](#概述)
- [系统架构](#系统架构)
- [GatewayRunner](#gatewayrunner)
- [BasePlatformAdapter](#baseplatformadapter)
- [支持的平台](#支持的平台)
- [会话管理](#会话管理)
- [消息路由与投递](#消息路由与投递)
- [认证与授权](#认证与授权)
- [配置系统](#配置系统)
- [生命周期管理](#生命周期管理)
- [添加新平台](#添加新平台)

---

## 概述

网关系统是 Hermes Agent 的多平台消息接入层，允许用户通过 19 个不同的消息平台与 Agent 交互。核心设计理念：

- **单进程多平台**: 所有平台适配器在同一个 asyncio 事件循环中并发运行
- **统一会话模型**: 不同平台的会话使用统一的 SessionContext 抽象
- **安全隔离**: 消息平台默认启用执行审批 (HERMES_EXEC_ASK=1)
- **静默模式**: 网关运行时抑制调试输出 (HERMES_QUIET=1)

```
+------------------------------------------------------------------+
|                      GatewayRunner                                |
|                                                                  |
|  +----------+  +----------+  +-------+  +--------+  +--------+  |
|  | Telegram |  | Discord  |  | Slack |  | 飞书   |  | 企微   |  |
|  +----+-----+  +----+-----+  +---+---+  +---+----+  +---+----+  |
|       |              |            |          |            |       |
|       +------+-------+------+----+----+-----+------+-----+       |
|              |              |         |            |             |
|              v              v         v            v             |
|       +----------------------------------------------+           |
|       |          消息路由 (handle_message)             |           |
|       +------+-----------------------------------+---+           |
|              |                                   |               |
|              v                                   v               |
|       +--------------+                   +---------------+       |
|       | SessionStore |                   | AIAgent Pool  |       |
|       | (会话查找)    |                   | (Agent 创建)   |       |
|       +--------------+                   +---------------+       |
+------------------------------------------------------------------+
```

---

## 系统架构

### 文件组织

```
gateway/
|-- run.py              # GatewayRunner 主控 (341KB)
|-- config.py           # 配置加载与验证 (40KB)
|-- session.py          # 会话管理与上下文 (41KB)
|-- delivery.py         # 消息投递路由器 (10KB)
|-- hooks.py            # 内置钩子 (6KB)
|-- mirror.py           # 消息镜像 (4KB)
|-- pairing.py          # 设备配对 (11KB)
|-- status.py           # 状态查询 (12KB)
|-- stream_consumer.py  # 流式消费器 (11KB)
|-- channel_directory.py # 频道目录 (9KB)
|-- sticker_cache.py    # 贴纸缓存 (3KB)
|
+-- platforms/
    |-- base.py             # BasePlatformAdapter 抽象基类 (66KB)
    |-- telegram.py         # Telegram (113KB)
    |-- discord.py          # Discord (117KB)
    |-- slack.py            # Slack (53KB)
    |-- whatsapp.py         # WhatsApp (39KB)
    |-- signal.py           # Signal (32KB)
    |-- matrix.py           # Matrix (81KB)
    |-- feishu.py           # 飞书 (141KB)
    |-- wecom.py            # 企业微信 (52KB)
    |-- dingtalk.py         # 钉钉 (13KB)
    |-- email.py            # Email (23KB)
    |-- mattermost.py       # Mattermost (27KB)
    |-- homeassistant.py    # Home Assistant (16KB)
    |-- sms.py              # SMS (10KB)
    |-- webhook.py          # Webhook (23KB)
    |-- api_server.py       # HTTP API 服务器 (66KB)
    +-- ADDING_A_PLATFORM.md # 添加平台指南 (9KB)
```

---

## GatewayRunner

`GatewayRunner` 是网关的主控类，管理所有平台适配器的生命周期：

```python
class GatewayRunner:
    """
    网关主控制器。
    管理所有平台适配器的生命周期，路由消息到/从 Agent。
    """
    
    def __init__(self, config: GatewayConfig = None):
        self.config = config or load_gateway_config()
        self.adapters: Dict[Platform, BasePlatformAdapter] = {}
        
        # 从 config.yaml / 环境变量加载临时配置
        self._prefill_messages = self._load_prefill_messages()
        self._ephemeral_system_prompt = self._load_ephemeral_system_prompt()
        self._reasoning_config = self._load_reasoning_config()
```

### 启动流程

```
GatewayRunner.start()
    |
    |-- 1. 加载配置
    |       load_gateway_config() -> GatewayConfig
    |       包含各平台的 token、配置、授权用户列表
    |
    |-- 2. 解析运行时 Agent 参数
    |       _resolve_runtime_agent_kwargs()
    |       -> 确定 API 端点、密钥、模型
    |
    |-- 3. 初始化平台适配器
    |       for platform in config.enabled_platforms:
    |           adapter = create_adapter(platform, config)
    |           adapter.set_message_handler(self.handle_message)
    |           self.adapters[platform] = adapter
    |
    |-- 4. 启动定时任务调度器
    |       cron.scheduler 后台线程 (每 60 秒 tick)
    |
    |-- 5. 并发启动所有适配器
    |       await asyncio.gather(
    |           adapter1.start(),
    |           adapter2.start(),
    |           adapter3.start(),
    |           ...
    |       )
    |
    +-- 6. 运行直到收到停止信号
            SIGINT / SIGTERM -> graceful shutdown
```

### 消息处理

```
handle_message(event: MessageEvent, adapter: BasePlatformAdapter)
    |
    |-- 1. 认证检查
    |       检查发送者是否在授权列表中
    |
    |-- 2. 构建会话键
    |       session_key = build_session_key(platform, chat_id, user_id)
    |
    |-- 3. 检查是否有运行中的 Agent
    |       if session_key in _running_agents:
    |           -> 中断当前 Agent 或排队
    |
    |-- 4. 创建/恢复 AIAgent
    |       agent = AIAgent(
    |           session_id=session_id,
    |           platform=platform.value,
    |           user_id=user_id,
    |           enabled_toolsets=platform_toolsets,
    |           ...
    |       )
    |
    |-- 5. 执行对话
    |       response = agent.chat(message)
    |
    |-- 6. 发送响应
    |       await adapter.send_response(chat_id, response)
    |       支持: 文本分片、Markdown 格式、媒体附件
    |
    +-- 7. 清理
            从 _running_agents 中移除
```

---

## BasePlatformAdapter

所有平台适配器继承自 `BasePlatformAdapter` 抽象基类：

```python
class BasePlatformAdapter(ABC):
    """
    平台适配器基类。
    子类实现平台特定的逻辑：连接、认证、接收/发送消息、媒体处理。
    """
    
    def __init__(self, config: PlatformConfig, platform: Platform):
        self.config = config
        self.platform = platform
        self._message_handler = None
        self._running = False
        self._active_sessions: Dict[str, asyncio.Event] = {}
        self._pending_messages: Dict[str, MessageEvent] = {}
        self._background_tasks: set[asyncio.Task] = set()
```

### 必须实现的方法

| 方法 | 说明 |
|------|------|
| `start()` | 连接平台并开始监听消息 |
| `stop()` | 断开连接并清理资源 |
| `send_text(chat_id, text)` | 发送文本消息 |
| `send_media(chat_id, media)` | 发送媒体文件 |
| `get_display_name(user_id)` | 获取用户显示名 |

### MessageEvent 数据模型

```python
@dataclass
class MessageEvent:
    """跨平台消息事件的统一表示。"""
    platform: Platform
    chat_id: str
    user_id: str
    username: str
    text: str
    message_type: MessageType  # TEXT, IMAGE, VOICE, VIDEO, DOCUMENT, ...
    reply_to_message_id: str = None
    attachments: List[Any] = field(default_factory=list)
    metadata: Dict[str, Any] = field(default_factory=dict)
```

### MessageType 枚举

```python
class MessageType(Enum):
    TEXT = "text"
    IMAGE = "image"
    VOICE = "voice"
    VIDEO = "video"
    DOCUMENT = "document"
    STICKER = "sticker"
    LOCATION = "location"
    CONTACT = "contact"
```

---

## 支持的平台

| 平台 | 文件 | 大小 | 传输方式 | 特性 |
|------|------|------|----------|------|
| Telegram | `telegram.py` | 113KB | Long Polling / Webhook | 富文本, 内联键盘, 语音, 贴纸 |
| Discord | `discord.py` | 117KB | WebSocket | 嵌入, 线程, 反应, 斜杠命令 |
| Slack | `slack.py` | 53KB | Socket Mode / Events API | Block Kit, 线程, 文件共享 |
| WhatsApp | `whatsapp.py` | 39KB | WhatsApp Bridge | 端到端加密, 媒体, LID 映射 |
| Signal | `signal.py` | 32KB | Signal CLI / REST API | 端到端加密, 群组 |
| Matrix | `matrix.py` | 81KB | Matrix Client-Server API | 联邦, 端到端加密, 房间 |
| 飞书 | `feishu.py` | 141KB | 飞书开放平台 API | 卡片消息, 群聊, 机器人 |
| 企业微信 | `wecom.py` | 52KB | 企微回调 / 主动推送 | 应用消息, 群聊 |
| 钉钉 | `dingtalk.py` | 13KB | 钉钉 Stream API | 群聊机器人 |
| Email | `email.py` | 23KB | IMAP + SMTP | 邮件解析, HTML 响应 |
| Mattermost | `mattermost.py` | 27KB | WebSocket | 频道, 线程 |
| Home Assistant | `homeassistant.py` | 16KB | HA Conversation API | 语音助手集成 |
| SMS | `sms.py` | 10KB | Twilio / 自定义 | 短信收发 |
| Webhook | `webhook.py` | 23KB | HTTP POST / GET | 自定义集成 |
| HTTP API | `api_server.py` | 66KB | REST API | 通用 API 接口 |

---

## 会话管理

### SessionStore

`gateway/session.py` 提供会话查找与上下文构建：

```python
class SessionStore:
    """管理网关会话的查找与创建。"""
    
    def find_session(self, platform, chat_id, user_id) -> Optional[str]:
        """查找现有会话。"""
    
    def create_session(self, platform, chat_id, user_id, model) -> str:
        """创建新会话。"""
```

### SessionContext

```python
@dataclass
class SessionContext:
    """会话上下文 -- 注入到 Agent 系统提示词中。"""
    platform: str           # "telegram", "discord", ...
    chat_id: str            # 平台特定的聊天 ID
    user_id: str            # 用户标识
    username: str           # 用户显示名
    is_group: bool          # 是否群聊
    group_name: str = ""    # 群聊名称
    reply_context: str = "" # 引用消息内容
```

### 会话键构建

```python
def build_session_key(platform: str, chat_id: str, user_id: str = "") -> str:
    """构建会话唯一键。
    
    DM (私聊): "{platform}:{user_id}"
    群聊:     "{platform}:{chat_id}"
    """
```

### 会话源标记

每个会话记录其来源平台：

```python
class SessionSource(Enum):
    CLI = "cli"
    TELEGRAM = "telegram"
    DISCORD = "discord"
    SLACK = "slack"
    WHATSAPP = "whatsapp"
    SIGNAL = "signal"
    MATRIX = "matrix"
    FEISHU = "feishu"
    WECOM = "wecom"
    EMAIL = "email"
    ACP = "acp"
    CRON = "cron"
    API = "api"
    ...
```

---

## 消息路由与投递

### DeliveryRouter

`gateway/delivery.py` 实现跨平台消息投递：

```python
class DeliveryRouter:
    """将消息投递到目标平台/聊天。
    
    主要用于:
    - 定时任务结果投递 (cron -> telegram/discord)
    - 跨平台消息发送 (send_message 工具)
    - 消息镜像
    """
```

### 投递流程

```
DeliveryRouter.deliver(target, message)
    |
    |-- 1. 解析目标
    |       target = "telegram:123456" 或 "discord:channel_id"
    |
    |-- 2. 查找适配器
    |       adapter = gateway.adapters[platform]
    |
    |-- 3. 格式化消息
    |       - 平台特定的格式转换
    |       - Markdown -> 平台原生格式
    |       - 长消息分片
    |
    +-- 4. 发送
            await adapter.send_text(chat_id, formatted_message)
```

### 消息分片

长响应在发送前会被自动分片，以适应各平台的消息长度限制：

| 平台 | 最大长度 | 分片策略 |
|------|----------|----------|
| Telegram | 4096 字符 | 按段落/代码块边界分片 |
| Discord | 2000 字符 | 按段落边界分片 |
| Slack | 40000 字符 | 按 Block 分片 |
| WhatsApp | 65536 字符 | 按段落分片 |
| SMS | 160/1600 字符 | 按字符分片 |

---

## 认证与授权

### 用户白名单

网关通过配置文件中的用户白名单控制访问：

```yaml
# config.yaml
gateway:
  telegram:
    token: "BOT_TOKEN"
    authorized_users:
      - "123456789"
      - "987654321"
  discord:
    token: "BOT_TOKEN"
    authorized_users:
      - "user_id_1"
```

### WhatsApp LID 解析

WhatsApp 使用 LID (Link ID) 系统，同一用户可能有多个标识：

```python
def _expand_whatsapp_auth_aliases(identifier: str) -> set:
    """解析 WhatsApp 电话号码/LID 别名。
    
    通过 session 目录中的 lid-mapping-*.json 文件
    递归解析所有关联的标识符。
    """
```

---

## 配置系统

网关配置从 `~/.hermes/config.yaml` 加载：

```yaml
gateway:
  # 全局设置
  model: "anthropic/claude-sonnet-4-20250514"
  system_prompt: "You are a helpful assistant."
  
  # 平台配置
  telegram:
    enabled: true
    token: "${TELEGRAM_BOT_TOKEN}"
    authorized_users: ["123456"]
  
  discord:
    enabled: true
    token: "${DISCORD_BOT_TOKEN}"
    authorized_users: ["user_id"]
  
  feishu:
    enabled: true
    app_id: "${FEISHU_APP_ID}"
    app_secret: "${FEISHU_APP_SECRET}"
  
  # 工具配置
  toolsets:
    enabled: ["web", "terminal", "file"]
    disabled: ["browser"]
```

### 环境变量展开

配置值支持 `${ENV_VAR}` 语法引用环境变量，在加载时自动展开。

---

## 生命周期管理

### 服务模式

网关支持作为系统服务运行：

```bash
hermes gateway start      # 启动后台服务
hermes gateway stop       # 停止服务
hermes gateway status     # 查看状态
hermes gateway install    # 安装 systemd 服务
hermes gateway uninstall  # 卸载服务
hermes gateway --replace  # 替换运行中的实例
```

### 优雅关闭

```
SIGINT/SIGTERM
    |
    |-- 1. 设置 _running = False
    |
    |-- 2. 取消所有后台消息处理任务
    |       for task in adapter._background_tasks:
    |           task.cancel()
    |
    |-- 3. 停止所有适配器
    |       for adapter in self.adapters.values():
    |           await adapter.stop()
    |
    |-- 4. 等待运行中的 Agent 完成
    |
    +-- 5. 清理资源
```

### 并发控制

网关使用哨兵值防止同一会话的并发处理：

```python
_AGENT_PENDING_SENTINEL = object()

# 消息到达时立即占位
_running_agents[session_key] = _AGENT_PENDING_SENTINEL

# 防止第二条消息在异步间隙中绕过检查
if _running_agents.get(session_key) is not None:
    # 中断当前 Agent 或排队
```

---

## 添加新平台

参考 `gateway/platforms/ADDING_A_PLATFORM.md`，添加新平台的步骤：

1. 创建 `gateway/platforms/my_platform.py`
2. 继承 `BasePlatformAdapter`
3. 实现必要的抽象方法
4. 在 `gateway/config.py` 中注册平台枚举值
5. 在 `GatewayRunner` 的适配器工厂中添加创建逻辑

```python
# gateway/platforms/my_platform.py
from gateway.platforms.base import BasePlatformAdapter, MessageEvent

class MyPlatformAdapter(BasePlatformAdapter):
    async def start(self):
        """连接到平台并开始监听。"""
        ...
    
    async def stop(self):
        """断开连接。"""
        ...
    
    async def send_text(self, chat_id: str, text: str):
        """发送文本消息。"""
        ...
```

---

## 相关文档

- [CLI 系统](cli-system.md) -- 网关管理命令
- [定时任务系统](cron-system.md) -- 定时任务结果投递
- [核心 Agent 循环](core-agent-loop.md) -- 网关创建的 Agent 实例
- [数据流图](data-flow.md) -- 消息路由数据流
