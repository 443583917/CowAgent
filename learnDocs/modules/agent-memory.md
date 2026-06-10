# Memory 三层记忆系统 — 模块深度分析

> **所属项目**: CowAgent v2.1.1  
> **分析日期**: 2026-06-10  
> **模块路径**: `agent/memory/`  
> **依赖模块**: `agent/memory/embedding/`, `common/`, `config.py`

---

## 1. 模块概述

Memory 模块实现了 CowAgent 的**三层长期记忆架构**，让 Agent 在多次对话之间保持对用户的持续认知。它是 Agent Harness 中最具创新性的子系统之一。

**核心职责**:
- **三层记忆架构**: 上下文记忆（会话内）→ 每日记忆（中期聚合）→ 核心记忆 MEMORY.md（长期保留）
- **混合检索**: 关键词全文搜索（SQLite FTS5）+ 向量语义搜索
- **Deep Dream 蒸馏**: 夜间后台任务，将散落的每日记忆精炼为结构化的 MEMORY.md 条目
- **记忆自动写入**: Agent 在对话中可通过工具主动检索和写入记忆
- **对话历史持久化**: 完整的 JSON 格式对话存储和恢复

在 Agent Harness 架构中，Memory 模块为 Agent Core 提供 **长期上下文感知能力**。

---

## 2. 架构设计

### 2.1 三层记忆模型

```
┌───────────────────────────────────────────────────────────┐
│ Layer 1: 上下文记忆 (Context Memory)                       │
│ • 存储位置: Agent.messages[] 内存中                        │
│ • 生命周期: 当前会话                                        │
│ • 容量限制: max_context_tokens (默认50K) / max_context_turns│
│ • 管理方式: AgentStreamExecutor._trim_messages()自动裁剪    │
├───────────────────────────────────────────────────────────┤
│ Layer 2: 每日记忆 (Daily Memory)                           │
│ • 存储位置: SQLite memory/chunks.db (含FTS5全文索引)        │
│ • 生命周期: 按天聚合，长期保留                               │
│ • 写入时机: 上下文裁剪时 flush_to_memory / Deep Dream       │
│ • 检索方式: 关键词 (FTS5) + 向量 (Embedding) 混合搜索       │
├───────────────────────────────────────────────────────────┤
│ Layer 3: 核心记忆 (Core Memory / MEMORY.md)                │
│ • 存储位置: ~/cow/MEMORY.md Markdown文件                    │
│ • 生命周期: 永久保留                                        │
│ • 写入时机: Deep Dream 夜间蒸馏                              │
│ • 格式: 结构化 Markdown（含日期/标签/内容/关联）             │
└───────────────────────────────────────────────────────────┘
```

### 2.2 内部组件关系

```
MemoryManager (manager.py)
├── MemoryConfig (config.py)           # 配置管理
├── MemoryStorage (storage.py)         # SQLite + FTS5 引擎
│   ├── add_chunk() / add_chunks()     # 写入分块
│   ├── keyword_search()               # FTS5 关键词搜索
│   ├── get_recent()                   # 获取最近记忆
│   └── delete_chunk()                 # 删除
├── TextChunker (chunker.py)           # 智能文本分块
│   ├── max_tokens: int                # 块大小
│   └── overlap_tokens: int            # 块间重叠
├── EmbeddingProvider (embedding/provider.py)  # 向量嵌入
│   ├── embed(texts) → List[Vector]    # 批量生成嵌入
│   └── 支持: OpenAI / DashScope / ZhipuAI
├── EmbeddingCache (embedding/)        # 嵌入缓存
├── MemoryFlushManager (summarizer.py) # Deep Dream 蒸馏
└── ConversationStore (conversation_store.py)  # 对话存储
```

### 2.3 类/组件关系

| 类 | 文件 | 职责 |
|----|------|------|
| `MemoryManager` | `manager.py` | 高层统一接口：初始化存储/嵌入/分块器，提供增删查操作 |
| `MemoryConfig` | `config.py` | 配置管理：DB路径、分块大小、嵌入设置 |
| `MemoryStorage` | `storage.py` | SQLite 引擎：建表、FTS5索引、关键词/混合搜索、CRUD |
| `TextChunker` | `chunker.py` | 文本分块：按 token 预算切分，块间可重叠 |
| `EmbeddingProvider` | `embedding/provider.py` | 嵌入抽象：支持 OpenAI/DashScope/ZhipuAI，含缓存和状态管理 |
| `MemoryFlushManager` | `summarizer.py` | Deep Dream 编排：获取每日记忆→LLM总结→写入 MEMORY.md |
| `ConversationStore` | `conversation_store.py` | 对话存储：将 Agent.messages[] 持久化到 SQLite，支持恢复 |

---

## 3. 源码深度解析

### 3.1 `manager.py` — MemoryManager 类

**初始化流程**:

```python
class MemoryManager:
    def __init__(self, config=None, embedding_provider=None, llm_model=None):
        self.config = config or get_default_memory_config()
        self.storage = MemoryStorage(db_path)         # SQLite 引擎
        self.chunker = TextChunker(                    # 文本分块器
            max_tokens=self.config.chunk_max_tokens, 
            overlap_tokens=self.config.chunk_overlap_tokens
        )
        self.embedding_provider = embedding_provider   # 嵌入提供者
        
        # 关键设计: embedding_provider 由调用者 (agent_initializer) 负责创建
        # 当为 None 时降级为纯关键词搜索，不会在此处重新初始化
```

**核心方法**:

```python
def add_memory(self, text: str, metadata: dict = None) -> str:
    """添加记忆条目"""
    # 1. 分块
    chunks = self.chunker.chunk_text(text)
    # 2. 生成嵌入向量（如果有 embedding_provider）
    embeddings = self.embedding_provider.embed(chunks) if self.embedding_provider else None
    # 3. 写入存储
    chunk_ids = self.storage.add_chunks(chunks, embeddings, metadata)
    return chunk_ids

def search(self, query: str, k: int = 5, hybrid: bool = True) -> list:
    """混合搜索记忆"""
    if hybrid and self.embedding_provider:
        # 向量搜索
        query_embedding = self.embedding_provider.embed([query])[0]
        vector_results = self.storage.vector_search(query_embedding, k=k*2)
        # 关键词搜索
        keyword_results = self.storage.keyword_search(query, k=k*2)
        # RRF (Reciprocal Rank Fusion) 融合排序
        return self._fuse_results(vector_results, keyword_results, k=k)
    else:
        return self.storage.keyword_search(query, k=k)

def flush_memory(self, messages, user_id, reason, max_messages, 
                 context_summary_callback=None):
    """将消息刷入每日记忆（由上下文裁剪触发）"""
    # 1. 格式化为可读文本
    # 2. 写入每日记忆
    # 3. 触发异步 LLM 摘要生成（用于上下文注入）
    # 4. 如果 context_summary_callback 存在，调用之注入摘要
```

### 3.2 `storage.py` — MemoryStorage 引擎

**数据库 Schema**:

```sql
-- 记忆分块表
CREATE TABLE memory_chunks (
    id TEXT PRIMARY KEY,
    content TEXT NOT NULL,
    metadata TEXT,              -- JSON格式元数据
    embedding BLOB,             -- 向量嵌入（可选）
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- FTS5 全文索引（支持中文分词）
CREATE VIRTUAL TABLE memory_chunks_fts USING fts5(
    content,
    content='memory_chunks',
    content_rowid='rowid'
);

-- 嵌入向量表（独立存储以优化查询）
CREATE TABLE embeddings (
    chunk_id TEXT PRIMARY KEY,
    vector BLOB NOT NULL,
    model TEXT,
    dimensions INTEGER
);
```

**混合搜索实现**:

```python
def keyword_search(self, query: str, k: int = 5) -> list:
    """FTS5 全文搜索，支持中文"""
    # 使用 FTS5 BM25 排序
    return self.db.execute("""
        SELECT mc.*, rank FROM memory_chunks_fts 
        WHERE memory_chunks_fts MATCH ? 
        ORDER BY rank LIMIT ?
    """, (query, k)).fetchall()

def vector_search(self, query_embedding, k: int = 5) -> list:
    """余弦相似度搜索"""
    # 动态加载所有向量并计算相似度
    # 对于大规模数据建议升级为向量数据库
    rows = self.db.execute("SELECT chunk_id, vector FROM embeddings").fetchall()
    scored = [(row[0], cosine_similarity(query_embedding, row[1])) for row in rows]
    scored.sort(key=lambda x: x[1], reverse=True)
    return [self.get_chunk(cid) for cid, _ in scored[:k]]
```

### 3.3 `summarizer.py` — Deep Dream 记忆蒸馏

**工作流程**:

```python
class MemoryFlushManager:
    def run_deep_dream(self, user_id=None):
        """夜间执行: 将每日记忆蒸馏为核心记忆"""
        # 1. 获取最近一天的记忆条目
        recent_memories = self.storage.get_recent(days=1)
        
        # 2. 构建提示词要求 LLM:
        #    - 识别重要的、持久的个人信息
        #    - 合并重复或相关的条目
        #    - 以结构化 Markdown 格式输出
        #    - 保留日期和来源引用
        
        # 3. 调用 LLM 生成 MEMORY.md 追加内容
        summary = self.llm_model.reply(prompt)
        
        # 4. 追加到 ~/cow/MEMORY.md
        # 5. 标记已蒸馏的记忆条目（避免重复处理）
```

### 3.4 `embedding/provider.py` — EmbeddingProvider

**多厂商支持**:

```python
class EmbeddingProvider:
    PROVIDERS = {
        "openai": {"model": "text-embedding-3-small", "dimensions": 1536},
        "dashscope": {"model": "text-embedding-v2", "dimensions": 1536},
        "zhipu": {"model": "embedding-2", "dimensions": 1024},
    }
    
    def embed(self, texts: list[str]) -> list[list[float]]:
        """批量生成嵌入向量，含缓存检查"""
        # 1. 检查缓存（基于文本哈希）
        # 2. 批量调用 API
        # 3. 写入缓存
        # 4. 返回向量列表
```

---

## 4. 数据流与工具链

### 4.1 输入输出

**输入**:
- `messages: list[dict]` — 被裁剪的消息历史
- `query: str` — 搜索查询
- `text: str` — 待写入的记忆文本
- `user_id: str` — 用户标识

**输出**:
- `SearchResult[]` — 搜索结果（含相似度分数和来源）
- `chunk_id: str` — 记忆分块 ID
- `summary: str` — Deep Dream 摘要文本

### 4.2 调用链

```
上下文裁剪触发:
AgentStreamExecutor._trim_messages()
  → MemoryManager.flush_memory(discarded_messages, user_id, reason="trim")
    → MemoryStorage.add_chunks(chunks)       # 写入每日记忆
    → LLM.summarize(discarded_messages)       # 异步生成摘要
    → context_summary_callback(summary)       # 注入上下文

Agent 工具调用:
memory_search tool
  → MemoryManager.search(query, k=5, hybrid=True)
    → MemoryStorage.keyword_search() + vector_search()
    → RRF 融合排序

memory_get tool
  → MemoryManager.get_recent(n=5)

Deep Dream (定时触发):
SchedulerService 定时任务
  → MemoryFlushManager.run_deep_dream()
    → 获取每日记忆
    → LLM 蒸馏总结
    → 写入 MEMORY.md
```

### 4.3 与其他模块的交互

| 被调用模块 | 交互方式 |
|-----------|---------|
| `agent/tools/memory/` | `memory_get` 和 `memory_search` 工具直接调用 `MemoryManager` |
| `agent/protocol/agent_stream.py` | 上下文裁剪时调用 `flush_memory()` |
| `bridge/agent_initializer.py` | 创建 `MemoryManager` 实例并注入到 Agent |
| `agent/tools/scheduler/` | 触发 Deep Dream 定时任务 |
| `agent/prompt/builder.py` | 将 MEMORY.md 内容注入系统提示词 |

---

## 5. 配置与扩展点

### 5.1 相关配置项

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `agent_workspace` | `~/cow` | 工作空间路径，MEMORY.md 和 memory/ 位于此目录下 |
| `embedding_model` | 自动选择 | 嵌入模型选择（openai/dashscope/zhipu） |
| `memory_chunk_max_tokens` | 512 | 记忆分块最大 token 数 |
| `memory_chunk_overlap_tokens` | 50 | 分块间重叠 token 数 |
| `deep_dream_enabled` | true | 是否启用 Deep Dream 蒸馏 |

### 5.2 扩展点

| 扩展点 | 说明 |
|--------|------|
| 嵌入模型注册 | `EmbeddingProvider.PROVIDERS` 字典中添加新厂商 |
| 搜索算法替换 | 重写 `MemoryManager.search()` 的融合策略 |
| 新存储后端 | 实现 `MemoryStorage` 的替代（如 pgvector/ChromaDB） |
| 自定义蒸馏策略 | 修改 `MemoryFlushManager.run_deep_dream()` 的提示词和处理逻辑 |

---

## 6. 二次开发规范

### 6.1 代码规范

- **嵌入降级**: 当 `embedding_provider` 为 None 时必须优雅降级为纯关键词搜索
- **分块幂等**: 同一文本的多次分块结果应保持一致
- **嵌入缓存**: 相同文本不重复调用嵌入 API（基于哈希去重）
- **异步写入**: 记忆写入不应阻塞 Agent 主循环
- **DB 迁移**: Schema 变更需要版本号管理和自动迁移

### 6.2 常见开发场景

**场景 1: 接入新的向量数据库（如 pgvector）**

```python
# 1. 实现新的存储类
class PgvectorMemoryStorage:
    def add_chunks(self, chunks, embeddings, metadata): ...
    def vector_search(self, query_embedding, k=5): ...
    def keyword_search(self, query, k=5): ...

# 2. 在 MemoryManager 中添加路由
if self.config.vector_backend == "pgvector":
    self.storage = PgvectorMemoryStorage(...)

# 3. 在 config.py 中添加配置项
# "memory_vector_backend": "sqlite"  # options: sqlite, pgvector, chromadb
```

**场景 2: 自定义 Deep Dream 蒸馏策略**

```python
# 修改 summarizer.py 中的提示词模板
DEEP_DREAM_PROMPT = """
你是一个记忆整理助手。请从以下日常记忆中提取:
1. 用户的持久偏好和习惯
2. 重要的个人信息（姓名、职业、兴趣）
3. 待办事项和承诺
4. 关键日期和事件

格式要求:
- 每条记忆以 ## 标题开头
- 包含日期标签: <!-- date: 2026-06-10 -->
- 包含类型标签: <!-- type: preference / fact / todo / event -->
"""
```

### 6.3 注意事项

1. **嵌入 API 费用**: 每次调用都有成本，务必使用缓存
2. **FTS5 中文支持**: SQLite 的 FTS5 默认不支持中文分词，需使用 ICU tokenizer 或自行分词
3. **向量维度一致性**: 切换嵌入模型时需重建向量索引（维度可能不同）
4. **MEMORY.md 并发写入**: 多个会话同时触发 Deep Dream 时需加锁
5. **存储膨胀**: 每日记忆可能快速增长，建议设置保留天数上限

---

## 7. 性能与安全

### 性能考虑

- **向量搜索**: 当前是全量加载+内存计算，数据量 >10K 条时建议迁移到专用向量数据库
- **FTS5 索引**: 写入时自动更新索引，大量写入时会有性能影响
- **嵌入缓存**: 基于文本内容哈希，避免重复 API 调用

### 安全注意事项

- **敏感信息**: MEMORY.md 和 memory/ 目录可能包含用户隐私，注意文件权限
- **嵌入数据**: 向量嵌入可能包含语义信息，传输时使用 HTTPS

---

## 8. 总结

### 优势
- 三层架构自然模拟人类记忆的遗忘曲线
- 混合检索兼顾精确匹配和语义理解
- Deep Dream 实现自动化记忆整理，无需用户干预

### 局限
- 向量搜索不支持大规模数据（需外部向量数据库）
- FTS5 中文分词依赖外部 tokenizer
- Deep Dream 依赖 LLM 质量

---

> **相关模块文档**:
> - [agent-protocol.md](agent-protocol.md) — Agent Core 执行引擎（记忆 flush 的触发者）
> - [agent-tools.md](agent-tools.md) — 工具系统（memory_get/memory_search 工具）
> - [agent-knowledge.md](agent-knowledge.md) — Knowledge 知识库（与记忆互补）
> - [agent-evolution.md](agent-evolution.md) — 自我进化（进化过程中更新记忆）
