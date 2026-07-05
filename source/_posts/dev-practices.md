---
title: "后端开发实践: 从代码规范到 CI/CD"
date: 2026-07-05 00:00:00
updated: 2026-07-05 00:00:00
tags:
  - 开发规范
  - CI/CD
  - 单元测试
  - 工程素养
categories:
  - 基础设施
---
## 一、代码规范

代码规范是让团队用相同的方式思考和表达，降低认知成本。

### 1.1 命名规范

核心原则：**名副其实**（`elapsedTimeMs` 优于 `time`）；**避免误导**（`accountList` 若是 Set 用 `accounts`）；**可读性优先**（`Customer` 非 `Cust`）；**可搜索**（`MAX_RETRY_COUNT=3` 优于到处 `3`）；**布尔变量**以 `is`/`has` 开头。

约定：包名全小写（`com.company.order.service`）；类名大驼峰名词（`OrderService`）；接口不加 `I` 前缀（`UserRepository`）；方法小驼峰动宾（`findById()`）；常量全大写+下划线（`MAX_RETRY_COUNT`）；变量见名知意（`unpaidOrderCount`）。

```java
// ❌ public void Proc(List<Order> l) { for (Order o:l) if (o.status==1) o.status=2; }
// ✅ 见名知意、方法短小、布尔方法自解释
public void markOrdersAsShipped(List<Order> unpaidOrders) {
    for (Order order : unpaidOrders) {
        if (order.isUnpaid()) order.markAsShipped();
    }
}
```

### 1.2 注释规范

**原则：注释解释"为什么"而非"做什么"。** 公开 API 写 Javadoc，业务规则写注释，违背直觉的决策备注原因。错误注释比没有注释更危险。

```java
/** <b>注意：此方法走从库，存在主从延迟。</b> */
public List<Order> findOrdersByUser(Long userId, OrderStatus status, Pageable page) { ... }
```

### 1.3 异常处理规范

```java
// ① 不要吞异常
try { doSomethingRisky(); } catch (Exception e) { /* 禁止空 catch */ }
// 至少：log.error("...", e); throw new ServiceException("...", e);

// ② 不要用异常控制流程——堆栈收集有性能开销
// 错误：try { return Long.parseLong(input); } catch (...) { return 0L; }
// 正确：if (input.matches("\\d+")) return Long.parseLong(input); else return 0L;

// ③ 保持异常链——不要丢失根因
catch (DuplicateKeyException e) {
    throw new ServiceException("创建用户失败, username=" + username, e); // 必须传入 e
}

// ④ 区分业务异常 vs 技术异常
throw new BusinessException(ErrorCode.INSUFFICIENT_BALANCE, "余额不足");    // 用户可理解
throw new ServiceException("调用支付网关超时, orderId=" + orderId, e);      // 需开发排查
```

用 `@RestControllerAdvice` 统一兜底，避免 Controller 里到处 try-catch：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(BusinessException.class)
    public ApiResult<Void> handleBusiness(BusinessException e) {
        log.warn("业务异常: code={}, msg={}", e.getCode(), e.getMessage());
        return ApiResult.fail(e.getCode(), e.getMessage());
    }
    @ExceptionHandler(Exception.class)
    public ApiResult<Void> handleUnknown(Exception e) {
        log.error("未知异常", e);
        return ApiResult.fail(500, "系统繁忙");
    }
}
```

### 1.4 日志规范

日志是生产环境的眼睛，线上问题时日志质量直接决定恢复时间。

| 级别 | 使用场景 | 生产 |
|------|----------|:----:|
| **ERROR** | 需人工介入：支付失败、DB 断开、重试耗尽 | 开 |
| **WARN** | 潜在问题未影响主流程：重试成功、降级 | 开 |
| **INFO** | 关键节点：入参出参、状态流转、调用耗时 | 开 |
| **DEBUG** | 开发调试：参数值、中间结果、SQL 绑定 | 关 |
| **TRACE** | 逐行追踪 | 关 |

```java
// ① 参数占位符——不要字符串拼接
log.info("创建订单, userId={}, skuId={}, amount={}", userId, skuId, amount);
// ② 异常传入 Throwable，否则堆栈不输出
log.error("支付失败, orderId={}", orderId, e);
// ③ 不打印敏感信息
log.info("用户登录: mobile={}", maskMobile(mobile)); // 138****5678
// ④ traceId 从 MDC 注入
MDC.put("traceId", span.context().toTraceId());
// ⑤ 高频日志加条件判断
if (log.isDebugEnabled()) { log.debug("明细: {}", toJson(detail)); }
```

```xml
<pattern>[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%X{traceId}] [%thread] %-5level %logger{36} - %msg%n</pattern>
```

> 日志系统完整方案参考 [可观测性](observability.md)。

### 1.5 集合使用规范

```java
// ① 指定初始容量——避免 HashMap 频繁扩容
Map<String, Order> map = new HashMap<>(expectedSize / 0.75 + 1);

// ② Arrays.asList 返回固定大小 List（不可 add/remove）
List<String> mutable = new ArrayList<>(Arrays.asList("a", "b", "c"));

// ③ subList 是视图，修改相互影响——要独立副本
List<Integer> copy = new ArrayList<>(original.subList(1, 3));

// ④ ToArray 传零长度数组（优于预分配）
String[] array = list.toArray(new String[0]);
```

### 1.6 并发编程规范

```java
// ① SimpleDateFormat → DateTimeFormatter（线程安全）
private static final DateTimeFormatter DTF = DateTimeFormatter.ofPattern("yyyy-MM-dd");
// ② HashMap → ConcurrentHashMap
Map<String, Object> cache = new ConcurrentHashMap<>();
// ③ 线程池禁止 Executors——必须手动创建有界队列
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10, 20, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(1000),
    new ThreadFactoryBuilder().setNameFormat("order-pool-%d").build(),
    new ThreadPoolExecutor.CallerRunsPolicy()
);
// ④ volatile 只保证可见性——count++ 用 AtomicInteger
// ⑤ 锁必须在 finally 释放：lock.lock(); try {...} finally { lock.unlock(); }
```

---

## 二、Git 工作流

### 2.1 分支策略选型

| 策略 | 核心思想 | 适用团队 |
|------|----------|----------|
| **Git Flow** | `main`+`develop`+`feature/*`+`release/*`+`hotfix/*` | 固定发布周期的传统项目 |
| **Trunk-Based** | 所有人直接往 main 提交小变更 | 精英团队（需强测试+feature flag） |

**务实选择**：简化 Git Flow——`main`（生产）+ `feature/*`（开发），hotfix 从 main 拉。```

### 2.2 Commit Message 规范

采用 **Conventional Commits**：

```
<type>(<scope>): <subject>
```

| type | 用途 | 示例 |
|------|------|------|
| `feat` | 新功能 | `feat(order): 支持批量取消订单` |
| `fix` | 修 Bug | `fix(pay): 修复微信支付签名校验` |
| `refactor` | 重构 | `refactor(dao): 提取公共分页逻辑` |
| `docs` / `test` | 文档/测试 | `docs(api): 补充错误码说明` |
| `chore` / `perf` | 构建/性能 | `chore(deps): 升级 Spring Boot 3.2` |

Subject 用祈使句，不超过 50 字符；一个 commit 只做一件事。

```bash
git commit -m "feat(order): 新增订单超时自动取消

超30分钟未支付自动取消，释放库存。Redis ZSet 实现。

Closes #456"
# ❌: git commit -m "fix bug" / git commit -m "修改"
```

### 2.3 合并策略

| 策略 | 效果 | 适用 |
|------|------|------|
| **merge** | 保留分支拓扑，完整历史 | 公共分支合并 |
| **squash** | 所有 commit 压成 1 个 | feature→main |
| **rebase** | 重放提交到最新基线 | 个人分支追平 |

**推荐组合**：个人分支追平用 `rebase`，合入 main 用 `squash merge`，公共分支间用 `merge`。

### 2.4 Code Review 规范

Code Review 不是找茬，是知识传递和质量兜底。PR 至少一人 Approve 后方可合并。

**Review Checklist**：逻辑正确性（边界/空值/并发/幂等）；异常处理（有无吞异常）；性能（N+1/大事务/锁范围）；安全（注入/敏感数据）；测试覆盖；可维护性。一次 PR ≤400 行。提意见用疑问句，区分 `[blocker]`/`[nit]`。

### 2.5 常见应急操作

```bash
git revert <hash>              # 撤销提交（生成反向 commit）
git reset --soft HEAD~1        # 撤销 commit 留暂存区（--hard 慎用）
git reflog && git checkout <hash>  # 恢复"丢失"的提交
git cherry-pick <hash>         # 挑 commit 到当前分支
git stash save "WIP" && git stash pop  # 暂存/恢复
```

---

## 三、项目结构规范

### 3.1 标准目录结构

以 Maven 多模块为例：

```
ecommerce/
├── pom.xml
├── ecommerce-api/              # DTO / Feign / RPC 接口
├── ecommerce-service/          # Service / Manager / Converter
├── ecommerce-dao/              # Mapper / Repository / Entity
├── ecommerce-common/           # 常量 / 异常 / 工具类
├── ecommerce-web/              # Controller
└── ecommerce-sdk/              # 对外 SDK（独立）
```

**依赖规则**：`web → api → service → dao → common`。上层依赖下层，下层不感知上层。

### 3.2 配置文件管理

```
src/main/resources/
├── application.yml              # 公共配置
├── application-dev.yml          # 开发环境
├── application-test.yml         # 测试环境
├── application-staging.yml      # 预发环境
└── application-prod.yml         # 生产环境
```

```yaml
# application.yml — 激活 profile 从环境变量读取，本地默认 dev
spring:
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}

# application-prod.yml — 敏感信息走环境变量或配置中心
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

**配置中心 vs 本地**：小型项目本地文件即可。微服务用配置中心（Nacos/Apollo）实现动态刷新。推荐**本地默认值 + 配置中心覆盖**。

### 3.3 版本号规范

语义化版本 `MAJOR.MINOR.PATCH`：MAJOR=不兼容 API，MINOR=向后兼容新功能，PATCH=向后兼容 Bug 修复。`<revision>2.3.1</revision>`，预发布：`1.0.0-rc.2`。

---

## 四、单元测试

### 4.1 JUnit 5 + Mockito 基础

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock private OrderRepository orderRepository;
    @Mock private PaymentGateway paymentGateway;
    @InjectMocks private OrderService orderService;

    @Test
    @DisplayName("支付成功应更新订单状态")
    void shouldUpdateOrderStatusWhenPaymentSucceeds() {
        Order order = mock(Order.class);
        when(orderRepository.findById(1L)).thenReturn(Optional.of(order));
        when(paymentGateway.pay(1L)).thenReturn(PaymentResult.success());

        orderService.pay(1L);
        verify(order).markAsPaid();
    }

    @Test
    @DisplayName("订单不存在时应抛出异常")
    void shouldThrowWhenOrderNotFound() {
        when(orderRepository.findById(1L)).thenReturn(Optional.empty());
        assertThrows(OrderNotFoundException.class, () -> orderService.pay(1L));
    }
}
```

### 4.2 测试金字塔

```
         ┌──────┐
         │ E2E  │ ← 极少
         │ 集成  │ ← 较少
         │ 单元  │ ← 大量
         └──────┘
```

| 层级 | 速度 | 稳定性 | 占比 | 工具 |
|------|:---:|:---:|:---:|------|
| 单元测试 | 毫秒 | 极高 | 70%+ | JUnit 5 + Mockito |
| 集成测试 | 秒 | 中 | 20% | Spring Boot Test + Testcontainers |
| E2E 测试 | 分钟 | 低 | 10% | REST Assured / Selenium |

不要追求精确比例，**追求反馈速度**：单元测试 200ms 内完成，发现错误立即反馈。

### 4.3 什么值得测

```
不值得测 →                        ← 最值得测 →
简单 getter/setter   核心算法/金额计算   复杂状态机/边界条件/异常路径
```

核心规则：涉及钱必测；边界（null/空/零/负/超大值）必测；异常路径必测。不测 getter/setter、框架代码、一眼可确认的简单逻辑。

### 4.4 TDD 争议与实践

TDD 的理想流程：写失败的测试 → 写最小代码通过 → 重构。现实态度：**在复杂逻辑和核心算法上 TDD，在简单 CRUD 上后补测试。** TDD 真正的价值不在"先写测试"，而在于**驱动你思考接口设计**——你必须先回答"输入是什么？输出是什么？边界是什么？"——这本身就是一次 mini 设计评审。

### 4.5 测试覆盖率

```java
// 行覆盖 ≠ 分支覆盖
if (score >= 90) return "A";
if (score >= 80) return "B";   // 只测 95,75 → 行 100%，分支 67%
return "C";
```

核心业务 90%+/85%+（行/分支），公共库 90%+，CRUD 70%+，Controller 40%+。**覆盖率是手段不是目的**——用它发现未测试代码。CI 中设新代码阈值 ≥80%。

---

## 五、CI/CD

### 5.1 CI/CD 流水线概念

```
代码提交 → [ 构建 → 单测 → 扫描 → 集成测试 → 制品 ] → 部署
           └─────────── CI ──────────────┘         └── CD ──┘
```
CI：代码提交后自动构建/测试，10 分钟反馈；CD（交付）：CI 通过自动打包，手动部署；CD（部署）：自动部署到生产。

### 5.2 GitHub Actions 示例

```yaml
# .github/workflows/ci.yml
name: Java CI
on:
  push:
    branches: [ main, 'feature/*' ]
  pull_request:
    branches: [ main ]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin', cache: maven }
      - name: Build & Unit Test
        run: mvn clean verify -pl '!e2e-tests'
      - name: Static Analysis
        run: mvn pmd:check spotbugs:check
      - name: Test Report
        uses: dorny/test-reporter@v1
        if: always()
        with: { name: JUnit Tests, path: '**/target/surefire-reports/*.xml', reporter: java-junit }
  docker:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v5
        with: { push: true, tags: registry.example.com/ecommerce:${{ github.sha }} }
      - run: kubectl set image deployment/ecommerce ecommerce=registry.example.com/ecommerce:${{ github.sha }} -n staging
```

### 5.3 Jenkins Pipeline

```groovy
pipeline {
    agent any
    environment { DOCKER_REGISTRY = 'registry.example.com' }
    stages {
        stage('Build & Test') {
            steps { sh 'mvn clean verify -DskipITs' }
            post { always { junit '**/target/surefire-reports/*.xml' } }
        }
        stage('SonarQube') {
            steps {
                withSonarQubeEnv('SonarQube') { sh 'mvn sonar:sonar' }
                timeout(time: 5, unit: 'MINUTES') { waitForQualityGate abortPipeline: true }
            }
        }
        stage('Docker Build & Push') {
            when { branch 'main' }
            steps {
                script {
                    def tag = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
                    docker.build("${DOCKER_REGISTRY}/ecommerce:${tag}").push()
                }
            }
        }
        stage('Deploy') {
            when { branch 'main' }
            steps {
                input message: '确认部署？', ok: '确认'
                sh "helm upgrade ecommerce ./helm -n staging --set image.tag=${tag}"
            }
        }
    }
    post { failure { emailext(subject: "构建失败", to: "${env.CHANGE_AUTHOR_EMAIL}") } }
}
```

### 5.4 部署策略

| 策略 | 优点 | 缺点 |
|------|------|------|
| **滚动更新** | 零停机 | 回滚慢 |
| **蓝绿部署** | 秒级回滚 | 资源翻倍 |
| **金丝雀发布** | 影响可控 | 实现复杂 |

推荐：普遍用滚动（K8s 默认），核心用金丝雀（Istio），关键备蓝绿。金丝雀流程：部署→切 5%→观察→逐步 25%→50%→100%。

### 5.5 容器化 CI/CD

`Git Push → CI → Docker Build → Registry → CD → K8s`。多阶段构建：

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /build
COPY . .
RUN mvn clean package -DskipTests -pl ecommerce-web -am

FROM eclipse-temurin:21-jre-alpine
COPY --from=builder /build/ecommerce-web/target/*.jar /app/app.jar
ENTRYPOINT ["java", "-XX:+UseZGC", "-jar", "/app/app.jar"]
```

> 容器化深入内容参考 [虚拟化与容器化](virtualization.md)。

---

## 六、工程素养

工程素养是区分"能写代码"和"能写好代码"的分水岭。

### 6.1 防御性编程

**所有外部输入不可信，所有外部调用可能失败。**

```java
// ① 入口校验
public Order createOrder(Long userId, List<OrderItem> items, BigDecimal amount) {
    Objects.requireNonNull(userId, "userId");
    if (items == null || items.isEmpty()) throw new IllegalArgumentException("items");
    if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0) throw new IllegalArgumentException();
}

// ② @NonNull/@Nullable 明确契约
public interface UserService {
    @Nullable User findById(@NonNull Long id);
}

// ③ 永远不返回 null
public List<Order> getOrders(Long userId) {
    List<Order> orders = orderRepository.findByUserId(userId);
    return orders != null ? orders : Collections.emptyList();
}
```

### 6.2 Fail Fast 原则

**让错误尽早暴露，而非在远离源头的地方才崩溃。** 三个落地习惯：构造器中立即校验；方法入口 precondition check；switch 必须有 default。

```java
// ❌ 错误被延迟：user 可能是 null，100 行后才 NPE
sendEmail(order.getUser().getEmail(), "确认");

// ✅ 入口集中校验
Objects.requireNonNull(order.getUser(), "order.user");
// 后续代码放心使用
```

### 6.3 技术债务管理

技术债务像房贷——适当使用是杠杆，但必须持续偿还。每个 sprint 预留 15%-20% 时间清理债务，季度做技术债务评审。

```java
// TODO(PROJ-1234, zhangsan): 临时内存缓存，Q2 迁移 Redis
// FIXME(PROJ-5678): 订单超 1000 行时 O(n²)，需重构批处理
```

### 6.4 文档文化

文档是异步沟通的最高形式。

| 文档 | 内容 |
|------|------|
| **README** | 项目是什么、怎么跑、目录结构 |
| **Changelog** | 版本变更日志（可由 conventional commits 自动生成） |
| **ADR** | 重大架构决策记录：背景→选项→决策→后果 |
| **运维手册** | 故障排查、回滚流程 |

ADR 模板：

```markdown
# ADR-003: 选择 Redis 作为会话存储
- **状态**: 已采纳  **日期**: 2026-04-10
- **背景**: 迁微服务，需集中式会话存储
- **决策**: Redis，TTL 天然满足过期需求
- **后果**: 引入 Redis 运维复杂度
```

### 6.5 版本兼容性

API 一旦发布就像泼出去的水。

```java
// ✅ 兼容：新增字段——旧客户端忽略即可
private String extInfo;
// ❌ 不兼容：修改字段语义、删除已有接口

@Deprecated(since = "2.0", forRemoval = true)
public Order createOrder(Long userId, Long skuId, int quantity) {
    return createOrder(CreateOrderRequest.of(userId, skuId, quantity));
}
```

**版本策略**：中小团队用 URL 路径版本（`/api/v1/orders`），成本最低。

### 6.6 面向故障设计

分布式系统中，故障不是"会不会发生"，而是"什么时候发生"。

| 模式 | 落地方案 |
|------|----------|
| **超时** | 每次外部调用设超时，逐级递缩 Client(5s)→Gateway(4s)→Svc(3s)→Pay(2s) |
| **重试** | 幂等可重试非幂等写禁止 Spring Retry + 1s→2s→4s 退避 |
| **降级/熔断/限流** | Sentinel fallback + 慢调用熔断 + QPS 限流 |
| **隔离** | 独立线程池防扩散（Bulkhead） |

```java
@Retryable(value = RemoteAccessException.class, maxAttempts = 3,
           backoff = @Backoff(delay = 1000, multiplier = 2.0))
public PaymentResult callPaymentGateway(PaymentRequest req) { ... }
@Recover
public PaymentResult fallback(PaymentRequest req, RemoteAccessException e) {
    log.error("支付网关重试耗尽, orderId={}", req.getOrderId(), e);
    return PaymentResult.fail("支付服务暂不可用");
}
```

> 流量治理、Sentinel 配置参考 [微服务架构](..\distributed\microservice.md) 的"流量治理"章节。限流算法参考 [分布式系统理论基础](..\distributed\distributed-theory.md) 的"流量控制"章节。

---

## 总结

六个维度的工程规范，核心落在三件事：

1. **防御性编程**：永远不信任外部输入，Fail Fast，为最坏情况做准备
2. **让规范自动化**：Commit 模板、Checkstyle、CI 静态扫描——人靠意志力，机器靠配置
3. **面向故障设计**：超时、重试、降级、熔断不是"高级功能"，是每个调用外部依赖的接口都必须认真对待的基本功
