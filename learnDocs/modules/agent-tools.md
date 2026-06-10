# Tools 工具系统与 MCP 集成 — 模块深度分析

> **所属项目**: CowAgent v2.1.1  
> **分析日期**: 2026-06-10  
> **模块路径**: `agent/tools/`  
> **依赖模块**: `common/`, `config.py`

---

## 1. 模块概述

Tools（工具系统）是 Agent 与外部世界交互的**原子操作层**。每个工具提供一个精确定义的能力——读文件、执行命令、搜索网络——Agent 在推理过程中根据需要选择和调用。工具系统还包括完整的 **MCP (Model Context Protocol)** 集成，将外部 MCP 生态的工具无缝引入 Agent。

**核心职责**:
- 14 个内置工具：覆盖文件 I/O、终端、搜索、浏览器、视觉、记忆、调度等
- MCP 协议集成：支持 stdio / SSE / Streamable HTTP 三种传输
- 工具热加载：MCP 工具随 `mcp.json` 变更自动重载
- 执行安全：重复调用检测、无限循环防护、失败重试策略
- 工具管理器单例：全局唯一，管理所有工具的注册和执行

---

## 2. 架构设计

### 2.1 工具分类

```
┌─────────────────────────────────────────────────────────────┐
│                     内置工具 (14个)                           │
├───────────────┬─────────────────────────────────────────────┤
│ 文件操作       │ read, write, edit, ls                       │
│ 终端执行       │ bash                                        │
│ 数据交互       │ send (文件发送)                              │
│ Web能力        │ web_search, web_fetch, browser              │
│ AI能力         │ vision (图像识别)                            │
│ 记忆检索       │ memory_get, memory_search                   │
│ 系统管理       │ scheduler (定时任务), env_config (环境变量)   │
│ 进化管理       │ evolution_undo (回滚)                        │
├───────────────┴─────────────────────────────────────────────┤
│                     MCP工具 (动态)                            │
│  stdio: npx/uvx/python 子进程                                │
│  SSE: HTTP 远程服务                                          │
│  Streamable HTTP: 双向流式 HTTP                              │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 BaseTool 接口

```python
class BaseTool:
    name: str           # 工具唯一名称
    description: str    # 工具描述（供 LLM 理解用途）
    params: dict        # JSON Schema 参数定义
    stage: ToolStage    # PRE_PROCESS (正常) / POST_PROCESS (自动执行)
    model: Any          # LLM 模型引用（工具可调用 LLM）

    def execute(self, params: dict) -> ToolResult: ...
    def execute_tool(self, params: dict) -> ToolResult:  # 包装异常
    def get_json_schema(self) -> dict:  # 返回 name+description+params
```

### 2.3 ToolManager 单例

```python
class ToolManager:
    _instance = None           # 全局单例

    def load_tools(tools_dir, config_dict):
        """扫描 agent/tools/ 下的所有 Python 模块，自动发现 BaseTool 子类"""

    def _load_mcp_tools():
        """从 mcp.json 加载 MCP 服务器，异步启动子进程"""

    def sync_mcp_into_agent(agent):
        """将新加载的 MCP 工具同步到 Agent 的工具列表（每轮 LLM 调用前）"""

    def refresh_mcp_if_changed():
        """检测 mcp.json 变更并热重载"""
```

### 2.4 MCP 架构

```
mcp.json / config.json (mcp_servers[])
    │
    ▼
ToolManager._load_mcp_tools()
    │
    ├─ stdio 传输: subprocess.Popen + JSON-RPC 2.0
    │   └─ McpClient (agent/tools/mcp/mcp_client.py)
    │       ├─ initialize → list_tools → 注册为 McpTool
    │       └─ call_tool → JSON-RPC request → response
    │
    ├─ SSE 传输: HTTP GET /sse (event stream) + POST /message
    │   └─ 同上 McpClient，使用 urllib + 事件流解析
    │
    └─ Streamable HTTP: HTTP POST (双向)
        └─ 同上 McpClient，无状态或会话模式
```

---

## 3. 工具详解

### 3.1 文件操作工具

| 工具 | 核心能力 | 安全措施 |
|------|---------|---------|
| `read` | 读取文件（支持分页 offset/limit） | 路径在工作空间内 |
| `write` | 覆盖写入文件 | 路径在工作空间内 |
| `edit` | 精确字符串替换（old_string → new_string） | 唯一匹配校验、路径在工作空间内 |
| `ls` | 目录列表 | 路径在工作空间内 |

### 3.2 `bash` — 终端执行

- 执行任意 shell 命令
- 支持 timeout（默认 120s）
- cwd 默认为工作空间
- 进化 Agent 使用时受 `_BashWorkspaceGuard` 限制

### 3.3 Web 工具

| 工具 | 实现 | 说明 |
|------|------|------|
| `web_search` | 搜索引擎 API | 返回标题+URL+摘要 |
| `web_fetch` | HTTP GET → HTML → Markdown | 最大 20KB，15 秒超时 |
| `browser` | Playwright 自动化 | 需先执行 `cow install-browser` 安装依赖 |

### 3.4 MCP 集成

**配置格式** (`mcp.json`):

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path"]
    },
    "my-api": {
      "type": "sse",
      "url": "http://localhost:8000/sse"
    }
  }
}
```

**热加载**: `ToolManager.refresh_mcp_if_changed()` 检查 `mcp.json` 的 mtime+sha256，变更时自动重启 MCP 子进程。

**MCP 工具适配器** (`mcp_tool.py`):
```python
class McpTool(BaseTool):
    def __init__(self, name, description, params, mcp_client):
        self.name = name
        self.description = description
        self.params = params
        self._client = mcp_client  # 持有 McpClient 引用

    def execute(self, args):
        return self._client.call_tool(self.name, args)
```

---

## 4. 数据流与工具链

### 4.1 工具执行流程

```
AgentStreamExecutor._call_llm_stream()
  → LLM 返回 tool_calls: [{name: "read", arguments: {path: "..."}}]
    → AgentStreamExecutor._execute_tool(tool_call)
      → 重复调用检测 (_check_consecutive_failures)
      → tool.execute_tool(arguments) → ToolResult
      → 结果截断 (当前轮50KB / 历史轮20KB)
      → 技能创建检测 (bash + init_skill.py → refresh_skills)
      → 追加 tool_result block 到 messages[]
```

### 4.2 MCP 工具同步

```
每轮 LLM 调用前:
AgentStreamExecutor._call_llm_stream()
  → ToolManager().sync_mcp_into_agent(self)
    → 检查 agent.tools 中是否已有所有 MCP 工具
    → 缺失的工具 → agent.add_tool(mcp_tool)
    → 已加载的 MCP 服务器 → tool.name → McpTool
```

---

## 5. 配置与扩展点

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `mcp_servers` | `[]` | MCP 服务器配置列表 |
| `agent_max_steps` | 100 | 单次最大工具调用步数 |
| `bash_timeout` | 120 | bash 命令超时（秒） |
| `web_search_api` | — | 搜索引擎配置 |

### 扩展点

| 扩展点 | 位置 | 说明 |
|--------|------|------|
| 新内置工具 | `agent/tools/<name>/` | 继承 `BaseTool`，自动被发现 |
| 新 MCP 传输 | `agent/tools/mcp/mcp_client.py` | 添加 `elif transport == "xxx"` 分支 |
| 自定义工具发现 | `ToolManager.load_tools()` | 修改扫描逻辑 |

---

## 6. 二次开发规范

### 6.1 工具开发模板

```python
# agent/tools/my_tool/my_tool.py
from agent.tools.base_tool import BaseTool, ToolResult

class MyTool(BaseTool):
    name = "my_tool"
    description = "工具描述——清晰说明做什么、何时使用"
    params = {
        "type": "object",
        "properties": {
            "param1": {"type": "string", "description": "参数说明"}
        },
        "required": ["param1"]
    }

    def execute(self, args: dict) -> ToolResult:
        try:
            result = do_work(args["param1"])
            return ToolResult.success(result)
        except Exception as e:
            return ToolResult.fail(str(e))
```

### 6.2 注意事项

1. **name 全局唯一**: 内置工具和 MCP 工具共享命名空间
2. **params 是 JSON Schema**: 必须符合 JSON Schema 规范
3. **execute 不能抛出异常**: 使用 `ToolResult.fail()` 返回错误
4. **大结果要截断**: AgentStreamExecutor 会截断 >50KB 的结果
5. **MCP 工具需要预热**: 启动时异步加载，首条消息可能不可用

---

> **相关模块文档**:
> - [agent-protocol.md](agent-protocol.md) — Agent Core（工具的执行者）
> - [agent-skills.md](agent-skills.md) — Skills（技能依赖底层工具）
> - [bridge.md](bridge.md) — Bridge（工具初始化在 agent_initializer）
