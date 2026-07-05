---
title: LangGraph 实战
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - AI
  - Agent
  - LangGraph
  - LangChain
  - 工作流
categories:
  - AI
---

LangGraph 是 LangChain 团队推出的 Agent 编排框架，核心思想是把 Agent 的决策过程建模为**有状态的有向图**。相比传统的线性 Chain（A→B→C→D），LangGraph 支持循环、条件分支、并行执行和人在回路（Human-in-the-Loop）。

## 一、LangChain 的局限

LangChain 的 Chain 抽象是线性的：

```python
chain = prompt | llm | parser | database_lookup | response
# A → B → C → D → E，一路走到黑
```

这在简单场景下够用，但 Agent 的真正能力在于**循环决策**——执行一个操作 → 观察结果 → 决定下一步。这就是 Agent 的核心循环：`Plan → Execute → Observe → Decide → Plan → ...`。LangChain 的 Chain 模型很难优雅地表达这种循环和分支。

## 二、LangGraph 核心概念

### StateGraph（状态图）

LangGraph 的核心是 `StateGraph`——一个带共享状态的有向图：

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

# 1. 定义共享状态
class AgentState(TypedDict):
    messages: Annotated[list, operator.add]  # 对话历史
    next_step: str                            # 下一步跳转
    tool_results: list                        # 工具调用结果

# 2. 创建图
graph = StateGraph(AgentState)

# 3. 添加节点
def agent(state):
    """调用 LLM 决定下一步"""
    response = llm.invoke(state["messages"])
    return {"messages": [response], "next_step": response.tool_calls[0].name if response.tool_calls else "end"}

def search_tool(state):
    """执行搜索"""
    query = state["messages"][-1].tool_calls[0].args["query"]
    result = search_api(query)
    return {"tool_results": [result]}

def generate_response(state):
    """根据工具结果生成最终回复"""
    context = state["tool_results"][-1]
    prompt = f"Based on: {context}, answer the question"
    return {"messages": [llm.invoke(prompt)]}

graph.add_node("agent", agent)
graph.add_node("search", search_tool)
graph.add_node("generate", generate_response)

# 4. 添加边（包括条件边）
graph.set_entry_point("agent")
graph.add_conditional_edges(
    "agent",
    lambda state: state["next_step"],  # 根据 LLM 决策跳转
    {"search": "search", "end": END}
)
graph.add_edge("search", "agent")  # 搜索后回到 agent 重新决策

# 5. 编译运行
app = graph.compile()
result = app.invoke({"messages": ["搜索今天北京的天气"]})
```

### 关键机制

| 概念 | 说明 |
|------|------|
| **StateGraph** | 有状态的有向图，节点共享一个类型化的 state 对象 |
| **条件边** | `add_conditional_edges` 根据 state 内容动态决定下一步 |
| **循环** | 节点可以指向之前的节点，形成 Agent 的 Plan-Execute-Observe 循环 |
| **Human-in-the-Loop** | 在关键节点设置 `interrupt_before`，暂停执行等待人工审批 |
| **检查点** | 自动保存每个节点的 state 快照，支持回溯和重放 |

## 三、Agent 循环模式

### 3.1 ReAct 模式（Reasoning + Acting）

```
LLM 思考 → 调用工具 → 观察结果 → LLM 再思考 → ... → 最终回答
```

LangGraph 中实现：

```python
graph.add_node("llm", call_llm)
graph.add_node("tools", execute_tools)
graph.add_conditional_edges("llm", decide_next, {"tools": "tools", "end": END})
graph.add_edge("tools", "llm")  # 工具结果返回 LLM 继续思考
```

### 3.2 Plan-and-Execute 模式

```
先制定计划 → 逐步执行 → 每步检查 → 修正计划
```

适用于复杂多步骤任务（如"帮我调研三家竞品并按季度做对比分析"）。Agent 先生成执行计划，然后逐步执行并在每步后评估是否需要修正。

## 四、Skill 模式

Skill 是 LangGraph 中的可复用工具单元。一个 Skill 封装了完成特定任务的完整逻辑：

```python
class WebSearchSkill:
    def __init__(self, api_key):
        self.search = TavilySearchAPI(api_key)
    
    def as_tool(self):
        """暴露给 Agent 的标准 tool 接口"""
        return {
            "name": "web_search",
            "description": "搜索互联网获取最新信息",
            "parameters": {
                "query": {"type": "string", "description": "搜索关键词"}
            }
        }
    
    def execute(self, query: str) -> str:
        return self.search.query(query)
```

Skill vs Tool：
- Tool 是**原子操作**——一次调用、一个结果
- Skill 是**可组合的工作流**——可能包含多个 Tool 调用 + 自身状态管理

## 五、Human-in-the-Loop

关键操作在执行前需要人工确认：

```python
# 在发送邮件节点前暂停
graph.add_node("send_email", send_email)
graph.compile(interrupt_before=["send_email"])

# 运行时会在 send_email 前暂停
result = app.invoke(state)
# 人工审查通过后
app.invoke(Command(resume="approve"))
```

## 六、小结

LangGraph 的核心价值在于把 Agent 的决策过程从"写死的 if-else"升级为"图结构 + 条件路由 + 循环"。Agent 框架的真正挑战不是调用 LLM，而是管理 LLM 调用的**状态流转**——LangGraph 用图抽象的优雅方式解决了这个问题。
