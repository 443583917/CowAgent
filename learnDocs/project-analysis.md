# 🐮 CowAgent 项目全面分析报告

> **生成日期**: 2026-06-10  
> **分析版本**: v2.1.1  
> **仓库地址**: [github.com/zhayujie/CowAgent](https://github.com/zhayujie/CowAgent)

---

## 目录

- [模块深度文档索引](#模块深度文档索引)
- [一、项目概述](#一项目概述)
- [二、架构设计](#二架构设计)
  - [2.1 整体架构（Agent Harness 模式）](#21-整体架构agent-harness-模式)
  - [2.2 核心运行流程](#22-核心运行流程)
  - [2.3 架构图](#23-架构图)
- [三、目录结构详解](#三目录结构详解)
  - [3.1 项目根目录](#31-项目根目录)
  - [3.2 agent/ — 智能体核心](#32-agent--智能体核心)
  - [3.3 bridge/ — 桥接层](#33-bridge--桥接层)
  - [3.4 channel/ — 通道层](#34-channel--通道层)
  - [3.5 models/ — 模型适配层](#35-models--模型适配层)
  - [3.6 其他重要目录](#36-其他重要目录)
- [四、核心功能模块深度解析](#四核心功能模块深度解析)
  - [4.1 Agent Core（智能体核心）](#41-agent-core智能体核心)
  - [4.2 Bridge 桥接层](#42-bridge-桥接层)
  - [4.3 Memory 三层记忆](#43-memory-三层记忆)
  - [4.4 Knowledge 知识库](#44-knowledge-知识库)
  - [4.5 Evolution 自我进化](#45-evolution-自我进化)
  - [4.6 Skills 技能系统](#46-skills-技能系统)
  - [4.7 Tools 工具系统](#47-tools-工具系统)
  - [4.8 Channels 通道系统](#48-channels-通道系统)
  - [4.9 Models 模型层](#49-models-模型层)
  - [4.10 CLI 命令行](#410-cli-命令行)
- [五、开发规范](#五开发规范)
  - [5.1 代码风格](#51-代码风格)
  - [5.2 架构设计模式](#52-架构设计模式)
  - [5.3 目录约定](#53-目录约定)
  - [5.4 扩展点汇总](#54-扩展点汇总)
- [六、项目功能亮点](#六项目功能亮点)
- [七、二次开发指南](#七二次开发指南)
  - [7.1 开发环境搭建](#71-开发环境搭建)
  - [7.2 源码阅读路径建议](#72-源码阅读路径建议)
  - [7.3 常见二次开发场景](#73-常见二次开发场景)
    - [场景 1: 添加新的 LLM 提供商](#场景-1添加新的-llm-提供商)
    - [场景 2: 添加新的 IM 通道](#场景-2添加新的-im-通道)
    - [场景 3: 开发自定义工具](#场景-3开发自定义工具)
    - [场景 4: 创建自定义技能](#场景-4创建自定义技能)
    - [场景 5: 开发插件](#场景-5开发插件)
    - [场景 6: 修改系统提示词 / Agent 人设](#场景-6修改系统提示词--agent-人设)
    - [场景 7: 集成新的语音/翻译服务](#场景-7集成新的语音翻译服务)
  - [7.4 配置关键路径](#74-配置关键路径)
  - [7.5 工作空间结构](#75-工作空间结构)
- [八、技术栈总览](#八技术栈总览)
- [九、版本历史摘要](#九版本历史摘要)

---

## 模块深度文档索引

以下是为每个核心模块编写的独立深度分析文档。点击链接查看详细的架构设计、源码解析、数据流和二次开发规范：

| 模块 | 文档 | 核心内容 |
|------|------|---------|
| Agent Core 执行引擎 | [agent-protocol.md](modules/agent-protocol.md) | Agent类、AgentStreamExecutor多轮循环、取消机制、上下文管理、消息完整性 |
| Memory 三层记忆 | [agent-memory.md](modules/agent-memory.md) | 三层架构、SQLite+FTS5、混合检索、Deep Dream蒸馏、EmbeddingProvider |
| Knowledge 知识库 | [agent-knowledge.md](modules/agent-knowledge.md) | Markdown Wiki、知识图谱、自动整理、交叉引用 |
| Evolution 自我进化 | [agent-evolution.md](modules/agent-evolution.md) | 空闲触发、隔离Agent、硬工程防护、变更检测、回滚 |
| Skills 技能系统 | [agent-skills.md](modules/agent-skills.md) | SKILL.md规范、SkillLoader、SkillManager、技能市场 |
| Tools 工具+MCP | [agent-tools.md](modules/agent-tools.md) | BaseTool接口、14个内置工具、MCP协议(stdio/SSE/HTTP)、热加载 |
| Prompt 提示词系统 | [agent-prompt.md](modules/agent-prompt.md) | PromptBuilder模块化构建、工作空间文件、模板系统、多语言 |
| Bridge 桥接层 | [bridge.md](modules/bridge.md) | Bridge单例路由、AgentBridge适配、AgentInitializer编排、事件转换 |
| Channel 通道系统 | [channel.md](modules/channel.md) | Channel接口、12种通道、ChannelFactory、多通道并行 |
| Models 模型适配 | [models.md](modules/models.md) | bot_factory工厂、15+厂商、OpenAI兼容增强、六维独立路由 |
| Plugins 插件系统 | [plugins.md](modules/plugins.md) | PluginManager、消息中间件、10个内置插件 |
| CLI 命令行 | [cli.md](modules/cli.md) | Click框架、cow命令体系、进程管理、技能CLI |
| Voice+Translate | [voice-translate.md](modules/voice-translate.md) | ASR/TTS工厂、13种语音、2种翻译、接口规范 |
| App+Config+Common | [app-config-common.md](modules/app-config-common.md) | 启动流程、配置体系(800+行)、常量、i18n、单例 |

---

## 一、项目概述

**CowAgent**（原名 `chatgpt-on-wechat`）是一个开源超级 AI 助手，是 **Agent Harness（智能体套件）工程化的参考实现**。它能够主动规划任务、控制计算机和外部服务、创建和运行技能（Skills）、构建个人知识库和长期记忆，并通过**自我进化（Self-Evolution）**与用户一同成长——越用越聪明。

| 属性 | 说明 |
|------|------|
| 许可证 | MIT |
| 最新版本 | v2.1.1（2026.06.09） |
| Python 版本 | ≥ 3.7（最新支持 3.13） |
| 官网 | [cowagent.ai](https://cowagent.ai) |
| 文档站 | [docs.cowagent.ai](https://docs.cowagent.ai) |
| 技能市场 | [skills.cowagent.ai](https://skills.cowagent.ai) |
| 定位 | Agent Harness — 通道/核心/模型三层完全解耦 |

**核心能力一览**：

| 能力 | 简介 |
|------|------|
| 任务规划 | 分解复杂任务，多步执行，工具循环直到达成目标 |
| 长期记忆 | 三层架构（上下文 → 每日 → 核心），Deep Dream 夜间蒸馏 |
| 知识整理 | 自动将对话中有价值的信息整理为结构化 Markdown Wiki |
| 自我进化 | 自动审查对话，改进技能、整理记忆、跟进未完成任务 |
| 技能系统 | 开放技能市场，一键安装，对话式创作 |
| 工具生态 | 内置 10+ 工具 + 原生 MCP 协议集成 |
| 多通道 | 12 种 IM 平台接入 |
| 多模态 | 文本、图像、语音、文件全支持 |
| 多模型 | 15+ LLM 厂商，Chat/Vision/Image/ASR/TTS/Embed 独立路由 |

---

## 二、架构设计

### 2.1 整体架构（Agent Harness 模式）

CowAgent 本质上是一个 **Agent Harness**，四个核心层次完全解耦，每一层都可独立扩展：

```
┌──────────────────────────────────────────────────────────────────┐
│                        🏗️ Channels 层                            │
│  Web · 微信 · 飞书 · 钉钉 · QQ · 企微 · Telegram · Slack · ...   │
│      消息流入 ↓                                     ↑ 回复流出     │
├──────────────────────────────────────────────────────────────────┤
│                      🧠 Agent Core 层                            │
│  ┌──────────┐ ┌───────────┐ ┌─────────┐ ┌──────────────────┐    │
│  │  Memory  │ │ Knowledge │ │ Skills  │ │     Tools        │    │
│  │ 三层记忆 │ │  知识库   │ │ 技能市场 │ │ 内置工具 + MCP   │    │
│  │ SQLite   │ │   Wiki    │ │ 热插拔  │ │ 14个工具+生态    │    │
│  └──────────┘ └───────────┘ └─────────┘ └──────────────────┘    │
│       ↓ 规划推理 + 多轮工具调用循环（Agent Stream Executor） ↓      │
├──────────────────────────────────────────────────────────────────┤
│                      🤖 Models 层                                │
│  Claude · GPT · Gemini · DeepSeek · Qwen · GLM · Kimi · ...     │
│  Chat │ Vision │ ImageGen │ ASR │ TTS │ Embedding  ← 独立路由    │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 核心运行流程

```
app.py:run()
├── load_config()                           # 加载 config.json
├── sigterm_handler_wrap()                  # 注册信号处理
├── _sync_builtin_skills()                  # 同步内置技能到工作空间
├── _warmup_mcp_tools()                     # 预热 MCP 子进程
├── _warmup_scheduler()                     # 预热调度器
│
└── ChannelManager.start()
    ├── PluginManager.load_plugins()        # 加载插件
    ├── channel_factory.create_channel()    # 为每个通道类型创建实例
    └── Thread(target=channel.startup)      # 每个通道跑在独立 daemon 线程
        │
        └── [收到用户消息]
            └── Bridge.get_agent_bridge()
                └── AgentBridge.agent_reply(query, context, on_event)
                    ├── AgentInitializer.initialize_agent(session_id)
                    │   ├── 迁移 API 密钥到 .env
                    │   ├── 加载环境变量
                    │   ├── 初始化工作空间 (~/cow/)
                    │   │   ├── AGENT.md   (Agent 人格)
                    │   │   ├── USER.md    (用户信息)
                    │   │   ├── RULE.md    (行为规则)
                    │   │   └── BOOTSTRAP.md (启动引导)
                    │   ├── 初始化记忆系统 (MemoryManager)
                    │   ├── 加载工具 (ToolManager → 内置 + MCP)
                    │   ├── 加载技能 (SkillManager)
                    │   │   ├── skills/          (内置)
                    │   │   └── workspace/skills/ (自定义)
                    │   ├── 初始化调度器 (SchedulerService)
                    │   └── 构建系统提示词 (PromptBuilder)
                    │       ├── 基础人设
                    │       ├── 上下文文件 (AGENT.md 等)
                    │       ├── 工具描述
                    │       ├── 技能描述
                    │       ├── 记忆摘要
                    │       └── 运行时信息 (时间/系统)
                    │
                    └── AgentStreamExecutor.run()
                        ┌─────────────────────────────────┐
                        │  while step < max_steps:        │
                        │  1. LLM.chat(messages + tools)  │
                        │  2. 解析响应:                    │
                        │     - finish_reason=stop → 返回  │
                        │     - tool_calls → 执行工具      │
                        │  3. ToolManager.execute()       │
                        │  4. 结果追加到 messages          │
                        │  5. step += 1                   │
                        └─────────────────────────────────┘
                        ↓
                    返回 Reply → Channel.send()
```

### 2.3 架构图

```
                    ┌──────────────┐
                    │   用户消息    │
                    └──────┬───────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
        ┌───▼──┐     ┌────▼───┐    ┌────▼───┐
        │ Web  │     │ Feishu │    │  ...   │   ← Channel 层
        └───┬──┘     └────┬───┘    └────┬───┘
            │              │              │
            └──────────────┼──────────────┘
                           │
                    ┌──────▼──────┐
                    │   Bridge    │   ← 桥接层（路由 + 适配）
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ AgentBridge │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼────┐ ┌────▼────┐ ┌────▼────┐
        │ Memory   │ │ Tools   │ │ Skills  │  ← Agent Core
        └──────────┘ └─────────┘ └─────────┘
                           │
                    ┌──────▼──────┐
                    │   Models    │   ← LLM 适配层
                    └─────────────┘
```

---

## 三、目录结构详解

### 3.1 项目根目录

| 文件/目录 | 用途 |
|-----------|------|
| `app.py` | 🔴 **应用入口**：配置加载、信号处理、ChannelManager 多通道生命周期管理、MCP/调度器预热 |
| `config.py` | 🔴 **配置中心**（800+ 行）：所有可用配置项的声明与默认值，含自动迁移和验证逻辑 |
| `config-template.json` | 配置模板文件 |
| `run.sh` | Linux/macOS 一键安装启动脚本 |
| `run.ps1` | Windows PowerShell 一键安装脚本 |
| `pyproject.toml` | Python 包定义，CLI 入口 `cow = cli.cli:main` |
| `requirements.txt` | Python 核心依赖 |
| `requirements-optional.txt` | 可选依赖（语音、浏览器等） |
| `Dockerfile` | Docker 镜像构建文件 |
| `README.md` | 项目主文档 |
| `CONTRIBUTING.md` | 贡献指南 |
| `LICENSE` | MIT 许可证 |

### 3.2 agent/ — 智能体核心

这是整个项目最核心的目录，所有 AI 能力都在此实现：

```
agent/
├── protocol/                         # 🔴 核心执行引擎
│   ├── agent.py                      # Agent 类：系统提示词、工具注册、技能集成
│   ├── agent_stream.py               # AgentStreamExecutor：多轮工具调用循环、流式执行
│   ├── models.py                     # LLMRequest / LLMModel 数据模型
│   ├── context.py                    # 上下文管理（token 计算、压缩）
│   ├── result.py                     # AgentAction / ToolResult / AgentResult 数据类
│   ├── task.py                       # 任务抽象
│   ├── cancel.py                     # 用户取消机制（线程安全）
│   └── message_utils.py              # 消息清洗、压缩（Claude/GPT 格式兼容、reasoning 截断）
│
├── chat/                             # 会话管理
│   ├── service.py                    # 聊天服务
│   └── session_service.py            # 会话持久化（SQLite 存储）
│
├── memory/                           # 🧠 三层长期记忆系统
│   ├── manager.py                    # MemoryManager：高层统一接口
│   ├── storage.py                    # MemoryStorage：SQLite + FTS5 全文索引
│   ├── config.py                     # MemoryConfig：记忆系统配置
│   ├── chunker.py                    # TextChunker：智能文本分块（可配置重叠）
│   ├── summarizer.py                 # MemoryFlushManager：Deep Dream 记忆蒸馏
│   ├── conversation_store.py         # 对话历史存储
│   ├── service.py                    # 记忆服务接口
│   ├── rebuild_index.py              # 手动重建索引
│   └── embedding/                    # 向量嵌入子系统
│       ├── provider.py               # EmbeddingProvider（多厂商抽象）
│       ├── state.py                  # 嵌入索引状态管理（增量/全量）
│       └── rebuild.py                # 索引重建逻辑
│
├── knowledge/                        # 📚 个人知识库
│   └── service.py                    # KnowledgeService：Markdown Wiki 管理、知识图谱
│
├── evolution/                        # 🧬 自我进化系统
│   ├── trigger.py                    # 空闲触发扫描线程（每60s，≥6轮 & 10min空闲 or 上下文压力）
│   ├── executor.py                   # 进化执行器：审查对话 → 改进技能/记忆/知识
│   ├── prompts.py                    # 进化审查的提示词模板
│   ├── config.py                     # EvolutionConfig：进化参数配置
│   ├── record.py                     # 进化历史记录
│   └── backup.py                     # 回滚/备份（安全的技能修改）
│
├── skills/                           # 🔧 技能系统
│   ├── manager.py                    # SkillManager：注册、安装、刷新、配置管理
│   ├── loader.py                     # SkillLoader：从目录加载 SKILL.md
│   ├── types.py                      # Skill / SkillEntry / SkillSnapshot 数据类型
│   ├── formatter.py                  # 为 LLM 格式化技能提示（注入到系统提示词）
│   ├── frontmatter.py                # YAML frontmatter 解析
│   ├── config.py                     # 技能配置
│   └── service.py                    # 技能服务接口（安装、搜索、列表）
│
├── tools/                            # 🛠️ 工具系统
│   ├── base_tool.py                  # BaseTool 抽象基类（定义工具标准接口）
│   ├── tool_manager.py               # ToolManager 单例：工具加载 + MCP 集成 + 热重载
│   ├── bash/bash.py                  # 终端命令执行
│   ├── read/read.py                  # 文件读取
│   ├── write/write.py                # 文件写入
│   ├── edit/edit.py                  # 精确字符串替换编辑
│   ├── ls/ls.py                      # 目录列表
│   ├── send/send.py                  # 文件发送
│   ├── web_fetch/web_fetch.py        # 网页内容抓取
│   ├── web_search/web_search.py      # Web 搜索
│   ├── vision/vision.py              # 视觉识别（图片理解）
│   ├── browser/                      # 浏览器自动化（Playwright）
│   │   ├── browser_tool.py           # 浏览器工具定义
│   │   └── browser_service.py        # 浏览器服务管理
│   ├── memory/                       # 记忆检索工具
│   │   ├── memory_get.py             # 获取记忆
│   │   └── memory_search.py          # 搜索记忆
│   ├── scheduler/                    # 定时任务调度
│   │   ├── scheduler_tool.py         # 调度工具定义
│   │   ├── scheduler_service.py      # 调度服务（后台线程）
│   │   ├── task_store.py             # 任务持久化存储
│   │   └── integration.py            # 与 Agent 的集成
│   ├── env_config/env_config.py      # 环境变量管理
│   ├── mcp/                          # MCP 协议集成
│   │   ├── mcp_client.py             # MCP 客户端（stdio/SSE/Streamable HTTP）
│   │   └── mcp_tool.py               # MCP 工具适配器
│   ├── evolution_undo/               # 进化回滚工具
│   └── utils/                        # 工具辅助函数
│       ├── diff.py                   # 文本差异
│       └── truncate.py               # 输出截断
│
└── prompt/                           # 💬 提示词系统 [📖 详见](modules/agent-prompt.md)
    ├── builder.py                    # PromptBuilder：模块化构建完整系统提示词
    └── workspace.py                  # 工作空间文件管理（AGENT.md/USER.md/RULE.md 等）
```

### 3.3 bridge/ — 桥接层

连接模型/通道/Agent 的中枢路由层：

```
bridge/
├── bridge.py                # Bridge 单例：多模型路由（chat/voice/translate），
│                            #   根据模型名自动路由到正确的 LLM 适配器
├── agent_bridge.py          # AgentBridge：将 Agent 系统与 Channel 基础设施对接，
│                            #   支持动态为 bot 添加 tool calling 能力
├── agent_event_handler.py   # AgentEventHandler：Agent 事件 → Channel 回复的转换，
│                            #   处理流式输出、图片、文件等
├── agent_initializer.py     # AgentInitializer：编排 Agent 初始化全流程
│                            #   （工作空间→记忆→工具→技能→提示词→调度器）
├── context.py               # Context：用户/会话/消息上下文封装
└── reply.py                 # Reply / ReplyType：回复数据类型定义
```

### 3.4 channel/ — 通道层

所有 IM 平台的消息入口和出口：

```
channel/
├── channel.py               # Channel 抽象基类（startup/send/stop 接口）
├── channel_factory.py       # 通道工厂：create_channel(type) → Channel 实例
├── web/                     # 🌐 Web Console（默认通道，HTTP + WebSocket + SSE）
│   └── static/              # 前端静态资源
├── weixin/                  # 💬 个人微信（基于 itchat）
├── wechatmp/                # 📱 微信公众号
├── wechatcom/               # 🏢 企业微信应用
├── wecom_bot/               # 🤖 企业微信群机器人
├── wechat_kf/               # 🎧 微信客服
├── feishu/                  # 🐦 飞书 / Lark
├── dingtalk/                # 📌 钉钉
├── qq/                      # 🐧 QQ
├── telegram/                # ✈️ Telegram
├── slack/                   # 💼 Slack
├── discord/                 # 🎮 Discord
└── terminal/                # ⌨️ 终端交互模式（--cmd 参数）
```

### 3.5 models/ — 模型适配层

每种 LLM 厂商一个独立子目录，统一实现 `reply()` 接口：

```
models/
├── bot_factory.py           # 模型工厂：create_bot(type) → Bot 实例
├── openai_compatible_bot.py # OpenAI 兼容协议通用 bot（tool calling 增强基类）
├── openai/                  # OpenAI 原生（GPT 全系列）
├── chatgpt/                 # OpenAI 兼容通用适配
├── gemini/                  # Google Gemini
├── claude/ → gemini/        # Anthropic Claude（通过 Gemini API 代理）
├── deepseek/                # DeepSeek
├── qianfan/                 # 百度千帆 / ERNIE
├── doubao/                  # 字节豆包
├── moonshot/                # Moonshot / Kimi
├── zhipuai/                 # 智谱 GLM
├── dashscope/               # 阿里通义千问（DashScope）
├── minimax/                 # MiniMax
├── mimo/                    # 小米 MiMo
├── xunfei/                  # 讯飞星火
├── linkai/                  # LinkAI 平台
├── modelscope/              # ModelScope
└── baidu/                   # 百度文心
```

### 3.6 其他重要目录

| 目录 | 用途 |
|------|------|
| `voice/` | 🎤 语音服务工厂：OpenAI / Azure / Google / Baidu / Ali / Xunfei / Tencent / DashScope / ZhipuAI / MiniMax / MiMo / LinkAI / ElevenLabs / pytts / Edge TTS |
| `translate/` | 🌐 翻译服务工厂：Baidu / Youdao |
| `plugins/` | 🔌 插件系统：banwords(敏感词) / keyword(关键词) / role(角色扮演) / godcmd(管理员命令) / dungeon(文字游戏) / tool / finish / cow_cli / hello / linkai |
| `skills/` | 📦 内置技能模板：skill-creator(技能创建器) / knowledge-wiki(知识库管理) / image-generation(图片生成) |
| `cli/` | 🖥️ Cow CLI 工具：start/stop/restart + skill install/search/list + knowledge + install-browser |
| `common/` | 📋 公共模块：const(全局常量) / log(日志) / i18n(国际化) / utils(工具函数) / singleton(单例装饰器) |
| `docs/` | 📖 Mintlify 文档站源码 |
| `tests/` | 🧪 单元测试 |
| `scripts/` | 🔧 部署/运维脚本 |
| `docker/` | 🐳 Docker Compose 配置 |
| `translate/` | 🌐 翻译适配器 |

---

## 四、核心功能模块深度解析

### 4.1 Agent Core（智能体核心）

**位置**: [`agent/protocol/`](agent/protocol/)

> 📖 **详见**: [agent-protocol.md](modules/agent-protocol.md) — Agent Core 执行引擎深度分析

这是整个项目最核心的部分，实现了标准的 **Agent Loop**（工具调用循环）：

**关键类**：

| 类 | 文件 | 职责 |
|----|------|------|
| `Agent` | `agent.py` | 封装系统提示词、工具列表、技能管理器、记忆管理器，提供 `run()` 入口 |
| `AgentStreamExecutor` | `agent_stream.py` | 驱动多轮推理+工具调用循环，支持流式 SSE 输出 |
| `LLMModel` | `models.py` | LLM 抽象接口，屏蔽不同模型 API 差异 |
| `BaseTool` | `tools/base_tool.py` | 工具抽象基类，定义 name/description/parameters + execute() 标准接口 |

**执行循环**（在 `AgentStreamExecutor.run()` 中）：

```python
while step < max_steps:
    # 1. 调用 LLM（带 tools 定义）
    response = model.chat(messages, tools)

    # 2. 解析响应
    if finish_reason == "stop":
        return final_reply  # Agent 认为任务完成
    elif finish_reason == "tool_calls":
        for tool_call in response.tool_calls:
            # 3. 执行工具
            result = tool_manager.execute(tool_call.name, tool_call.args)
            # 4. 结果追加到 messages
            messages.append({"role": "tool", "content": result})
        # 5. 继续下一轮推理
        step += 1
```

**关键设计**：
- **取消机制** (`cancel.py`): 线程安全的取消注册表，用户可随时中断执行
- **上下文压缩** (`message_utils.py`): 自动截断过长 reasoning 内容（保留头尾 2KB），压缩历史消息为纯文本
- **JSON 修复**: 使用 `json_repair` 库修复非严格厂商返回的畸形 JSON

### 4.2 Bridge 桥接层

**位置**: [`bridge/`](bridge/)

> 📖 **详见**: [bridge.md](modules/bridge.md) — Bridge 桥接层深度分析

Bridge 是整个系统的**中枢路由层**，负责将 Channel 消息路由到正确的处理链路：

```
Channel 消息
  → Bridge.get_agent_bridge()
    → AgentBridge.agent_reply(query, context, on_event)
      → AgentInitializer.initialize_agent(session_id)
        → AgentStreamExecutor.run()
          → 返回 Reply
            → Channel.send(reply)
```

**关键类**：

| 类 | 职责 |
|----|------|
| `Bridge` | 单例，管理所有 bot 实例（Chat/Voice/Translate），根据模型名自动路由 |
| `AgentBridge` | Agent 系统与 Channel 的适配层，动态为 bot 添加 OpenAI tool calling 能力 |
| `AgentInitializer` | 编排 Agent 初始化全流程（8 个步骤） |
| `AgentEventHandler` | 将 Agent 事件转换为 Channel 回复（文本/图片/文件/错误） |

**模型路由逻辑**（`Bridge.__init__`）：

```python
# 根据 model 名称前缀自动判断 bot_type：
if model.startswith("claude"):   → CLAUDEAPI
if model.startswith("gemini"):   → GEMINI
if model.startswith("glm"):      → ZHIPU_AI
if model.startswith("deepseek"): → DEEPSEEK
if model.startswith("kimi"):     → MOONSHOT
if model.startswith("doubao"):   → DOUBAO
# ... 等等
```

### 4.3 Memory 三层记忆

**位置**: [`agent/memory/`](agent/memory/)

> 📖 **详见**: [agent-memory.md](modules/agent-memory.md) — Memory 三层记忆系统深度分析

CowAgent 实现了业界领先的三层长期记忆架构：

```
┌─────────────────────────────────────────────┐
│  Layer 1: 上下文记忆 (Context Memory)        │
│  • 会话内消息历史                            │
│  • 自动压缩：超出 token 上限时智能截断        │
├─────────────────────────────────────────────┤
│  Layer 2: 每日记忆 (Daily Memory)            │
│  • 按天自动聚合                              │
│  • 混合检索：FTS5 关键词 + 向量语义          │
│  • Deep Dream 夜间蒸馏 → Layer 3             │
├─────────────────────────────────────────────┤
│  Layer 3: 核心记忆 (Core Memory / MEMORY.md) │
│  • 永久保留的精华                             │
│  • 结构化 Markdown 格式                       │
│  • Agent 可直接检索引用                       │
└─────────────────────────────────────────────┘
```

**技术栈**：
- **存储**: SQLite + FTS5 全文搜索
- **检索**: 混合检索 = 关键词匹配 (FTS5) + 向量语义相似度 (Embedding)
- **向量嵌入**: 多厂商支持（OpenAI / DashScope / ZhipuAI 等），带缓存和增量重建
- **Deep Dream**: 后台任务，将散落的每日记忆蒸馏为精炼的 MEMORY.md 条目和叙事日志

**关键类**：

| 类 | 职责 |
|----|------|
| `MemoryManager` | 高层统一接口 |
| `MemoryStorage` | SQLite + FTS5 存储引擎 |
| `TextChunker` | 智能文本分块（可配置块大小和重叠） |
| `MemoryFlushManager` | Deep Dream 记忆蒸馏调度 |
| `EmbeddingProvider` | 多厂商嵌入向量抽象 |

### 4.4 Knowledge 知识库

**位置**: [`agent/knowledge/`](agent/knowledge/)

> 📖 **详见**: [agent-knowledge.md](modules/agent-knowledge.md) — Knowledge 知识库系统深度分析

与记忆系统互补——记忆按**时间**组织，知识按**主题**组织：

```
workspace/
└── knowledge/
    ├── index.md                    # 知识库索引
    ├── log.md                      # 更新日志
    ├── concepts/                   # 概念类知识
    │   └── reinforcement-learning.md
    ├── tools/                      # 工具使用知识
    │   └── docker-commands.md
    └── projects/                   # 项目相关知识
        └── my-app-architecture.md
```

**核心能力**：
- AI 自动从对话中提取有价值的信息，整理为 Markdown 文档
- 自动维护交叉引用和索引 (`index.md`)
- Web Console 提供交互式**知识图谱可视化**
- 支持手动编辑和 AI 辅助编辑

### 4.5 Evolution 自我进化

**位置**: [`agent/evolution/`](agent/evolution/)

> 📖 **详见**: [agent-evolution.md](modules/agent-evolution.md) — Evolution 自我进化系统深度分析

这是 v2.1.1 的标志性功能——让 Agent 在后台自动成长：

**触发条件**（满足任一即触发）：

| 条件 | 默认值 |
|------|--------|
| 用户对话轮数 | ≥ 6 轮（自上次进化后） |
| 会话空闲时间 | ≥ 10 分钟 |
| 上下文压力 | 活跃上下文超过 token 预算的 80% |

**进化执行流程**：

```
1. 后台扫描线程 (每60s)
   └── 发现符合条件的空闲会话
       └── 启动进化 Agent（独立 LLM 实例）
           ├── 审查本轮对话
           ├── 提出改进建议
           │   ├── 技能改进（修改 SKILL.md）
           │   ├── 记忆整理（更新 MEMORY.md）
           │   ├── 知识补充（创建/更新知识文档）
           │   └── 任务跟进（未完成的待办事项）
           ├── 可信度评估 → 决定是否自动应用
           └── 通知用户（Web UI 中标记 🧬）
```

**安全设计**：
- 进化结果标记可信度（LOW/MEDIUM/HIGH）
- 低可信度建议不自动应用，仅提示用户
- 技能修改前自动备份（`backup.py`）
- 支持回滚（`evolution_undo` 工具）

### 4.6 Skills 技能系统

**位置**: [`agent/skills/`](agent/skills/)

> 📖 **详见**: [agent-skills.md](modules/agent-skills.md) — Skills 技能系统深度分析

技能是比工具更高层的工作流定义，每个技能是一个包含 `SKILL.md` 的目录：

```
my-skill/
├── SKILL.md         # 技能定义（YAML frontmatter + Markdown 指令）
├── script.py        # 可选的可执行脚本
└── assets/          # 可选的资源文件
```

**SKILL.md 格式**：

```markdown
---
name: my-skill
description: 描述这个技能做什么
version: 1.0.0
tools:
  - bash
  - read
  - write
---

# 技能指令

当用户要求 XXX 时，请按以下步骤执行...
```

**技能来源**：
1. **内置技能** (`skills/`): skill-creator, knowledge-wiki, image-generation
2. **自定义技能** (`workspace/skills/`): 用户自行安装或创建
3. **Skill Hub** ([skills.cowagent.ai](https://skills.cowagent.ai)): 开放技能市场
4. **GitHub / ClawHub / URL**: 从任意源安装
5. **对话式创作**: 通过 `skill-creator` 技能与 Agent 对话生成

**关键类**：

| 类 | 职责 |
|----|------|
| `SkillManager` | 技能注册、安装、刷新、启用/禁用配置 |
| `SkillLoader` | 从目录加载和验证 SKILL.md |
| `SkillFormatter` | 为 LLM 格式化技能描述，注入到系统提示词 |

### 4.7 Tools 工具系统

**位置**: [`agent/tools/`](agent/tools/)

> 📖 **详见**: [agent-tools.md](modules/agent-tools.md) — Tools 工具系统与 MCP 集成深度分析

**内置工具完整清单**：

| 工具名 | 文件 | 功能 | 阶段 |
|--------|------|------|------|
| `bash` | `bash/bash.py` | 终端命令执行（沙箱化） | 运行时 |
| `read` | `read/read.py` | 文件读取（支持分页） | 运行时 |
| `write` | `write/write.py` | 文件写入（覆盖） | 运行时 |
| `edit` | `edit/edit.py` | 精确字符串替换编辑 | 运行时 |
| `ls` | `ls/ls.py` | 目录列表 | 运行时 |
| `send` | `send/send.py` | 文件发送到用户 | 运行时 |
| `web_search` | `web_search/web_search.py` | 网络搜索 | 运行时 |
| `web_fetch` | `web_fetch/web_fetch.py` | 网页内容抓取（HTML→Markdown） | 运行时 |
| `vision` | `vision/vision.py` | 图像识别/理解 | 运行时 |
| `browser` | `browser/` | 浏览器自动化（Playwright） | 运行时 |
| `memory_get` | `memory/memory_get.py` | 获取记忆条目 | 运行时 |
| `memory_search` | `memory/memory_search.py` | 搜索记忆（混合检索） | 运行时 |
| `scheduler` | `scheduler/` | 定时任务调度 | 运行时 |
| `env_config` | `env_config/env_config.py` | 环境变量读写 | 运行时 |
| `evolution_undo` | `evolution_undo/` | 撤销进化修改 | 运行时 |

**MCP 集成**：
- 支持 **stdio** 传输（本地子进程：npx/uvx/python）
- 支持 **SSE** 传输（远程 HTTP 服务）
- 支持 **Streamable HTTP** 传输
- 配置文件 `mcp.json`
- **热加载**：检测文件变更后自动重载，无需重启
- MCP 工具与内置工具使用统一接口，Agent 无感

### 4.8 Channels 通道系统

**位置**: [`channel/`](channel/)

> 📖 **详见**: [channel.md](modules/channel.md) — Channel 通道系统深度分析

所有通道遵循统一接口：

```python
class Channel:
    def startup(self):
        """启动通道，开始阻塞式消息监听循环"""

    def send(self, reply: Reply, context: Context):
        """发送回复到用户"""

    def stop(self):
        """优雅关闭通道"""
```

**已支持的 12 种通道**：

| 通道 | 目录 | 文本 | 图片 | 文件 | 语音 | 群聊 |
|------|------|:----:|:----:|:----:|:----:|:----:|
| Web Console | `web/` | ✅ | ✅ | ✅ | ✅ | — |
| 个人微信 | `weixin/` | ✅ | ✅ | ✅ | ✅ | — |
| 微信公众号 | `wechatmp/` | ✅ | ✅ | — | ✅ | — |
| 企业微信应用 | `wechatcom/` | ✅ | ✅ | ✅ | ✅ | — |
| 企业微信机器人 | `wecom_bot/` | ✅ | ✅ | ✅ | ✅ | ✅ |
| 微信客服 | `wechat_kf/` | ✅ | ✅ | ✅ | ✅ | — |
| 飞书 | `feishu/` | ✅ | ✅ | ✅ | ✅ | ✅ |
| 钉钉 | `dingtalk/` | ✅ | ✅ | ✅ | ✅ | ✅ |
| QQ | `qq/` | ✅ | ✅ | ✅ | — | ✅ |
| Telegram | `telegram/` | ✅ | ✅ | ✅ | ✅ | ✅ |
| Slack | `slack/` | ✅ | ✅ | ✅ | — | ✅ |
| Discord | `discord/` | ✅ | ✅ | ✅ | — | ✅ |
| 终端模式 | `terminal/` | ✅ | — | — | — | — |

**多通道并行**：ChannelManager 管理所有通道的生命周期，每个通道跑在独立 daemon 线程，互不干扰。

### 4.9 Models 模型层

**位置**: [`models/`](models/)

> 📖 **详见**: [models.md](modules/models.md) — Models 模型适配层深度分析

通过 `bot_factory.create_bot(type)` 工厂模式统一创建：

| 厂商 | 目录 | Chat | Vision | ImageGen | ASR | TTS | Embed |
|------|------|:----:|:------:|:--------:|:---:|:---:|:-----:|
| Claude | `claude/` → `gemini/` | ✅ | ✅ | — | — | — | — |
| OpenAI | `openai/` `chatgpt/` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Gemini | `gemini/` | ✅ | ✅ | ✅ | — | — | — |
| DeepSeek | `deepseek/` | ✅ | — | — | — | — | — |
| Qwen | `dashscope/` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| GLM | `zhipuai/` | ✅ | ✅ | — | ✅ | — | ✅ |
| Kimi | `moonshot/` | ✅ | ✅ | — | — | — | — |
| MiniMax | `minimax/` | ✅ | ✅ | ✅ | — | ✅ | — |
| Doubao | `doubao/` | ✅ | ✅ | ✅ | — | — | ✅ |
| ERNIE | `qianfan/` `baidu/` | ✅ | ✅ | — | — | — | — |
| MiMo | `mimo/` | ✅ | ✅ | — | — | ✅ | — |
| LinkAI | `linkai/` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Custom | `chatgpt/` | ✅ | — | — | — | — | — |

**六个维度独立路由**：Chat / Vision / ImageGen / ASR / TTS / Embedding 可分别配置不同供应商。

### 4.10 CLI 命令行

**位置**: [`cli/`](cli/)

基于 Click 框架，入口命令 `cow`：

| 命令 | 功能 |
|------|------|
| `cow start` | 启动 CowAgent 服务 |
| `cow stop` | 停止服务 |
| `cow restart` | 重启服务 |
| `cow status` | 查看运行状态 |
| `cow logs` | 查看日志（支持 -f 实时跟踪） |
| `cow update` | 拉取最新代码并重启 |
| `cow skill install <name>` | 安装技能 |
| `cow skill search <keyword>` | 搜索技能市场 |
| `cow skill list` | 列出已安装技能 |
| `cow install-browser` | 安装浏览器自动化依赖 |
| `cow knowledge` | 知识库管理 |

---

## 五、开发规范

### 5.1 代码风格

**语言要求**（来自 `CONTRIBUTING.md`）：
- 代码注释和文档字符串：**英文**
- Issue / PR 标题和描述：**英文**
- Commit message：**英文**

**提交规范**：推荐 [Conventional Commits](https://www.conventionalcommits.org/) 格式，但不强制：

```
feat: add web search tool
fix: reconnect telegram websocket on timeout
docs: clarify Docker setup
chore: bump dependencies
refactor: extract memory chunker
```

**分支策略**：
- 从 `master` 创建特性分支
- 命名风格：`feat/xxx`, `fix/xxx`, `docs/xxx`, `chore/xxx`
- PR 合并回 `master`
- 一个 PR 做一件事（保持聚焦）

**Python 版本**：
- 最低支持：Python 3.7
- 最新支持：Python 3.13
- 包管理：pip + setuptools

### 5.2 架构设计模式

| 模式 | 应用位置 | 说明 |
|------|---------|------|
| **工厂模式** | `channel/channel_factory.py`, `models/bot_factory.py`, `voice/factory.py`, `translate/factory.py` | 统一创建各类组件实例 |
| **单例模式** | `Bridge`, `ToolManager` | 全局唯一实例，通过 `@singleton` 装饰器实现 |
| **适配器模式** | `agent/protocol/models.py` 中的 `LLMModel` | 屏蔽不同 LLM API 的差异 |
| **观察者模式** | `AgentEventHandler` | 事件驱动的流式输出 |
| **策略模式** | `agent/memory/embedding/provider.py` | 多厂商嵌入向量策略切换 |
| **模板方法** | `BaseTool` | 定义工具标准接口，子类实现具体逻辑 |
| **插件化** | `plugins/` | 热插拔式插件扩展 |

### 5.3 目录约定

新增功能时遵循以下约定：

| 新增类型 | 创建位置 | 需要修改的注册点 |
|----------|---------|----------------|
| 新 LLM 厂商 | `models/<vendor>/` | `models/bot_factory.py` + `common/const.py` |
| 新 IM 通道 | `channel/<name>/` | `channel/channel_factory.py` + `app.py` |
| 新工具 | `agent/tools/<name>/` | `agent/tools/tool_manager.py` |
| 新技能 | `skills/<name>/` 或外部仓库 | 无需修改代码（热加载） |
| 新插件 | `plugins/<name>/` | 无需修改代码（自动发现） |
| 新语音服务 | `voice/<name>/` | `voice/factory.py` |
| 新翻译服务 | `translate/<name>/` | `translate/factory.py` |

### 5.4 扩展点汇总

| 扩展点 | 接口 | 说明 |
|--------|------|------|
| `Channel` | `startup()`, `send()`, `stop()` | 接入新 IM 平台 |
| `Bot` | `reply(query, context) → Reply` | 接入新 LLM 厂商 |
| `BaseTool` | `name`, `description`, `parameters`, `execute()` | 添加新工具能力 |
| `Skill` | `SKILL.md` (frontmatter + instructions) | 创建新技能工作流 |
| `Plugin` | 插件接口 | 添加消息处理中间件 |
| `Voice` | `voiceToText()` / `textToVoice()` | 接入新语音服务 |
| `Translator` | `translate()` | 接入新翻译服务 |
| `EmbeddingProvider` | `embed(texts)` | 接入新嵌入模型 |

---

## 六、项目功能亮点

### 6.1 技术创新

| 亮点 | 详细说明 |
|------|---------|
| 🤖 **Agent Harness 架构** | 通道/核心/模型三层完全解耦，每层可独立扩展，定义了 Agent 工程化的参考标准 |
| 🧬 **自我进化 (Self-Evolution)** | Agent 在后台自动审查对话、改进技能、整理记忆、补充知识、跟进任务——越用越聪明 |
| 🧠 **三层长期记忆** | 上下文→每日→核心的三层架构 + Deep Dream 夜间蒸馏，实现从短期到长期的自然记忆沉淀 |
| 📚 **自动知识整理** | AI 自动将对话中有价值的信息按主题整理为结构化 Markdown Wiki，支持知识图谱可视化 |
| 🔧 **技能生态系统** | 开放技能市场（Skill Hub），支持从 GitHub/ClawHub/URL 安装，以及对话式自然语言创作 |
| 🔌 **原生 MCP 集成** | 完整支持 Model Context Protocol（stdio/SSE/Streamable HTTP），热加载，零代码扩展工具 |
| 🌍 **全平台覆盖** | 12 种 IM 通道统一接入，Web 控制台一站式管理 |
| 🎯 **六维独立路由** | Chat / Vision / ImageGen / ASR / TTS / Embedding 六个 AI 维度可独立配置不同厂商 |
| 💬 **智能上下文压缩** | 自动识别上下文压力，智能截断 reasoning 内容，保留关键信息同时控制 token 消耗 |

### 6.2 工程亮点

| 亮点 | 详细说明 |
|------|---------|
| 🚀 **一键安装** | 单行命令完成所有依赖安装和配置，支持 Linux/macOS/Windows/Docker |
| 🎛️ **Web 统一控制台** | 可视化配置模型、通道、技能、记忆、知识，无需手动编辑配置文件 |
| 🔄 **并行会话** | Web Console 支持多个独立会话同时运行，互不干扰 |
| ⚡ **流式输出** | 全通道支持 SSE 流式响应，实时展示 Agent 思考过程 |
| 🌏 **国际化** | CLI、提示词、回复均支持中/英/日三语切换 |
| 🔥 **MCP 热加载** | 修改 `mcp.json` 后自动检测并重载，无需重启服务 |
| 🛡️ **优雅降级** | 嵌入向量失败时自动回退到纯关键词搜索，不影响核心功能 |
| 📦 **技能同步** | 启动时自动将项目内置技能同步到用户工作空间，确保最新版本 |
| 🔐 **安全设计** | Web 密码保护、敏感词插件、进化操作自动备份可回滚 |

---

## 七、二次开发指南

### 7.1 开发环境搭建

```bash
# 1. 克隆仓库
git clone https://github.com/zhayujie/CowAgent.git
cd CowAgent

# 2. 安装依赖
pip install -r requirements.txt

# 3. 安装可选依赖（语音、浏览器等）
pip install -r requirements-optional.txt

# 4. 安装 CLI 工具
pip install -e .

# 5. 创建配置文件
cp config-template.json config.json
# 编辑 config.json，填入 API Key 等配置

# 6. 启动服务
cow start

# 7. 查看日志
cow logs -f
```

### 7.2 源码阅读路径建议

对于二次开发者，建议按以下顺序阅读源码：

```
1. app.py                  ← 入口：理解整体启动流程
2. config.py               ← 配置：了解所有可配置项
3. common/const.py         ← 常量：了解支持的模型/通道类型
4. bridge/bridge.py        ← 路由：理解模型/语音/翻译的路由逻辑
5. bridge/agent_bridge.py  ← 适配：理解 Agent 如何对接 Channel
6. bridge/agent_initializer.py ← 初始化：理解 Agent 的完整创建流程
7. agent/protocol/agent.py      ← 核心：Agent 类的完整定义
8. agent/protocol/agent_stream.py ← 执行：多轮工具调用循环的实现
9. agent/tools/base_tool.py      ← 工具：理解工具接口规范
10. agent/tools/tool_manager.py  ← 管理：理解工具的加载和 MCP 集成
11. agent/skills/manager.py      ← 技能：理解技能系统的生命周期
12. agent/memory/manager.py      ← 记忆：理解三层记忆架构
13. agent/evolution/trigger.py   ← 进化：理解自我进化的触发和执行
14. channel/channel_factory.py   ← 通道：理解多平台接入机制
```

### 7.3 常见二次开发场景

#### 场景 1: 添加新的 LLM 提供商

以 "添加 Cohere" 为例：

```
步骤 1: 在 models/ 下创建 cohere/ 目录
步骤 2: 实现 models/cohere/cohere_bot.py
        - 继承或参考 OpenAICompatibleBot
        - 实现 reply(query, context) → Reply 方法
        - 处理流式/非流式两种模式

步骤 3: 在 models/bot_factory.py 中注册
        def create_bot(bot_type):
            ...
            elif bot_type == const.COHERE:
                from models.cohere.cohere_bot import CohereBot
                return CohereBot()

步骤 4: 在 common/const.py 中添加常量
        COHERE = "cohere"
        COHERE_COMMAND_R = "command-r"
        COHERE_COMMAND_R_PLUS = "command-r-plus"

步骤 5: 在 config.py 的 available_setting 中添加
        "cohere_api_key": "",
        "cohere_api_base": "https://api.cohere.com/v1",

步骤 6: 在 common/const.py 的模型列表中注册模型名

步骤 7: 如果需要 tool calling，参考 agent_bridge.py 中的
        add_openai_compatible_support() 方法进行增强
```

#### 场景 2: 添加新的 IM 通道

以 "添加 LINE" 为例：

```
步骤 1: 在 channel/ 下创建 line/ 目录

步骤 2: 实现 channel/line/line_channel.py
        from channel.channel import Channel

        class LineChannel(Channel):
            def startup(self):
                # 启动 LINE Bot 的消息监听循环
                # 收到消息后调用 self.handle(msg, context)
                pass

            def send(self, reply, context):
                # 将 Reply 转换为 LINE 消息格式并发送
                if reply.type == ReplyType.TEXT:
                    line_bot_api.reply_message(...)
                elif reply.type == ReplyType.IMAGE:
                    ...
                pass

步骤 3: 在 channel/channel_factory.py 中注册
        elif channel_type == "line":
            from channel.line.line_channel import LineChannel
            ch = LineChannel()

步骤 4: 在 app.py 的 _clear_singleton_cache() 中添加条目

步骤 5: 在 config.py 中添加 LINE 相关配置项（token, secret 等）
```

#### 场景 3: 开发自定义工具

```
步骤 1: 在 agent/tools/ 下创建 my_tool/ 目录

步骤 2: 创建 agent/tools/my_tool/my_tool.py
        from agent.tools.base_tool import BaseTool

        class MyTool(BaseTool):
            @property
            def name(self) -> str:
                return "my_tool"

            @property
            def description(self) -> str:
                return "这个工具用来做 XXX"

            @property
            def parameters(self) -> dict:
                return {
                    "type": "object",
                    "properties": {
                        "param1": {
                            "type": "string",
                            "description": "参数1的描述"
                        }
                    },
                    "required": ["param1"]
                }

            def execute(self, args: dict) -> ToolResult:
                param1 = args.get("param1")
                # 实现工具逻辑
                result = do_something(param1)
                return ToolResult(success=True, output=result)

步骤 3: 工具会被 ToolManager 自动发现并加载（无需手动注册）
        ToolManager 会扫描 agent/tools/ 下所有 Python 文件，
        自动发现 BaseTool 的子类并注册。
```

#### 场景 4: 创建自定义技能

**方式 A：手动创建**

```
1. 在 workspace/skills/ 下创建目录
   mkdir -p ~/cow/skills/my-skill

2. 创建 SKILL.md
---
name: my-skill
description: 我的自定义技能 - 自动生成周报
version: 1.0.0
tools:
  - bash
  - read
  - write
---

# 自动生成周报

当用户要求生成周报时：
1. 先读取 ~/cow/knowledge/log.md 获取本周活动
2. 用 bash 查询 git log 获取代码提交记录
3. 合并为结构化的周报 Markdown
4. 保存到 ~/cow/knowledge/reports/weekly-{date}.md
```

**方式 B：对话式创作**
```
直接对 Agent 说：
"帮我创建一个技能，它能自动从 git log 和对话记录中
 提取本周工作，生成格式化周报"

Agent 会调用 skill-creator 技能来创建它。
```

**方式 C：从市场安装**
```bash
cow skill search "周报"
cow skill install weekly-report
```

#### 场景 5: 开发插件

```
步骤 1: 在 plugins/ 下创建 my_plugin/ 目录

步骤 2: 创建 plugins/my_plugin/__init__.py
        from plugins.plugin import Plugin

        class MyPlugin(Plugin):
            """我的自定义插件"""

            def __init__(self):
                super().__init__()
                self.name = "MyPlugin"

            def handle_message(self, msg, context):
                """处理每条消息的钩子"""
                # 这里可以实现：
                # - 日志记录
                # - 内容过滤
                # - 消息增强
                # - 路由决策
                return msg  # 返回原始消息或修改后的消息

步骤 3: 插件会被 PluginManager 自动发现并加载
        启动时自动扫描 plugins/ 目录
```

#### 场景 6: 修改系统提示词 / Agent 人设

**方式 A：通过对话自然修改**（推荐）
```
直接告诉 Agent：
"你的名字是小牛，你是一个擅长 Python 开发的 AI 助手，
 回复风格简洁专业"

Agent 会自动将你的偏好写入 ~/cow/AGENT.md。
```

**方式 B：直接编辑文件**
```
编辑 ~/cow/AGENT.md    → 修改 Agent 人设
编辑 ~/cow/USER.md     → 修改用户信息
编辑 ~/cow/RULE.md     → 修改行为规则
编辑 ~/cow/BOOTSTRAP.md → 修改启动引导
```

#### 场景 7: 集成新的语音/翻译服务

**添加语音服务**：
```
1. 在 voice/<name>/ 下创建目录
2. 实现 voiceToText() 和/或 textToVoice() 方法
3. 在 voice/factory.py 中注册
```

**添加翻译服务**：
```
1. 在 translate/<name>/ 下创建目录
2. 实现 translate(text, from_lang, to_lang) → Reply 方法
3. 在 translate/factory.py 中注册
```

### 7.4 配置关键路径

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `channel_type` | `"web"` | 通道配置，支持多通道逗号分隔如 `"web, feishu, dingtalk"` |
| `bot_type` | 自动推断 | LLM 提供商，留空则根据 model 名称自动判断 |
| `model` | `"gpt-3.5-turbo"` | 模型名称，支持所有注册的模型 |
| `agent_workspace` | `"~/cow"` | 工作空间根目录 |
| `web_console` | `true` | 是否自动启动 Web 控制台 |
| `web_host` | `"127.0.0.1"` | Web 监听地址（部署服务器时设为 `0.0.0.0`） |
| `web_port` | `9899` | Web 端口 |
| `web_password` | `""` | Web 控制台密码（空为不设密码） |
| `agent_max_context_tokens` | `50000` | Agent 上下文 token 上限 |
| `agent_max_steps` | `100` | Agent 单次最大工具调用步数 |
| `cow_lang` | `"auto"` | 语言：`auto`/`en`/`zh` |
| `conversation_max_tokens` | `1000` | 会话上下文字符数上限 |
| `expires_in_seconds` | `3600` | 会话空闲过期时间 |

### 7.5 工作空间结构

CowAgent 的工作空间（默认 `~/cow/`）是持久化存储的核心：

```
~/cow/
├── AGENT.md                  # Agent 人格定义（系统提示词核心部分）
├── USER.md                   # 用户身份信息
├── RULE.md                   # 行为规则和限制
├── BOOTSTRAP.md              # 启动引导指令
├── MEMORY.md                 # 🔴 长期核心记忆（Deep Dream 蒸馏后的精华）
│
├── memory/                   # 记忆系统存储
│   ├── conversations.db      # 对话历史（SQLite）
│   ├── chunks.db             # 记忆分块（SQLite + FTS5）
│   └── embedding/            # 向量嵌入索引
│
├── knowledge/                # 个人知识库（Markdown Wiki）
│   ├── index.md              # 知识库索引
│   ├── log.md                # 知识更新日志
│   └── <category>/           # 按主题分目录
│       └── <topic>.md        # 具体知识文档
│
├── skills/                   # 自定义技能
│   ├── skills_config.json    # 技能配置（启用/禁用状态）
│   └── <skill-name>/         # 每个技能一个目录
│       └── SKILL.md          # 技能定义
│
├── sessions/                 # 多会话持久化
│   └── <session-id>/         # 每个会话的独立状态
│
├── scheduler/                # 定时任务存储
│   └── tasks.json            # 已调度的任务
│
└── .env                      # 环境变量（API 密钥等）
```

---

## 八、技术栈总览

| 层级 | 技术 |
|------|------|
| **语言** | Python 3.7+ |
| **前端** | 原生 HTML/JS/CSS + WebSocket + SSE（Web Console） |
| **存储** | SQLite + FTS5（记忆/会话/任务） |
| **向量嵌入** | 多厂商 API（OpenAI / DashScope / ZhipuAI 等） |
| **Web 框架** | 自研轻量 HTTP/WebSocket 服务（`channel/web/`） |
| **自动化** | Playwright（浏览器工具） |
| **CLI** | Click 框架 |
| **包管理** | pip + setuptools |
| **文档** | Mintlify |
| **消息协议** | MCP（Model Context Protocol）、SSE |
| **IM 协议** | itchat（微信）、飞书/钉钉/QQ/Telegram/Slack/Discord SDK |

---

## 九、版本历史摘要

| 版本 | 日期 | 关键特性 |
|------|------|---------|
| v2.0.0 | 2026.02.03 | 重大升级：超级 Agent 助手、多步任务规划、长期记忆、Skills 框架 |
| v2.0.5 | 2026.04.01 | Cow CLI、Skill Hub 开源、浏览器工具、企微机器人二维码接入 |
| v2.0.6 | 2026.04.14 | 知识库、Deep Dream 记忆蒸馏、智能上下文压缩、多会话 Web Console |
| v2.0.7 | 2026.04.22 | 内置图片生成（GPT Image 2）、新模型（Kimi K2.6、Claude Opus 4.7、GLM 5.1） |
| v2.0.8 | 2026.05.06 | 飞书通道全面升级（语音、流式、二维码接入）、DeepSeek V4、百度千帆 |
| v2.0.9 | 2026.05.22 | 模型管理面板、MCP 协议支持、持久化浏览器会话、新模型（GPT-5.5、Gemini 3.5 Flash） |
| v2.1.0 | 2026.06.01 | 国际化、新通道（Telegram/Discord/Slack/微信客服）、CLI 交互升级、MCP Streamable HTTP |
| v2.1.1 | 2026.06.09 | 🧬 自我进化、Web Console 升级（消息管理、并行会话优化）、MCP 增强、Python 3.13 支持 |

---

> 📖 **更多资源**：
> - 官方网站：[cowagent.ai](https://cowagent.ai)
> - 文档站：[docs.cowagent.ai](https://docs.cowagent.ai)
> - 技能市场：[skills.cowagent.ai](https://skills.cowagent.ai)
> - GitHub：[github.com/zhayujie/CowAgent](https://github.com/zhayujie/CowAgent)
> - 贡献指南：[CONTRIBUTING.md](../CONTRIBUTING.md)

---

*文档由 Claude 基于 CowAgent v2.1.1 源码分析生成。如有更新，请以官方文档为准。*
