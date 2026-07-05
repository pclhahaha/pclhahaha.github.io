---
title: RAG 体系详解
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - AI
  - RAG
  - 向量检索
  - LLM
categories:
  - AI
---

RAG（Retrieval-Augmented Generation）是目前最实用的 LLM 应用范式。核心思想很简单：LLM 的知识截止于训练数据，RAG 在 LLM 回答前先检索相关知识片段拼接给模型，让模型基于"新知识"生成答案。

## 一、基础 RAG 流水线

```
用户提问 "公司Q3的营收是多少？"
    ↓
[Embedding 模型] → 将提问转为向量
    ↓
[向量数据库] → 检索 Top-K 相关文档片段
    ↓
[Prompt 拼接] → 提问 + 检索到的文档片段 → 发给 LLM
    ↓
[LLM] → "根据财报，Q3营收为3.2亿美元，同比增长15%"
```

## 二、文档预处理（Ingestion）

RAG 系统的质量高度依赖文档切分策略：

| 策略 | 做法 | 适用场景 |
|------|------|----------|
| **固定长度切分** | 每 512 token 一块，重叠 50 token | 通用场景 |
| **语义切分** | 根据段落/章节边界切分 | 长文档 |
| **递归切分** | 先用大分隔符（章节），再细分 | 层级结构文档 |
| **Agentic 切分** | 让 LLM 决定切分边界 | 高质量要求、低吞吐 |

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,        # 每块 512 token
    chunk_overlap=50,      # 相邻块重叠 50 token
    separators=["\n\n", "\n", "。", "，", " "]  # 优先按段落切
)
chunks = splitter.split_text(document)
```

**Metadata 很重要**：每块文档需要保留来源信息（文件名、页码、章节标题），供 LLM 生成答案时引用来源。

## 三、Embedding 模型选型

| 模型 | 维度 | 适用场景 |
|------|------|----------|
| text-embedding-3-small (OpenAI) | 1536 | 通用，性价比高 |
| bge-large-zh (BAAI) | 1024 | 中文场景首选 |
| voyage-2 (Voyage AI) | 1024 | 长文本、高精度 |
| jina-embeddings-v3 | 1024 | 多语言、长文本 |

Embedding 质量直接决定检索精度。选择时关注 MTEB 排行榜上的检索任务评分。

## 四、向量数据库选型

| 数据库 | 类型 | 适用场景 |
|--------|------|----------|
| **Pinecone** | 托管服务 | 不想运维、快速启动 |
| **Milvus** | 开源 | 大规模（亿级向量）、私有部署 |
| **Chroma** | 嵌入式 | 原型开发、小规模 |
| **pgvector** | PostgreSQL 插件 | 已有 PG，不想引入新组件 |
| **Qdrant** | 开源 | 高性能、过滤查询强 |

检索速度主要看索引类型：
- **Flat**（暴力搜索）：精确但慢，10 万以下向量可用
- **HNSW**（分层小世界图）：速度快 10-100x，召回率 95%+
- **IVF**（倒排文件）：速度与精度的折中

## 五、高级 RAG 变体

### 5.1 Agentic RAG

传统 RAG 是一次检索 → 一次生成。Agentic RAG 让 LLM 在检索过程中多轮思考：

```
提问 → LLM 拆分为子问题 → 分别检索 → LLM 评估结果
    → 信息不足？重新检索 → 足够？合并生成答案
```

### 5.2 Graph RAG

普通 RAG 检索的是文本片段。Graph RAG 先用 LLM 从文档中提取**实体和关系**构建知识图谱，检索时从图谱中查找相关实体及其关联：

```
提问："供应链问题如何影响Q3收入？"
    → Graph RAG 检索：供应链 ←影响→ 产能 ←关联→ Q3收入
    → 返回的不仅是文本片段，还有实体关系的上下文图
```

### 5.3 Hybrid Search（混合检索）

向量检索擅长语义匹配，但不擅长精确关键词匹配。混合方案：向量检索（语义） + BM25（关键词）→ RRF 融合排序。

## 六、RAG 评估

RAGAS 是最常用的 RAG 评估框架，四个核心指标：

| 指标 | 衡量什么 | 健康值 |
|------|----------|--------|
| **Faithfulness** | 答案是否基于检索到的文档（非幻觉） | > 0.9 |
| **Context Relevance** | 检索到的文档是否与问题相关 | > 0.8 |
| **Answer Relevance** | 答案是否准确回答了问题 | > 0.8 |
| **Context Recall** | 检索是否覆盖了回答问题所需的信息 | > 0.9 |

## 七、小结

RAG 是 LLM 应用的基石——它用"检索→上下文→生成"三步弥补了 LLM 知识时效性和幻觉率的问题。随着 Agentic RAG 和 Graph RAG 的成熟，RAG 正从"简单的检索增强"演变为"智能知识管理系统"。
