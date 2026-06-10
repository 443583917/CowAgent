# Channel 通道系统 — 模块深度分析

> **所属项目**: CowAgent v2.1.1  
> **分析日期**: 2026-06-10  
> **模块路径**: `channel/`  
> **依赖模块**: `bridge/`, `common/`, `config.py`

---

## 1. 模块概述

Channel（通道系统）是 CowAgent 的**消息入口和出口层**，负责对接 12 种 IM 平台。所有通道实现统一的 `Channel` 接口，在 `ChannelManager` 的管理下并行运行。新增通道只需实现三个方法即可接入整个 Agent Harness。

**核心职责**:
- 消息接收：监听各自平台的消息（HTTP/WebSocket/长轮询/SDK回调）
- 消息路由：将用户消息转换为标准 `Context`，调用 `Bridge` 处理
- 回复发送：将 `Reply` 转换回各平台的原生消息格式
- 生命周期管理：`startup()` / `stop()` 支持优雅启停

---

## 2. 架构设计

### 2.1 Channel 接口

```python
class Channel:
    channel_type: str   # 通道类型标识

    def startup(self):
        """启动通道，开始监听消息（阻塞式）"""
        raise NotImplementedError

    def send(self, reply: Reply, context: Context):
        """发送回复到用户"""
        raise NotImplementedError

    def stop(self):
        """优雅关闭（可选）"""
        pass
```

### 2.2 通道工厂

```python
# channel/channel_factory.py
def create_channel(channel_type) -> Channel:
    if channel_type == "terminal":    return TerminalChannel()
    elif channel_type == "web":       return WebChannel()
    elif channel_type == "wechatmp":  return WechatMPChannel()
    elif channel_type == "feishu":    return FeiShuChanel()
    # ... 12 种通道
    else: raise RuntimeError
```

### 2.3 ChannelManager 多通道管理

```python
class ChannelManager:
    def start(channel_names, first_start):
        """为每个通道类型创建实例，各跑在独立 daemon 线程"""
        for name in channel_names:
            ch = channel_factory.create_channel(name)
            Thread(target=ch.startup, daemon=True).start()

    def stop(channel_name=None):
        """停止指定通道或全部通道"""

    def add_channel(name):
        """动态添加新通道（运行时热加载）"""

    def remove_channel(name):
        """动态移除运行中的通道"""
```

---

## 3. 通道详解

### 3.1 Web Console (`channel/web/`) — 默认通道

- **传输**: HTTP + WebSocket + SSE
- **认证**: 可配置 `web_password`
- **功能**: 完整的管理控制台（模型配置、通道管理、技能安装、记忆浏览、知识图谱）
- **并行会话**: 支持多标签页独立会话
- **流式输出**: SSE 实时推送 Agent 推理过程

### 3.2 即时通讯通道

| 通道 | SDK/协议 | 特有功能 |
|------|---------|---------|
| 个人微信 | itchat | — |
| 微信公众号 | HTTP API | 被动回复/主动推送 |
| 企业微信应用 | 企微 SDK | — |
| 企业微信机器人 | Webhook | 群聊支持、二维码接入 |
| 微信客服 | 微信客服 API | — |
| 飞书/Lark | 飞书 SDK | 语音、流式、二维码接入 |
| 钉钉 | 钉钉 SDK | 群聊、语音 |
| QQ | QQ Bot SDK | 群聊 |
| Telegram | python-telegram-bot | 群聊、文件、语音 |
| Slack | Slack SDK | 群聊 |
| Discord | Discord.py | 群聊 |

---

## 4. 数据流

```
[IM平台] → 消息到达
  → Channel.handle_message(msg)
    → Context(msg_type, content, session_id, receiver, ...)
    → Bridge.fetch_agent_reply(query, context, on_event)
      → Agent 处理...
    → Reply(type, content)
  → Channel.send(reply, context)
    → [IM平台] → 用户收到回复
```

### 4.1 Web Console 特殊处理

Web Console 使用**请求-响应模型**而非长连接：
- `request_id` 映射到 SSE 事件队列
- 用户发送消息 → 生成 `request_id` → Agent 执行 → SSE 推送到该 request_id 的队列
- 前端通过 EventSource 接收事件流

---

## 5. 配置与扩展点

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `channel_type` | `"web"` | 通道类型，支持逗号分隔多通道 |
| `web_host` | `"127.0.0.1"` | Web 监听地址 |
| `web_port` | `9899` | Web 端口 |
| `web_password` | `""` | Web 密码 |
| `single_chat_prefix` | `["bot", "@bot"]` | 单聊触发前缀 |
| `group_chat_prefix` | `["@bot"]` | 群聊触发前缀 |

### 扩展点

| 扩展点 | 实现方式 |
|--------|---------|
| 新 IM 通道 | 1. 创建 `channel/<name>/` 2. 实现 `Channel` 接口 3. 在 `channel_factory.py` 注册 4. 在 `app.py` 的 `_clear_singleton_cache` 注册 |
| 消息中间件 | 通过 `Plugin` 系统的 `handle_message` 钩子 |
| 自定义回复格式 | 覆写 `Channel.send()` 方法 |

---

## 6. 二次开发规范

### 6.1 新通道开发模板

```python
# channel/my_platform/my_platform_channel.py
from channel.channel import Channel
from bridge.context import Context, ContextType
from bridge.reply import Reply, ReplyType

class MyPlatformChannel(Channel):
    def startup(self):
        # 初始化 SDK、注册回调
        # 收到消息时调用:
        context = Context(ContextType.TEXT, msg_content)
        context["session_id"] = user_id
        context["receiver"] = user_id
        
        reply = self._bridge.fetch_agent_reply(msg_content, context)
        self.send(reply, context)

    def send(self, reply, context):
        if reply.type == ReplyType.TEXT:
            platform_sdk.send_text(context["receiver"], reply.content)
        elif reply.type == ReplyType.IMAGE:
            platform_sdk.send_image(context["receiver"], reply.content)
        # ...
```

### 6.2 注意事项

1. **startup() 是阻塞的**: 它在独立 daemon 线程中运行
2. **session_id 是必需的**: 用于 Agent 的会话隔离
3. **群聊需要区分**: `context["isgroup"] = True` + `context["msg"]` 引用原始消息对象
4. **Web 通道需要 request_to_session 映射**: 后台推送时需要合成 request_id

---

> **相关模块文档**:
> - [bridge.md](bridge.md) — Bridge（通道的下游调用者）
> - [app-config-common.md](app-config-common.md) — App入口（ChannelManager 的位置）
