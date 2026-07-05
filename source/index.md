---
title: 后端知识图谱
date: 2026-07-05 00:00:00
type: index
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
.kg-section-title .emoji { margin-right: 6px; }

.kg-category { margin-bottom: 20px; }
.kg-cat-title { font-size: 15px; font-weight: 600; color: #555; margin-bottom: 8px; }

.kg-links { display: flex; flex-wrap: wrap; gap: 6px 12px; }
.kg-link { font-size: 13px; color: #333; text-decoration: none; padding: 3px 10px; background: #f7f7f7; border-radius: 4px; transition: all 0.15s; white-space: nowrap; }
.kg-link:hover { background: #e3e3e3; color: #000; }
.kg-link.tag { background: #eef5ff; color: #3b82f6; font-size: 12px; padding: 2px 8px; }
.kg-link.tag:hover { background: #dbeafe; }

@media (max-width: 600px) {
  .kg-stats { gap: 16px; }
  .kg-stat-num { font-size: 24px; }
}
</style>

<div class="knowledge-graph">

<div class="kg-header">
  <h1>后端知识图谱</h1>
  <p>全栈后端工程师知识体系 · 持续更新</p>
  <div class="kg-stats">
    <div class="kg-stat"><div class="kg-stat-num">113</div><div class="kg-stat-label">篇文章</div></div>
    <div class="kg-stat"><div class="kg-stat-num">13</div><div class="kg-stat-label">分类</div></div>
    <div class="kg-stat"><div class="kg-stat-num">100+</div><div class="kg-stat-label">标签</div></div>
  </div>
</div>


<div class="kg-section">
  <div class="kg-section-title"><span class="emoji">🔬</span>CS 基础</div>

  <div class="kg-category">
    <div class="kg-cat-title">数据结构</div>
    <div class="kg-links">
      <a class="kg-link" href="/2026/07/05/skip-list/">跳表 (SkipList)</a>
      <a class="kg-link" href="/2026/07/05/b-plus-tree/">B+Tree</a>
      <a class="kg-link" href="/2026/07/05/lsm-tree/">LSM-Tree</a>
      <a class="kg-link" href="/2026/07/05/red-black-tree/">红黑树</a>
      <a class="kg-link" href="/2026/07/05/trie-ac-automaton/">Trie 与 AC 自动机</a>
      <a class="kg-link" href="/2026/07/05/fst-finite-state-transducer/">FST 有限状态转换器</a>
      <a class="kg-link" href="/2026/07/05/heap-and-priority-queue/">堆与 TopK</a>
      <a class="kg-link" href="/2026/07/05/consistent-hashing/">一致性哈希</a>
      <a class="kg-link" href="/2026/07/05/incremental-rehash/">渐进式 Rehash</a>
      <a class="kg-link" href="/2026/07/05/timing-wheel/">时间轮</a>
      <a class="kg-link" href="/2026/07/05/bloomfilter/">布隆过滤器</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">算法</div>
    <div class="kg-links">
      <a class="kg-link" href="/2026/07/05/dijkstra-shortest-path/">Dijkstra 最短路径</a>
      <a class="kg-link" href="/2026/07/05/topological-sort/">拓扑排序</a>
      <a class="kg-link" href="/2026/07/05/sliding-window/">滑动窗口</a>
      <a class="kg-link" href="/2026/07/05/sorting-algorithms/">JDK 排序工程智慧</a>
      <a class="kg-link" href="/2026/07/05/external-merge-sort/">分治与外部归并排序</a>
      <a class="kg-link" href="/2026/07/05/algorithms/">📖 数据结构与算法（全文）</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">计算机网络</div>
    <div class="kg-links">
      <a class="kg-link" href="/2020/01/01/network/">计算机网络（全文）</a>
      <a class="kg-link" href="/2019/12/01/HTTP/">HTTPS 与 TLS/SSL</a>
      <a class="kg-link" href="/2026/07/05/tcp-handshake/">TCP 三次握手与四次挥手</a>
      <a class="kg-link" href="/2026/07/05/dns-system/">DNS 域名系统</a>
      <a class="kg-link" href="/2026/07/05/io-multiplexing/">IO 多路复用</a>
      <a class="kg-link" href="/2026/07/05/java-nio-channel-buffer/">Java NIO</a>
      <a class="kg-link" href="/2026/07/05/netty-thread-model/">Netty 线程模型</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">操作系统</div>
    <div class="kg-links">
      <a class="kg-link" href="/2020/01/01/os/">操作系统（全文）</a>
      <a class="kg-link" href="/2021/02/21/cgroups/">Linux Cgroups</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">设计模式</div>
    <div class="kg-links">
      <a class="kg-link" href="/2026/07/05/singleton-pattern/">单例模式</a>
      <a class="kg-link" href="/2026/07/05/proxy-pattern/">代理模式</a>
      <a class="kg-link" href="/2026/07/05/observer-pattern/">观察者模式</a>
      <a class="kg-link" href="/2026/07/05/strategy-pattern/">策略模式</a>
      <a class="kg-link" href="/2026/07/05/template-method-pattern/">模板方法</a>
      <a class="kg-link" href="/2026/07/05/chain-of-responsibility/">责任链</a>
      <a class="kg-link" href="/2026/07/05/state-pattern/">状态模式</a>
      <a class="kg-link" href="/2026/07/05/design-patterns/">📖 设计模式（全文）</a>
    </div>
  </div>
</div>


<div class="kg-section">
  <div class="kg-section-title"><span class="emoji">☕</span>Java 生态</div>

  <div class="kg-category">
    <div class="kg-cat-title">Java 核心</div>
    <div class="kg-links">
      <a class="kg-link" href="/2020/01/01/java-core/">Java 语言基础</a>
      <a class="kg-link" href="/2026/07/05/java-generics/">泛型与类型擦除</a>
      <a class="kg-link" href="/2026/07/05/java-stream-api/">Stream API</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">集合框架</div>
    <div class="kg-links">
      <a class="kg-link" href="/2026/07/05/java-hashmap/">HashMap (JDK8)</a>
      <a class="kg-link" href="/2026/07/05/java-concurrenthashmap/">ConcurrentHashMap</a>
      <a class="kg-link" href="/2026/07/05/java-linkedhashmap-lru/">LinkedHashMap 与 LRU</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">并发编程</div>
    <div class="kg-links">
      <a class="kg-link" href="/2026/07/05/java-memory-model/">JMM</a>
      <a class="kg-link" href="/2026/07/05/java-aqs/">AQS</a>
      <a class="kg-link" href="/2026/07/05/java-fork-join/">Fork/Join</a>
      <a class="kg-link" href="/2020/02/09/java-concurrency/">📖 Java 并发（全文）</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">JVM</div>
    <div class="kg-links">
      <a class="kg-link" href="/2026/07/05/jvm-classloader/">类加载器</a>
      <a class="kg-link" href="/2026/07/05/java-object-header/">对象头与锁升级</a>
      <a class="kg-link" href="/2026/07/05/jvm-gc-algorithms/">GC 算法与收集器</a>
      <a class="kg-link" href="/2026/07/05/jvm-compressed-oops/">压缩指针</a>
      <a class="kg-link" href="/2026/07/05/jvm-escape-analysis/">逃逸分析</a>
      <a class="kg-link" href="/2026/07/05/java-string-pool/">字符串常量池</a>
      <a class="kg-link" href="/2026/07/05/jvm-tuning/">JVM 调优实战</a>
      <a class="kg-link" href="/2020/01/05/java-jvm/">📖 JVM 基础（全文）</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">Spring 全家桶</div>
    <div class="kg-links">
      <a class="kg-link" href="/2026/07/05/spring-bean-lifecycle/">Bean 生命周期</a>
      <a class="kg-link" href="/2026/07/05/aop-jdk-vs-cglib/">AOP — JDK vs CGLIB</a>
      <a class="kg-link" href="/2026/07/05/spring-boot-autoconfig/">自动配置原理</a>
      <a class="kg-link" href="/2020/01/04/java-spring/">📖 Spring 源码解析（全文）</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">MyBatis</div>
    <div class="kg-links">
      <a class="kg-link" href="/2026/07/05/mybatis-plugin/">插件机制</a>
      <a class="kg-link" href="/2026/07/05/mybatis-mapper-proxy/">Mapper 动态代理</a>
      <a class="kg-link" href="/2026/07/05/mybatis-cache/">一/二级缓存</a>
      <a class="kg-link" href="/2025/01/01/java-mybatis/">📖 MyBatis 源码解析（全文）</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">RPC / 异步编程</div>
    <div class="kg-links">
      <a class="kg-link" href="/2026/07/05/grpc-protobuf/">gRPC 与 Protobuf</a>
      <a class="kg-link" href="/2026/07/05/java-completablefuture/">CompletableFuture</a>
      <a class="kg-link" href="/2026/07/05/reactive-programming/">响应式编程</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">面试</div>
    <div class="kg-links">
      <a class="kg-link" href="/2026/07/05/java-interview/">后端面试 FAQ</a>
    </div>
  </div>
</div>


<div class="kg-section">
  <div class="kg-section-title"><span class="emoji">🗄️</span>数据存储</div>

  <div class="kg-links">
    <a class="kg-link" href="/2026/07/05/mysql/">📖 MySQL 深度解析</a>
    <a class="kg-link" href="/2026/07/05/mysql/">MySQL 调优</a>
    <a class="kg-link" href="/2026/07/05/redis/">📖 Redis 深度解析</a>
    <a class="kg-link" href="/2026/07/05/elasticsearch/">📖 Elasticsearch 深度解析</a>
  </div>
</div>


<div class="kg-section">
  <div class="kg-section-title"><span class="emoji">🌐</span>分布式系统</div>

  <div class="kg-category">
    <div class="kg-cat-title">理论</div>
    <div class="kg-links">
      <a class="kg-link" href="/2026/07/05/cap-theorem/">CAP 定理</a>
      <a class="kg-link" href="/2026/07/05/base-theory/">BASE 理论</a>
      <a class="kg-link" href="/2026/07/05/consistency-models/">一致性模型</a>
      <a class="kg-link" href="/2026/07/05/distributed-transactions/">分布式事务</a>
      <a class="kg-link" href="/2026/07/05/logical-clocks/">逻辑时钟</a>
      <a class="kg-link" href="/2026/07/05/lease-mechanism/">Lease 租约</a>
      <a class="kg-link" href="/2026/07/05/rate-limiting-algorithms/">限流算法</a>
      <a class="kg-link" href="/2026/07/05/gossip-protocol/">Gossip 协议</a>
      <a class="kg-link" href="/2026/07/05/distributed-theory/">📖 分布式理论（全文）</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">共识算法</div>
    <div class="kg-links">
      <a class="kg-link" href="/2026/07/05/raft-algorithm/">Raft</a>
      <a class="kg-link" href="/2026/07/05/paxos-algorithm/">Paxos</a>
      <a class="kg-link" href="/2026/07/05/zab-protocol/">ZAB (ZooKeeper)</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">实战组件</div>
    <div class="kg-links">
      <a class="kg-link" href="/2026/07/05/distributed-locks/">分布式锁</a>
      <a class="kg-link" href="/2026/07/05/distributed-id/">分布式 ID</a>
      <a class="kg-link" href="/2025/01/01/mq/">📖 消息队列深度解析</a>
      <a class="kg-link" href="/2026/07/05/gfs-paper/">GFS 论文解读</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">微服务</div>
    <div class="kg-links">
      <a class="kg-link" href="/2026/07/05/service-discovery/">服务注册与发现</a>
      <a class="kg-link" href="/2026/07/05/api-gateway/">API 网关</a>
      <a class="kg-link" href="/2026/07/05/traffic-governance/">流量治理</a>
      <a class="kg-link" href="/2026/07/05/ddd/">📖 领域驱动设计</a>
      <a class="kg-link" href="/2026/07/05/cqrs-event-sourcing/">CQRS 与事件溯源</a>
      <a class="kg-link" href="/2026/07/05/microservice/">📖 微服务架构（全文）</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">安全</div>
    <div class="kg-links">
      <a class="kg-link" href="/2026/07/05/jwt-authentication/">JWT 认证</a>
      <a class="kg-link" href="/2026/07/05/oauth2-sso/">OAuth2 与 SSO</a>
      <a class="kg-link" href="/2026/07/05/web-vulnerabilities/">Web 漏洞攻防</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">系统设计</div>
    <div class="kg-links">
      <a class="kg-link" href="/2026/07/05/url-shortener-design/">短链系统</a>
      <a class="kg-link" href="/2026/07/05/flash-sale-design/">秒杀系统</a>
      <a class="kg-link" href="/2026/07/05/feed-stream-design/">Feed 流</a>
      <a class="kg-link" href="/2026/07/05/leaderboard-design/">排行榜</a>
      <a class="kg-link" href="/2026/07/05/system-design/">📖 系统设计方法论（全文）</a>
    </div>
  </div>
</div>


<div class="kg-section">
  <div class="kg-section-title"><span class="emoji">⚙️</span>基础设施</div>

  <div class="kg-links">
    <a class="kg-link" href="/2026/07/05/nginx/">📖 Nginx 深度解析</a>
    <a class="kg-link" href="/2026/07/05/docker-deep-dive/">Docker</a>
    <a class="kg-link" href="/2026/07/05/kubernetes-core/">Kubernetes</a>
    <a class="kg-link" href="/2026/07/05/virtualization/">📖 虚拟化与容器化（全文）</a>
    <a class="kg-link" href="/2026/07/05/prometheus-grafana/">Prometheus + Grafana</a>
    <a class="kg-link" href="/2026/07/05/distributed-tracing/">分布式链路追踪</a>
    <a class="kg-link" href="/2026/07/05/observability/">📖 可观测性（全文）</a>
    <a class="kg-link" href="/2026/07/05/performance/">📖 性能优化方法论（全文）</a>
    <a class="kg-link" href="/2026/07/05/cicd-practices/">CI/CD 实践</a>
    <a class="kg-link" href="/2026/07/05/unit-testing/">单元测试金字塔</a>
    <a class="kg-link" href="/2026/07/05/dev-practices/">📖 后端开发实践（全文）</a>
  </div>
</div>

</div>
