# Skills 技能系统 — 模块深度分析

> **所属项目**: CowAgent v2.1.1  
> **分析日期**: 2026-06-10  
> **模块路径**: `agent/skills/`, `skills/`  
> **依赖模块**: `agent/skills/` 自身包含完整的加载/管理/格式化系统

---

## 1. 模块概述

Skills（技能）是 CowAgent 中比 Tool 更高层的**工作流抽象**。如果说 Tool 是原子操作（"读文件"、"执行命令"），那么 Skill 就是组合工作流（"自动生成周报"、"管理知识库"）。技能通过 `SKILL.md` 文件定义，可以被 Agent 在需要时"调用"——Agent 读取技能的 Markdown 指令，然后使用底层工具逐步执行。

**核心职责**:
- 技能发现：从内置目录和自定义目录递归扫描 `SKILL.md`
- 技能注册：同名自定义技能覆盖内置技能
- 技能格式化：将技能描述注入到 Agent 的系统提示词中
- 技能配置：启用/禁用管理（`skills_config.json`）
- 技能安装：从 Skill Hub / GitHub / ClawHub / URL 获取
- 对话式创作：通过 `skill-creator` 技能自然语言生成新技能

---

## 2. 架构设计

### 2.1 技能生命周期

```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Discovery  │ →  │    Load      │ →  │   Register   │ →  │   Activate   │
│  扫描目录     │    │ 解析SKILL.md │    │ 合并到 registry│   │ 注入提示词    │
│  SkillLoader │    │ SkillLoader  │    │ SkillManager │    │ SkillFormatter│
└─────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
       ↑                                                          │
       │  refresh_skills() 循环刷新                                │
       └──────────────────────────────────────────────────────────┘
```

### 2.2 SKILL.md 规范

```markdown
---
name: my-skill
description: 描述这个技能做什么
version: 1.0.0
tools:
  - bash
  - read
  - write
disable-model-invocation: false
user-invocable: true
---

# 技能指令

当用户要求 XXX 时，请按以下步骤执行：
1. 读取配置文件
2. 执行数据提取
3. 生成报告
```

### 2.3 关键类

| 类 | 文件 | 职责 |
|----|------|------|
| `SkillManager` | `manager.py` | 技能生命周期管理：加载、刷新、启用/禁用、安装 |
| `SkillLoader` | `loader.py` | 从目录递归扫描和解析 SKILL.md |
| `SkillFormatter` | `formatter.py` | 将技能列表格式化为 LLM 可理解的提示词 |
| `Skill` / `SkillEntry` | `types.py` | 数据类型定义 |
| `parse_frontmatter()` | `frontmatter.py` | YAML frontmatter 解析 |

### 2.4 加载优先级

```
custom_dir (workspace/skills/) — 最高优先级，覆盖同名内置技能
builtin_dir (skills/)          — 基础优先级
```

---

## 3. 源码深度解析

### 3.1 `SkillLoader._load_skills_recursive()` — 递归发现

**发现规则**:
1. 如果目录包含 `SKILL.md`，将其作为一个技能加载，**不递归**子目录（子目录是该技能的内部资源）
2. 如果目录不包含 `SKILL.md`，递归扫描子目录
3. 跳过 `.` 开头的目录、`node_modules`、`__pycache__`、`venv`、`.git`
4. 根目录下的 `.md` 文件（非 `README.md`）也被视为技能

### 3.2 `SkillLoader._load_skill_from_file()` — 单文件解析

```python
def _load_skill_from_file(self, file_path, source):
    # 1. 读取文件内容
    content = f.read()
    
    # 2. 解析 YAML frontmatter
    frontmatter = parse_frontmatter(content)
    
    # 3. 提取 name（默认用父目录名）
    name = frontmatter.get('name', parent_dir_name)
    
    # 4. 提取 description（技能的核心标识，必须有）
    description = frontmatter.get('description', '')
    if not description:
        return LoadSkillsResult(skills=[], diagnostics=["No description"])
    
    # 5. 解析 disable-model-invocation 标志
    #    true = 模型不会主动调用，只能由用户用 /skill 触发
    
    # 6. 创建 Skill 对象
    return Skill(name=name, description=description, file_path=file_path, ...)
```

### 3.3 `SkillManager` — 技能管理

**`skills_config.json` 结构**:

```json
{
  "my-skill": {
    "name": "my-skill",
    "description": "...",
    "source": "custom",
    "enabled": true
  }
}
```

**安装流程**（通过 `service.py`）:
1. 从 Skill Hub API 获取技能元数据
2. 下载技能文件到 `workspace/skills/<name>/`
3. 调用 `refresh_skills()` 重新加载
4. 更新 `skills_config.json`

---

## 4. 数据流与工具链

### 4.1 调用链

```
Agent.get_full_system_prompt()
  → SkillManager.refresh_skills()
    → SkillLoader.load_all_skills(builtin_dir, custom_dir)
      → load_skills_from_dir() × 2
        → _load_skills_recursive()
          → _load_skill_from_file() × N
  → SkillManager.build_skills_prompt(skill_filter)
    → SkillFormatter.format_skill_entries_for_prompt(entries)
      → 返回格式化的技能描述文本
  → PromptBuilder.build(skill_manager=...)
    → 将技能提示注入系统提示词
```

---

## 5. 配置与扩展点

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `agent_workspace` | `~/cow` | 自定义技能存放于 `<workspace>/skills/` |

### 扩展点

| 扩展点 | 说明 |
|--------|------|
| 新安装源 | `service.py` 中添加新的安装源（如 GitLab、Gitee） |
| 技能发现规则 | `SkillLoader._load_skills_recursive()` 中修改扫描策略 |
| 提示词格式化 | `SkillFormatter` 中自定义技能在系统提示词中的呈现方式 |

---

## 6. 二次开发规范

### 6.1 SKILL.md 编写规范

- **必须有 description**: 无 description 的技能会被静默跳过
- **name 唯一**: 同名自定义技能会覆盖内置技能
- **tools 声明**: 声明技能需要使用的工具列表
- **指令清晰**: Markdown 正文包含清晰的分步指令

### 6.2 常见开发场景

**场景: 创建新技能**
```bash
mkdir -p ~/cow/skills/my-weekly-report
# 编写 SKILL.md
# 重启或通过 Agent 对话触发 refresh_skills()
```

**场景: 对话式创建**
```
对 Agent 说: "帮我创建一个自动生成周报的技能"
Agent 会使用 skill-creator 技能来引导创建过程
```

### 6.3 注意事项

1. 技能加载失败不会报错，只会记录到 diagnostics
2. 内置技能（`skills/`）在启动时被同步到工作空间，同名自定义技能覆盖内置版本
3. `disable-model-invocation: true` 的技能只能通过 `/skill` 命令手动触发

---

> **相关模块文档**:
> - [agent-tools.md](agent-tools.md) — 工具系统（技能依赖的底层工具）
> - [agent-protocol.md](agent-protocol.md) — Agent Core（技能提示词的注入点）
> - [agent-evolution.md](agent-evolution.md) — 进化系统（进化过程中改进技能）
