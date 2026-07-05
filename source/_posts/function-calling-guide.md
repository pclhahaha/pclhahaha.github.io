---
title: Function Calling 与 Tool Use
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - AI
  - Agent
  - Function Calling
  - Tool Use
categories:
  - AI
---

Function Calling 是 LLM 连接外部世界的桥梁——不是让 LLM 直接执行代码，而是让 LLM **告诉系统**应该调用哪个函数和什么参数，系统执行后再把结果返回给 LLM。

## 一、工作原理

```
用户: "今天北京天气怎么样？"

LLM 判断: 需要调用 get_weather 函数
    → 返回 function_call: {name: "get_weather", arguments: {city: "北京"}}

系统: 执行 get_weather(city="北京") → "25°C，晴"

LLM 接收函数结果: "今天北京天气晴朗，温度25°C"
```

### OpenAI 格式

```json
{
  "model": "gpt-4",
  "messages": [{"role": "user", "content": "今天北京天气怎么样？"}],
  "tools": [{
    "type": "function",
    "function": {
      "name": "get_weather",
      "description": "获取指定城市的天气信息",
      "parameters": {
        "type": "object",
        "properties": {
          "city": {"type": "string", "description": "城市名称"}
        },
        "required": ["city"]
      }
    }
  }]
}
```

LLM 返回：

```json
{
  "role": "assistant",
  "tool_calls": [{
    "function": {
      "name": "get_weather",
      "arguments": "{\"city\": \"北京\"}"
    }
  }]
}
```

## 二、实现循环

真正的 Agent 需要多次调用 Function Calling：

```python
def run_agent(user_query, tools, max_iterations=5):
    messages = [{"role": "user", "content": user_query}]
    
    for _ in range(max_iterations):
        response = llm.invoke(messages, tools=tools)
        msg = response.choices[0].message
        
        if msg.tool_calls:
            # LLM 要求调用工具
            messages.append(msg)
            for tool_call in msg.tool_calls:
                result = execute_tool(tool_call)
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": result
                })
            # 循环回去，LLM 看到工具结果后决定下一步
        else:
            # LLM 认为任务完成
            return msg.content
```

## 三、关键设计

### Tool Description 很重要

LLM 根据 description 决定何时调用哪个工具——一个模糊的 description 会导致 LLM 不调用或调用错误。

```python
# ❌ 差：描述太模糊
{"name": "process", "description": "处理数据"}

# ✅ 好：描述清晰具体
{
    "name": "query_database",
    "description": "执行 SQL 查询并返回结果。仅在用户明确要求查询数据库时使用。",
    "parameters": {
        "sql": {"type": "string", "description": "要执行的 SQL 语句"}
    }
}
```

### 强制调用

有些场景需要强制调用工具，可以设置 `tool_choice`:

```json
{
  "tool_choice": {
    "type": "function",
    "function": {"name": "get_weather"}
  }
}
```

### 并行调用

LLM 可以同时返回多个 tool_calls（并行执行无依赖的工具）：

```json
{
  "tool_calls": [
    {"function": {"name": "get_weather", "arguments": "{\"city\":\"北京\"}"}},
    {"function": {"name": "get_weather", "arguments": "{\"city\":\"上海\"}"}}
  ]
}
```

## 四、与 LangGraph 的结合

```python
from langgraph.graph import StateGraph, END

def decide_next(state):
    response = llm.invoke(state["messages"], tools=tools)
    if response.tool_calls:
        return "execute_tools"
    return "end"

graph.add_node("llm", call_llm)
graph.add_node("tools", execute_tools)
graph.add_conditional_edges("llm", decide_next, {"execute_tools": "tools", "end": END})
graph.add_edge("tools", "llm")

# LLM → 需要工具 → 执行 → 结果给 LLM → 需要更多工具？→ 循环 → 完成
```

## 五、小结

Function Calling 的核心不是"LLM 调用函数"，而是"LLM 说出它想调用什么"。执行权始终在系统端——这意味着你可以加入权限控制、人工审批、调用审计等安全机制。这是 Agent 安全性的重要保障。
