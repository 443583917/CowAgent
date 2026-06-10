# Knowledge 知识库系统 — 模块深度分析

> **所属项目**: CowAgent v2.1.1  
> **分析日期**: 2026-06-10  
> **模块路径**: `agent/knowledge/`  
> **依赖模块**: `common/`, `config.py`

---

## 1. 模块概述

Knowledge（知识库）与 Memory（记忆系统）互补——记忆按**时间**组织（"昨天讨论了什么"），知识按**主题**组织（"什么是微服务架构"）。知识库自动将对话中有价值的信息整理为结构化的 Markdown Wiki，并通过知识图谱可视化让用户直观浏览。

**核心职责**:
- 自动整理：Agent 在对话中主动将知识写入 Markdown 文件
- 目录索引：通过 `index.md` 维护完整的知识目录
- 操作日志：`log.md` 记录所有知识操作
- 树状导航：Web Console 提供交互式知识图谱浏览
- 交叉引用：知识页面间通过 Markdown 链接构建知识网络

---

## 2. 架构设计

### 2.1 知识库布局

```
workspace/knowledge/
├── index.md              # 知识目录索引（Agent 必须维护）
├── log.md                # 知识操作日志
├── concepts/             # 概念类知识
│   └── agent-harness.md
├── tools/                # 工具使用知识
│   └── docker-commands.md
├── projects/             # 项目相关知识
└── sources/              # 外部资源引用
```

### 2.2 写入流程

```
对话中产生有价值的知识
  → Agent 调用 write 工具写入 knowledge/<category>/<slug>.md
  → Agent 调用 edit 工具更新 knowledge/index.md（添加索引条目）
  → Agent 调用 edit 工具更新 knowledge/log.md（记录操作）
  → Web Console 自动刷新知识图谱
```

### 2.3 `knowledge-wiki` 技能

内置技能 `knowledge-wiki` 提供知识库操作的详细规范：
- 文件命名约定（小写、连字符分隔）
- 索引格式（每行 `[标题](路径) — 摘要`）
- 交叉引用规范（只链接已存在页面）
- 目录组织原则（一致性优先）

---

## 3. 源码解析

### 3.1 `KnowledgeService` (`service.py`)

```python
class KnowledgeService:
    def list_tree(self) -> dict:
        """列出知识库的完整目录树，支持嵌套子目录"""
        # 返回 {"tree": [{dir, files, children}]}
    
    def read_page(self, path: str) -> dict:
        """读取单个知识页面的内容"""
    
    def get_graph(self) -> dict:
        """构建知识图谱（节点+边），用于前端可视化"""
        # 解析所有 .md 文件中的 [[wikilinks]] 和 [markdown](links)
    
    def search(self, query: str) -> list:
        """全文搜索知识库内容"""
```

---

## 4. 数据流

```
[Agent 判断知识值得保存]
  → write("knowledge/concepts/new-concept.md", content)
  → read("knowledge/index.md")
  → edit("knowledge/index.md", old_str, old_str + "\n- [New Concept](concepts/new-concept.md) — 一句话描述")
  → edit("knowledge/log.md", old_str, old_str + "\n- 2026-06-10: 新增 concepts/new-concept.md")

[Web Console 请求]
  → KnowledgeService.list_tree()
  → KnowledgeService.read_page()
  → KnowledgeService.get_graph()
```

---

## 5. 配置与扩展点

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `knowledge` | true | 是否启用知识库 |
| `agent_workspace` | `~/cow` | 知识库位于 `<workspace>/knowledge/` |

### 扩展点

| 扩展点 | 说明 |
|--------|------|
| 知识图谱优化 | 替换 `get_graph()` 的双向链接检测算法 |
| 全文搜索增强 | 集成 FTS5 或 Elasticsearch |
| 导出格式 | 添加 PDF/HTML 导出功能 |

---

## 6. 二次开发规范

### 6.1 注意事项

1. **index.md 是知识库的核心**: 每次增删改知识页面后必须同步更新
2. **不要创建孤立页面**: 新建页面时检查是否有已有页面应反向链接
3. **只链接已存在页面**: 不预先创建死链接
4. **目录组织一致性**: 同一知识库保持统一的分类风格

---

> **相关模块文档**:
> - [agent-memory.md](agent-memory.md) — Memory（与知识按"时间vs主题"互补）
> - [agent-skills.md](agent-skills.md) — Skills（knowledge-wiki 技能提供操作规范）
> - [agent-prompt.md](agent-prompt.md) — Prompt（RULE.md 中的知识系统操作规则）
