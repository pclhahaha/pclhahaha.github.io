---
title: AI 与 LLM Agent 知识图谱
date: 2026-07-05 12:00:00
type: page
comments: false
---

<style>
.knowledge-graph { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; max-width: 960px; margin: 0 auto; }
.kg-header { text-align: center; margin-bottom: 32px; }
.kg-header h1 { font-size: 28px; margin-bottom: 8px; }
.kg-header p { color: #666; font-size: 14px; }
.kg-stats { display: flex; justify-content: center; gap: 32px; margin: 20px 0; }
.kg-stat { text-align: center; }
.kg-stat-num { font-size: 32px; font-weight: 700; color: #333; }
.kg-stat-label { font-size: 12px; color: #999; margin-top: 2px; }
.kg-section { margin-bottom: 36px; }
.kg-section-title { font-size: 20px; font-weight: 700; padding-bottom: 8px; border-bottom: 2px solid #eee; margin-bottom: 16px; }
.kg-category { margin-bottom: 20px; }
.kg-cat-title { font-size: 15px; font-weight: 600; color: #555; margin-bottom: 8px; }
.kg-links { display: flex; flex-wrap: wrap; gap: 6px 12px; }
.kg-link { font-size: 13px; color: #333; text-decoration: none; padding: 3px 10px; background: #f7f7f7; border-radius: 4px; transition: all 0.15s; white-space: nowrap; }
.kg-link:hover { background: #e3e3e3; color: #000; }
@media (max-width: 600px) { .kg-stats { gap: 16px; } .kg-stat-num { font-size: 24px; } }
</style>

<div class="knowledge-graph">

<div class="kg-header">
  <h1>AI 与 LLM Agent 知识图谱</h1>
  <p>大语言模型 · Agent 框架 · RAG · 推理优化</p>
  <div class="kg-stats">
    <div class="kg-stat"><div class="kg-stat-num">4</div><div class="kg-stat-label">篇文章</div></div>
    <div class="kg-stat"><div class="kg-stat-num">3</div><div class="kg-stat-label">分类</div></div>
  </div>
</div>

<div class="kg-section">
  <div class="kg-section-title">🧠 LLM 基础</div>
  <div class="kg-category">
    <div class="kg-cat-title">核心架构</div>
    <div class="kg-links">
      <a class="kg-link" href="/transformer-architecture/">Transformer 架构详解</a>
    </div>
  </div>
  <div class="kg-category">
    <div class="kg-cat-title">训练与微调</div>
    <div class="kg-links">
      <a class="kg-link" href="#" style="color:#999">预训练 / SFT / RLHF（待补充）</a>
    </div>
  </div>
  <div class="kg-category">
    <div class="kg-cat-title">模型选型</div>
    <div class="kg-links">
      <a class="kg-link" href="#" style="color:#999">开源模型对比（待补充）</a>
    </div>
  </div>
</div>

<div class="kg-section">
  <div class="kg-section-title">🤖 Agent 框架</div>
  <div class="kg-category">
    <div class="kg-cat-title">编排框架</div>
    <div class="kg-links">
      <a class="kg-link" href="/langgraph-guide/">LangGraph 实战</a>
    </div>
  </div>
  <div class="kg-category">
    <div class="kg-cat-title">协议标准</div>
    <div class="kg-links">
      <a class="kg-link" href="/mcp-protocol-guide/">MCP 协议详解</a>
    </div>
  </div>
  <div class="kg-category">
    <div class="kg-cat-title">进阶</div>
    <div class="kg-links">
      <a class="kg-link" href="#" style="color:#999">Function Calling / Tool Use（待补充）</a>
      <a class="kg-link" href="#" style="color:#999">Multi-Agent 协作（待补充）</a>
    </div>
  </div>
</div>

<div class="kg-section">
  <div class="kg-section-title">🔍 RAG 体系</div>
  <div class="kg-links">
    <a class="kg-link" href="/rag-system-design/">RAG 体系详解</a>
    <a class="kg-link" href="#" style="color:#999">Agentic RAG（待补充）</a>
    <a class="kg-link" href="#" style="color:#999">Graph RAG（待补充）</a>
    <a class="kg-link" href="#" style="color:#999">RAG 评估 RAGAS（待补充）</a>
  </div>
</div>

<div class="kg-section">
  <div class="kg-section-title">⚡ 推理优化</div>
  <div class="kg-links">
    <a class="kg-link" href="#" style="color:#999">vLLM / TensorRT（待补充）</a>
    <a class="kg-link" href="#" style="color:#999">量化 GPTQ / AWQ（待补充）</a>
    <a class="kg-link" href="#" style="color:#999">Prompt Caching（待补充）</a>
  </div>
</div>

<div class="kg-section">
  <div class="kg-section-title">🛡️ 工程与安全</div>
  <div class="kg-links">
    <a class="kg-link" href="#" style="color:#999">LLMOps 监控（待补充）</a>
    <a class="kg-link" href="#" style="color:#999">Prompt Injection 防御（待补充）</a>
  </div>
</div>

<div style="text-align:center; margin-top:48px;">
  <a href="/" style="color:#555; text-decoration:none; font-size:14px; border:1px solid #ddd; padding:8px 24px; border-radius:4px;">← 返回后端知识图谱</a>
</div>

</div>
