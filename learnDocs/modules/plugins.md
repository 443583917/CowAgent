# Plugins 插件系统 — 模块深度分析

> **所属项目**: CowAgent v2.1.1  
> **分析日期**: 2026-06-10  
> **模块路径**: `plugins/`  
> **依赖模块**: `bridge/`, `channel/`, `config.py`

---

## 1. 模块概述

Plugins（插件系统）提供**热插拔式**的消息处理中间件。插件在 Channel 收到消息后、交给 Bridge 处理前执行，可用于内容过滤、关键词触发、角色扮演、管理员命令等场景。

**核心职责**:
- 自动发现：扫描 `plugins/` 目录，自动加载所有插件
- 消息拦截：在 Bridge 处理前修改/过滤消息
- 内置插件：10 个官方插件覆盖常见需求

---

## 2. 架构设计

### 2.1 插件接口

```python
class Plugin:
    name: str = "BasePlugin"

    def handle_message(self, msg, context):
        """消息处理钩子——返回原始msg或修改后的msg，返回None则拦截"""
        return msg
```

### 2.2 内置插件

| 插件 | 目录 | 功能 |
|------|------|------|
| `banwords` | `banwords/` | 敏感词过滤 |
| `keyword` | `keyword/` | 关键词触发特定回复 |
| `role` | `role/` | 角色扮演（切换人设） |
| `godcmd` | `godcmd/` | 管理员命令（#help, #reset, #config） |
| `dungeon` | `dungeon/` | 文字冒险游戏 |
| `tool` | `tool/` | 工具类插件（时间、天气等） |
| `finish` | `finish/` | 会话结束处理 |
| `cow_cli` | `cow_cli/` | CLI 命令集成 |
| `hello` | `hello/` | 示例插件（开发模板） |
| `linkai` | `linkai/` | LinkAI 平台集成 |

### 2.3 PluginManager

```python
class PluginManager:
    def load_plugins(self):
        """扫描 plugins/ 目录，加载所有插件类"""
        for dir in listdir("plugins/"):
            if has_init_file(dir):
                module = importlib.import_module(f"plugins.{dir}")
                plugin = module.PluginClass()
                self.plugins.append(plugin)

    def handle_message(self, msg, context):
        """依次执行所有插件的 handle_message"""
        for plugin in self.plugins:
            msg = plugin.handle_message(msg, context)
            if msg is None:  # 插件拦截
                return None
        return msg
```

---

## 3. 数据流

```
Channel 收到消息
  → PluginManager.handle_message(msg, context)
    → banwords.handle_message()    # 敏感词过滤
    → keyword.handle_message()     # 关键词匹配
    → role.handle_message()        # 角色切换
    → godcmd.handle_message()      # 管理员命令
    → ...
  → 如果 msg 未被拦截 → Bridge.fetch_agent_reply(msg, context)
  → 如果 msg 被拦截 (返回 None) → 跳过 Bridge 处理
```

---

## 4. 扩展点与二次开发

### 4.1 插件开发模板

```python
# plugins/my_plugin/__init__.py
from plugins.plugin import Plugin

class MyPlugin(Plugin):
    def __init__(self):
        super().__init__()
        self.name = "MyPlugin"

    def handle_message(self, msg, context):
        # 示例: 记录所有消息到日志
        logger.info(f"[MyPlugin] {context.get('session_id')}: {msg}")
        return msg  # 不拦截，继续传递
```

### 4.2 注意事项

1. 插件按加载顺序执行，后执行的插件看到的是前一个插件可能已修改的消息
2. 返回 `None` 会拦截消息（不交给 Bridge），返回原 `msg` 则放行
3. 插件目录必须有 `__init__.py` 才会被自动发现
4. 插件在 `ChannelManager.start()` 的 `first_start` 阶段加载（只加载一次）

---

> **相关模块文档**:
> - [channel.md](channel.md) — Channel（插件的上游）
> - [bridge.md](bridge.md) — Bridge（插件的下游）
