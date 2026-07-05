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
      <a class="kg-link" href="/skip-list/">跳表 (SkipList)</a>
      <a class="kg-link" href="/b-plus-tree/">B+Tree</a>
      <a class="kg-link" href="/lsm-tree/">LSM-Tree</a>
      <a class="kg-link" href="/red-black-tree/">红黑树</a>
      <a class="kg-link" href="/trie-ac-automaton/">Trie 与 AC 自动机</a>
      <a class="kg-link" href="/fst-finite-state-transducer/">FST 有限状态转换器</a>
      <a class="kg-link" href="/heap-and-priority-queue/">堆与 TopK</a>
      <a class="kg-link" href="/consistent-hashing/">一致性哈希</a>
      <a class="kg-link" href="/incremental-rehash/">渐进式 Rehash</a>
      <a class="kg-link" href="/timing-wheel/">时间轮</a>
      <a class="kg-link" href="/bloomfilter/">布隆过滤器</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">算法</div>
    <div class="kg-links">
      <a class="kg-link" href="/dijkstra-shortest-path/">Dijkstra 最短路径</a>
      <a class="kg-link" href="/topological-sort/">拓扑排序</a>
      <a class="kg-link" href="/sliding-window/">滑动窗口</a>
      <a class="kg-link" href="/sorting-algorithms/">JDK 排序工程智慧</a>
      <a class="kg-link" href="/external-merge-sort/">分治与外部归并排序</a>
      <a class="kg-link" href="/algorithms/">📖 数据结构与算法（全文）</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">计算机网络</div>
    <div class="kg-links">
      <a class="kg-link" href="/network/">计算机网络（全文）</a>
      <a class="kg-link" href="/HTTP/">HTTPS 与 TLS/SSL</a>
      <a class="kg-link" href="/tcp-handshake/">TCP 三次握手与四次挥手</a>
      <a class="kg-link" href="/dns-system/">DNS 域名系统</a>
      <a class="kg-link" href="/io-multiplexing/">IO 多路复用</a>
      <a class="kg-link" href="/java-nio-channel-buffer/">Java NIO</a>
      <a class="kg-link" href="/netty-thread-model/">Netty 线程模型</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">操作系统</div>
    <div class="kg-links">
      <a class="kg-link" href="/os/">操作系统（全文）</a>
      <a class="kg-link" href="/cgroups/">Linux Cgroups</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">设计模式</div>
    <div class="kg-links">
      <a class="kg-link" href="/singleton-pattern/">单例模式</a>
      <a class="kg-link" href="/proxy-pattern/">代理模式</a>
      <a class="kg-link" href="/observer-pattern/">观察者模式</a>
      <a class="kg-link" href="/strategy-pattern/">策略模式</a>
      <a class="kg-link" href="/template-method-pattern/">模板方法</a>
      <a class="kg-link" href="/chain-of-responsibility/">责任链</a>
      <a class="kg-link" href="/state-pattern/">状态模式</a>
      <a class="kg-link" href="/design-patterns/">📖 设计模式（全文）</a>
    </div>
  </div>
</div>


<div class="kg-section">
  <div class="kg-section-title"><span class="emoji">☕</span>Java 生态</div>

  <div class="kg-category">
    <div class="kg-cat-title">Java 核心</div>
    <div class="kg-links">
      <a class="kg-link" href="/java-core/">Java 语言基础</a>
      <a class="kg-link" href="/java-generics/">泛型与类型擦除</a>
      <a class="kg-link" href="/java-stream-api/">Stream API</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">集合框架</div>
    <div class="kg-links">
      <a class="kg-link" href="/java-hashmap/">HashMap (JDK8)</a>
      <a class="kg-link" href="/java-concurrenthashmap/">ConcurrentHashMap</a>
      <a class="kg-link" href="/java-linkedhashmap-lru/">LinkedHashMap 与 LRU</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">并发编程</div>
    <div class="kg-links">
      <a class="kg-link" href="/java-memory-model/">JMM</a>
      <a class="kg-link" href="/java-aqs/">AQS</a>
      <a class="kg-link" href="/java-fork-join/">Fork/Join</a>
      <a class="kg-link" href="/java-concurrency/">📖 Java 并发（全文）</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">JVM</div>
    <div class="kg-links">
      <a class="kg-link" href="/jvm-classloader/">类加载器</a>
      <a class="kg-link" href="/java-object-header/">对象头与锁升级</a>
      <a class="kg-link" href="/jvm-gc-algorithms/">GC 算法与收集器</a>
      <a class="kg-link" href="/jvm-compressed-oops/">压缩指针</a>
      <a class="kg-link" href="/jvm-escape-analysis/">逃逸分析</a>
      <a class="kg-link" href="/java-string-pool/">字符串常量池</a>
      <a class="kg-link" href="/jvm-tuning/">JVM 调优实战</a>
      <a class="kg-link" href="/java-jvm/">📖 JVM 基础（全文）</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">Spring 全家桶</div>
    <div class="kg-links">
      <a class="kg-link" href="/spring-bean-lifecycle/">Bean 生命周期</a>
      <a class="kg-link" href="/aop-jdk-vs-cglib/">AOP — JDK vs CGLIB</a>
      <a class="kg-link" href="/spring-boot-autoconfig/">自动配置原理</a>
      <a class="kg-link" href="/java-spring/">📖 Spring 源码解析（全文）</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">MyBatis</div>
    <div class="kg-links">
      <a class="kg-link" href="/mybatis-plugin/">插件机制</a>
      <a class="kg-link" href="/mybatis-mapper-proxy/">Mapper 动态代理</a>
      <a class="kg-link" href="/mybatis-cache/">一/二级缓存</a>
      <a class="kg-link" href="/java-mybatis/">📖 MyBatis 源码解析（全文）</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">RPC / 异步编程</div>
    <div class="kg-links">
      <a class="kg-link" href="/grpc-protobuf/">gRPC 与 Protobuf</a>
      <a class="kg-link" href="/java-completablefuture/">CompletableFuture</a>
      <a class="kg-link" href="/reactive-programming/">响应式编程</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">面试</div>
    <div class="kg-links">
      <a class="kg-link" href="/java-interview/">后端面试 FAQ</a>
    </div>
  </div>
</div>


<div class="kg-section">
  <div class="kg-section-title"><span class="emoji">🗄️</span>数据存储</div>

  <div class="kg-links">
    <a class="kg-link" href="/mysql/">📖 MySQL 深度解析</a>
    <a class="kg-link" href="/mysql-tuning/">MySQL 调优</a>
    <a class="kg-link" href="/redis/">📖 Redis 深度解析</a>
    <a class="kg-link" href="/elasticsearch/">📖 Elasticsearch 深度解析</a>
  </div>
</div>


<div class="kg-section">
  <div class="kg-section-title"><span class="emoji">🌐</span>分布式系统</div>

  <div class="kg-category">
    <div class="kg-cat-title">理论</div>
    <div class="kg-links">
      <a class="kg-link" href="/cap-theorem/">CAP 定理</a>
      <a class="kg-link" href="/base-theory/">BASE 理论</a>
      <a class="kg-link" href="/consistency-models/">一致性模型</a>
      <a class="kg-link" href="/distributed-transactions/">分布式事务</a>
      <a class="kg-link" href="/logical-clocks/">逻辑时钟</a>
      <a class="kg-link" href="/lease-mechanism/">Lease 租约</a>
      <a class="kg-link" href="/rate-limiting-algorithms/">限流算法</a>
      <a class="kg-link" href="/gossip-protocol/">Gossip 协议</a>
      <a class="kg-link" href="/distributed-theory/">📖 分布式理论（全文）</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">共识算法</div>
    <div class="kg-links">
      <a class="kg-link" href="/raft-algorithm/">Raft</a>
      <a class="kg-link" href="/paxos-algorithm/">Paxos</a>
      <a class="kg-link" href="/zab-protocol/">ZAB (ZooKeeper)</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">实战组件</div>
    <div class="kg-links">
      <a class="kg-link" href="/distributed-locks/">分布式锁</a>
      <a class="kg-link" href="/distributed-id/">分布式 ID</a>
      <a class="kg-link" href="/mq/">📖 消息队列深度解析</a>
      <a class="kg-link" href="/gfs-paper/">GFS 论文解读</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">微服务</div>
    <div class="kg-links">
      <a class="kg-link" href="/service-discovery/">服务注册与发现</a>
      <a class="kg-link" href="/api-gateway/">API 网关</a>
      <a class="kg-link" href="/traffic-governance/">流量治理</a>
      <a class="kg-link" href="/ddd/">📖 领域驱动设计</a>
      <a class="kg-link" href="/cqrs-event-sourcing/">CQRS 与事件溯源</a>
      <a class="kg-link" href="/microservice/">📖 微服务架构（全文）</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">安全</div>
    <div class="kg-links">
      <a class="kg-link" href="/jwt-authentication/">JWT 认证</a>
      <a class="kg-link" href="/oauth2-sso/">OAuth2 与 SSO</a>
      <a class="kg-link" href="/web-vulnerabilities/">Web 漏洞攻防</a>
    </div>
  </div>

  <div class="kg-category">
    <div class="kg-cat-title">系统设计</div>
    <div class="kg-links">
      <a class="kg-link" href="/url-shortener-design/">短链系统</a>
      <a class="kg-link" href="/flash-sale-design/">秒杀系统</a>
      <a class="kg-link" href="/feed-stream-design/">Feed 流</a>
      <a class="kg-link" href="/leaderboard-design/">排行榜</a>
      <a class="kg-link" href="/system-design/">📖 系统设计方法论（全文）</a>
    </div>
  </div>
</div>


<div class="kg-section">
  <div class="kg-section-title"><span class="emoji">⚙️</span>基础设施</div>

  <div class="kg-links">
    <a class="kg-link" href="/nginx/">📖 Nginx 深度解析</a>
    <a class="kg-link" href="/docker-deep-dive/">Docker</a>
    <a class="kg-link" href="/kubernetes-core/">Kubernetes</a>
    <a class="kg-link" href="/virtualization/">📖 虚拟化与容器化（全文）</a>
    <a class="kg-link" href="/prometheus-grafana/">Prometheus + Grafana</a>
    <a class="kg-link" href="/distributed-tracing/">分布式链路追踪</a>
    <a class="kg-link" href="/observability/">📖 可观测性（全文）</a>
    <a class="kg-link" href="/performance/">📖 性能优化方法论（全文）</a>
    <a class="kg-link" href="/cicd-practices/">CI/CD 实践</a>
    <a class="kg-link" href="/unit-testing/">单元测试金字塔</a>
    <a class="kg-link" href="/dev-practices/">📖 后端开发实践（全文）</a>
  </div>
</div>

</div>
