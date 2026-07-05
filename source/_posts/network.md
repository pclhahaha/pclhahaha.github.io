---
title: 计算机网络
date: 2020-01-01 00:00:00
updated: 2020-01-01 00:00:00
tags:
  - 计算机网络
  - TCP
  - HTTP
  - HTTPS
  - DNS
categories:
  - CS基础
---
## 一、TCP/IP 协议栈

### 1.1 OSI 七层模型 vs TCP/IP 四层模型

OSI 七层模型是学术上的理想参考模型，TCP/IP 四层模型是事实上的工业标准：

```
+--------------------+--------------------+
|    OSI 七层模型    |   TCP/IP 四层模型   |
+--------------------+--------------------+
| 7. 应用层          |                    |
| 6. 表示层          |   应用层            |
| 5. 会话层          |                    |
+--------------------+--------------------+
| 4. 传输层          |   传输层 (TCP/UDP)  |
+--------------------+--------------------+
| 3. 网络层          |   网络层 (IP)       |
+--------------------+--------------------+
| 2. 数据链路层      |   链路层            |
| 1. 物理层          |                    |
+--------------------+--------------------+
```

OSI 七层做减法，TCP/IP 四层做加法——前者先定义完整模型再实现协议，后者先有协议再抽象模型。TCP/IP 将表示层和会话层合并到应用层，现实中 TLS、JSON/Protobuf 等逻辑直接在应用代码中处理，不需要独立协议层。

### 1.2 分层设计思想

每层只关心与对等层的逻辑通信：

| 层级 | 数据单位 | 核心协议 | 寻址方式 | 典型设备 |
|------|----------|----------|----------|----------|
| 应用层 | Message/Stream | HTTP, DNS, SMTP | 域名/URI | Nginx, Tomcat |
| 传输层 | Segment/Datagram | TCP, UDP | 端口号(16bit) | 操作系统内核 |
| 网络层 | Packet | IP, ICMP | IP 地址(32/128bit) | 路由器 |
| 链路层 | Frame | Ethernet, ARP | MAC 地址(48bit) | 交换机, 网卡 |

核心收益：解耦（切换 HTTP/2 不需改动 TCP 层）、可替换（IPv4→IPv6 上层无感知）、标准化（不同厂商设备在同一层面互操作）。

### 1.3 封装与解封装

数据逐层添加/剥离头部，过程如下：

```
发送端（封装）                             接收端（解封装）
应用层数据
  ↓ +TCP头  → [TCP Segment]
  ↓ +IP头   → [IP Packet]
  ↓ +MAC头  → [Ethernet Frame]
  ↓ 物理层（比特流）
==============================================>
                                            [Ethernet Frame] → 去MAC头
                                            [IP Packet]      → 去IP头
                                            [TCP Segment]    → 去TCP头
                                            应用层数据
```

**MTU 与 MSS**：链路层 MTU 通常 1500 字节，减去 IP 头(20)和 TCP 头(20)，MSS 通常 1460 字节。TCP 通过 MSS 协商避免 IP 分片——任何一个分片丢失都需重传整个 IP 包，性能急剧下降。

---

## 二、TCP 传输控制协议

### 2.1 三次握手

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
# 全连接队列大小：min(backlog, somaxconn)
```

### 2.2 四次挥手

```
主动关闭方                          被动关闭方
   |------ FIN (seq=u) ---------------->|  FIN_WAIT1 → CLOSE_WAIT
   |<----- ACK (ack=u+1) --------------|  FIN_WAIT2
   |<----- FIN (seq=v, ack=u+1) -------|  LAST_ACK
   |------ ACK (ack=v+1) ------------->|  TIME_WAIT(等待2MSL→CLOSED)
```

**为什么是四次？**

TCP 是全双工通信，每个方向需单独关闭。被动方收到 FIN 仅表示对方不再发数据，自己可能还有数据未发完，因此先回 ACK，等数据发完再发 FIN——FIN 和 ACK 分开发送，所以是四次。若被动方恰好也没有数据要发了，可合并 FIN+ACK，表现为"三次挥手"，但逻辑上仍是四次。

**TIME_WAIT 与 2MSL**

主动关闭方发送最后 ACK 后进入 TIME_WAIT，持续 2MSL（Linux 默认 60s）：

1. 确保最后 ACK 被收到：若 ACK 丢失，可接收重发的 FIN 并重发 ACK
2. 让旧连接的所有报文消失：防止旧连接延迟报文被新连接错误接收

**TIME_WAIT 过多的危害**：客户端端口耗尽。解决：

```bash
sysctl -w net.ipv4.tcp_tw_reuse=1              # 客户端端口快速回收
sysctl -w net.ipv4.ip_local_port_range="1024 65000"  # 扩大端口范围
```

服务端应让反向代理(Nginx)作为主动关闭方，保护好应用服务器。

**CLOSE_WAIT 过多排查**

CLOSE_WAIT 表示收到 FIN 但应用层未调用 close()——是**应用层 Bug**：

```bash
ss -tan state close-wait | awk '{print $4}' | awk -F: '{print $1}' | sort | uniq -c | sort -rn
```

| 原因 | 修复 |
|------|------|
| finally 块未关闭连接 | try-finally 保证 close() |
| 连接池泄露 | 用完归还，超时回收 |
| 异步回调未处理 | complete/error 路径都关闭 |

### 2.3 TCP 状态机速查

| 状态 | 含义 | 常见出现场景 |
|------|------|-------------|
| LISTEN | 等待连接请求 | 正常 |
| SYN_SENT | 已发送 SYN | 连接目标不可达 |
| SYN_RCVD | 收到 SYN，已回复 SYN+ACK | 半连接队列中 |
| ESTABLISHED | 已建立连接 | 正常 |
| FIN_WAIT1 | 已发 FIN | 短暂过渡 |
| FIN_WAIT2 | 收到 ACK，等对方 FIN | 对方应用层未 close() |
| CLOSE_WAIT | 收到 FIN，等应用层 close() | **应用层 Bug** |
| LAST_ACK | 已发 FIN，等 ACK | 短暂过渡 |
| TIME_WAIT | 等待 2MSL | 短连接过多 |

### 2.4 滑动窗口与流量控制

**为什么需要滑动窗口？** 停等协议每发一个包等一个 ACK，吞吐量极低。滑动窗口允许连续发送多个报文段：

```
已确认 | 已发送未确认 | 可发送未发送 | 不可发送
 [  ][  ][  ][  ] [  ][  ][  ][  ] [  ][  ][  ]
  ↑              ↑                ↑
 左边界         指针            右边界
```

**流量控制(rwnd)**：接收方在 ACK 头中的 Window 字段(16bit, 启用 Window Scale 可达 1GB)通告接收窗口。`rwnd = 接收缓冲区 - 已缓存未读取`。

**零窗口探测**：当 rwnd=0，发送方启动 Persist Timer 周期性发送 1 字节探测报文。

**糊涂窗口综合征**：接收方每次只消耗几个字节，导致大量小包。解决方案：接收方不通告微量窗口（Clark 方案），发送方使用 Nagle 算法。

### 2.5 拥塞控制

拥塞控制关心**网络容量**，流量控制关心**对方容量**。

**(1) 慢启动**：cwnd 初始 1~10 个 MSS(Linux 3.0+ init_cwnd=10)，每收到 ACK cwnd+=1 MSS，指数增长，直到达 ssthresh。

**(2) 拥塞避免**：cwnd≥ssthresh 后，每 RTT cwnd+=1 MSS，线性增长。

**(3) 快速重传**：收到 3 个重复 ACK 时不等超时直接重传。为什么是 3 次？乱序也会产生 1-2 个 Dup ACK。

**(4) 快速恢复**：ssthresh=cwnd/2, cwnd=ssthresh+3, 重传丢包, 此后每收一个 Dup ACK 窗口膨胀 cwnd+=1, 收新 ACK 后 cwnd=ssthresh 进入拥塞避免。

`实际发送窗口 = min(rwnd, cwnd)`

**(5) BBR 简介**：传统算法基于丢包判断拥塞，但缓冲区膨胀(Bufferbloat)导致高延迟无丢包。BBR 通过持续探测**最大带宽(BtlBw)**和**最小 RTT(RTprop)**建模管道瓶颈，在跨洋链路等有一定丢包的环境下性能优于 CUBIC。启用：

```bash
sysctl -w net.core.default_qdisc=fq
sysctl -w net.ipv4.tcp_congestion_control=bbr
```

### 2.6 TCP 选项

| 选项 | 作用 | 说明 |
|------|------|------|
| MSS | 协商最大数据段 | 通常 1460（以太网），避免 IP 分片 |
| SACK | 选择性确认 | 告知已收到的具体段，避免整窗口重传 |
| Timestamp | 精确 RTT + 防序列号回绕 | PAWS 机制 |
| Window Scale | 窗口缩放(左移0-14位) | 突破 65535 字节限制，典型 8MB |

SACK 对长肥管道尤其重要：无 SACK 时一个窗口内丢一个段需重传其后所有段(Go-Back-N)，有 SACK 只重传丢失的段。

### 2.7 Nagle 算法与延迟确认

**Nagle**：一个连接上最多只能有一个未确认的小包(< MSS)，在该包 ACK 到达前不能发其他小包。目的是减少小包数量。

**TCP_NODELAY**：禁用 Nagle，低延迟场景必须设置：

```java
Socket socket = new Socket();
socket.setTcpNoDelay(true);
```

**延迟确认(Delayed ACK)**：收到数据后等 200ms(Linux)，期待有反向数据捎带 ACK 或合并后续段的 ACK。

**死亡组合**：Nagle(等 ACK) + Delayed ACK(等 200ms) → 双向死锁 200ms。几乎所有实时通信场景都应设置 `TCP_NODELAY`。

### 2.8 Keep-Alive 机制

TCP Keep-Alive 是系统层面的保活，默认 2 小时才开始探测，对于应用层太慢：

```bash
# 仅用于检测死连接，不用于应用层心跳
net.ipv4.tcp_keepalive_time = 7200   # 2h 后开始探测
net.ipv4.tcp_keepalive_probes = 9    # 探测 9 次
net.ipv4.tcp_keepalive_intvl = 75    # 每次间隔 75s
```

**应用层心跳**更可控：能检测应用层僵死，跨代理和 NAT 更可靠：

```java
// Netty
pipeline.addLast(new IdleStateHandler(60, 0, 0, TimeUnit.SECONDS));
```

---

## 三、UDP 用户数据报协议

UDP 无连接、不可靠、面向报文，头部仅 8 字节（源端口、目的端口、长度、校验和）。

| 特性 | TCP | UDP |
|------|-----|-----|
| 连接 | 面向连接，全双工 | 无连接 |
| 可靠性 | 确认/重传/排序 | 不保证 |
| 顺序 | 有序交付 | 不保证 |
| 速度 | 慢（握手/拥塞控制） | 快 |
| 头部大小 | 20~60 字节 | 8 字节 |
| 控制 | 流量控制+拥塞控制 | 无（应用层负责） |
| 适用 | HTTP, SSH, MySQL | DNS, VoIP, 游戏, QUIC |
| 传输模式 | 字节流 | 报文 |

**UDP 适用场景**：
- **DNS**：单次请求-响应，报文 < 512B(EDNS0 4096)，超限回退 TCP
- **实时音视频**：丢几帧不影响，重传增加延迟
- **在线游戏**：状态更新频繁，丢一两个包关系不大
- **QUIC/HTTP3**：在 UDP 之上实现可靠性+多路复用+加密，避免 TCP 队头阻塞

---

## 四、HTTP 超文本传输协议

### 4.1 HTTP/1.1

**持久连接(Keep-Alive)**：HTTP/1.1 默认复用 TCP 连接，减少握手和慢启动开销。

**管道化(Pipelining)**：一次发多个请求不等响应，但有队头阻塞(HOL Blocking)——第一个响应慢则后续全部阻塞，主流浏览器默认禁用。

**队头阻塞**：单 TCP 连接上按序处理，是 HTTP/1.1 最大性能瓶颈。

**分块传输(Chunked)**：响应体大小未知时，`Transfer-Encoding: chunked` 分块发送：

```
HTTP/1.1 200 OK
Transfer-Encoding: chunked

5\r\nHello\r\n
6\r\n World\r\n
0\r\n\r\n       ← 结束标记
```

每块：`十六进制长度\r\n数据\r\n`。

### 4.2 HTTP/2

二进制协议，核心改进：

```
Stream 1 (HEADERS + DATA) ← /index.html
Stream 3 (HEADERS + DATA) ← /style.css        → 多路复用到单一 TCP 连接
Stream 5 (HEADERS + DATA) ← /app.js
```

**多路复用**：多 Stream 在一条 TCP 连接上交错传输帧，解决 HTTP 层面的 HOL Blocking。

**头部压缩(HPACK)**：静态字典(61 个常见头部) + 动态字典(通信中增量更新) + 哈夫曼编码，可压缩 85% 以上。

**服务器推送(Server Push)**：服务端主动推送客户端尚未请求的资源。但生产中使用率很低(浏览器缓存命中时浪费带宽)，Chrome 已移除支持。

**HTTP/2 的 TCP 队头阻塞**：解决了 HTTP 层面 HOL Blocking 但 TCP 层面仍有——底层 TCP 丢包时整个链路阻塞，所有 Stream 等重传。这是 HTTP/3 改用 QUIC 的核心动机。

### 4.3 HTTP/3

基于 QUIC(UDP 之上)，在用户态实现可靠传输+多路复用+TLS 1.3 加密：

| 特性 | HTTP/2 (TCP+TLS) | HTTP/3 (QUIC) |
|------|-------------------|---------------|
| 传输层 | TCP(内核态) | UDP+QUIC(用户态) |
| 连接迁移 | 不支持(四元组绑定) | 支持(Connection ID) |
| 多路复用 | TCP 层 HOL Blocking | Stream 独立，无 HOL |
| 握手 | 2~3 RTT | 0-RTT / 1-RTT |
| 加密 | TLS 可选 | 默认加密(TLS 1.3 内建) |

**连接迁移**：QUIC 通过 Connection ID 标识连接，不受 IP/端口变化影响，网络切换(WiFi→4G)无需重连。

**0-RTT**：若客户端曾连接过服务端(缓存 PSK)，第一个包即带业务数据。但存在重放攻击风险，仅适用幂等请求。

> 生产：内网优先 HTTP/2+gRPC，公网可在 CDN/网关启用 HTTP/3。

### 4.4 常见状态码

| 状态码 | 含义 | 后端关注 |
|--------|------|----------|
| 200 | OK | - |
| 201 | Created | RESTful POST 返回 |
| 204 | No Content | DELETE 常见 |
| 206 | Partial Content | 断点续传/Range |
| 301 | Moved Permanently | SEO 影响，浏览器缓存 |
| 302 | Found | 登录后跳转 |
| 304 | Not Modified | 协商缓存命中 |
| 400 | Bad Request | 参数校验 |
| 401 | Unauthorized | JWT 过期 |
| 403 | Forbidden | 权限不足 |
| 404 | Not Found | 资源/路由不匹配 |
| 429 | Too Many Requests | 触达限流 |
| 499 | Client Closed | Nginx 自定义：客户端断开 |
| 500 | Internal Server Error | 未捕获异常 |
| 502 | Bad Gateway | 上游不可达/非法响应 |
| 503 | Service Unavailable | 熔断/过载 |
| 504 | Gateway Timeout | 上游超时 |

### 4.5 Cookie、Session、Token

**Cookie**：`Set-Cookie` 响应在客户端存小段数据(4KB)。`HttpOnly`(防 XSS)、`Secure`(仅 HTTPS)、`SameSite`(防 CSRF)。

**Session**：服务端状态，Cookie 只存 Session ID。登录后服务端创建 Session(存 Redis/DB)，客户端带 Session ID Cookie。

**JWT**：无状态认证，`Header.Payload.Signature`。

| 方案 | 优点 | 缺点 |
|------|------|------|
| Session | 服务端可控，可主动失效 | 有状态，扩展需共享存储 |
| JWT | 无状态，天然分布式 | 无法主动失效，体积大 |

**生产实践**：双 Token 模式——短期 Access Token(JWT, 15min) + 长期 Refresh Token(Opaque, 存服务端可撤销)。

### 4.6 跨域(CORS)

浏览器的同源策略限制跨域资源访问。CORS 通过新增 HTTP 头声明允许的源：

```
GET /api/data
Origin: https://frontend.example.com

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://frontend.example.com
Access-Control-Allow-Credentials: true
```

**预检请求(Preflight)**：非简单请求(如 `Content-Type: application/json`)先发 OPTIONS 检查：

```
OPTIONS /api/data
Origin: https://frontend.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type

HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://frontend.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

注意：`Allow-Origin: *` 与 `Allow-Credentials: true` 互斥。可通过反向代理统一处理 CORS。**JSONP**(仅 GET)已基本被 CORS 取代，生产不要使用。

### 4.7 HTTP 缓存

**强缓存**——浏览器直接用缓存，不发请求：

| 头 | 含义 |
|----|------|
| `Cache-Control: max-age=3600` | 缓存 3600 秒(优先级 > Expires) |
| `public` | 可被代理/CDN 缓存 |
| `private` | 仅浏览器缓存 |
| `no-cache` | 每次协商(不是不缓存) |
| `no-store` | 完全不缓存 |
| `immutable` | 资源永不变化，不验证 |

**协商缓存**——发条件请求，服务端判断：

```
# 基于时间
If-Modified-Since: Mon, 01 Jan 2024 00:00:00 GMT
→ 304 Not Modified  或  200 OK + Last-Modified

# 基于内容摘要(优先级更高)
If-None-Match: "abc123"
→ 304 Not Modified  或  200 OK + ETag: "def456"
```

| 维度 | ETag | Last-Modified |
|------|------|----------------|
| 精度 | 内容变化即变 | 秒级，同秒修改检测不到 |
| 开销 | 计算 ETag 消耗 CPU | 开销小 |

**生产实践**：带 hash 的静态资源(`app.a1b2c3.js`)→ `Cache-Control: max-age=31536000, immutable`；HTML 入口 → `no-cache`；API 数据 → `private, max-age=0`。

---

## 五、HTTPS

### 5.1 TLS 1.2 握手 (ECDHE)

```
客户端                                          服务端
  |-- ClientHello(密码套件,随机数1) ------------>|
  |<- ServerHello(选定套件,随机数2)              |
  |   Certificate(证书链)                       |
  |   ServerKeyExchange(ECDHE公钥,签名) --------|
  |-- ClientKeyExchange(ECDHE公钥)              |
  |   ChangeCipherSpec(切换加密)                |
  |   Finished(加密校验) ----------------------->|
  |<- ChangeCipherSpec + Finished -------------|
  |======== 安全通信 (2 RTT) ====================|
```

**ECDHE 密钥交换**：双方各自生成临时 ECDHE 密钥对→交换公钥→各自算共享密钥→结合随机数用 PRF 派生会话密钥。E("Ephemeral") 表示每次握手的密钥对临时生成，提供**前向安全性(Forward Secrecy)**——证书私钥泄露也不影响历史会话。

RSA 密钥交换由客户端用证书公钥加密 Premaster Secret——证书私钥泄露则所有历史会话可解密，**无前向安全**。生产环境应禁用 RSA 密钥交换。

### 5.2 TLS 1.3 握手

```
客户端                                          服务端
  |-- ClientHello(密钥分享参数,随机数) --------->|
  |<- ServerHello(密钥分享参数)                 |
  |   {EncryptedExtensions}                    |
  |   {Certificate}                            |
  |   {Finished} ------------------------------|
  |-- {Finished} ----------------------------->|
  |======== 安全通信 (1-RTT) ====================|
```

**1-RTT**：ClientHello 中直接带 DH 密钥分享参数，无需额外往返。

**0-RTT**：客户端缓存 PSK，在 ClientHello 带 Early Data 直接通信。注意存在重放攻击风险，仅用幂等请求。

**关键改进**：移除 RSA 密钥交换、静态 DH、SHA-1/MD5、CBC 模式、RC4；仅保留 AEAD 套件；大部分握手消息加密。

### 5.3 证书链与 CA

```
根 CA (Root CA) — 自签名，预置在操作系统/浏览器
  ↓ 用根 CA 私钥签名
中间 CA (Intermediate CA) — 离线保存
  ↓ 用中间 CA 私钥签名
终端证书 (End-entity) — 绑定域名和公钥
```

验证：用上级证书公钥验证下级签名 → 递归到根 CA(信任锚点)。证书吊销通过 CRL/OCSP。OCSP Stapling 让服务端握手时附带 OCSP 响应，避免客户端额外查询。

### 5.4 中间人攻击防御

MITM 核心：攻击者分别与客户端和服务端建立加密连接，在中间解密篡改。

**证书锁定(Certificate Pinning)**：客户端预置证书/公钥指纹，握手时验证匹配。即使攻击者获取合法 CA 签发的伪造证书也能被检测。

```java
CertificatePinner pinner = new CertificatePinner.Builder()
    .add("api.example.com", "sha256/AAAA...").build();
```

Pinning 管理成本高(证书更换需客户端发版)。现代实践倾向 DNS CAA 记录(指定哪些 CA 可为该域名签发) + Certificate Transparency(公开可审计)。

---

## 六、DNS 域名系统

### 6.1 迭代查询 vs 递归查询

- **递归查询**：客户端→本地 DNS→根 DNS→.com NS→最终结果一层层回传。最简单但对递归 DNS 负担大
- **迭代查询**：本地 DNS 逐个查询根→`.com` NS→权威 NS，组装结果后返回。运营商 DNS→权威 DNS 通常用迭代

### 6.2 DNS 缓存层次

| 缓存位置 | TTL 行为 | 说明 |
|----------|----------|------|
| 浏览器缓存 | 独立，~1min | Chrome: `chrome://net-internals/#dns` |
| OS 缓存 | 遵从记录 TTL | `ipconfig /displaydns` |
| 递归 DNS | 可能延长 TTL | 8.8.8.8, 114.114.114.114 |
| 权威 DNS | 无缓存 | 直接返回查询结果 |

注意：部分递归 DNS 会忽略过短的 TTL(TTL 延展)，也存在负缓存(NXDOMAIN 结果)。

### 6.3 CDN 原理

CDN 通过 DNS 将用户导向最近的边缘节点：

```
用户 → 递归 DNS → CDN 权威 DNS(智能 DNS)
    → 根据来源 IP(地理位置/运营商/负载) 返回最近边缘节点 IP
    → 用户访问边缘节点
    → 未命中 → 回源 → 缓存
```

**后端关注**：
- 缓存穿透：大量 CDN miss 直接打回源站，需做好限流降级
- 缓存一致性：内容更新后需 Purge 或等 TTL 过期
- 日志：源站通过 `X-Forwarded-For` / `X-Real-IP` 获取真实 IP

### 6.4 DNS 优化

1. **减少查询次数**：域名收敛、HTTP/2 多路复用
2. **DNS Prefetch**：`<link rel="dns-prefetch" href="//api.example.com">`
3. **HTTPDNS**：绕过运营商 DNS，直接向 HTTPDNS 服务商请求，避免劫持并获得准确调度
4. **Anycast DNS**：多节点共享 IP，BGP 自动路由最近节点
5. **长连接+连接池**：减少建连频率

### 6.5 DNS 负载均衡

DNS Round-Robin 返回多个 A 记录轮询排序，实现粗粒度负载均衡：

```
example.com. 300 IN A 192.168.1.1
example.com. 300 IN A 192.168.1.2
example.com. 300 IN A 192.168.1.3
```

局限：缓存导致不均匀、无故障剔除、无法感知实时负载。GSLB 结合健康检查+地理位置更智能，但后端内部仍依赖 L4/L7 负载均衡器。

---

## 七、其他重要主题

### 7.1 WebSocket

通过 HTTP 升级握手建立全双工 WebSocket 连接：

```
客户端 → 服务端:
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==

服务端 → 客户端:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

`Sec-WebSocket-Accept = Base64(SHA1(Key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"))`，RFC 6455 固定魔数，防误升级。

**帧格式**：FIN(1bit) + opcode(Text=1, Binary=2, Close=8, Ping=9, Pong=10) + MASK(客户端→服务端必须掩码，服务端可选) + Payload。

**心跳**：协议内置 Ping/Pong 帧保活：
```java
ctx.writeAndFlush(new PingWebSocketFrame(Unpooled.EMPTY_BUFFER));
```

### 7.2 TCP 粘包与拆包

TCP 是字节流，无消息边界：

```
发送: write(AAA) write(BBB)  →  接收可能: AAABBB(粘包) / AA ABBB(拆包+粘包) / ...
```

**解决方案**：

| 方案 | 原理 | 示例 |
|------|------|------|
| 固定长度 | 每消息 N 字节 | `FixedLengthFrameDecoder` |
| 分隔符 | 特殊字符分隔 | `\r\n`(Redis, SMTP), `\0` |
| 长度字段 | 固定头含消息长度 | gRPC, Dubbo, Netty |

```java
// Netty 最通用的方案
pipeline.addLast(new LengthFieldBasedFrameDecoder(
    65535,   // maxFrameLength
    0,       // lengthFieldOffset
    4,       // lengthFieldLength
    0,       // lengthAdjustment
    4));     // initialBytesToStrip
```

**最佳实践**：魔术字(0xCAFEBABE) + 长度字段 + 消息体。魔术字用于快速校验和协议识别。

### 7.3 零拷贝(Zero Copy)

文件传输场景的数据路径对比：

```
传统: 磁盘→内核缓冲区→(CPU拷贝)→用户缓冲区→(CPU拷贝)→Socket缓冲区→网卡
      2 次 CPU 拷贝 + 4 次上下文切换

sendfile: 磁盘→内核缓冲区→(SG-DMA聚合)→Socket缓冲区→网卡
          0 次 CPU 拷贝 + 2 次上下文切换
```

```java
// Java NIO 零拷贝
FileChannel fc = new FileInputStream("file.txt").getChannel();
SocketChannel sc = SocketChannel.open();
fc.transferTo(0, fc.size(), sc);  // 内部调用 sendfile()
```

| 技术 | CPU 复制 | 适用场景 | 限制 |
|------|---------|----------|------|
| read+write | 2 | 通用 | 性能最差 |
| mmap+write | 1 | 大文件 | mmap 有开销 |
| sendfile | 0(SG-DMA) | 文件→Socket | 不能改数据 |
| splice | 0 | 两个 fd 间 | Linux 特有，管道中转 |

Kafka、Nginx 大量使用 sendfile 推送静态文件/日志消费。

---

## 八、排查工具与思路

### 8.1 tcpdump

```bash
# 抓 HTTP 流量
tcpdump -i eth0 -A -s 0 'tcp port 8080'

# 写入文件用于 Wireshark 分析
tcpdump -i eth0 host 10.0.0.5 -w capture.pcap

# 排查三次握手——抓 SYN 包
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn) != 0'

# 抓特定标志位(SYN+ACK)
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn|tcp-ack) == (tcp-syn|tcp-ack)'

# 抓 DNS
tcpdump -i eth0 -n udp port 53

# 读取 pcap 文件并过滤
tcpdump -r capture.pcap 'tcp port 443'
```

### 8.2 Wireshark

Server 上用 `tcpdump -w` 保存 pcap，本地分析。关键过滤器：
- `tcp.stream eq 0`——追踪单条 TCP 流
- `tcp.analysis.retransmission`——分析重传
- `tcp.analysis.out_of_order`——分析乱序
- Statistics → Flow Graph——TCP 流时间线

### 8.3 ss / netstat

```bash
# 按状态过滤与统计
ss -tan state time-wait
ss -tan | awk 'NR>1{print $1}' | sort | uniq -c | sort -rn

# Socket 内存使用
ss -tam

# 全局统计
netstat -s | grep -E "timeout|retrans|reset"

# 查看监听
ss -tlnp
```

### 8.4 mtr

实时展示各跳丢包率和延迟：

```bash
mtr -r -c 100 10.0.0.5
mtr -r -c 100 -n 10.0.0.5   # 不解析域名
```

解读：某一跳丢包但后续不丢——该跳限制 ICMP 速率，非真丢包；某一跳及以后全丢——该跳可能是故障点。

### 8.5 常见问题排查思路

**(1) 连接被拒绝(ECONNREFUSED)**
```
ss -tlnp | grep 端口        # 检查是否监听
iptables -L -n -v           # 检查防火墙
```

**(2) 连接超时(SYN 无响应)**
```
ping 目标IP                 # 三层可达性
telnet 目标IP 端口          # 端口可达性
traceroute 目标IP           # 路由路径
tcpdump -i eth0 host 目标IP # 看 SYN 发出/RST 回应
```

**(3) 大量 TIME_WAIT**
```bash
ss -tan state time-wait | wc -l
# 排查：是否启用连接池(HTTP Client/DB 连接池)
# 解决：tcp_tw_reuse, 扩大端口范围, 让 Nginx 做主动关闭方
```

**(4) 大量重传**
```bash
ss -ti                       # 每个连接 TCP 信息(rtt,rto,cwnd,retrans)
ethtool -S eth0 | grep drop  # 网卡丢包
ip -s link show eth0         # 接口统计
```

**(5) 负载不均**
```
原因：长连接聚拢/ DNS 缓存集中/ Keep-Alive timeout 过大
排查：查看各 upstream ESTABLISHED 分布
修复：调整 keepalive_timeout / keepalive_requests
```

**排查口诀**：先看连通(ping/telnet)→再看握手(tcpdump SYN/RST)→再看传输(netstat -s/ss -ti)→再看应用(日志/监控)。

---

## 参考资料

- RFC 793 - Transmission Control Protocol
- RFC 7540 - HTTP/2
- RFC 9000 - QUIC
- RFC 8446 - TLS 1.3
- RFC 6455 - WebSocket Protocol
- RFC 1034/1035 - DNS
- 《TCP/IP 详解 卷一：协议》— Kevin Fall, W. Richard Stevens
- 《HTTP/2 in Action》— Barry Pollard
- 《图解 HTTP》— 上野宣
