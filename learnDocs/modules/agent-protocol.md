# Agent Core 执行引擎 — 模块深度分析

> **所属项目**: CowAgent v2.1.1  
> **分析日期**: 2026-06-10  
> **模块路径**: `agent/protocol/`, `agent/chat/`  
> **依赖模块**: `agent/tools/`, `agent/skills/`, `agent/memory/`, `agent/prompt/`, `common/`, `config.py`

---

## 1. 模块概述

Agent Protocol 是 CowAgent 的**心脏**——它实现了完整的 Agent Loop（多轮推理 + 工具调用循环）。消息从 Channel 流入后，经过 Bridge 路由，最终由本模块驱动 LLM 逐步推理、调用工具、整合结果，直到任务完成或达到步数上限。

**核心职责**:
- 封装 Agent 实例：管理系统提示词、工具列表、技能管理器、记忆管理器
- 驱动多轮工具调用循环（AgentStreamExecutor）
- 流式输出管理（SSE 事件发射）
- 上下文生命周期管理（token 计算、智能裁剪、溢出恢复）
- 取消机制（线程安全的用户中断）
- 消息历史完整性保障（格式修复、孤儿 tool_result 修补）
- 会话持久化（SQLite 存储）

在整个 Agent Harness 架构中，Agent Protocol 位于 **Agent Core 层的中心**，向下调用 Models 层，向上被 Bridge 层驱动。

---

## 2. 架构设计

### 2.1 内部架构图

```
┌──────────────────────────────────────────────────────────────┐
│                      Agent (agent.py)                        │
│  ┌──────────────────────────────────────────────────────────┐│
│  │  系统提示词  │  工具列表  │  技能管理器  │  记忆管理器    ││
│  │  system_     │  tools[]  │  skill_      │  memory_       ││
│  │  prompt      │           │  manager     │  manager       ││
│  └──────────────────────────────────────────────────────────┘│
│                            │                                  │
│              run_stream(user_msg, on_event)                   │
│              ↓                                                │
│  ┌──────────────────────────────────────────────────────────┐│
│  │              AgentStreamExecutor (agent_stream.py)        ││
│  │                                                           ││
│  │  while turn < max_turns:                                  ││
│  │    ┌─────────────────────────────────────────────┐       ││
│  │    │ _call_llm_stream()                           │       ││
│  │    │  ├─ _prepare_messages()                      │       ││
│  │    │  ├─ sanitize_claude_messages()               │       ││
│  │    │  ├─ model.call_stream(request) → chunk loop  │       ││
│  │    │  │  ├─ cancel_event 探测 (每8个chunk)        │       ││
│  │    │  │  ├─ reasoning_content → 截断存储          │       ││
│  │    │  │  ├─ content delta → SSE emit              │       ││
│  │    │  │  └─ tool_calls buffer → 聚合              │       ││
│  │    │  ├─ JSON 解析 + json_repair                  │       ││
│  │    │  └─ 追加 assistant_msg 到 messages[]         │       ││
│  │    └─────────────────────────────────────────────┘       ││
│  │    ↓ 如果有 tool_calls                                    ││
│  │    ┌─────────────────────────────────────────────┐       ││
│  │    │ _execute_tool(tool_call)                     │       ││
│  │    │  ├─ 重复调用检测 (相同args 5次→停止)         │       ││
│  │    │  ├─ 连续失败检测 (相同tools 8次→critical)    │       ││
│  │    │  ├─ tool.execute_tool(args) → ToolResult      │       ││
│  │    │  ├─ 超大结果截断 (50KB current / 20KB hist)  │       ││
│  │    │  ├─ 技能创建检测 → 自动 refresh_skills()     │       ││
│  │    │  └─ 追加 tool_result 到 messages[]           │       ││
│  │    └─────────────────────────────────────────────┘       ││
│  │    turn += 1                                              ││
│  │                                                           ││
│  │  异常处理:                                                 ││
│  │   ├─ AgentCancelledError → _handle_cancelled()            ││
│  │   ├─ Context Overflow → _aggressive_trim_for_overflow()   ││
│  │   ├─ API Error → 重试 (最多3次,退避等待)                  ││
│  │   └─ 空响应 → 显式请求回复                                ││
│  └──────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘

辅助模块:
  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
  │ cancel.py        │  │ message_utils.py │  │ models.py        │
  │ CancelTokenReg-  │  │ sanitize_claude_ │  │ LLMRequest       │
  │ istry 单例       │  │ messages()       │  │ LLMModel (抽象)  │
  │ request_id→Event │  │ compress_turn_to │  │ ModelFactory     │
  └──────────────────┘  │ _text_only()     │  └──────────────────┘
                         └──────────────────┘
  ┌──────────────────┐  ┌──────────────────┐
  │ result.py        │  │ context.py       │
  │ AgentAction      │  │ (token计算)      │
  │ AgentActionType  │  │                  │
  │ ToolResult       │  │                  │
  │ AgentResult      │  │                  │
  └──────────────────┘  └──────────────────┘
```

### 2.2 类/组件关系

| 类 | 文件 | 职责 | 依赖 |
|----|------|------|------|
| `Agent` | `agent.py` | Agent 顶层封装：系统提示词、工具注册、技能/记忆集成、历史管理 | `LLMModel`, `BaseTool`, `SkillManager`, `MemoryManager`, `PromptBuilder` |
| `AgentStreamExecutor` | `agent_stream.py` | 多轮推理执行器：调用 LLM → 解析工具调用 → 执行工具 → 循环 | `Agent`, `LLMModel`, `BaseTool`, `message_utils` |
| `LLMModel` | `models.py` | LLM 抽象基类：`call()` / `call_stream()` | 由 `bridge/agent_bridge.py` 中 `AgentLLMModel` 实现 |
| `LLMRequest` | `models.py` | 请求数据类：messages、temperature、stream、tools、system | — |
| `CancelTokenRegistry` | `cancel.py` | 取消令牌注册表：request_id → threading.Event | — |
| `AgentCancelledError` | `cancel.py` | 取消异常：被 AgentStreamExecutor 捕获后优雅退出 | — |
| `AgentAction` | `result.py` | 行为记录：tool_use / thinking / final_response | — |
| `ToolResult` | `result.py` | 工具执行结果：status / result / execution_time | — |

---

## 3. 源码深度解析

### 3.1 `agent/protocol/agent.py` — Agent 类

**文件职责**: 定义 Agent 顶层实例，封装所有子系统引用，提供 `run_stream()` 入口。

**核心属性**:

```python
class Agent:
    system_prompt: str          # 系统提示词（可能是缓存的旧版本）
    model: LLMModel              # LLM 适配器实例
    tools: list[BaseTool]        # 已注册工具列表
    max_steps: int = 100         # 最大工具调用步数
    max_context_tokens: int      # 最大上下文 token 数
    context_reserve_tokens: int  # 预留给新响应的 token 缓冲区
    messages: list[dict]         # 持久化消息历史（Claude content-blocks 格式）
    messages_lock: threading.Lock # 消息历史线程安全锁
    memory_manager: MemoryManager # 记忆管理器引用
    skill_manager: SkillManager  # 技能管理器引用
    workspace_dir: str           # 工作空间路径
    runtime_info: dict           # 运行时信息（含动态时间）
    extra_system_suffix: str     # 附加到系统提示词末尾的指令（进化Agent使用）
```

**关键方法详解**:

**`get_full_system_prompt(skill_filter=None) → str`** (第106-142行)

这是系统提示词的**动态重建**方法。与构造函数中传入的 `system_prompt` 不同，此方法每次调用都从磁盘重新读取 `AGENT.md`/`USER.md`/`RULE.md`，刷新技能列表，并调用 `PromptBuilder.build()` 组装完整提示词。这样任何文件变更都立即生效，无需重启。

```python
def get_full_system_prompt(self, skill_filter=None) -> str:
    # 1. 刷新技能列表
    if self.skill_manager:
        self.skill_manager.refresh_skills()
    
    # 2. 重新读取上下文文件 (AGENT.md, USER.md, RULE.md)
    context_files = load_context_files(self.workspace_dir)
    
    # 3. 构建完整提示词
    builder = PromptBuilder(workspace_dir=self.workspace_dir, language=lang)
    full = builder.build(
        tools=self.tools,
        context_files=context_files,
        skill_manager=self.skill_manager,
        memory_manager=self.memory_manager,
        runtime_info=self.runtime_info,
    )
    
    # 4. 附加额外后缀（进化Agent使用）
    if self.extra_system_suffix:
        full = f"{full}\n\n{self.extra_system_suffix}"
    return full
```

**`run_stream(user_message, on_event, clear_history, skill_filter, cancel_event) → str`** (第383-482行)

Agent 的主入口方法。关键流程:

1. **清空历史**: 如果 `clear_history=True`，重置消息列表
2. **重建系统提示词**: 调用 `get_full_system_prompt()` 获取最新版本
3. **消息拷贝**: 从 `self.messages` 拷贝一份给 Executor（避免并发修改）
4. **创建 Executor**: 传入拷贝后的消息历史、工具列表、cancel_event
5. **执行**: `executor.run_stream(user_message)`
6. **同步回写**: 将 Executor 修改后的消息列表同步回 Agent（线程安全）
7. **后处理**: 执行所有 POST_PROCESS 阶段的工具
8. **异常恢复**: 如果 Executor 清空了消息列表（上下文溢出），同步清空 Agent 的消息

**Token 估算系统** (第227-288行):

Agent 内置了精确的 token 估算逻辑，无需调用外部 API:

```python
def _estimate_text_tokens(text: str) -> int:
    """CJK 字符 ~1.5 tokens/char, ASCII ~0.25 tokens/char"""
    non_ascii = sum(1 for c in text if ord(c) > 127)
    ascii_count = len(text) - non_ascii
    return int(non_ascii * 1.5 + ascii_count * 0.25) + 1

def _get_model_context_window(self) -> int:
    """根据模型名返回上下文窗口大小"""
    if 'claude-3' in model_name or 'claude-sonnet' in model_name:
        return 200000    # Claude: 200K
    elif 'gpt-4' in model_name:
        if 'turbo' in model_name: return 128000
        elif '32k' in model_name: return 32000
        else: return 8000
    elif 'deepseek' in model_name:
        return 64000      # DeepSeek: 64K
    elif 'gemini' in model_name:
        if '2.0' in model_name: return 2000000  # Gemini 2.0: 2M
        else: return 1000000  # Gemini 1.5: 1M
    return 128000  # 默认保守值
```

### 3.2 `agent/protocol/agent_stream.py` — AgentStreamExecutor 类

**文件职责**: 实现多轮工具调用循环的完整逻辑，是项目中最复杂、代码量最大的单文件（~1700行）。

**核心循环逻辑** (`run_stream()`, 第340-703行):

```
while turn < max_turns:
    # 1. 检查取消信号（每轮开始）
    self._check_cancelled()
    
    # 2. 调用 LLM（流式）
    assistant_msg, tool_calls = self._call_llm_stream(retry_on_empty=True)
    
    # 3. 无工具调用 → 任务完成
    if not tool_calls:
        if not assistant_msg and turn > 1:
            # 空响应 → 注入提示要求 LLM 显式回复
            self.messages.append({"role": "user", "content": [{"text": "请向用户说明..."}]})
            assistant_msg, tool_calls = self._call_llm_stream(retry_on_empty=False)
            # 移除注入的提示
        break
    
    # 4. 执行工具
    for tool_call in tool_calls:
        self._check_cancelled()  # 工具间也可取消
        result = self._execute_tool(tool_call)
        # 检查: 文件发送 / critical_error / 超大结果截断
        # 构建 tool_result block
    
    # 5. 追加 tool_result 到消息历史
    # 使用 try/finally 确保即使异常也追加（保持消息格式完整性）
    
    # 6. 无限循环检测: 相同工具+参数成功 3 次 → 注入停止提示

# 达到 max_turns → 注入提示强制 LLM 总结
```

**关键设计决策**:

1. **`try/finally` 保护 tool_result** (第577-628行): 即使工具执行异常，也必须追加 `tool_result` 到消息历史。否则 Claude/OpenAI 会因为 `tool_use` 没有对应的 `tool_result` 而报错。

2. **重复调用检测** (第332-338行 `_record_tool_result` + 第269-330行 `_check_consecutive_failures`):
   - 相同工具+相同参数 **5 次** → 停止（无论成功失败，防无限循环）
   - 相同工具+相同参数 **连续失败 3 次** → 停止
   - 相同工具（不同参数）**连续失败 8 次** → **critical error，终止整个会话**
   - 相同工具（不同参数）连续失败 6 次 → 停止并提示换方法

3. **LLM 错误恢复策略** (`_call_llm_stream()`, 第705-1116行):
   - **Context Overflow**: 先尝试激进裁剪（`_aggressive_trim_for_overflow`），失败则清空历史 + 清空 DB + 抛出友好错误
   - **Message Format Error**: 直接清空历史 + 清空 DB（防止脏数据重新加载）
   - **Rate Limit (429)**: 退避重试（30s/45s/60s）
   - **其他可重试错误** (timeout, 500, 502, 503): 退避重试（2s/4s/6s）
   - **空响应**: 重试一次（`retry_on_empty` 标志）

4. **上下文管理三级策略** (`_trim_messages()`, 第1538-1694行):
   - **第一级**: 截断历史 tool_result（30K → 20K chars）
   - **第二级**: 轮次限制（超过 `max_context_turns` 时保留后半）
   - **第三级**: Token 限制 — 少于 5 轮时**压缩为纯文本**（保留语义），5+ 轮时**丢弃前半部分**
   - 被丢弃的轮次会 **flush 到每日记忆** + 注入**上下文摘要**到保留轮次的第一条消息

5. **取消机制** (`_check_cancelled()`, 第139-146行):
   - 每轮开始检查
   - 工具间检查
   - LLM 流式输出期间每 8 个 chunk 检查一次
   - 取消时 `_handle_cancelled()` 会合成缺失的 `tool_result` blocks（保证消息历史有效），然后追加 `_(Cancelled by user)_` 标记

### 3.3 `agent/protocol/cancel.py` — 取消令牌注册表

**文件职责**: 提供线程安全的请求取消机制。Web Console 的 "Cancel" 按钮和 `/cancel` 命令最终都通过此模块工作。

**设计要点**:
- `CancelTokenRegistry` 是**模块级单例**（`_registry`）
- 支持两种取消粒度：
  - `cancel_request(request_id)`: 取消单个请求
  - `cancel_session(session_id)`: 取消某个会话的所有进行中请求
- `threading.Event` 作为信号载体，Agent 循环在安全点轮询 `event.is_set()`
- 请求完成后自动 `unregister()` 清理

### 3.4 `agent/protocol/message_utils.py` — 消息完整性保障

**核心函数**:

**`sanitize_claude_messages(messages)`**: 修复 Clode API 要求的 `tool_use → tool_result` 配对。在以下场景自动修复:
- 末尾有孤儿 `tool_use`（无对应 `tool_result`）→ 合成错误 tool_result
- 连续的 assistant `tool_use` 消息之间缺少 user `tool_result` → 插入合成结果
- 重复的 `tool_result` IDs → 去重保留最后一个

**`compress_turn_to_text_only(turn)`**: 将包含工具调用链的轮次压缩为纯文本格式，用于上下文 token 超限时的**无损语义保留**。

### 3.5 `agent/chat/session_service.py` — 会话持久化

**文件职责**: 将 Agent 的 `messages` 列表持久化到 SQLite，支持:
- 多会话并行（每个 `session_id` 独立存储）
- 会话切换/恢复
- 会话删除
- 会话列表展示

---

## 4. 数据流与工具链

### 4.1 输入输出

**输入**:
- `user_message: str` — 用户消息文本
- `on_event: Callable` — 事件回调函数（可选，用于 SSE 流式推送）
- `cancel_event: threading.Event` — 取消信号（可选）
- `skill_filter: list[str]` — 技能过滤列表（可选）
- `clear_history: bool` — 是否清空历史（默认 False）

**输出**:
- `str` — LLM 最终回复文本
- 通过 `on_event` 回调流式发射事件:
  - `agent_start` / `agent_end`
  - `message_start` / `message_update` / `message_end`
  - `tool_execution_start` / `tool_execution_end`
  - `reasoning_update`（思考内容增量）
  - `error` / `agent_cancelled`

### 4.2 调用链

```
Channel.handle_message(msg)
  → Bridge.fetch_agent_reply(query, context, on_event)
    → AgentBridge.agent_reply(query, context, on_event)
      → AgentInitializer.initialize_agent(session_id)  # 首次或缓存失效时
        → Agent(...) 创建
      → agent.run_stream(user_message, on_event, cancel_event=cancel_event)
        → AgentStreamExecutor(...) 创建
        → executor.run_stream(user_message)
          ┌─ while turn < max_turns:
          │   _call_llm_stream()
          │     → model.call_stream(LLMRequest)
          │     → 解析 chunk stream
          │   _execute_tool(tool_call)
          │     → tool.execute_tool(args)
          └─ return final_response
      → Reply 对象返回给 Channel
```

### 4.3 与其他模块的交互

| 被调用模块 | 调用位置 | 调用方式 |
|-----------|---------|---------|
| `agent/tools/tool_manager.py` | `_call_llm_stream()` L733-737 | `ToolManager().sync_mcp_into_agent(self)` — 每轮同步 MCP 工具 |
| `agent/tools/base_tool.py` | `_execute_tool()` L1180 | `tool.execute_tool(arguments) → ToolResult` |
| `agent/skills/manager.py` | `Agent.get_full_system_prompt()` L118 | `self.skill_manager.refresh_skills()` |
| `agent/prompt/builder.py` | `Agent.get_full_system_prompt()` L127-134 | `PromptBuilder.build(...)` |
| `agent/memory/manager.py` | `_trim_messages()` L1670-1681, `_call_llm_stream()` L957-961 | `memory_manager.flush_memory(...)` |
| `agent/memory/conversation_store.py` | `_clear_session_db()` L1707-1710 | `store.clear_session(session_id)` |
| `agent/protocol/message_utils.py` | `_validate_and_fix_messages()` L1269 | `sanitize_claude_messages(self.messages)` |
| `common/i18n.py` | 多处 | `_t(cn_text, en_text)` — 双语错误消息 |
| `config.py` | 多处 | `conf().get(key)` — 读取运行时配置 |

---

## 5. 配置与扩展点

### 5.1 相关配置项

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `agent_max_steps` | 100 | 单次运行最大工具调用步数 |
| `agent_max_context_tokens` | 50000 | Agent 上下文 token 上限 |
| `agent_max_context_turns` | 20 | 最大保留对话轮次 |
| `enable_thinking` | false | 是否启用深度思考模式（控制 reasoning 内容的处理） |
| `debug` | false | 调试模式（打印完整系统提示词和消息） |
| `conversation_max_tokens` | 1000 | 会话上下文字符数上限（旧版配置，Agent 模式不完全依赖此项） |
| `temperature` | 0.9 | LLM 温度参数 |
| `request_timeout` | 180 | API 请求超时（秒） |
| `timeout` | 120 | 重试超时窗口（秒） |

### 5.2 扩展点

| 扩展点 | 位置 | 说明 |
|--------|------|------|
| 新的事件类型 | `_emit_event()` | 在 `agent_stream.py` 中通过 `self._emit_event(type, data)` 发射自定义事件 |
| 新的工具阶段 | `BaseTool.stage` | 支持 `PRE_PROCESS` 和 `POST_PROCESS`，可在 `run_stream()` 前后插入逻辑 |
| 自定义错误恢复 | `_call_llm_stream()` 异常处理 | 可在 `is_context_overflow` / `is_message_format_error` 检测后插入自定义恢复逻辑 |
| Token 估算优化 | `_estimate_text_tokens()` | 可替换为更精确的 tiktoken 等库 |
| 模型上下文窗口 | `_get_model_context_window()` | 添加新模型时需在此注册上下文窗口大小 |

---

## 6. 二次开发规范

### 6.1 代码规范

- **消息格式**: 统一使用 Claude content-blocks 格式 `[{"type": "text", "text": "..."}, {"type": "tool_use", ...}]`
- **线程安全**: 访问 `self.messages` 时必须持有 `self.messages_lock`
- **异常处理**: 工具执行异常不应中断 Agent 循环，使用 `try/finally` 保证 tool_result 完整性
- **取消检查**: 在任何 I/O 或长时间操作前调用 `self._check_cancelled()`
- **事件发射**: 每个状态转换都应发射对应事件（开始/更新/结束）
- **日志级别**: 使用 `logger.info` 记录关键流转，`logger.warning` 记录可恢复错误，`logger.error` 记录不可恢复错误

### 6.2 开发流程

**添加新的工具阶段**:
1. 在 `agent/tools/base_tool.py` 的 `ToolStage` 枚举中添加新阶段
2. 在 `Agent.run_stream()` 中（第479行附近）添加阶段执行逻辑
3. 确保新阶段的工具不影响消息历史完整性

**修改 Agent Loop 行为**:
1. 在 `AgentStreamExecutor.run_stream()` 中修改循环逻辑
2. 任何新增的 `self.messages.append()` 必须保持 Claude content-blocks 格式
3. 修改后必须验证 `tool_use → tool_result` 配对仍正确

### 6.3 常见开发场景

**场景 1: 添加 Agent 执行中间件（如请求日志/审计）**

```python
# 在 AgentStreamExecutor.run_stream() 的 while 循环开头插入:
def run_stream(self, user_message):
    # ... existing code ...
    while turn < self.max_turns:
        # --- 自定义中间件 ---
        self._emit_event("custom_audit", {
            "turn": turn, 
            "message_count": len(self.messages),
            "timestamp": time.time()
        })
        # --- 原有逻辑 ---
        self._check_cancelled()
        # ...
```

**场景 2: 自定义 token 计算**

```python
# 替换 Agent._estimate_text_tokens() 为 tiktoken 实现:
def _estimate_text_tokens(self, text: str) -> int:
    import tiktoken
    enc = tiktoken.encoding_for_model(self.model.model or "gpt-4")
    return len(enc.encode(text))
```

**场景 3: 添加 LLM 响应的后处理**

```python
# 在 _call_llm_stream() 返回前 (L1116附近):
def _post_process_response(self, content: str) -> str:
    """对所有 LLM 文本回复进行后处理"""
    # 敏感词过滤
    # 格式化修正
    # 语言检测和转换
    return processed_content

# 在 full_content 赋值后调用:
full_content = self._post_process_response(full_content)
```

### 6.4 注意事项

1. **消息历史完整性是底线**: 任何时候都不能破坏 `tool_use → tool_result` 的配对关系，否则下一次 LLM 调用会直接报错
2. **不要在循环中裁剪上下文**: 上下文裁剪（`_trim_messages()`）只在 `run_stream()` 开始时执行一次，循环内部不裁剪。因为循环中裁剪会破坏当前 tool_use/tool_result 链
3. **取消处理要完整**: `AgentCancelledError` 必须在 `finally` 块中处理好消息历史，否则下次请求可能因格式错误而失败
4. **空响应不代表错误**: LLM 可能在工具执行完毕后返回空文本（认为工具结果已经足够），此时需要注入提示要求显式回复
5. **MCP 工具的热同步**: 每轮 LLM 调用前都会 `sync_mcp_into_agent()`，确保新加载的 MCP 工具立即可用

---

## 7. 性能与安全

### 性能考虑

- **Token 估算开销**: 每次 `_trim_messages()` 都会遍历所有消息并估算 token，对于长会话（100+ 轮）可能耗时数十毫秒
- **消息拷贝**: `run_stream()` 中 `self.messages.copy()` 在长会话中可能产生较大的内存分配
- **工具结果截断**: 当前轮 50KB、历史轮 20KB 的截断限制可在 `MAX_CURRENT_TURN_RESULT_CHARS` 和 `MAX_HISTORY_RESULT_CHARS` 中调整
- **失败历史**: 只保留最近 50 条记录，避免内存膨胀

### 安全注意事项

- **取消令牌泄漏**: 确保每次 Agent 运行结束后调用 `CancelTokenRegistry.unregister()`
- **消息注入**: LLM 不能直接修改消息历史（`messages` 列表仅由 AgentStreamExecutor 管理）
- **会话隔离**: 不同 `session_id` 的消息历史完全隔离

### 资源管理

- **消息列表生命周期**: `messages` 在 Agent 实例生命周期内持续增长，由 `_trim_messages()` 定期裁剪
- **会话 DB**: 每次上下文溢出或格式错误后，对应的 session DB 数据被清空以防脏数据重新加载

---

## 8. 总结

### 优势
- **健壮的 Agent Loop**: 多级异常恢复（溢出→裁剪→清空→友好提示），容错性极高
- **完善的取消机制**: 线程安全、多粒度（单请求/全会话）、消息历史自动修复
- **智能上下文管理**: 三级策略（截断→轮次限制→Token限制），结合记忆 flush 和摘要注入
- **流式优先**: 全流程流式，用户体验好

### 局限
- **Token 估算非精确**: 使用启发式算法而非 tokenizer，可能偏差较大
- **消息格式限制**: 仅支持 Claude content-blocks 格式，OpenAI 格式需要在 Bot 层转换
- **单线程执行**: 工具串行执行，不支持并行工具调用

### 未来演进方向
- 支持并行工具调用（多个独立工具同时执行）
- 引入精确的 tokenizer 库
- 支持更多消息格式（原生 OpenAI multi-content 格式）

---

> **相关模块文档**:
> - [agent-tools.md](agent-tools.md) — 工具系统与 MCP 集成
> - [agent-memory.md](agent-memory.md) — Memory 三层记忆系统
> - [agent-skills.md](agent-skills.md) — Skills 技能系统
> - [agent-prompt.md](agent-prompt.md) — Prompt 提示词系统
> - [bridge.md](bridge.md) — Bridge 桥接层
