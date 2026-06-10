# Bridge 桥接层 — 模块深度分析

> **所属项目**: CowAgent v2.1.1  
> **分析日期**: 2026-06-10  
> **模块路径**: `bridge/`  
> **依赖模块**: `agent/`, `models/`, `voice/`, `translate/`, `channel/`, `config.py`

---

## 1. 模块概述

Bridge（桥接层）是 CowAgent 的**中枢神经系统**。它连接三个独立层次——Channel（消息入口）、Agent Core（智能处理）、Models（LLM 适配）——负责消息路由、Agent 初始化编排、事件转换和模型选择。

**核心职责**:
- **Bridge**: 单例路由中心，管理 chat/voice/translate bot 实例
- **AgentBridge**: Agent 系统与 Channel 基础设施的适配层
- **AgentInitializer**: 编排 Agent 创建的 8 个步骤
- **AgentEventHandler**: 将 Agent 流式事件转换为 Channel Reply
- **Context/Reply**: 消息上下文的标准化数据模型

---

## 2. 架构设计

```
┌─────────────────────────────────────────────────────────────┐
│                      Channel 层                              │
│         Web | WeChat | Feishu | DingTalk | ...               │
└────────────────────────┬────────────────────────────────────┘
                         │ query + context
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                      Bridge 层                                │
│                                                              │
│  Bridge (bridge.py) — 单例                                   │
│  ├─ get_bot("chat")        → LLM Bot                        │
│  ├─ get_bot("voice_to_text") → ASR Bot                      │
│  ├─ get_bot("text_to_voice") → TTS Bot                      │
│  ├─ get_bot("translate")   → Translator Bot                 │
│  └─ get_agent_bridge()     → AgentBridge (Agent模式)        │
│                                                              │
│  AgentBridge (agent_bridge.py)                               │
│  ├─ agent_reply(query, context, on_event)                   │
│  │   ├─ AgentInitializer.initialize_agent(session_id)       │
│  │   │   ├─ 1. 工作空间初始化                                │
│  │   │   ├─ 2. API密钥迁移 (.env)                           │
│  │   │   ├─ 3. 记忆系统初始化                                │
│  │   │   ├─ 4. 工具加载                                      │
│  │   │   ├─ 5. 技能加载                                      │
│  │   │   ├─ 6. 调度器初始化                                  │
│  │   │   └─ 7. 系统提示词构建                                │
│  │   ├─ agent.run_stream(user_msg, on_event, cancel_event)  │
│  │   └─ AgentEventHandler → Reply                           │
│  └─ create_agent(...) — 为进化/调度创建隔离Agent             │
│                                                              │
│  AgentEventHandler (agent_event_handler.py)                  │
│  ├─ 文本流 → ReplyType.TEXT                                 │
│  ├─ 图片生成 → ReplyType.IMAGE                              │
│  ├─ 文件发送 → ReplyType.FILE                               │
│  └─ 错误 → ReplyType.ERROR                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. 源码深度解析

### 3.1 Bridge — 模型路由

```python
@singleton
class Bridge:
    def __init__(self):
        # 根据 model 名称自动推断 bot_type:
        # claude-* → CLAUDEAPI
        # gpt-* / o1-* → OPENAI
        # gemini-* → GEMINI
        # deepseek-* → DEEPSEEK
        # glm-* → ZHIPU_AI
        # ... 等等
        
    def get_agent_bridge(self):
        if self._agent_bridge is None:
            from bridge.agent_bridge import AgentBridge
            self._agent_bridge = AgentBridge(self)
        return self._agent_bridge
```

### 3.2 AgentBridge — Agent 适配

```python
class AgentBridge:
    def agent_reply(self, query, context, on_event, clear_history):
        # 1. 获取或创建 session agent
        agent = self.agents.get(session_id) or self.default_agent
        
        # 2. 如果 agent 不存在或配置变化 → 重新初始化
        if agent is None or self._config_changed():
            agent = self.initializer.initialize_agent(session_id)
            self.agents[session_id] = agent
        
        # 3. 记录进化信号
        note_user_turn(agent, channel_type, receiver)
        
        # 4. 执行 Agent
        cancel_event = CancelTokenRegistry().register(request_id)
        response = agent.run_stream(query, on_event=handler, cancel_event=cancel_event)
        
        # 5. 返回最终 Reply
        return handler.build_final_reply()
```

### 3.3 AgentInitializer — 初始化编排

初始化顺序（不可调换，有依赖关系）:
1. 迁移 API 密钥到 `.env` 文件
2. 创建/验证工作空间目录结构
3. 初始化记忆系统（`MemoryManager`）
4. 加载工具（`ToolManager` + MCP）
5. 初始化调度器（`SchedulerService`）
6. 加载技能（`SkillManager`）
7. 构建系统提示词（`PromptBuilder`）
8. 创建 Agent 实例

---

## 4. 数据流

```
Channel.handle_message(msg)
  → Bridge.get_agent_bridge()
    → AgentBridge.agent_reply(query, context, on_event)
      → AgentInitializer.initialize_agent(session_id)
      → agent.run_stream(query)
        → AgentStreamExecutor.run()
        → AgentEventHandler 处理事件流
      → return Reply
    → Channel.send(reply)
```

---

## 5. 配置与扩展点

| 配置项 | 说明 |
|--------|------|
| `bot_type` | 强制指定 LLM 提供商（留空自动推断） |
| `model` | 模型名称（用于自动推断 bot_type） |
| `agent_workspace` | 工作空间路径 |

### 扩展点

| 扩展点 | 说明 |
|--------|------|
| 新模型路由规则 | `Bridge.__init__()` 中添加 `model.startswith("xxx")` |
| 新 bot 类型 | `Bridge.btype` 字典 + `bot_factory.create_bot()` |
| 新事件类型 | `AgentEventHandler` 中添加事件处理分支 |

---

## 6. 二次开发规范

### 注意事项

1. **Bridge 是单例**: 全局唯一，通过 `@singleton` 装饰器保证
2. **初始化顺序不可变**: AgentInitializer 的 8 个步骤有先后依赖
3. **Agent 缓存**: session_id → Agent 的映射缓存在 `AgentBridge.agents`，配置变更时需重建
4. **线程安全**: 多 Channel 并行时，同一 session_id 的 Agent 可能被多个线程访问（但同一时刻只有一个请求在运行，因为有 `concurrency_in_session=1` 的约束）

---

> **相关模块文档**:
> - [agent-protocol.md](agent-protocol.md) — Agent Core（bridge 的主要下游）
> - [channel.md](channel.md) — Channel（bridge 的上游调用者）
> - [models.md](models.md) — Models（bot 工厂）
> - [agent-evolution.md](agent-evolution.md) — Evolution（通过 bridge 创建隔离 Agent）
