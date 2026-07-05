---
title: Multi-Agent 协作
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - AI
  - Agent
  - Multi-Agent
categories:
  - AI
---

单个 LLM Agent 的能力有限——模型有幻觉、上下文窗口不够、单一角色难以处理复杂任务。Multi-Agent 系统让多个 Agent 分工协作，每个 Agent 专注于自己的领域。

## 一、为什么需要 Multi-Agent

| 场景 | 单 Agent | Multi-Agent |
|------|----------|-------------|
| 复杂多步骤任务 | 容易迷路 | 分解为子任务，并行执行 |
| 跨领域知识 | 一个模型难全知 | 不同 Agent 加载不同知识库 |
| 交叉验证 | 无法自我质疑 | 一个 Agent 生成，另一个审查 |
| 角色扮演 | 无法自我博弈 | 面试官 Agent + 候选人 Agent |

## 二、协作模式

### 2.1 顺序流水线

```
[需求分析 Agent] → [架构设计 Agent] → [代码编写 Agent] → [代码审查 Agent]
```

适合：有明确阶段划分的任务。

### 2.2 辩论/博弈

```
[Agent A: 方案1] → [Agent B: 挑战方案1] → [Agent A: 改进方案] → [达成共识]
```

适合：需要批判性思考的场景（如安全审查、投资分析）。

### 2.3 层级调度

```
         [Orchestrator Agent]
        /        |        \
[Agent A]  [Agent B]  [Agent C]
  搜索      数据处理     写报告
```

## 三、LangGraph 实现 Multi-Agent

```python
from langgraph.graph import StateGraph

class MultiAgentState(TypedDict):
    messages: list
    next_agent: str
    task_results: dict

# 创建多个 Agent 节点
def researcher(state):
    result = llm.invoke("research this topic: " + state["task"])
    return {"task_results": {"research": result}, "next_agent": "analyst"}

def analyst(state):
    insight = llm.invoke("analyze: " + state["task_results"]["research"])
    return {"task_results": {**state["task_results"], "analysis": insight}, "next_agent": "writer"}

def writer(state):
    report = llm.invoke("write report based on: " + str(state["task_results"]))
    return {"messages": [report], "next_agent": "end"}

graph = StateGraph(MultiAgentState)
graph.add_node("researcher", researcher)
graph.add_node("analyst", analyst)
graph.add_node("writer", writer)
graph.add_edge("researcher", "analyst")
graph.add_edge("analyst", "writer")
```

## 四、Agent 间通信

| 方式 | 说明 |
|------|------|
| **共享 State** | 所有 Agent 共享一个 state 对象，读/写公共字段 |
| **消息传递** | Agent A 发送消息给 Agent B，类似 Actor 模型 |
| **黑板模式** | 中央"黑板"存储中间结果，Agent 按需读写 |

LangGraph 默认使用共享 State 模式——简单直观，但需要注意并发写入的冲突（可通过 `operator.add` 合并列表字段）。

## 五、生产级考虑

| 挑战 | 方案 |
|------|------|
| **死循环** | 设置最大 Agent 调度次数（max_iterations） |
| **幻觉传播** | 关键步骤让多个 Agent 交叉验证 |
| **Token 成本** | 小任务用小模型（如 Qwen3-7B），大任务才用大模型 |
| **延迟** | 无依赖的 Agent 并行执行（`graph.add_edge(["agent_a","agent_b"], "merger")`） |

## 六、小结

Multi-Agent 的精髓是"专注产生质量"。不要让一个 Agent 做所有事——给每个 Agent 一个清晰的角色和有限的任务。LangGraph 的图抽象天然适合表达 Agent 间的顺序、条件分支和并行关系。
