---
title: 预训练 / SFT / RLHF 详解
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - AI
  - 训练
  - SFT
  - RLHF
categories:
  - AI
---

大语言模型的训练分为三个关键阶段：预训练（Pre-training）→ 监督微调（SFT）→ 人类反馈强化学习（RLHF）。理解每个阶段做了什么、为什么需要它，是 LLM 工程的基础。

## 一、预训练（Pre-training）

预训练是**让模型学会语言**的阶段。投入海量文本（数万亿 token），用"预测下一个 token"作为训练目标。

```
输入：  "今天天气真"
目标：  "好"
损失：  CrossEntropyLoss(P(好), 1)

训练数据：Common Crawl, Wikipedia, Books, GitHub code...
数据量：Llama 3 用了 15T tokens，GPT-4 据传用了 ~13T tokens
算力：Llama 3 405B 用了 ~30,840 GPU-hours (H100)
成本：Llama 3 70B 约 $2-3M，GPT-4 据传 > $100M
```

预训练的输出是一个**Base Model**——它能续写文本，但不会遵循指令。你问它"1+1=?"它可能续写成"1+1=2 是一个非常基础的数学问题..."

## 二、监督微调（SFT）

SFT 让模型**学会遵循指令**。收集大量（问题, 理想回答）对，在 Base Model 上继续训练。

```
训练样本：
{
  "instruction": "将以下英文翻译成中文",
  "input": "The weather is nice today.",
  "output": "今天天气不错。"
}
```

关键：
- SFT 数据质量比数量重要——1 万条高质量人工标注 > 10 万条 ChatGPT 生成数据
- 训练 1-3 个 epoch 即可，过度 SFT 会导致模型失去预训练学到的多样性（"模式坍缩"）
- 数据覆盖越广越好：代码、数学、推理、创作、翻译...

## 三、RLHF（人类反馈强化学习）

RLHF 解决 SFT 的一个核心问题：SFT 让模型学会"什么是好的回答"，但很难量化"多好算好"。RLHF 通过人类偏好信号来训练一个**打分模型**，再用强化学习优化主模型。

### 阶段 1：训练奖励模型（Reward Model）

收集人类对同一 prompt 的多个回答的偏好排序：

```
Prompt: "写一首关于春天的诗"

回答 A: "春风拂面暖，花开满园香..."  (人类选为更好的)
回答 B: "春天来了..."              (人类选为较差的)

→ 训练 Reward Model 学会预测人类偏好
```

### 阶段 2：PPO 强化学习

用 Reward Model 作为奖励信号，通过 PPO（Proximal Policy Optimization）算法优化主模型：

```
模型生成回答 → Reward Model 打分 → PPO 更新模型参数 → 
   约束条件：不要偏离原始 SFT 模型太远（KL 惩罚项）
```

PPO 中的 KL 惩罚项很关键——它防止模型为了追求高分而生成胡言乱语。如果没有 KL 惩罚，模型可能发现"输出特殊 token"能获得高分（Reward Hacking），但完全没有实际意义。

## 四、DPO：RLHF 的简化替代

DPO（Direct Preference Optimization）是 2023 年提出的更简洁方案——不需要训练单独的 Reward Model，直接通过偏好数据优化模型。

```python
# DPO 的核心损失函数（简化概念）
loss = -log(sigmoid(beta * (log P_preferred - log P_dispreferred)))
```

DPO 相比 RLHF：
- 不需要训练和维护 Reward Model（节省 30-50% 计算资源）
- 不需要 PPO 的在线采样和 KL 惩罚项调参
- 效果在多数 benchmark 上相当，但复杂推理任务上 RLHF 仍占优

## 五、训练流程全景

```
语料数据                    指令数据              偏好数据
  (trillions)               (thousands)          (tens of thousands)
  ↓                         ↓                    ↓
[预训练] → Base Model → [SFT] → SFT Model → [RLHF/DPO] → 最终模型
  数周-数月                数小时-数天             数天
  最大成本                 中等成本               最小成本
```

## 六、小结

三阶段训练的精髓：预训练给模型"知识"（知道世界上有什么），SFT 给模型"技能"（知道怎么回答问题），RLHF 给模型"品味"（知道什么是好回答）。三者缺一不可——跳过任一阶段都会导致模型要么不懂指令、要么回答质量低下。
