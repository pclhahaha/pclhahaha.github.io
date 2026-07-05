---
title: TCP 三次握手与四次挥手
date: 2026-07-05
updated: 2026-07-05
tags:
  - TCP
  - 三次握手
  - 四次挥手
  - 网络协议
categories:
  - CS基础
  - 网络
---

```
客户端                             服务端
   |-- SYN (seq=x) ---------------->|  SYN_SENT → SYN_RCVD
   |<- SYN+ACK (seq=y, ack=x+1) ---|  SYN_RCVD → ESTABLISHED
   |-- ACK (seq=x+1, ack=y+1) ---->|  ESTABLISHED
```

**为什么是三次，不是两次？**

核心原因：**防止历史连接导致服务端资源浪费**（RFC 793）。假设两次握手：

1. 客户端发送 SYN(seq=90)，网络延迟未到达
2. 客户端超时重发 SYN(seq=100)，两次握手建立连接，数据传输完毕关闭
3. 旧的 SYN(seq=90) 到达服务端，服务端直接进入 ESTABLISHED 分配资源，等待客户端发数据
4. 客户端认为这不是合法连接，不理会，服务端资源白白浪费

三次握手让客户端在最终 ACK 阶段有机会拒绝旧的连接请求。另外，三次握手恰好让双方都能确认对方的初始序列号(ISN)，两次握手只能确认一方的 ISN。

**SYN Flood 攻击及防御**

攻击者伪造大量不存在的源 IP 发送 SYN，服务端回复 SYN+ACK 后进入 SYN_RCVD 分配半连接资源。半连接队列被占满，正常用户 SYN 被丢弃。

| 防御方式 | 原理 | 命令 |
|----------|------|------|
| SYN Cookie | 连接信息编码进 ISN，不预分配资源 | `sysctl -w net.ipv4.tcp_syncookies=1` |
| 增大半连接队列 | 提高 backlog | `sysctl -w net.ipv4.tcp_max_syn_backlog=8192` |
| 缩短重试 | 减少 SYN 重传次数 | `sysctl -w net.ipv4.tcp_syn_retries=3` |

**全连接队列溢出**：当 accept() 速度跟不上新连接到达速度时，全连接队列溢出：

```bash
netstat -s | grep -i "listen.*overflow"
`
```
