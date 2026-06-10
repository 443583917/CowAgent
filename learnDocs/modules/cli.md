# CLI 命令行工具 — 模块深度分析

> **所属项目**: CowAgent v2.1.1  
> **分析日期**: 2026-06-10  
> **模块路径**: `cli/`  
> **依赖模块**: `agent/tools/scheduler/`, `agent/skills/`, `agent/knowledge/`, `config.py`

---

## 1. 模块概述

CLI（命令行工具）提供 `cow` 命令，是 CowAgent 的**运维入口**。基于 Click 框架，支持服务控制、技能管理、知识库操作、进程管理等。

**核心职责**:
- 服务控制：`cow start|stop|restart|status|logs|update`
- 技能管理：`cow skill install|search|list`
- 知识库管理：`cow knowledge`
- 工具安装：`cow install-browser`

---

## 2. 命令体系

```
cow
├── start              # 启动 CowAgent 服务
├── stop               # 停止服务
├── restart            # 重启服务
├── status             # 查看运行状态
├── logs [-f]          # 查看日志（-f 实时跟踪）
├── update             # 拉取最新代码并重启
│
├── skill              # 技能管理
│   ├── install <name> # 安装技能（从 Skill Hub / GitHub / URL）
│   ├── search <kw>    # 搜索技能市场
│   └── list           # 列出已安装技能
│
├── knowledge          # 知识库管理（树状浏览/搜索/查看）
├── context            # 上下文管理
└── install-browser    # 安装浏览器自动化依赖
```

---

## 3. 源码解析

### 3.1 CLI 入口

```python
# cli/cli.py
import click

@click.group()
def main():
    """CowAgent CLI"""

@main.command()
def start():
    """Start CowAgent service"""
    # 检查 Python 版本
    # 检查配置文件
    # 启动子进程运行 app.py
    # 写入 PID 文件

@main.command()
def stop():
    """Stop CowAgent service"""
    # 读取 PID 文件
    # 发送 SIGTERM
    # 等待进程退出

@main.command()
def status():
    """Show service status"""
    # 检查 PID 文件和进程存活
    # 显示运行状态、端口、PID
```

### 3.2 技能安装命令

```python
# cli/commands/skill.py
def skill_install(name):
    """从多个源安装技能"""
    # 1. 尝试 Skill Hub API
    # 2. 尝试 GitHub (github.com/xxx/xxx)
    # 3. 尝试 URL 直接下载
    # 4. 解压到 workspace/skills/
    # 5. 调用 SkillManager.refresh_skills()
```

---

## 4. 扩展点

| 扩展点 | 实现方式 |
|--------|---------|
| 新命令 | `@main.command()` 装饰器添加到 `cli/cli.py` |
| 新技能安装源 | 在 `cli/commands/skill.py` 的安装流程中添加 |
| 自定义运维操作 | 添加新的子命令到 `cli/commands/` |

---

## 5. 二次开发规范

### 5.1 添加新 CLI 命令

```python
# cli/commands/my_command.py
@click.command()
@click.argument('name')
def my_command(name):
    """My custom command description"""
    # 实现逻辑
    print(f"Running my-command with {name}")

# 在 cli/cli.py 中注册:
from cli.commands.my_command import my_command
main.add_command(my_command)
```

### 5.2 注意事项

1. CLI 与运行中的 Agent 是**独立进程**，通过 PID 文件和信号通信
2. `cow update` 执行 `git pull` + 重启，需确保本地修改已提交或 stash
3. 技能安装是幂等的，重复安装会覆盖

---

> **相关模块文档**:
> - [agent-skills.md](agent-skills.md) — Skills（CLI 技能操作的目标）
> - [agent-knowledge.md](agent-knowledge.md) — Knowledge（CLI 知识库操作的目标）
