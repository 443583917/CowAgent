# App 入口、Config 配置中心与 Common 公共模块 — 模块深度分析

> **所属项目**: CowAgent v2.1.1  
> **分析日期**: 2026-06-10  
> **模块路径**: `app.py`, `config.py`, `common/`  
> **依赖模块**: 所有其他模块

---

## 1. 模块概述

这三个组件构成了 CowAgent 的**基础骨架**——应用启动、配置管理和公共工具。

**核心职责**:
- **app.py**: 应用入口，编排整个启动流程
- **config.py**: 全局配置中心（800+ 行），所有可配置项的声明和默认值
- **common/**: 全局常量、日志、国际化、工具函数、单例装饰器

---

## 2. app.py — 应用入口

### 2.1 启动流程

```python
def run():
    # 1. 加载配置
    load_config()

    # 2. 注册信号处理 (SIGINT / SIGTERM)
    sigterm_handler_wrap(signal.SIGINT)
    sigterm_handler_wrap(signal.SIGTERM)

    # 3. 解析通道类型 ("--cmd" → terminal, 否则从配置读取)
    channel_names = _parse_channel_type(conf().get("channel_type", "web"))

    # 4. 自动启动 Web Console（除非显式禁用）
    if web_console_enabled and "web" not in channel_names:
        channel_names.append("web")

    # 5. 同步内置技能到工作空间
    _sync_builtin_skills()

    # 6. 预热 MCP 工具（后台加载子进程）
    _warmup_mcp_tools()

    # 7. 预热调度器
    _warmup_scheduler()

    # 8. 启动 ChannelManager（插件加载 + 所有通道）
    _channel_mgr = ChannelManager()
    _channel_mgr.start(channel_names, first_start=True)

    # 9. 保持主线程存活
    while True:
        time.sleep(1)
```

### 2.2 ChannelManager 设计

```python
class ChannelManager:
    def start(channel_names, first_start):
        """为每个通道创建实例并启动在独立 daemon 线程"""
        # 1. 创建通道实例
        for name in channel_names:
            ch = channel_factory.create_channel(name)
            self._channels[name] = ch

        # 2. 首次启动时加载插件
        if first_start:
            PluginManager().load_plugins()

        # 3. Web Console 先启动（保证日志清晰）
        # 4. 其他通道间隔 0.1s 启动

    def stop(channel_name=None):
        """停止通道（支持单个/全部）"""

    def add_channel(name) / remove_channel(name):
        """运行时动态添加/移除通道"""
```

### 2.3 内置技能同步

```python
def _sync_builtin_skills():
    """每次启动时将项目 skills/ 同步到 workspace/skills/"""
    for name in os.listdir("skills/"):
        if is_skill_dir(name):
            # 删除旧版本 → 复制新版本
            shutil.rmtree(dst)
            shutil.copytree(src, dst)
```

---

## 3. config.py — 配置中心

### 3.1 配置体系

```python
# available_setting 字典声明所有可配置项（800+ 行）
available_setting = {
    # 基础配置
    "cow_lang": "auto",
    "model": "gpt-3.5-turbo",
    "bot_type": "",

    # API 密钥
    "open_ai_api_key": "",
    "claude_api_key": "",
    "gemini_api_key": "",

    # Web 配置
    "web_host": "127.0.0.1",
    "web_port": 9899,
    "web_password": "",

    # Agent 配置
    "agent_workspace": "~/cow",
    "agent_max_steps": 100,
    "agent_max_context_tokens": 50000,
    "agent_max_context_turns": 20,

    # 通道触发
    "single_chat_prefix": ["bot", "@bot"],
    "group_chat_prefix": ["@bot"],

    # ... 200+ 配置项
}

# 配置加载优先级:
# 1. config.json 文件
# 2. 环境变量（自动映射）
# 3. 默认值 (available_setting)
```

### 3.2 配置迁移

`config.py` 包含自动迁移逻辑，处理旧版配置格式到新版的转换（如 `chatgpt-on-wechat` → `CowAgent` 的重命名）。

---

## 4. common/ — 公共模块

### 4.1 `const.py` — 全局常量

定义所有模型名、厂商类型、通道名常量：
```python
# 模型常量
CLAUDE_FABLE_5 = "claude-fable-5"
GPT_54 = "gpt-5.4"
GEMINI_35_FLASH = "gemini-3.5-flash"
DEEPSEEK_V4_PRO = "deepseek-v4-pro"

# 厂商常量
OPENAI = "openai"
CLAUDEAPI = "claudeAPI"
GEMINI = "gemini"
DEEPSEEK = "deepseek"

# 通道常量
FEISHU = "feishu"
DINGTALK = "dingtalk"
TELEGRAM = "telegram"
```

### 4.2 `i18n.py` — 国际化

```python
def get_language() -> str:
    """返回当前语言: "zh" / "en" / "auto" """
    lang = conf().get("cow_lang", "auto")
    if lang == "auto":
        return detect_system_lang()
    return lang

def t(cn_text: str, en_text: str) -> str:
    """根据当前语言选择中文或英文文本"""
    return cn_text if get_language() == "zh" else en_text
```

### 4.3 `singleton.py` — 单例装饰器

```python
def singleton(cls):
    instances = {}
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get_instance
```

### 4.4 `log.py` — 日志系统

统一的日志管理，支持文件轮转和级别控制。

### 4.5 `utils.py` — 工具函数

`expand_path()`: 将 `~` 扩展为用户主目录。

---

## 5. 配置与扩展点

### 5.1 关键配置项速查

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `channel_type` | `"web"` | 通道类型 |
| `bot_type` | 自动 | LLM 提供商 |
| `model` | `"gpt-3.5-turbo"` | 模型名 |
| `agent_workspace` | `"~/cow"` | 工作空间 |
| `agent_max_steps` | 100 | 最大步数 |
| `web_console` | true | Web 控制台 |
| `web_port` | 9899 | Web 端口 |
| `cow_lang` | `"auto"` | 语言 (auto/en/zh) |

### 5.2 扩展点

| 扩展点 | 位置 |
|--------|------|
| 新配置项 | `config.py:available_setting` |
| 新常量 | `common/const.py` |
| 新启动步骤 | `app.py:run()` |
| 新语言 | `common/i18n.py` |

---

## 6. 二次开发规范

### 6.1 添加新配置项

```python
# 在 config.py 的 available_setting 中添加:
"my_feature_enabled": True,  # 是否启用我的功能
"my_feature_option": "default_value",  # 功能选项

# 在代码中使用:
from config import conf
if conf().get("my_feature_enabled"):
    do_something(conf().get("my_feature_option"))
```

### 6.2 注意事项

1. **config.py 不要导入 agent 模块**: 避免循环依赖（config 是最底层模块）
2. **所有配置项必须有默认值**: 确保 `conf().get(key)` 永远不返回 None
3. **API 密钥不要硬编码**: 通过环境变量或 `config.json` 传入
4. **信号处理要兼容**: 同时处理 SIGINT (Ctrl+C) 和 SIGTERM (kill)
5. **daemon 线程**: Channel 线程设置为 daemon=True，主线程退出时自动回收

---

> **相关模块文档**:
> - 本模块是所有其他模块的基础，被所有模块依赖
> - [channel.md](channel.md) — ChannelManager 启动的通道
> - [bridge.md](bridge.md) — 由 app.py 预热调度器时初始化
> - [agent-tools.md](agent-tools.md) — MCP 工具预热的接收者
