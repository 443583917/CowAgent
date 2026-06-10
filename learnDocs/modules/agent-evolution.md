# Evolution 自我进化系统 — 模块深度分析

> **所属项目**: CowAgent v2.1.1  
> **分析日期**: 2026-06-10  
> **模块路径**: `agent/evolution/`  
> **依赖模块**: `agent/protocol/`, `agent/tools/`, `agent/skills/`, `agent/memory/`, `bridge/`, `channel/`

---

## 1. 模块概述

Evolution（自我进化）是 CowAgent v2.1.1 的标志性创新功能。它在**后台静默运行**，自动审查用户的空闲对话，发现问题后主动改进技能、整理记忆、补充知识、跟进未完成任务。Agent 真正做到了"越用越聪明"。

**核心职责**:
- 空闲扫描：后台线程每 60s 扫描所有活跃会话
- 信号累积：对话轮次 ≥ 6 或上下文压力 > 80% 时触发进化审查
- 隔离执行：使用受限工具集的独立 Agent 审查对话记录
- 安全写入：硬工程防护确保不会修改项目内置技能
- 变更检测：对比工作空间快照，只有真正修改了文件才通知用户
- 回滚支持：每次修改前自动备份，支持 `evolution_undo` 工具回滚

---

## 2. 架构设计

### 2.1 触发→执行→通知流程

```
┌─────────────────────────────────────────────────────────────┐
│  trigger.py: 后台扫描线程 (daemon, 每60s)                     │
│                                                              │
│  for each session in agent_bridge.agents:                    │
│    ├─ 检查 evolution enabled?                                │
│    ├─ 检查足够信号?                                           │
│    │   ├─ turns >= min_turns (默认6轮)  OR                   │
│    │   └─ 上下文压力 > 80% token预算                          │
│    ├─ 检查足够空闲? idle >= idle_seconds (默认10min)          │
│    └─ 所有条件满足 → run_evolution_for_session()              │
│                                                              │
│  并发控制: 最多 2 个进化任务同时运行 (_MAX_CONCURRENT=2)       │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│  executor.py: 进化执行                                        │
│                                                              │
│  1. 构建 transcript (最近未审查的消息, 截断至12KB)             │
│  2. 备份工作空间 (MEMORY.md + 每日记忆 + 可编辑技能 + AGENT.md)│
│  3. 创建隔离审查 Agent:                                       │
│     - 相同 model (复用用户配置的LLM)                           │
│     - 受限工具集: {read, write, edit, ls, bash, memory_*}    │
│     - 硬工程防护: _WorkspaceWriteGuard 限制写入工作空间内      │
│     - 硬工程防护: _BashWorkspaceGuard 限制命令在工作空间执行   │
│  4. 快照工作空间 (path → mtime/size)                          │
│  5. 运行审查 Agent                                            │
│  6. 解析输出:                                                 │
│     - [SILENT] → 无变更，返回 False                           │
│     - 无文件变更 → 静默，返回 False（防啰嗦）                   │
│     - 有文件变更 → 记录进化日志 + 注入 [EVOLUTION] 标记        │
│  7. 推送到用户通道                                             │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│  通知: _notify_user()                                         │
│  - 通过 create_channel() 创建通道实例                          │
│  - 向用户推送进化摘要                                          │
│  - Web 通道特殊处理: 合成 request_id + session 映射             │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 安全防护设计

| 防护层 | 实现 | 说明 |
|--------|------|------|
| 工具白名单 | `_ALLOWED_TOOLS = {read, write, edit, ls, bash, memory_search, memory_get}` | 审查 Agent 只有 7 个工具可用 |
| 写入路径防护 | `_WorkspaceWriteGuard` | write/edit 工具被包装，路径必须位于工作空间内 |
| 命令路径防护 | `_BashWorkspaceGuard` | bash 工具被包装，命令的 cwd 固定为工作空间，检测并拒绝绝对路径和 `..` 遍历 |
| 内置技能保护 | `_builtin_skill_names()` | 项目的 `skills/` 目录被识别为受保护，不会备份也不会被编辑 |
| 并发限制 | `_MAX_CONCURRENT = 2` | 最多同时运行 2 个进化任务 |
| 文件变更检测 | `_workspace_snapshot()` + `_workspace_changed()` | 只有文件真正变更时才通知，避免空跑 |
| 备份回滚 | `backup.py` | 每次修改前保存备份，`evolution_undo` 工具可恢复 |

### 2.3 类/组件关系

| 文件 | 职责 |
|------|------|
| `trigger.py` | 后台扫描线程：信号判断、触发决策、cursor 管理 |
| `executor.py` | 进化执行器：transcript 构建、隔离 Agent 创建、安全防护、变更检测、通知推送 |
| `prompts.py` | 提示词模板：`EVOLUTION_SYSTEM_PROMPT`、`SILENT_TOKEN`、`build_review_user_message()` |
| `config.py` | 进化配置：`enabled`/`min_turns`/`idle_seconds`/`max_steps` |
| `backup.py` | 备份管理：`create_backup()` 在进化前保存文件副本 |
| `record.py` | 进化记录：`append_session_evolution()` 写入进化日志 |

---

## 3. 源码深度解析

### 3.1 `trigger.py` — 触发机制

**信号模型**（`note_user_turn()` 在每次用户消息后被调用）:

```python
# 存储在 Agent 实例上的轻量属性:
agent._evo_last_active   = time.time()  # 最后活跃时间
agent._evo_turns         = 6            # 自上次进化后的轮次计数
agent._evo_channel_type  = "web"        # 来源通道
agent._evo_receiver      = "user123"    # 推送目标
agent._evo_done_msg_count = 150         # 已审查的消息数(cursor)
```

**触发条件**（`_scan_once()`）：

```python
def _scan_once(agent_bridge, cfg):
    for session_id, agent in sessions:
        last_active = agent._evo_last_active
        turns = agent._evo_turns
        
        # 足够信号 = 足够轮次 OR 上下文压力
        enough_signal = turns >= cfg.min_turns or _context_pressure_reached(agent)
        
        # 足够空闲 = 距最后活跃 >= idle_seconds
        idle = now - last_active
        if enough_signal and idle >= cfg.idle_seconds:
            # 触发进化前重置基线，避免重复触发
            agent._evo_turns = 0
            run_evolution_for_session(agent_bridge, session_id, ...)
```

### 3.2 `executor.py` — 执行引擎

**关键设计决策**:

1. **Cursor 机制**（第344-350行）: 使用 `_evo_done_msg_count` 作为游标，只审查上次进化后的**新消息**。避免长会话中反复审查旧内容。

2. **硬工程防护 vs 提示词防护**（`_WorkspaceWriteGuard`、`_BashWorkspaceGuard`）:
   - 不仅依赖提示词限制，更在工具层面硬编码路径检查
   - `_WorkspaceWriteGuard.execute()` 在执行前检查目标路径是否在工作空间内
   - `_BashWorkspaceGuard.execute()` 在执行前解析命令中的路径引用，检测逃逸尝试

3. **双重去噪**（第448-465行）:
   - 第一重：输出以 `[SILENT]` 开头 → 无变更
   - 第二重：即使 LLM 返回了文本，但实际没有文件变更 → 静默（防啰嗦）

4. **进化后 cursor 更新**（第476-479行）: 进化注入的 `[EVOLUTION]` 消息也会被计入 cursor，避免下次扫描重新触发。

### 3.3 `prompts.py` — 提示词模板

进化审查 Agent 的系统提示词包含:
- Agent 的完整身份和职责描述
- 工作空间结构说明
- 判断标准：什么时候应该 [SILENT]，什么时候应该行动
- 可执行的操作：更新记忆、改进技能、补充知识、跟进任务
- 输出格式要求

---

## 4. 数据流与工具链

### 4.1 输入输出

**输入**:
- `session_id`: 会话标识
- `channel_type`: 通道类型（用于推送通知）
- `receiver`: 推送目标（用户 ID）
- `idle_minutes`: 空闲分钟数（日志用）

**输出**:
- `bool`: 是否产生了实际变更
- 副作用: 工作空间文件修改、进化日志写入、用户通知推送

### 4.2 调用链

```
[用户发送消息]
  → AgentBridge.agent_reply()
    → trigger.note_user_turn(agent, channel_type, receiver)
      → agent._evo_turns += 1

[后台扫描线程 - 每60s]
  → trigger._scan_once()
    → executor.run_evolution_for_session()
      → AgentBridge.create_agent(restricted_tools)  # 隔离审查Agent
      → review_agent.run_stream(user_msg)           # 执行审查
      → _workspace_changed()                        # 检测变更
      → _notify_user()                              # 推送通知
```

---

## 5. 配置与扩展点

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `evolution_enabled` | true（新安装默认开启） | 总开关 |
| `evolution_min_turns` | 6 | 最少对话轮次 |
| `evolution_idle_minutes` | 10 | 最少空闲分钟 |
| `evolution_max_steps` | 20 | 审查 Agent 最大步数 |

### 扩展点

| 扩展点 | 位置 | 说明 |
|--------|------|------|
| 新增审查维度 | `prompts.py` | 在 `EVOLUTION_SYSTEM_PROMPT` 中添加新的审查维度 |
| 新增安全防护 | `executor.py` | 在 `_guard_tools()` 中添加新的工具包装器 |
| 自定义触发条件 | `trigger.py` | 在 `_scan_once()` 中添加新的触发条件 |
| 通知渠道自定义 | `executor.py:_notify_user()` | 自定义通知格式和推送逻辑 |

---

## 6. 二次开发规范

### 6.1 常见开发场景

**场景: 添加新的进化操作类型（如自动归档旧对话）**

```python
# 1. 在 prompts.py 的 EVOLUTION_SYSTEM_PROMPT 中添加操作说明
# 2. 将新工具添加到 _ALLOWED_TOOLS 集合
_ALLOWED_TOOLS = {"read", "write", "edit", "ls", "bash", "memory_search", "memory_get", "archive"}

# 3. 在 executor.py 的 _WATCH_SUBDIRS 中添加新文件类型
_WATCH_SUBDIRS = ("MEMORY.md", "AGENT.md", "skills", "knowledge", "output", "archive")
```

### 6.2 注意事项

1. **进化 Agent 使用用户配置的相同模型**，会产生 API 费用
2. **并发限制为 2**，多会话时部分会话可能跳过本轮扫描
3. **内置技能永远不应该被进化修改**
4. **进化注入的消息会计入 cursor**，避免循环触发

---

> **相关模块文档**:
> - [agent-protocol.md](agent-protocol.md) — Agent Core（run_stream 的使用者）
> - [agent-memory.md](agent-memory.md) — Memory（进化更新记忆文件）
> - [agent-skills.md](agent-skills.md) — Skills（进化改进技能）
> - [bridge.md](bridge.md) — Bridge（create_agent 的提供者）
