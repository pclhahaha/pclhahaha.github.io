---
title: Transformer 架构详解
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - AI
  - Transformer
  - LLM
  - 深度学习
categories:
  - AI
---

Transformer 是当前所有大语言模型（GPT、Claude、Llama、Gemini）的底层架构。2017 年 Google 在《Attention Is All You Need》论文中提出它时，目标是解决机器翻译问题，但它的自注意力机制展现出惊人的通用性，最终彻底取代了 RNN/LSTM，成为 NLP 乃至多模态领域的主导架构。

## 一、为什么需要 Transformer

在 Transformer 之前，序列建模的主流方案是 RNN/LSTM。它们的核心问题：

- **串行计算**：处理第 N 个词必须等前 N-1 个词处理完——无法并行，训练慢
- **长程依赖衰减**：处理 1000 个 token 的文本时，第 1 个 token 的信息经过 1000 步传递后衰减严重
- **梯度消失/爆炸**：反向传播经过几十步后梯度趋近于零或无穷大

Transformer 用自注意力机制一举解决了这三个问题。

## 二、核心架构

Transformer 由编码器（Encoder）和解码器（Decoder）堆叠而成，但 GPT 只用了 Decoder，BERT 只用了 Encoder。

```
输入文本 → [Input Embedding + Positional Encoding]
              ↓
    ┌─────────────────────────┐
    │  Multi-Head Attention   │ ← 自注意力：每个词关注所有词
    │  Add & Norm             │
    │  Feed Forward (MLP)     │
    │  Add & Norm             │
    └─────────────────────────┘
              ↓ (× N 层堆叠)
         输出向量 → 下游任务
```

## 三、自注意力机制

自注意力（Self-Attention）是 Transformer 的灵魂。核心公式：

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

对每个 token，生成三个向量：
- **Q（Query）**：我在找什么？
- **K（Key）**：我能提供什么？
- **V（Value）**：我的实际内容是什么？

计算过程：
1. 计算 Q 和所有 K 的点积，得到注意力分数矩阵
2. 除以 $\sqrt{d_k}$（缩放因子，防止点积过大导致 softmax 梯度消失）
3. Softmax 归一化为概率分布
4. 用概率分布对 V 加权求和——得到当前 token 的"上下文感知"表示

**实际例子**：处理句子"我昨天在书店买了一本关于分布式系统的书"时，token"书"的注意力权重分布中，"买"和"书店"的权重会很高，而"昨天"、"关于"的权重相对低——模型学会了关注语义相关的词。

## 四、多头注意力

单头注意力只能捕捉一种关系模式。多头注意力并行运行多个注意力头，每个头关注不同的语义维度：

- Head 1：关注语法结构（主谓宾）
- Head 2：关注指代关系（代词→先行词）
- Head 3：关注语义相似性
- Head N：关注位置邻近关系

不同头关注不同维度，最终拼接起来通过线性层融合——这使得模型能同时捕捉多种语言特征。

## 五、位置编码

自注意力本身是**位置无关**的——"我爱你"和"你爱我"在注意力机制下完全等价（只是 token 顺序不同，但注意力权重对称）。因此需要额外注入位置信息。

**正弦位置编码**（原论文方案）：

$$PE_{(pos, 2i)} = \sin\left(\frac{pos}{10000^{2i/d_{model}}}\right)$$

$$PE_{(pos, 2i+1)} = \cos\left(\frac{pos}{10000^{2i/d_{model}}}\right)$$

偶数维度用正弦，奇数维度用余弦。这种设计使得模型可以外推到训练时未见过的序列长度。

现代模型（GPT-3+、Llama）多使用**可学习位置编码**或 **RoPE（旋转位置编码）**——RoPE 通过旋转变换将位置信息编码到注意力计算中，在长文本外推方面表现更好。

## 六、关键变体

| 架构 | 使用部分 | 代表模型 | 特点 |
|------|----------|----------|------|
| Encoder-only | 仅编码器 | BERT | 双向注意力，适合理解任务 |
| Decoder-only | 仅解码器 | GPT 系列、Llama | 单向（因果）注意力，适合生成任务 |
| Encoder-Decoder | 完整架构 | T5、BART | 适合翻译、摘要等 Seq2Seq 任务 |

GPT 系列之所以只用 Decoder，是因为它的核心任务是"预测下一个 token"——这是一个自回归生成任务，只需要单向注意力（每个 token 只能看到它之前的 token，不能看到未来的 token）。

## 七、小结

Transformer 的突破不在于复杂——它的结构出奇简单（注意力 + 前馈网络 + 残差连接 + LayerNorm，堆叠 N 层）。突破在于**可并行性**（所有 token 同时计算注意力，不需要像 RNN 那样串行等待）和**长程依赖建模**（任意两个 token 之间的注意力路径长度为 O(1)，而非 RNN 的 O(N)）。这两点使得 Transformer 可以训练到千亿甚至万亿参数的规模——而这正是大语言模型能力涌现的基础。
