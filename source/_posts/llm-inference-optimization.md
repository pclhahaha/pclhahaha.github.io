---
title: LLM 推理优化
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - AI
  - 推理优化
  - vLLM
  - 量化
categories:
  - AI
---

LLM 推理速度直接决定用户体验。从原始的 HuggingFace `model.generate()` 到生产级的 vLLM 部署，推理速度可以提升 10-50 倍。

## 一、KV Cache——最基础的优化

LLM 生成 token 时，每个新 token 都要关注之前的所有 token。如果不缓存，每生成一个新 token 都要重新计算所有历史 token 的 Key 和 Value 矩阵——这是 O(N²) 的计算量。

```
生成第 1 个 token: 计算 token_1 的 K,V
生成第 2 个 token: 计算 token_2 的 K,V, 复用 token_1 的 K,V
生成第 N 个 token: 计算 token_N 的 K,V, 复用 token_1..token_{N-1} 的 K,V

→ 不需要每次重新计算所有历史 token 的注意力
```

KV Cache 把每个新 token 的计算量从 O(N²) 降到 O(N)，这是所有 LLM 推理引擎的基础优化。

KV Cache 的内存占用是推理的主要瓶颈：一个 70B 模型在 128K 上下文下，KV Cache 需要约 160GB 显存（FP16）。这就是为什么大模型推理对显存要求如此之高。

## 二、vLLM——PagedAttention

vLLM 的核心创新是 **PagedAttention**——把 KV Cache 按"页"（block）管理，类似操作系统的虚拟内存：

```
传统模式：KV Cache 是一整块连续内存
  → 显存碎片严重，利用率 < 40%

vLLM 模式：KV Cache 分成固定大小的 block（16-32 token）
  → 按需分配，显存利用率 > 90%
  → 多个请求可以共享相同的 prefix block
```

```bash
# 启动 vLLM 服务
vllm serve Qwen/Qwen3-72B \
  --tensor-parallel-size 4 \    # 4 卡张量并行
  --max-model-len 32768 \
  --gpu-memory-utilization 0.95 \
  --enable-prefix-caching       # 共享 prompt prefix
```

## 三、连续批处理（Continuous Batching）

传统批处理等所有请求完成后才开始下一批。vLLM 的连续批处理允许"来一个处理一个，完成一个退出一个"：

```
传统批处理:
  Batch1: [Req1, Req2, Req3] → 等全部完成 → Batch2: [Req4, Req5]
  
连续批处理:
  [Req1, Req2] → Req1完成 → [Req2, Req3] → Req2完成 → [Req3, Req4, Req5]
```

吞吐量提升 5-10 倍，因为 GPU 空转时间大幅减少。

## 四、量化

| 方法 | 位宽 | 压缩比 | 质量损失 | 方案 |
|------|------|--------|----------|------|
| **GPTQ** | INT4/INT8 | 4x/2x | < 2% | 离线一次性量化 |
| **AWQ** | INT4 | 4x | < 1% | 离线，保护关键权重 |
| **GGUF/GGML** | Q4_K_M | 4x | < 5% | llama.cpp CPU推理 |
| **FP8** | FP8 | 2x | 极小 | H100 原生支持 |

AWQ 相比 GPTQ 的核心改进：保护"显著权重"（salient weights，即绝对值较大的权重通道）不被过度量化，因为它们在推理中贡献更大。

## 五、Prompt Caching

当大量用户的 prompt 共享相同的前缀（如系统提示词），缓存这部分 prefix 的 KV Cache 可以避免重复计算：

```
System Prompt (共享) + User Query (不同)
    ↓
[缓存 System Prompt 的 KV Cache]
    ↓
每次只计算 User Query 部分的 KV Cache
```

对于有固定长 system prompt 的 Agent，这可以将首次 token 延迟降低 80%。LangChain、vLLM 和 Anthropic API 都支持此功能。

## 六、延迟 vs 吞吐量

| 优化 | 降低延迟 | 提升吞吐 |
|------|---------|---------|
| KV Cache | ✅ | ✅ |
| PagedAttention | — | ✅ |
| 连续批处理 | — | ✅ |
| 量化 | ✅ | ✅ |
| Tensor Parallelism | ✅ | — |
| Prompt Caching | ✅ | — |

## 七、小结

推理优化的核心矛盾是"显存 vs 速度"——KV Cache 省计算但占显存，PagedAttention 优化显存但增加实现复杂度，量化压缩显存但轻微损失质量。生产级部署通常组合使用：vLLM + AWQ INT4 + Prompt Caching。
