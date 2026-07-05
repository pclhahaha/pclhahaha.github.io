---
title: MCP 协议详解
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - AI
  - MCP
  - Agent
  - LLM
categories:
  - AI
---

MCP（Model Context Protocol）是 Anthropic 在 2024 年底发布的开源协议，目标是为 LLM 和外部工具/数据源之间建立统一的连接标准。它的核心理念类似 USB-C——不是创造新的外设，而是提供一个通用接口，让任何 LLM 客户端都能连接任何 MCP 服务端。

## 一、为什么需要 MCP

在 MCP 之前，每个 LLM 应用都要为不同的数据源编写定制的连接代码：

```
Claude → 自定义插件 → Google Drive
ChatGPT → 自定义插件 → Google Drive
Llama → 自定义适配器 → Google Drive
```

N 个 LLM × M 个数据源 = N×M 个集成。MCP 的目标是将 N×M 降为 N+M：

```
Claude → MCP 客户端 → MCP 协议 ← MCP 服务端 ← Google Drive
ChatGPT → MCP 客户端 → MCP 协议 ← MCP 服务端 ← GitHub
Llama → MCP 客户端 → MCP 协议 ← MCP 服务端 ← Slack
```

## 二、协议架构

MCP 采用客户端-服务端架构，基于 JSON-RPC 2.0 通信：

```
┌─────────────────┐         ┌─────────────────┐
│   MCP 客户端     │ ←JSON→  │   MCP 服务端     │
│  (LLM 应用)      │  RPC    │  (数据源连接器)  │
│                  │         │                  │
│  - 发起请求      │         │  - 暴露资源       │
│  - 调用工具      │         │  - 执行工具       │
│  - 读取资源      │         │  - 提供 Prompt    │
└─────────────────┘         └─────────────────┘
```

### 三种核心能力

| 能力 | 作用 | 示例 |
|------|------|------|
| **Tools** | LLM 可调用的工具函数 | `search_files`, `send_email`, `run_query` |
| **Resources** | 暴露给 LLM 的数据资源 | `file://documents/report.pdf`, `postgres://db/users` |
| **Prompts** | 预定义的 Prompt 模板 | "请总结以下文档", "分析这个数据库表" |

### 生命周期

```
1. 初始化：客户端和服务端协商协议版本和能力
2. 发现：客户端请求 tools/resources/prompts 列表
3. 调用：LLM 通过客户端调用服务端提供的工具
4. 返回：服务端返回结果
5. 关闭：连接断开
```

## 三、MCP vs API vs RAG vs Agent

| 概念 | 是什么 | 谁用 |
|------|--------|------|
| **API** | 固定的 HTTP/gRPC 接口 | 开发者手动调用 |
| **RAG** | 检索增强生成 | LLM 通过检索器获取知识 |
| **MCP** | LLM 连接外部世界的协议标准 | LLM 通过 MCP 调用工具和数据 |
| **Agent** | 自主决策 + 工具调用的系统 | 最终用户与 Agent 交互 |

关系：Agent 使用 RAG 获取知识，通过 MCP 协议调用外部工具和 API——三者是协作关系而非竞争关系。

## 四、简单示例

**MCP 服务端**（暴露 SQLite 数据库）：

```python
from mcp.server import Server, StdioServerTransport
from mcp.types import Tool, TextContent
import sqlite3

app = Server("sqlite-server")

@app.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(name="query", description="执行 SQL 查询", 
             inputSchema={"type": "object", "properties": {"sql": {"type": "string"}}})
    ]

@app.call_tool()
async def call_tool(name: str, args: dict) -> list[TextContent]:
    if name == "query":
        conn = sqlite3.connect("data.db")
        result = conn.execute(args["sql"]).fetchall()
        return [TextContent(type="text", text=str(result))]
    raise ValueError(f"Unknown tool: {name}")

if __name__ == "__main__":
    StdioServerTransport().run(app)
```

**MCP 客户端**（LLM 侧）：

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async with stdio_client(StdioServerParameters(command="python", args=["sqlite_server.py"])) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()
        tools = await session.list_tools()
        # 将 tools 的 schema 传给 LLM
        # LLM 决定调用 query 工具
        result = await session.call_tool("query", {"sql": "SELECT * FROM users LIMIT 5"})
```

## 五、生产级实践

**Composio** 等平台已将 MCP 封装为生产可用的 Agent 连接器：

```python
from composio_langgraph import ComposioToolSet

toolset = ComposioToolSet()
tools = toolset.get_tools(apps=[App.GITHUB, App.SLACK, App.JIRA])
# Agent 现在可以操作 GitHub PR、发 Slack 消息、创建 JIRA 工单
```

**安全考虑**：
- Agent 不能无限制地调用工具——需要用户预先授权（User Consent）
- 敏感操作（发送邮件、付款）必须走 Human-in-the-Loop 确认
- 每个 Tool 需要声明它需要的权限，客户端展示给用户审批

## 六、小结

MCP 的价值在于**标准化**——让 LLM 应用从"每个数据源写一个定制连接器"变成"实现一次 MCP，连接所有"。对开发者来说，这意味着更多时间花在 Agent 逻辑上，而非集成胶水代码上。
