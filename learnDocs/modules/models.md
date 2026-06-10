# Models 模型适配层 — 模块深度分析

> **所属项目**: CowAgent v2.1.1  
> **分析日期**: 2026-06-10  
> **模块路径**: `models/`  
> **依赖模块**: `common/const.py`, `config.py`, `bridge/`

---

## 1. 模块概述

Models（模型适配层）封装了 15+ LLM 厂商的 API 差异，对外暴露统一的 `reply()` / `reply_stream()` 接口。通过 `bot_factory.create_bot()` 工厂模式，上层代码无需关心底层是 Claude 还是 GPT。

**核心职责**:
- 多厂商适配：每个厂商一个独立子目录
- 统一接口：`Bot.reply(query, context) → Reply`
- 流式支持：`Bot.reply_stream(query, context) → Iterator[Reply]`
- Tool Calling：OpenAI 兼容协议的工具调用增强
- 六维独立路由：Chat / Vision / ImageGen / ASR / TTS / Embedding

---

## 2. 架构设计

### 2.1 厂商适配模式

```
bot_factory.create_bot(bot_type) → Bot实例
├── openai/        → OpenAIBot       (OpenAI 原生)
├── chatgpt/       → ChatGPTBot      (OpenAI 兼容通用)
├── gemini/        → GeminiBot       (Google Gemini)
├── claude/        → ClaudeBot       (Anthropic Claude, 通过Gemini代理)
├── deepseek/      → DeepSeekBot     (DeepSeek)
├── qianfan/       → QianfanBot      (百度千帆/ERNIE)
├── doubao/        → DoubaoBot       (字节豆包)
├── moonshot/      → MoonshotBot     (Kimi/Moonshot)
├── zhipuai/       → ZhipuAIBot     (智谱GLM, 原生tool calling)
├── dashscope/     → DashscopeBot    (阿里通义千问)
├── minimax/       → MiniMaxBot      (MiniMax)
├── mimo/          → MiMoBot         (小米MiMo)
├── xunfei/        → XunfeiBot       (讯飞星火)
├── linkai/        → LinkAIBot       (LinkAI平台)
├── modelscope/    → ModelScopeBot   (ModelScope)
└── baidu/         → BaiduBot        (百度文心)
```

### 2.2 OpenAI 兼容协议增强

```python
# bridge/agent_bridge.py
def add_openai_compatible_support(bot_instance):
    """为任意 OpenAI 兼容 bot 动态添加 tool calling 能力"""
    class EnhancedBot(bot_instance.__class__, OpenAICompatibleBot):
        def get_api_config(self):
            return {
                'api_key': conf().get("open_ai_api_key"),
                'api_base': conf().get("open_ai_api_base"),
                'model': conf().get("model"),
            }
    bot_instance.__class__ = EnhancedBot  # 运行时替换类
```

### 2.3 自动路由逻辑

```python
# bridge/bridge.py Bridge.__init__()
if model.startswith("claude"):   → CLAUDEAPI
if model.startswith("gemini"):   → GEMINI
if model.startswith("deepseek"): → DEEPSEEK
if model.startswith("glm"):      → ZHIPU_AI
# ... 20+ 条路由规则
```

---

## 3. Bot 接口规范

```python
class Bot:
    def reply(self, query: str, context: Context = None) -> Reply:
        """非流式回复"""
        raise NotImplementedError

    def reply_stream(self, query: str, context: Context = None) -> Iterator[Reply]:
        """流式回复（可选实现）"""
        raise NotImplementedError
```

---

## 4. 数据流

```
Bridge.get_bot("chat")
  → bot_factory.create_bot(bot_type) → Bot实例
    → Bot.reply(query, context)
      → 构建厂商特有的 HTTP 请求
      → 解析厂商特有的 HTTP 响应
      → 返回统一 Reply
```

---

## 5. 配置与扩展点

### 5.1 各厂商配置

每种模型通过 `config.json` 中的专属字段配置 API key 和 base URL:
- `open_ai_api_key` / `open_ai_api_base`
- `claude_api_key` / `claude_api_base`
- `gemini_api_key` / `gemini_api_base`
- `deepseek_api_key` / `deepseek_api_base`
- ...

### 5.2 扩展点

| 扩展点 | 实现方式 |
|--------|---------|
| 新厂商 | 1. `models/<vendor>/` 创建 2. 实现 `reply()` 3. `bot_factory.py` 注册 4. `const.py` 添加常量 |
| Tool calling 增强 | 参考 `add_openai_compatible_support()` 动态混入 |
| 自定义模型参数 | 覆写 `get_api_config()` 返回模型特有参数 |

---

## 6. 二次开发规范

### 6.1 新厂商适配模板

```python
# models/my_vendor/my_vendor_bot.py
class MyVendorBot:
    def __init__(self):
        self.api_key = conf().get("my_vendor_api_key")
        self.api_base = conf().get("my_vendor_api_base", "https://api.myvendor.com/v1")
        self.model = conf().get("model")

    def reply(self, query, context=None):
        # 1. 构建请求
        headers = {"Authorization": f"Bearer {self.api_key}"}
        payload = {"model": self.model, "messages": [{"role": "user", "content": query}]}
        
        # 2. 发送请求
        resp = requests.post(f"{self.api_base}/chat/completions", json=payload, headers=headers)
        
        # 3. 解析为统一 Reply
        return Reply(ReplyType.TEXT, resp.json()["choices"][0]["message"]["content"])
```

### 6.2 注意事项

1. **Tool Calling 需要特殊处理**: 非 OpenAI 厂商的 tool calling 格式可能完全不同
2. **流式响应的 chunk 格式**: 每家厂商的 SSE chunk 格式不同
3. **API 重试**: 建议内置退避重试（参考 `AgentStreamExecutor._call_llm_stream`）
4. **模型名称常量**: 必须在 `common/const.py` 中注册

---

> **相关模块文档**:
> - [bridge.md](bridge.md) — Bridge（bot 的使用者和路由者）
> - [agent-protocol.md](agent-protocol.md) — Agent Core（LLMModel 的使用者）
