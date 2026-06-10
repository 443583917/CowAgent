# Voice 语音与 Translate 翻译服务 — 模块深度分析

> **所属项目**: CowAgent v2.1.1  
> **分析日期**: 2026-06-10  
> **模块路径**: `voice/`, `translate/`  
> **依赖模块**: `bridge/`, `config.py`

---

## 1. 模块概述

Voice 和 Translate 模块为 CowAgent 提供**语音交互**和**多语言翻译**能力。两个模块都使用工厂模式，通过 `Bridge` 统一路由。

**核心职责**:
- **ASR (语音→文本)**: 13 种语音识别服务
- **TTS (文本→语音)**: 13 种语音合成服务
- **Translate (翻译)**: 2 种翻译服务

---

## 2. 架构设计

### 2.1 语音工厂

```python
# voice/factory.py
def create_voice(voice_type) -> Voice:
    if voice_type == "openai":    return OpenaiVoice()
    elif voice_type == "azure":   return AzureVoice()
    elif voice_type == "google":  return GoogleVoice()
    elif voice_type == "baidu":   return BaiduVoice()
    elif voice_type == "dashscope": return DashscopeVoice()
    elif voice_type == "zhipu":   return ZhipuVoice()
    # ... 13 种
```

### 2.2 Voice 接口

```python
class Voice:
    def voiceToText(self, voice_file) -> Reply:
        """语音转文本"""
        raise NotImplementedError

    def textToVoice(self, text) -> Reply:
        """文本转语音"""
        raise NotImplementedError
```

### 2.3 翻译接口

```python
class Translator:
    def translate(self, text, from_lang="", to_lang="en") -> Reply:
        raise NotImplementedError
```

---

## 3. 已支持的提供商

| 类别 | 提供商 |
|------|--------|
| ASR + TTS | OpenAI, Azure, Google, Baidu, Ali(DashScope), Xunfei, Tencent, ZhipuAI |
| TTS Only | MiniMax, MiMo, ElevenLabs, Edge TTS, pytts |
| 全功能代理 | LinkAI (统一接入) |
| 翻译 | Baidu Translate, Youdao Translate |

---

## 4. 数据流

```
# ASR
Channel 收到语音消息
  → Bridge.get_bot("voice_to_text")
    → voice_factory.create_voice(voice_type)
    → voice.voiceToText(audio_file) → Reply(text)
  → 继续传递给 Bridge.fetch_agent_reply(text)

# TTS
Agent 返回文本回复
  → Bridge.get_bot("text_to_voice")
    → voice.textToVoice(text) → Reply(audio)
  → Channel.send(audio)

# 翻译
Bridge.get_bot("translate")
  → translator.translate(text, from_lang, to_lang)
```

---

## 5. 配置与扩展点

| 配置项 | 说明 |
|--------|------|
| `voice_to_text` | ASR 提供商 |
| `text_to_voice` | TTS 提供商 |
| `translate` | 翻译提供商 |
| 各厂商 API Key | `open_ai_api_key`, `azure_voice_key` 等 |

### 扩展点

| 扩展点 | 实现方式 |
|--------|---------|
| 新语音服务 | 1. `voice/<name>/` 创建 2. 实现 `Voice` 接口 3. `voice/factory.py` 注册 |
| 新翻译服务 | 同上，在 `translate/factory.py` 注册 |

---

## 6. 二次开发规范

### 注意事项

1. **API Key 自动选择**: 当 `voice_to_text` 为空时，Bridge 自动选择第一个已配 Key 的提供商
2. **LinkAI 优先级**: 如果启用了 LinkAI，voice/translate 都会自动路由到 LinkAI
3. **语音文件格式**: 各厂商支持的音频格式不同，需在适配器中处理转换
4. **TTS 缓存**: 相同文本的 TTS 结果建议缓存

---

> **相关模块文档**:
> - [bridge.md](bridge.md) — Bridge（语音/翻译的路由者）
> - [models.md](models.md) — Models（与语音模型共享 API Key 体系）
