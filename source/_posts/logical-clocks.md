---
title: Lamport 逻辑时钟 / Vector Clock
date: 2026-07-05
updated: 2026-07-05
tags:
  - 分布式
  - 逻辑时钟
  - Lamport
  - Vector Clock
  - HLC
categories:
  - 分布式
---

Lamport 在 1978 年的论文《Time, Clocks, and the Ordering of Events》中提出了逻辑时钟（Logical Clock）：

**规则**：
1. 每个进程维护一个计数器 `C`，初始为 0
2. 每发生一个本地事件，`C = C + 1`
3. 发送消息时，将 `C` 值附带在消息中
4. 接收消息时，`C = max(本地C, 消息中的C) + 1`

**Happens-Before 关系**（记作 `a → b`）：
- 同一进程内：若 a 在 b 之前发生，则 `a → b`
- 跨进程：若 a 是发送事件，b 是对应的接收事件，则 `a → b`
- 传递性：若 `a → b` 且 `b → c`，则 `a → c`

```
进程 P1:  [a:1] → [b:2] ──────── [e:5] → [f:6]
                  ↓  msg(2)        ↑  msg(4)
进程 P2:       [c:3] → [d:4] ────┘
                  Lamport Time: C(d) < C(e)，但 d 和 e 无因果关系！
```

**局限性**：Lamport Clock 只能判断"如果 `a → b` 则 `C(a) < C(b)`"，但**不能反向推断**——`C(a) < C(b)` 不能得出 `a → b`（它们可能只是并发）。
