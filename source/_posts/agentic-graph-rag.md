---
title: Agentic RAG 与 Graph RAG
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - AI
  - RAG
  - Agentic RAG
  - Graph RAG
categories:
  - AI
---

基础 RAG 的流程是固定的：检索一次 → 生成一次。Agentic RAG 和 Graph RAG 从不同方向突破了这一限制——一个让检索变得"智能"（多轮思考），一个让检索变得"有结构"（知识图谱）。

## 一、Agentic RAG

Agentic RAG 的核心是让 LLM 在检索过程中**多步推理**——不是一口气检索然后生成，而是检索 → 评估 → 重新检索 → 再评估 → 最终生成。

```
用户: "比较 iPhone 15 和 Galaxy S24 的续航"

基础 RAG:
  1. 检索 "iPhone 15 续航" + "Galaxy S24 续航"
  2. 生成对比

Agentic RAG:
  1. LLM: "我需要先查 iPhone 15 的续航数据"
     → 检索 "iPhone 15 battery life"
  2. LLM: "拿到了。再查 Galaxy S24"
     → 检索 "Galaxy S24 battery life hours"
  3. LLM: "两家用了不同的测试标准，需要找第三方对比"
     → 检索 "iPhone 15 vs Galaxy S24 battery comparison"
  4. LLM: "信息够了，可以对比了"
     → 生成最终对比
```

### LangGraph 实现

```python
def decide_next(state):
    response = llm.invoke(state["messages"])
    if "需要更多信息" in response:
        return "search"
    elif "可以回答了" in response:
        return "generate"
    return "reformulate"  # 改写检索词

graph.add_conditional_edges("agent", decide_next, {
    "search": "search", "reformulate": "reformulate", "generate": "generate"
})
graph.add_edge("search", "agent")
graph.add_edge("reformulate", "search")
```

## 二、Graph RAG

基础 RAG 检索文本片段。Graph RAG 先构建**知识图谱**，检索时利用图谱的关联结构提供更丰富的上下文。

```
原始文档:
 "张三是A公司CEO。A公司2024年营收10亿。李四是CTO。"

Graph RAG 构建的知识图谱:
 [张三] --CEO--> [A公司] --营收--> [10亿]
 [李四] --CTO--> [A公司]
```

查询"谁在A公司工作"时，Graph RAG 不仅检索到"张三是A公司CEO"这个片段，还会通过图结构找到"李四是CTO"——即使该片段中没有出现"A 公司"（因为"李四"和"A公司"通过图路径连通）。

### 构建流程

```python
# 1. LLM 提取实体和关系
entities = llm.extract_entities(document)
# → [{"entity": "张三", "type": "PERSON"}, ...]

# 2. 构建图
for entity in entities:
    graph.add_node(entity)
for relation in relations:
    graph.add_edge(relation.source, relation.target, relation.type)

# 3. 检索
query_entities = llm.extract_entities(query)
subgraph = graph.search(query_entities, depth=2)  # 2-hop 邻居

# 4. 生成
answer = llm.generate(query, context=subgraph)
```

## 三、对比

| 维度 | 基础 RAG | Agentic RAG | Graph RAG |
|------|----------|-------------|-----------|
| 检索次数 | 1 次 | 多次（自适应） | 1 次（图遍历） |
| 上下文来源 | 文本片段 | 文本片段 | 文本 + 图关联 |
| 适合 | 简单问答 | 多步推理 | 需要全局关联理解 |
| 额外成本 | 无 | 多次 LLM 调用 | 图构建与存储 |

## 四、小结

Agentic RAG 用多轮思考突破单次检索的限制，Graph RAG 用图结构弥补纯文本检索的语义盲区。两者可以结合——先用 Graph RAG 获取全局关联，再用 Agentic RAG 按需深入检索特定方向。
