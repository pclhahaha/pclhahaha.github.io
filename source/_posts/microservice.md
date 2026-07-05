---
title: 微服务架构
date: 2026-07-05 00:00:00
updated: 2026-07-05 00:00:00
tags:
  - 微服务
  - Spring Cloud
  - 服务治理
  - API网关
categories:
  - 分布式
---
## 一、微服务架构概述

### 1.1 架构演进：单体 → SOA → 微服务

**单体架构**是软件系统的起点——所有功能模块打包在一个进程内，部署在一个应用服务器上。早期项目规模小、团队人数少时，单体架构交付效率最高：本地启动即开发，打包即部署。

但随着业务膨胀，单体架构的劣化不可避免：
- **代码耦合**：修改订单模块可能意外影响库存模块，IDE 里一次类引用搜索能跳出几十个调用方，没人敢改。
- **部署瓶颈**：改一行文案也要全量重新部署，回归用例上千条，发布窗口从每周一次拖到每月一次。
- **扩展困难**：热点模块（如秒杀）无法独立扩容，只能整体水平扩展，资源浪费严重。
- **技术锁定**：Java 1.6 升 1.8 全量回归，不敢动。

**SOA（面向服务架构）**试图用 ESB（企业服务总线）解决问题，把系统拆成服务，通过总线通信。但 ESB 本身成了新的单点：所有流量经过总线，总线挂了全站瘫痪；所有服务依赖总线的 XML Schema，版本协调成本极高。SOA 的理念正确但落地太重，属于"治大国如烹小鲜"翻车现场。

**微服务架构**继承了 SOA 的拆分思想，但去掉了重量的 ESB，改用轻量级通信（HTTP/RPC），强调"去中心化"治理。每个服务独立部署、独立扩展、独立技术选型。Martin Fowler 在 2014 年的文章将微服务推向主流，Spring Cloud、Dubbo、K8s 等生态进一步降低了落地成本。

```
单体架构                        SOA                          微服务架构
┌─────────────────────┐    ┌──────────────┐        ┌──────┐ ┌──────┐ ┌──────┐
│                     │    │     ESB      │        │Order │ │User  │ │Pay   │
│  ┌───────┬───────┐  │    │  (智能管道)   │        │Svc   │ │Svc   │ │Svc   │
│  │ Order │ User  │  │    │              │        └──┬───┘ └──┬───┘ └──┬───┘
│  │ Pay   │ Stock │  │    ├──┼──┼──┼──┤           │       │        │
│  └───────┴───────┘  │    │  │  │  │  │      ─────┼───────┼────────┼──── (轻量通信)
│     单一进程+DB      │    │Order User  Pay│     ┌──┴──┐ ┌──┴──┐ ┌──┴──┐
└─────────────────────┘    └──────────────┘     │ DB  │ │ DB  │ │ DB  │
                                                └─────┘ └─────┘ └─────┘
```

### 1.2 微服务的核心优势

| 维度       | 说明                                                                 |
| ---------- | -------------------------------------------------------------------- |
| 独立部署   | 单服务发布不需要全量回归，发布时间从"月级"缩到"天级/小时级"         |
| 技术异构   | 推荐服务用 Python 跑模型，订单服务用 Java 保证事务，各取所长         |
| 弹性伸缩   | 大促前只扩容支付和下单服务，不用带着后台管理系统一起扩三倍           |
| 故障隔离   | 评论服务挂了不影响用户浏览商品，这一点单体做不到                     |
| 组织对齐   | 服务边界即团队边界，2-pizza team 拥有从开发到运维的完整所有权       |
| 可维护性   | 单个服务代码量可控（1-2 万行），新人两周能上手，不用看半年代码       |

### 1.3 微服务的代价（你不会从技术大会 PPT 上看到的）

- **网络开销**：一次本地方法调用变成一次 RPC 调用，延迟从纳秒级升到毫秒级。分布式系统中网络是不可靠的——延迟、丢包、超时，让你的代码必须处处处理"对方挂了怎么办"。
- **数据一致性**：单体的事务 ACID 变成分布式事务 BASE，扣库存和扣余额不在同一个数据库，最终一致性引入大量补偿逻辑和中间状态。"只是拆了两个服务而已，订单状态同步代码写了 2000 行"是真实故事。
- **运维复杂度**：10 个容器 vs 500 个容器，部署流水线、日志采集、链路追踪、监控告警的量级完全不同。K8s + Istio + Prometheus + ELK + Skywalking，光运维链路上的组件比业务服务还多。
- **调试困境**：一个请求经过 8 个服务，每一步都可能有独立的超时/重试策略，定位问题需要全链路追踪。本地启动一个带依赖的服务可能需要 16G 内存。
- **分布式数据管理**：每个服务有自己的数据库，跨服务的联合查询变成 API 聚合。DBA 习惯的 SQL join 不复存在。

### 1.4 何时用/何时不用微服务

| 场景                       | 建议               |
| -------------------------- | ------------------ |
| 创业初期 / MVP 验证        | 单体，跑起来再说   |
| 团队 < 5 人                | 单体，拆了没人维护 |
| 业务模型未稳定，频繁调整   | 单体，服务边界难定 |
| 性能敏感、低延迟要求       | 单体或合并部署     |
| 团队 > 20 人，多团队并行   | 微服务，独立迭代   |
| 部分模块有独立扩容需求     | 微服务，精准扩缩容 |
| 多种技术栈需求             | 微服务，技术异构   |
| 遗留系统改造               | 微服务，逐步替换   |

一句话总结：**当单体带来的组织成本大于微服务带来的技术成本时，就是切入微服务的拐点。**

### 1.5 康威定律

> "设计系统的组织，其产生的设计等同于这些组织之间的沟通结构。" —— Melvin Conway, 1967

微服务的边界划分不是纯技术决策，而是**组织结构决策**。如果两个服务分别由不同团队维护，但团队之间需要频繁沟通才能完成一个需求，那这个服务边界大概率划错了。反过来，任何试图绕过组织问题、纯靠技术手段"设计"微服务的行为，最终都会被组织沟通成本反噬。

引申结论：
- **不要让同一个团队维护超过 3 个核心服务**——你无法同时 deep dive 多个领域。
- **服务拆分要先拆团队**——先有人，再划边界，然后写代码。
- **跨团队 API 必须契约化**——接口就是团队间的合同，需要版本管理和兼容性承诺。

---

## 二、服务通信

### 2.1 RPC vs REST vs MQ 对比

微服务间通信主要三种方式，无绝对优劣，只有场景适配：

| 维度         | RPC (Dubbo/gRPC)        | REST (Spring MVC)     | MQ (RocketMQ/Kafka)      |
| ------------ | ----------------------- | --------------------- | ------------------------ |
| 通信模式     | 同步（可异步）          | 同步                  | 异步                     |
| 传输协议     | TCP/HTTP2              | HTTP/1.1 或 HTTP/2    | TCP（私有协议）          |
| 序列化       | Protobuf/Hessian 等     | JSON/XML              | 自定义（二进制）         |
| 性能         | 高（长连接+二进制）     | 中（短连接+文本）     | 高（批量+顺序写）        |
| 耦合度       | 强（接口依赖）          | 中（API 契约）        | 弱（事件驱动）           |
| 服务治理     | 丰富（注册/路由/限流）  | 需借助网关            | 依赖 MQ 自身能力         |
| 典型场景     | 内部服务间高频调用      | 对外 API/跨语言       | 异步解耦/削峰/事件驱动   |

**选择策略**：
- 核心交易链路、对延迟敏感 → RPC（Dubbo/gRPC）
- 对外开放 API、跨语言调用 → REST
- 非核心链路、异步解耦 → MQ（详见 [消息队列](mq.md)）

**关于性能**：不要一上来就"JSON 慢我要用 Protobuf"。先看你的业务瓶颈在哪——如果整个请求 90% 的时间花在数据库查询上，序列化省下的 5ms 毫无意义。性能优化要遵循阿姆达尔定律：先优化占比大的部分。

### 2.2 序列化框架对比

> 本节简要对比，序列化协议及 gRPC 详细实现见 [RPC 与异步编程](rpc-async.md)。

| 框架       | 序列化方式 | 跨语言 | 性能   | 可读性 | 适用场景           |
| ---------- | ---------- | ------ | ------ | ------ | ------------------ |
| Protobuf   | 二进制     | 是     | 极高   | 差     | gRPC，内部高性能   |
| Hessian    | 二进制     | 有限   | 高     | 差     | Dubbo 默认         |
| Kryo       | 二进制     | 否     | 极高   | 差     | Java 内部高性能    |
| JSON       | 文本       | 是     | 一般   | 好     | REST API，跨语言   |
| Fastjson   | 文本       | 否     | 较高   | 好     | Java REST 场景     |

生产建议：
- **内部服务** → Protobuf + gRPC，强类型契约，编译器保证接口一致性
- **开放 API** → JSON + REST，可读性 > 性能
- **Dubbo 项目** → 默认 Hessian，高并发可换 Protobuf 或 Kryo
- **避免 Fastjson** → 安全漏洞频发，迁移到 Jackson

### 2.3 服务调用链

一次 RPC 调用在微服务体系下经历的路由远比看上去复杂：

```
Client
  │
  ├─ 1. 服务发现 ────────────► 从注册中心获取服务列表
  ├─ 2. 负载均衡 ────────────► 选定目标节点
  ├─ 3. 序列化请求 ──────────► 对象 → 字节流
  ├─ 4. 网络发送 ────────────► TCP/HTTP 传输
  │
Server
  ├─ 5. 反序列化请求 ────────► 字节流 → 对象
  ├─ 6. 执行业务逻辑 ────────► 业务处理
  ├─ 7. 序列化响应 ──────────► 对象 → 字节流
  └─ 8. 返回结果
```

每一步都需要容错保障：

**超时控制**：每一次 RPC 调用必须设置超时，且超时时间应逐级递减——上游超时 3s，下游就要设 2s，给链路留出传播和重试的 buffer。放任不设超时等于说"这个线程愿意等一辈子"，线程池必炸。

```java
// Dubbo 超时配置（消费端）
@DubboReference(timeout = 3000, retries = 2)
private UserService userService;
```

**重试策略**：不是所有失败都应该重试。
- **幂等写操作** → 可重试（如库存预扣，有幂等键保证）
- **非幂等写操作** → 禁止重试（如扣款，可能重复扣）
- **读操作** → 可重试，但需控制次数（v2~3 次），避免雪崩

**负载均衡**：见第六章，客户端负载均衡（Ribbon/LoadBalancer）在调用前从注册中心获取的实例列表中选一个节点。

---


```yaml
# application.yml - Nacos 服务注册
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        namespace: dev          # 环境隔离
        group: DEFAULT_GROUP
        ephemeral: true         # AP 模式
```

```java
// 服务调用 + 负载均衡
@Bean
@LoadBalanced  // 启用 Ribbon/LoadBalancer 客户端负载均衡
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// 调用时使用服务名，自动从 Nacos 获取实例列表
String result = restTemplate.getForObject(
    "http://order-service/orders/1", String.class);
```

### 3.4 Eureka

Eureka 是 Netflix 开源的服务发现组件，坚定的 **AP** 设计——宁可返回过期数据也不拒绝服务。

**自我保护模式**（Self Preservation Mode）：Eureka Server 在 15 分钟内心跳续约率低于 85% 时进入自我保护模式，不再剔除任何实例。这是它 AP 哲学的集中体现：网络分区的另一端可能服务仍然健康，不能盲目剔除。

```
正常模式:
  Eureka Server: "A 心跳超时，剔除" → Consumer 不会再调用 A

自我保护模式:
  Eureka Server: "心跳丢失比例过高，可能是网络问题，我不剔除了"
  → Consumer 可能继续调用到不健康的 A（比调不到任何实例要好）
```

**与 Nacos 对比**：
| 特性         | Eureka                     | Nacos                      |
| ------------ | -------------------------- | -------------------------- |
| CAP 模型     | AP                         | AP + CP 切换               |
| 一致性协议   | Peer to Peer 异步复制      | Distro (AP) / Raft (CP)    |
| 服务健康检查 | 被动（心跳超时）           | 主动（TCP/HTTP/MySQL 探测） |
| 配置管理     | 无（需配合 Spring Config） | 内置                       |
| 自我保护     | 有（自动触发）             | 无                         |
| 控制台       | 简单，功能少               | 功能完善，中文友好         |
| 社区活跃度   | 停止维护（2.x）            | 持续活跃                   |

**结论**：新项目建议直接用 **Nacos**——它同时承担服务发现和配置中心，一个组件顶两个（Nacos = Eureka + Config）。Eureka 2.x 已停止维护，Spring Cloud 官方也推荐 Nacos 作为替代。

### 3.5 Consul / etcd

| 特性     | Consul                     | etcd                      |
| -------- | -------------------------- | ------------------------- |
| CAP      | CP                         | CP                        |
| 协议     | Raft                       | Raft                      |
| 健康检查 | 内置丰富                   | 需配合（Lease + Watch）   |
| KV 存储  | 有                         | 核心能力                  |
| 多数据中心| 原生支持                   | 需自行实现                |
| 生态     | HashiCorp 生态丰富         | K8s 默认存储，运维成熟    |

**选型建议**：
- 已用 K8s → 直接用 **K8s Service + CoreDNS** 做服务发现，etcd 是基础设施层
- 已用 Spring Cloud → 用 **Nacos**，与 Spring Cloud 集成最深度
- 多数据中心/混合云 → 考虑 **Consul**，原生多数据中心支持
- 内部基础设施/分布式锁 → etcd，K8s 标配

---

## 四、配置中心

### 4.1 为什么需要配置中心

配置中心解决的远不止"把配置从 jar 包里拿出来"这么简单：

| 需求           | 传统方式（application.yml 内置） | 配置中心                       |
| -------------- | -------------------------------- | ------------------------------ |
| 动态刷新       | 改配置 → 重新部署                | 改配置 → 实时生效              |
| 版本管理       | Git，但和发布耦合                 | 每次变更可追溯/回滚             |
| 灰度发布       | 靠发布系统控制                   | 配置层面精准控制（IP/标签/比例） |
| 权限审计       | 谁改了不知道                     | 操作人+时间+内容全记录          |
| 环境隔离       | 靠文件名（dev/prod）管理         | namespace 物理隔离              |
| 配置复用       | 公共配置每个服务复制一份         | 公共配置共享，改动同步          |

### 4.2 Apollo

Apollo（携程开源）在配置管理领域"出道即巅峰"，其设计理念至今是配置中心的最佳实践。

**架构组件**：

```
┌────────────────┐   ┌──────────────────┐   ┌───────────────────┐
│   Portal       │   │   Admin Service  │   │   Config Service  │
│ (管理界面)      │   │   (配置修改/发布) │   │   (配置读取/推送)   │
│                │   │                  │   │                   │
│  - 配置编辑     │──►│  - 校验/审计      │──►│  - 客户端拉配置     │
│  - 发布审批     │   │  - 发布/回滚      │   │  - 长轮询推送       │
└────────────────┘   └──────────────────┘   └───────────────────┘
         │                    │                        │
         └────────────────────┼────────────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │       MySQL       │
                    └───────────────────┘
```

**发布回滚机制**：
- 每一次发布生成一个 ReleaseKey
- Config Service 对比客户端上报的 ReleaseKey，有变更时推送新配置
- 发布后发现问题 → 一键回滚到上一版本（本质是发布一个历史版本的快照）
- 支持**灰度发布**：可以指定部分实例/IP 先接收新配置，验证通过后全量发布

**客户端长轮询**：Apollo 客户端不是定时轮询（浪费连接），而是长轮询——客户端发起 HTTP 请求，服务端 hold 住连接（默认 60s），在这期间配置有变更立即返回，没有变更则超时返回空，客户端立即发起下一次长轮询。

```java
// Apollo 客户端使用
@Value("${timeout:3000}")           // 支持热更新（需 @RefreshScope）
private int timeout;

@ApolloConfigChangeListener          // 监听配置变更
private void onChange(ConfigChangeEvent event) {
    for (String key : event.changedKeys()) {
        ConfigChange change = event.getChange(key);
        log.info("Config {} changed: {} -> {}", key, change.getOldValue(), change.getNewValue());
    }
}
```

### 4.3 Nacos Config

Nacos Config 的设计思想：**dataId + group + namespace** 三级隔离。

```
Namespace (环境隔离)
  └── Group (应用分组)
        └── Data ID (具体配置文件)
```

```properties
# Data ID 格式: ${prefix}-${spring.profiles.active}.${file-extension}
# 例: order-service-dev.yaml

# bootstrap.yml
spring:
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        namespace: prod                  # 生产环境
        group: ORDER_GROUP              # 订单域分组
        file-extension: yaml
        shared-configs:                  # 共享配置
          - data-id: common.yaml
            group: DEFAULT_GROUP
            refresh: true
```

**Nacos Config vs Apollo**：

| 维度       | Nacos Config        | Apollo                   |
| ---------- | ------------------- | ------------------------ |
| 强项       | 服务发现+配置一体   | 配置管理专精             |
| 灰度发布   | 支持，功能较简单    | 支持，功能完善           |
| 权限审批   | 基础                | 完善（发布审批/操作审计） |
| 版本对比   | 基础                | 可视化 diff              |
| 客户端语言 | Java/Go/Python/Node | 主要是 Java              |
| 选型建议   | 已用 Nacos 服务发现 | 对配置管理要求较高的场景  |

### 4.4 热更新原理

**Spring Cloud Config + Bus 刷新**：

Spring Cloud Config 通过 Git 存储配置，配合 Spring Cloud Bus（消息总线）实现广播刷新：

```
开发者 push 配置到 Git
    │
    ▼
POST /actuator/bus-refresh 到任意一个 Config Client
    │
    ▼
Spring Cloud Bus 通过 MQ 广播 RefreshRemoteApplicationEvent
    │
    ▼
所有监听此事件的 Client 执行 ContextRefresher.refresh()
    │
    ▼
被 @RefreshScope 注解的 Bean 重新初始化，新配置生效
```

**Nacos/Apollo 的推送机制**比 Spring Cloud Config 更优雅——不需要主动调用 refresh 端点，客户端长轮询自动感知并推送变更。生产环境建议优先使用 Nacos 或 Apollo 的推送方案，Spring Cloud Config + Bus 适合已部署 RabbitMQ/Kafka 且不希望引入额外组件的场景。

---


> 📖 独立文章：[API 网关](/api-gateway/)

## 六、负载均衡

### 6.1 客户端负载均衡 vs 服务端负载均衡

```
【服务端负载均衡】
Client ──► LVS/Nginx ──► Server A
                        ──► Server B
                        ──► Server C
特点: 请求先到 LB，LB 转发给后端。LB 成为单一入口，需要 HA。

【客户端负载均衡】
Client ──► 1. 从注册中心拉服务列表
        ──► 2. 本地选一个 Server
        ──► 3. 直连 Server A

特点: Client 自己做负载均衡，注册中心只在服务列表变更时通知。
      省去了中间 LB，但每个 Client 都要引入负载均衡逻辑。
```

**为什么微服务场景推荐客户端负载均衡？**
- 少一跳：Client → Server，省去 LB 中转，延迟更低
- 无单点：每个 Client 独立决策，LB 挂了不影响已拉取到列表的 Client
- 更灵活：可以按标签、权重、版本等维度定制路由策略

### 6.2 Ribbon / Spring Cloud LoadBalancer

Ribbon 是 Netflix 的客户端负载均衡器，Spring Cloud 后来推出了自己的 LoadBalancer 作为替代（Ribbon 已进入维护模式）。

> 以下以 Ribbon 为例说明策略，Spring Cloud LoadBalancer 原理类似。

**负载均衡策略**：

| 策略                     | 说明                                     |
| ------------------------ | ---------------------------------------- |
| RoundRobinRule           | 轮询，按顺序依次选择                     |
| RandomRule               | 随机                                     |
| WeightedResponseTimeRule | 加权，响应越快权重越高                   |
| BestAvailableRule        | 选择并发请求最小的实例                   |
| RetryRule                | 在轮询基础上加重试（超时/失败换下个）    |
| AvailabilityFilteringRule | 过滤掉熔断的/高并发的实例               |
| ZoneAvoidanceRule（默认）| 复合：先按 Zone 过滤，再轮询             |

```java
// Ribbon 自定义负载均衡规则（全局）
@Bean
public IRule ribbonRule() {
    return new WeightedResponseTimeRule();
}

// 针对特定服务
order-service:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
```

### 6.3 负载均衡算法实现思路

**加权轮询**：每个实例分配一个固定权重，权重大的实例被选中的概率更大。实现方式——维护一个选择序列：如权重 3:2:1，序列为 [A,A,A,B,B,C]，然后在序列上轮询。更优雅的实现是 Nginx 的平滑加权轮询算法，每次选出当前权重最大的实例后减去总权重，保证分布均匀。

**最小连接数**：实时维护每个实例的活跃连接数（注意不是累计请求数），请求进来时选连接数最小的那个。常用于长连接场景（如 WebSocket 网关），短连接场景差距不大。

**一致性哈希**：将请求的某个 key（如 userId）哈希映射到哈希环上，顺时针找到最近的服务器节点。优势是同一用户的请求永远落到同一服务器，缓存命中率高；缺点是增加/删除节点时只有相邻节点受影响。

```java
// 一致性哈希核心逻辑（简化）
public Server select(List<Server> servers, String key) {
    TreeMap<Integer, Server> ring = new TreeMap<>();
    for (Server s : servers) {
        for (int i = 0; i < VIRTUAL_NODES; i++) {
            ring.put(hash(s.getHost() + "#" + i), s);
        }
    }
    int hash = hash(key);
    Map.Entry<Integer, Server> entry = ring.ceilingEntry(hash);
    if (entry == null) {
        entry = ring.firstEntry(); // 超过最大值，取第一个
    }
    return entry.getValue();
}
```

**生产建议**：
- 默认用加权轮询即可，实现简单、分布均匀
- 实例性能不均匀时用加权，按 CPU 核数/内存比例配权重
- 有状态服务（如 WebSocket 会话）用一致性哈希，让同一用户固定落到同一实例

---


---

## 八、Spring Cloud 体系

### 8.1 核心组件全景图

```
                     ┌─────────────────────────┐
                     │     Spring Cloud         │
                     │     (分布式系统工具箱)     │
                     └───────────┬─────────────┘
                                 │
        ┌────────────┬───────────┼───────────┬────────────┐
        ▼            ▼           ▼           ▼            ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
   │  Config  │ │ Discovery│ │ Gateway  │ │CircuitBr │ │  Bus     │
   │   Center │ │  (注册中心)│ │  (网关)  │ │ (熔断器) │ │ (消息总线)│
   └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘
   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
   │  Sleuth  │ │  Stream  │ │OpenFeign │ │LoadBalan │ │  Config  │
   │ (链路追踪)│ │ (消息驱动)│ │(声明式调用)│ │ (负载均衡)│ │ (配置)   │
   └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘
```

**Netflix OSS 体系**（一代目，渐退役）：
- Eureka → 注册中心
- Ribbon → 客户端负载均衡
- Hystrix → 熔断器
- Zuul → 网关
- Feign → 声明式 HTTP 客户端

**Spring Cloud Alibaba 体系**（二代目，推荐）：
- Nacos → 注册中心 + 配置中心（取代 Eureka + Config）
- Sentinel → 流量治理（取代 Hystrix）
- Seata → 分布式事务
- RocketMQ → 消息驱动

### 8.2 Spring Cloud Alibaba 迁移指南

```
原 Netflix 栈                      Alibaba 替代
─────────────────                ──────────────────
Eureka         ───►   Nacos Discovery
Spring Config  ───►   Nacos Config
Hystrix        ───►   Sentinel
Ribbon         ───►   Spring Cloud LoadBalancer (Spring 官方)
Zuul           ───►   Spring Cloud Gateway (Spring 官方)
```

**迁移重点**：
1. Nacos 同时替代 Eureka + Config，一个组件解决两个问题，运维成本减半
2. Sentinel 比 Hystrix 更灵活——控制台动态下发规则，无需重启，细粒度到参数级别
3. Spring Cloud Gateway 比 Zuul 性能好，且是官方主推，生态兼容性最优
4. 保留 Feign（声明式调用）和 Sleuth（链路追踪），它们与底层框架解耦

### 8.3 Spring Cloud vs K8s Service

这是一个经典争论："我们用了 K8s，还需要 Spring Cloud 吗？"

| 能力             | Spring Cloud              | Kubernetes Service         |
| ---------------- | ------------------------- | -------------------------- |
| 服务发现         | Nacos/Eureka              | CoreDNS + Service          |
| 负载均衡         | Ribbon/LoadBalancer       | kube-proxy (iptables/IPVS) |
| 配置管理         | Nacos/Apollo/Config       | ConfigMap/Secret           |
| 熔断限流         | Sentinel/Hystrix          | 无原生支持                 |
| 网关             | Spring Cloud Gateway      | Ingress Controller         |
| 服务间通信       | Feign/RestTemplate        | Pod IP 直连                |
| 部署方式         | 虚拟机/Docker             | 仅 Pod                     |
| 语言限制         | Java（Spring 生态）       | 无限制                     |
| 灰度/泳道        | 丰富（Nacos+Gateway）     | 需 Istio/Linkerd 配合      |

**结论**：
- 纯 Java Spring 团队 → Spring Cloud 全套，上层应用治理能力更强
- 混合语言多 → K8s Service + Istio，用 Sidecar 透出流量治理
- 中小团队/快速交付 → Spring Cloud 全套，K8s 只做容器编排
- 大型企业多语言多团队 → K8s + Istio（Service Mesh），做基础设施层统一治理

两者不互斥——大多数公司是 Spring Cloud（应用层治理）+ K8s（容器编排）混用。

---

## 九、微服务治理实践

### 9.1 服务划分原则：DDD 限界上下文

服务划分是微服务最难的第一步。划得太大 → 退化成分布式单体（Distributed Monolith）；划得太小 → 调用链过长、事务协调爆炸、运维崩溃。

**DDD 限界上下文（Bounded Context）是微服务拆分的核心方法论**：

```
电商业务 - 限界上下文划分

┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│   商品上下文    │   │   订单上下文    │   │   用户上下文    │
│               │   │               │   │               │
│  - 商品 (SKU)  │   │  - 订单 (Order)│   │  - 用户 (User) │
│  - 类目        │   │  - 购物车      │   │  - 地址        │
│  - 库存        │   │  - 支付        │   │  - 积分        │
│  - 价格        │   │  - 物流        │   │               │
└───────┬───────┘   └───────┬───────┘   └───────┬───────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                    ┌───────▼───────┐
                    │   营销上下文    │
                    │               │
                    │  - 优惠券      │
                    │  - 活动        │
                    │  - 推荐        │
                    └───────────────┘
```

**判断拆分质量的几个指标**：
- 两个服务之间是否需要**分布式事务**？频繁需要 → 可能不该拆分
- 一个业务变更是否需要同时变更多个服务？几乎每次都连坐 → 边界有问题
- 两个服务的数据是否需要 join 查询？极其频繁 → 考虑合并

**实践口诀**：
> 先单体跑通业务 → 识别限界上下文 → 先拆边缘业务练手 → 再拆核心链路  
> **不要一上来就按"一个表一个服务"拆，那是地狱模式。**

### 9.2 灰度发布 / 蓝绿部署

**灰度发布**（金丝雀发布）：新版本只部署少量实例，按条件让部分用户流量打到新版本上。观察新版本无异常后，逐步扩大范围直到 100%。

```
生产环境
┌──────────────────────────────────────────┐
│ 实例 v1.0 (90% 流量)                      │
│ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐│
│ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘│
│                                          │
│ 金丝雀实例 v1.1 (10% 流量)                │
│ ┌────┐                                   │
│ └────┘  ← 只放 10% 用户（内部员工/白名单） │
└──────────────────────────────────────────┘
```

实现方式：
```yaml
# Nacos 元数据灰度路由
spring:
  cloud:
    nacos:
      discovery:
        metadata:
          version: v1.1        # 金丝雀版本标记
```

```java
// Gateway 灰度路由
@Bean
public GlobalFilter canaryFilter() {
    return (exchange, chain) -> {
        String version = exchange.getRequest().getHeaders().getFirst("X-Version");
        if ("v1.1".equals(version)) {
            // 路由到金丝雀实例
            exchange.getAttributes().put("nacos-version", "v1.1");
        }
        return chain.filter(exchange);
    };
}
```

**蓝绿部署**：同时维护两套完整环境（蓝 = 当前，绿 = 新版），新版验证通过后流量全部切换到绿。回滚就是切回蓝。

| 维度       | 灰度发布                           | 蓝绿部署                       |
| ---------- | ---------------------------------- | ------------------------------ |
| 资源消耗   | 小（少数金丝雀节点）               | 大（两套完整环境）             |
| 回滚速度   | 快（摘流量）                       | 快（切流量）                   |
| 验证覆盖   | 渐进式，发现隐性问题更好           | 全量时才有真实流量             |
| 适用场景   | 日常发布                           | 重大版本/不兼容升级            |

### 9.3 全链路压测

微服务架构下的全链路压测不再是"JMeter 打一个接口"那么简单：

**核心挑战**：
- 如何识别"压测流量"并隔离，避免污染线上数据？
- 如何让全链路（Gateway → A → B → C → DB）的压测标传递不丢失？
- 如何让 MQ 的压测消息不被消费者当真？

**解决思路**：
1. **流量染色**：在 Gateway 层为压测流量打标（HTTP Header: `X-Stress-Test: true`），并在 Dubbo 的 RpcContext、MQ 的 Message Header 中透传
2. **数据隔离**：压测请求使用独立的影子表（如 `order → order_shadow`）或通过 SQL hint 路由到另一个数据源
3. **中间件隔离**：Message → 独立的压测 Topic；Redis → 独立 DB index
4. **Mock 外部依赖**：外部支付/短信等调用在压测时走 Mock 通道

### 9.4 API 版本管理

微服务不可避免会遇到 API 不兼容升级。不做版本管理 → 所有调用方必须同步升级 → 本质上是分布式单体。

**三种常见方案**：

| 方案           | 示例                        | 优点             | 缺点                 |
| -------------- | --------------------------- | ---------------- | -------------------- |
| URL 版本       | `/api/v1/orders` `/api/v2/orders` | 清晰直观      | 路由膨胀，代码重复   |
| Header 版本    | `Accept: version=v1`        | URL 保持干净     | 调试不便，不易测试   |
| 请求参数版本   | `/api/orders?version=v1`    | 简单             | 参数污染，不够 REST  |

**推荐**：对外 API 使用 URL 版本（透明、好调试、方便网关路由），内部服务使用 Header 版本（干净），同时遵循以下原则：
- **向下兼容优先**：新增字段不删字段，默认值处理好
- **同一时刻只维护两个版本**：当前版本 + 上一个版本，更老的强制废弃
- **API 契约文档化**：用 OpenAPI/Swagger 描述接口，CI 中做兼容性检查

---

## 十、总结

微服务的本质不是"拆服务"，而是**用组织网络的复杂度换取技术系统的可维护性**。拆之前问自己三个问题：

1. **团队规模够大吗？** 小于 20 人，单体可能是最优解。
2. **业务边界清晰吗？** 模型还在迭代期，拆了就要不断地调整边界。
3. **运维能力跟得上吗？** 容器化、监控、链路追踪、CI/CD pipeline 是否就绪？

如果三条都满足，再看本章的组件选型——用 Nacos 做注册+配置，用 Gateway 做入口，用 Sentinel 做流量治理，用 Spring Cloud 做粘合剂。至于 RPC 和 MQ，那是服务通信层的选择（详见 [RPC 与异步编程](rpc-async.md) 和 [消息队列](mq.md)），而分布式理论如 CAP、BASE、分布式事务等在 [分布式系统理论基础](distributed-theory.md) 中有详细推导。

微服务不是银弹，它是分布式系统复杂度的某种转移。把问题从"单体内部的耦合"转移到了"服务之间的协调"——你要确保自己真正需要承接后者的复杂度，而不是为了技术潮流做迁移。
