# Prompt 提示词系统与工作空间 — 模块深度分析

> **所属项目**: CowAgent v2.1.1  
> **分析日期**: 2026-06-10  
> **模块路径**: `agent/prompt/`  
> **依赖模块**: `agent/skills/`, `agent/memory/`, `common/i18n.py`, `config.py`

---

## 1. 模块概述

Prompt 系统负责构建 Agent 的**完整系统提示词**——这是决定 Agent "是谁"、"能做什么"、"怎么做"的核心。工作空间（`~/cow/`）存储了 Agent 的持久化人格和规则文件。每次 Agent 对话前，`PromptBuilder` 都从磁盘重新读取这些文件，将它们与工具列表、技能列表、记忆摘要、运行时信息拼接为完整的系统提示词。

**核心职责**:
- **PromptBuilder**: 模块化构建系统提示词（11+ 个子模块）
- **工作空间初始化**: 首次运行时创建模板文件（AGENT.md/USER.md/RULE.md/BOOTSTRAP.md）
- **上下文文件加载**: 读取并验证工作空间文件
- **模板管理**: 中英文双语模板，自动检测语言
- **MEMORY.md 截断**: 防止长记忆文件占用过多上下文

---

## 2. 架构设计

### 2.1 系统提示词组成

```
完整系统提示词 = PromptBuilder.build()
├── 1. 基础人设 (base_persona / AGENT.md 覆盖)
├── 2. 用户身份 (USER.md)
├── 3. 行为规则 (RULE.md)
├── 4. 长期记忆 (MEMORY.md, 截断至200行/25KB)
├── 5. 启动引导 (BOOTSTRAP.md, 首次对话)
├── 6. 工具列表描述
├── 7. 技能列表描述 (SkillFormatter 格式化)
├── 8. 记忆管理器摘要
├── 9. 知识库操作规范 (来自 RULE.md)
├── 10. 运行时信息 (当前时间、系统信息)
└── 11. 额外后缀 (extra_system_suffix, 进化Agent使用)
```

### 2.2 工作空间文件

| 文件 | 用途 | 更新方式 |
|------|------|---------|
| `AGENT.md` | Agent 人格定义（名字、角色、性格、交流风格） | 首次对话填写 + Agent 自行 edit |
| `USER.md` | 用户静态身份（姓名、称呼、联系方式、生日） | 用户提供信息时更新 |
| `RULE.md` | 工作空间规则（目录结构、记忆系统、知识系统、安全） | Agent 学习教训后更新 |
| `MEMORY.md` | 长期记忆索引（自动加载到上下文） | Deep Dream + Agent edit |
| `BOOTSTRAP.md` | 首次启动引导脚本（完成后自动删除） | Agent 执行后 `rm` |

### 2.3 关键类

| 类/函数 | 文件 | 职责 |
|---------|------|------|
| `PromptBuilder` | `builder.py` | 模块化组装完整系统提示词 |
| `ContextFile` | `builder.py` | 上下文文件数据类 (path + content) |
| `ensure_workspace()` | `workspace.py` | 创建工作空间目录结构 + 模板文件 |
| `load_context_files()` | `workspace.py` | 读取并验证上下文文件 |
| `WorkspaceFiles` | `workspace.py` | 工作空间路径数据类 |

---

## 3. 源码深度解析

### 3.1 `PromptBuilder.build()` — 提示词组装

```python
class PromptBuilder:
    def build(self, base_persona, user_identity, tools, context_files, 
              skill_manager, memory_manager, runtime_info, **kwargs) -> str:
        sections = []
        
        # 1. 基础人设（用 AGENT.md 内容覆盖）
        persona = self._get_persona(base_persona, context_files)
        sections.append(persona)
        
        # 2. 用户身份
        if user_identity or self._has_user_file(context_files):
            sections.append(self._build_user_section(...))
        
        # 3. 行为规则 (RULE.md)
        sections.append(self._build_rules_section(context_files))
        
        # 4. 长期记忆 (MEMORY.md, 截断后)
        sections.append(self._build_memory_section(context_files, memory_manager))
        
        # 5. 工具列表
        if tools:
            sections.append(self._build_tools_section(tools))
        
        # 6. 技能列表
        if skill_manager:
            sections.append(self._build_skills_section(skill_manager))
        
        # 7. 运行时信息
        sections.append(self._build_runtime_section(runtime_info))
        
        return "\n\n".join(sections)
```

### 3.2 工作空间文件加载

```python
def load_context_files(workspace_dir, files_to_load=None):
    """按优先级加载上下文文件"""
    # 默认加载顺序（后面的可覆盖前面的）:
    # AGENT.md → USER.md → RULE.md → MEMORY.md → BOOTSTRAP.md
    
    for filename in files_to_load:
        content = read_file(filename)
        
        # 跳过空文件和模板占位符
        if not content or _is_template_placeholder(content):
            continue
        
        # MEMORY.md 截断（取最后200行或25KB）
        if filename == "MEMORY.md":
            content = _truncate_memory_content(content)
        
        # BOOTSTRAP.md 自动清理（AGENT.md已填写→删除）
        if filename == "BOOTSTRAP.md" and _is_onboarding_done(workspace_dir):
            os.remove(filepath)
            continue
        
        context_files.append(ContextFile(path=filename, content=content))
```

### 3.3 模板系统

所有模板文件都有中英文双语版本（通过 `i18n.get_language()` 自动选择）:

```
_get_agent_template()    → "我是谁？" / "Who am I?"
_get_user_template()     → "用户基本信息" / "User basics"
_get_rule_template()     → "工作空间规则" / "Workspace rules"
_get_memory_template()   → "长期记忆" / "Long-term memory"
_get_bootstrap_template() → "首次初始化引导" / "First-run onboarding"
```

---

## 4. 数据流

```
Agent.get_full_system_prompt()
  ├─ SkillManager.refresh_skills()
  ├─ load_context_files(workspace_dir)
  │   ├─ AGENT.md → 存在且有内容 → ContextFile
  │   ├─ USER.md → 同上
  │   ├─ RULE.md → 同上
  │   └─ MEMORY.md → 截断后加载
  └─ PromptBuilder.build(...)
      └─ 返回 ~5000-15000 字符的完整系统提示词
```

---

## 5. 配置与扩展点

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `agent_workspace` | `~/cow` | 工作空间根目录 |
| `cow_lang` | `auto` | 语言（决定模板语言） |
| `character_desc` | — | 旧版人设描述（被 AGENT.md 覆盖） |

### 扩展点

| 扩展点 | 说明 |
|--------|------|
| 新的上下文文件 | 在 `load_context_files()` 的 `files_to_load` 列表中添加 |
| 新的提示词模块 | 在 `PromptBuilder.build()` 中添加新的 section |
| 自定义模板 | 修改 `workspace.py` 中的模板函数 |

---

## 6. 二次开发规范

### 6.1 注意事项

1. **AGENT.md 覆盖 character_desc**: RULE.md 明确说明工作空间文件优先
2. **MEMORY.md 截断限制**: 200 行或 25KB，超出部分通过 `memory_search` 工具检索
3. **BOOTSTRAP.md 自动清理**: 首次对话完成后 Agent 执行 `rm BOOTSTRAP.md`
4. **模板占位符检测**: 含 `*(填写`、`*(filled during` 等模式的文件被视为未填写，不加载
5. **系统提示词每次重建**: `get_full_system_prompt()` 每次都从磁盘读取，文件变更即时生效

---

> **相关模块文档**:
> - [agent-protocol.md](agent-protocol.md) — Agent Core（提示词的消费者）
> - [agent-memory.md](agent-memory.md) — Memory（MEMORY.md 的写入者）
> - [agent-skills.md](agent-skills.md) — Skills（技能提示词的注入源）
