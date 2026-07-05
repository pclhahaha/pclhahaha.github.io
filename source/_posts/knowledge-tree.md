---
title: 后端工程师知识体系
date: 2026-07-05 00:00:00
updated: 2026-07-05 00:00:00
tags:
  - 知识体系
  - 后端
  - 学习路线
categories:
  - 索引
---

# 后端工程师知识体系

> 29 篇文章 · 5 大分类 · ~20,000 行 · 55 张配图 · 95%+ 覆盖后端核心知识

---

## 📁 目录结构

```
posts/
├── cs-basics/        # 计算机基础 (5篇)
├── java/             # Java 全家桶 (9篇)
├── storage/          # 数据存储 (3篇)
├── distributed/      # 分布式系统 (6篇)
├── infra/            # 基础设施与工程实践 (6篇)
└── knowledge-tree.md # 本文件
```

---

## 一、计算机基础 `cs-basics/`

| # | 文章 | 行数 | 深度 |
|:--|------|:---:|:---:|
| 1 | [计算机网络](cs-basics/network.md) | 635 | TCP/UDP/HTTP/HTTPS/DNS/WebSocket/排查 |
| 2 | [操作系统](cs-basics/os.md) | 411 | 进程线程/内存管理/文件系统/IO模型/调度 |
| 3 | [数据结构与算法](cs-basics/algorithms.md) | 892 | 后端视角：B+Tree/LSM/跳表/哈希→中间件底层 |
| 4 | [设计模式](cs-basics/design-patterns.md) | 870 | SOLID/单例/DCL/代理AOP/责任链→JDK+Spring |
| 5 | [布隆过滤器](cs-basics/bloomfilter.md) | 491 | 数学推导/Guava/Redis/Cuckoo/缓存穿透 |

---

## 二、Java 全家桶 `java/`

### 语言与虚拟机
| # | 文章 | 行数 | 深度 |
|:--|------|:---:|:---:|
| 6 | [Java 语言基础](java/core.md) | 489 | 数据类型/内部类/泛型/接口/函数式编程/Stream |
| 7 | [Java 集合框架](java/collections.md) | 665 | HashMap/ConcurrentHashMap 源码分析 |
| 8 | [Java 并发编程](java/concurrency.md) | 498 | JMM/AQS/锁升级/线程池/Fork-Join |
| 9 | [JVM 基础](java/jvm.md) | 661 | 内存/类加载/GC(G1/ZGC)/对象布局/finalize |

### 框架与通信
| # | 文章 | 行数 | 深度 |
|:--|------|:---:|:---:|
| 10 | [Spring 源码解析](java/spring.md) | 570 | IOC/AOP/Bean生命周期/Boot启动/自动装配 |
| 11 | [MyBatis 源码解析](java/mybatis.md) | 633 | 初始化/执行流程/插件/缓存/动态SQL/Mapper代理 |
| 12 | [Java NIO 与 Netty](java/netty.md) | 301 | IO模型/epoll/NIO/Netty线程模型/ChannelPipeline |
| 13 | [RPC 与异步编程](java/rpc-async.md) | 480 | Protobuf/gRPC/CompletableFuture/RxJava/Reactor |

### 面试
| # | 文章 | 行数 | 深度 |
|:--|------|:---:|:---:|
| 14 | [后端面试 FAQ](java/interview.md) | 301 | 8大主题 50+ 高频面试题速查 |

---

## 三、数据存储 `storage/`

| # | 文章 | 行数 | 深度 |
|:--|------|:---:|:---:|
| 15 | [MySQL 深度解析](storage/mysql.md) | 1301 | InnoDB/B+Tree/MVCC/锁/日志/优化/主从/分库分表 |
| 16 | [Redis 深度解析](storage/redis.md) | 758 | SDS/skiplist/持久化/哨兵/集群/缓存策略/分布式锁 |
| 17 | [Elasticsearch 深度解析](storage/elasticsearch.md) | 1380 | 倒排索引/FST/集群/写入/查询/聚合/ILM |

---

## 四、分布式系统 `distributed/`

| # | 文章 | 行数 | 深度 |
|:--|------|:---:|:---:|
| 18 | [分布式系统理论](distributed/distributed-theory.md) | 957 | CAP/BASE/一致性/分布式事务(2PC→Saga→Seata)/锁/ID/限流/GFS分析 |
| 19 | [消息队列深度解析](distributed/mq.md) | 1135 | Kafka/RocketMQ/RabbitMQ 架构/存储/事务/可靠性 |
| 20 | [微服务架构](distributed/microservice.md) | 713 | 注册发现/配置中心/网关/负载均衡/流量治理 |
| 21 | [领域驱动设计](distributed/ddd.md) | 622 | 战略设计/战术设计/六边形架构/CQRS/微服务拆分 |
| 22 | [系统设计案例](distributed/system-design.md) | 831 | 短链/秒杀/Feed流/排行榜 + 方法论 + 组件选型 |
| 23 | [后端安全](distributed/security.md) | 477 | JWT/OAuth2/SSO/SQL注入/XSS/CSRF/SSRF/API安全 |

---

## 五、基础设施与工程实践 `infra/`

| # | 文章 | 行数 | 深度 |
|:--|------|:---:|:---:|
| 24 | [虚拟化与容器化](infra/virtualization.md) | 859 | Hypervisor/Docker/K8s/ServiceMesh/Serverless |
| 25 | [Nginx 深度解析](infra/nginx.md) | 1433 | 反向代理/负载均衡/缓存/HTTPS/性能优化/OpenResty |
| 26 | [Linux Cgroups](infra/cgroups.md) | 301 | cgroups架构/控制器/规则系统/OOM |
| 27 | [性能优化方法论](infra/performance.md) | 1049 | USE/火焰图/Arthas/JVM/数据库/架构优化/压测 |
| 28 | [可观测性](infra/observability.md) | 594 | Prometheus+Grafana/ELK/Jaeger+SkyWalking/排查 |
| 29 | [开发实践](infra/dev-practices.md) | 497 | 代码规范/Git工作流/测试/CI-CD/工程素养 |

---

## 覆盖度

```
模块                 文章数    覆盖率    评价
────────────────────────────────────────────
计算机基础              5      95%     基础扎实，数据结构+算法视角独特
Java 语言与框架         9      95%     源码级分析贯穿，JDK→框架→中间件全链路
数据存储                3      90%     三大存储全覆盖，缺 MongoDB/TiDB
分布式系统              6      95%     理论+实践+案例，覆盖P6-P8面试
基础设施                6      95%     从K8s到Nginx到CI/CD，运维视角补全
────────────────────────────────────────────
总计                  29       ~95%
```

### 画像
- **目标**：3-5年 Java 后端 → 架构师
- **风格**：源码级分析 + 生产实践 + 面试导向
- **规模**：~20,000 行 Markdown，55 张配图
- **语言**：中文
