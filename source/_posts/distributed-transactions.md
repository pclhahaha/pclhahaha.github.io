---
title: 分布式事务方案对比
date: 2026-07-05
updated: 2026-07-05
tags:
  - 分布式
  - 分布式事务
  - 2PC
  - TCC
  - Saga
  - Seata
categories:
  - 分布式
---

分布式事务泛指跨多个节点的事务操作，目标是保证跨节点的数据一致性。传统单机数据库的 ACID 事务在分布式环境下无法直接使用，需要方案升级。

### 1.1 2PC（两阶段提交）

#### 流程

2PC 引入协调者（Coordinator），将事务分为两个阶段：

**阶段一 Prepare（投票阶段）**：
1. 协调者向所有参与者发送 prepare 请求
2. 参与者执行事务操作（但不提交），记录 undo 和 redo 日志
3. 参与者返回 YES（可以提交）或 NO（需要回滚）

**阶段二 Commit（提交阶段）**：
1. 若所有参与者返回 YES → 协调者发 commit 请求，参与者提交事务并释放锁
2. 若任一参与者返回 NO 或超时 → 协调者发 rollback 请求，参与者回滚并释放锁

```java
// 2PC 协调者伪代码
public boolean twoPhaseCommit(List<Participant> participants) {
    // Phase 1: Prepare
    for (Participant p : participants) {
        boolean ok = p.prepare();
        if (!ok) {
            rollbackAll(participants);
            return false;
        }
    }
    // Phase 2: Commit
    for (Participant p : participants) {
        p.commit();
    }
    return true;
}
```

#### 缺陷

| 缺陷 | 描述 |
|------|------|
| **同步阻塞** | Prepare 阶段参与者锁定资源后等待协调者指令，期间其他事务无法访问这些资源 |
| **单点故障** | 协调者宕机后，参与者无法知道应该 commit 还是 rollback，陷入"阻塞等待" |
| **脑裂** | commit/rollback 阶段，部分参与者收到指令、部分未收到，数据不一致 |
| **性能差** | 所有参与者必须串行等待，任一慢节点拖慢整个事务 |

### 1.2 3PC（三阶段提交）

3PC 在 2PC 基础上增加了一个 CanCommit 阶段，并为参与者和协调者分别引入超时机制：

1. **CanCommit**：协调者询问参与者"是否可以提交"（不执行任何操作，不持锁）
2. **PreCommit**：协调者发送预提交，参与者执行事务但不提交
3. **DoCommit**：协调者发送提交，参与者提交或超时自动提交

#### 改进
- 引入超时机制：参与者等待超时后**自动提交**（假设协调者已同意），减少阻塞时间
- CanCommit 阶段降低不必要持有的锁时间

#### 缺陷
- **网络分区问题未根本解决**：如果 PreCommit 后发生网络分区，部分参与者因超时自动提交、部分收到回滚——数据依然不一致
- 多一轮通信开销，在高可用系统中性能反而不如 2PC
- 在现实中几乎不被使用，更像一个学术上的改进尝试

### 1.3 TCC（Try-Confirm-Cancel）

TCC 是二阶段提交在业务层的变体，将事务拆分为三个操作：

- **Try**：预留资源（锁定库存、冻结余额）
- **Confirm**：确认提交（扣减库存、扣款）
- **Cancel**：取消释放（释放库存、解冻余额）

```java
// TCC 接口示例
public interface TccService {
    // Try: 冻结库存
    boolean tryFreezeInventory(String orderId, String skuId, int count);
    
    // Confirm: 确认扣减
    boolean confirmDeductInventory(String orderId, String skuId);
    
    // Cancel: 释放冻结
    boolean cancelReleaseInventory(String orderId, String skuId);
}
```

#### 完整实现：TCC 事务协调器

```
                    ┌─────────────┐
                    │ TCC 协调器   │
                    │ (统筹状态机)  │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
          [库存服务]   [账户服务]   [订单服务]
          Try/Confirm  Try/Confirm  Try/Confirm
          /Cancel      /Cancel      /Cancel
```

**事务日志表**——记录每个 TCC 事务的状态：

```sql
CREATE TABLE tcc_transaction (
    tcc_id      VARCHAR(64) PRIMARY KEY,   -- 全局事务 ID
    status      TINYINT DEFAULT 0,         -- 0-进行中 1-已确认 2-已取消
    create_time DATETIME,
    update_time DATETIME
);

CREATE TABLE tcc_branch (
    branch_id   VARCHAR(64) PRIMARY KEY,
    tcc_id      VARCHAR(64),
    service     VARCHAR(64),               -- 服务名
    status      TINYINT DEFAULT 0,         -- 0-TRY 1-CONFIRM 2-CANCEL
    retry_count INT DEFAULT 0,
    INDEX idx_tcc (tcc_id)
);
```

**协调器核心逻辑**：

```java
public class TccCoordinator {
    
    // 阶段1：Try 所有分支
    public boolean tryAll(String tccId, List<TccCall> calls) {
        // 记录全局事务
        logStart(tccId);
        
        List<TccCall> succeeded = new ArrayList<>();
        try {
            for (TccCall call : calls) {
                if (call.try()) {
                    succeeded.add(call);
                    logBranch(tccId, call.getBranchId(), BRANCH_TRY);
                } else {
                    throw new TccException("Try failed: " + call.getBranchId());
                }
            }
            // 全部 Try 成功 → Confirm 所有
            confirmAll(tccId, calls);
            return true;
        } catch (Exception e) {
            // 任一 Try 失败 → Cancel 已成功的分支
            cancelAll(tccId, succeeded);
            return false;
        }
    }
    
    // 阶段2a：Confirm 所有分支
    private void confirmAll(String tccId, List<TccCall> calls) {
        for (TccCall call : calls) {
            retryUntilSuccess(() -> call.confirm(), tccId, call.getBranchId());
        }
        logFinish(tccId, STATUS_CONFIRMED);
    }
    
    // 阶段2b：Cancel 已 Try 成功的分支
    private void cancelAll(String tccId, List<TccCall> succeeded) {
        for (TccCall call : succeeded) {
            retryUntilSuccess(() -> call.cancel(), tccId, call.getBranchId());
        }
        logFinish(tccId, STATUS_CANCELLED);
    }
    
    // 重试直到成功（Confirm/Cancel 必须成功）
    private void retryUntilSuccess(Runnable op, String tccId, String branchId) {
        int retries = 0;
        while (retries < MAX_RETRIES) {
            try {
                op.run(); return;
            } catch (Exception e) {
                retries++;
                sleep(Math.min(1000L << retries, 60000L));  // 指数退避，最大 60s
            }
        }
        alertHuman(tccId, branchId);  // 最终失败 → 人工介入
    }
}
```

**关键挑战及实现**：

**空回滚（Empty Rollback）**：Try 因网络超时未到达服务端，但协调者超时判定需要 Cancel。

```java
// 防悬挂/防空回滚实现
public class InventoryTccService {
    
    public boolean tryFreezeInventory(String orderId, String skuId, int count) {
        // 1. 先检查是否已被 Cancel（防悬挂）
        if (isCancelled(orderId)) return false;
        
        // 2. 幂等检查
        if (isTried(orderId)) return true;
        
        // 3. 执行业务 + 记录状态（同一本地事务）
        tccLogMapper.insert(orderId, "TRY");  // 记录 Try 状态
        inventoryMapper.freeze(skuId, count);
        return true;
    }
    
    public boolean cancelReleaseInventory(String orderId, String skuId) {
        // 1. 幂等检查
        if (isCancelled(orderId)) return true;
        
        // 2. 空回滚处理：如果从未 Try，也记录 Cancel 并返回成功
        if (!isTried(orderId)) {
            tccLogMapper.insert(orderId, "CANCEL");  // 防悬挂标记
            return true;
        }
        
        // 3. 执行回滚 + 记录状态
        inventoryMapper.unfreeze(skuId, getFrozenCount(orderId));
        tccLogMapper.updateStatus(orderId, "CANCEL");
        return true;
    }
}
```

### 1.4 Saga

Saga 的核心是"长事务拆为一组有序的本地事务 + 补偿事务"。与 TCC 不同，Saga 的本地事务直接提交（不先 Try），失败时执行补偿回滚。

#### 协调式 Saga 引擎实现

```java
public class SagaOrchestrator {
    
    // 定义 Saga 步骤
    public void executeOrderSaga(String orderId, OrderRequest request) {
        Saga saga = new Saga(orderId);
        
        saga.addStep("createOrder", 
            () -> orderService.createOrder(request),           // 正向
            () -> orderService.cancelOrder(request)            // 补偿
        );
        
        saga.addStep("reserveInventory",
            () -> inventoryService.deduct(request.getSkuId(), request.getCount()),
            () -> inventoryService.rollback(request.getSkuId(), request.getCount())
        );
        
        saga.addStep("processPayment",
            () -> paymentService.charge(request.getAmount()),
            () -> paymentService.refund(request.getAmount())
        );
        
        saga.execute();
    }
}
```

**Saga 内部状态机**：

```
    ┌────────┐
    │ START  │
    └───┬────┘
        │
   ┌────▼────┐    失败     ┌─────────┐
   │ EXECUTING│───────────→│COMPENSATING│
   └────┬────┘            └─────┬─────┘
        │ 成功                   │
   ┌────▼────┐             ┌────▼────┐
   │COMPLETED│             │ ABORTED │
   └─────────┘             └─────────┘
```

```java
public class Saga {
    private final List<SagaStep> steps = new ArrayList<>();
    private String state = "START";
    
    public void execute() {
        state = "EXECUTING";
        List<Integer> completedSteps = new ArrayList<>();
        
        try {
            for (int i = 0; i < steps.size(); i++) {
                steps.get(i).execute();      // 执行正向操作
                completedSteps.add(i);       // 记录已完成的步骤
                persistState(completedSteps); // 持久化到 DB
            }
            state = "COMPLETED";
        } catch (Exception e) {
            state = "COMPENSATING";
            // 逆序执行补偿（后执行的先回滚）
            for (int i = completedSteps.size() - 1; i >= 0; i--) {
                steps.get(completedSteps.get(i)).compensate();
            }
            state = "ABORTED";
        }
    }
}
```

**Saga 日志表**——支持宕机恢复：

```sql
CREATE TABLE saga_log (
    saga_id     VARCHAR(64) PRIMARY KEY,
    step_index  INT,                      -- 当前执行到第几步
    status      TINYINT,                  -- 0-进行中 1-已完成 2-已终止
    step_status VARCHAR(1024),            -- JSON: [{"step":0,"ok":true},{"step":1,"ok":false}]
    payload     TEXT,                     -- 请求参数，用于恢复时重试
    create_time DATETIME
);
```

**宕机恢复**：Saga 管理器启动时扫描状态为"进行中"的记录，根据 step_status 判断从哪一步继续执行补偿——被中断的事务如果有部分成功执行的步骤，从最后成功的步骤之后开始补偿。

#### 编排式 Saga（事件驱动）

不需要中心协调器，各服务通过事件总线通信：

```
订单服务 → [OrderCreated] → 库存服务 → [InventoryReserved] → 支付服务 → [PaymentCompleted]
                │ 失败                    │ 失败                    │ 失败
                ↓                         ↓                        ↓
          [OrderFailed]           [InventoryRelease]        [PaymentRefund]
```

各服务监听上游事件并执行自己的逻辑，失败时发布补偿事件。优点是无中心协调器、高度解耦；缺点是流程分散难以追踪——需要分布式链路追踪辅助调试。"

### 1.5 本地消息表 + 最大努力通知

这是一种通过消息中间件实现最终一致性的方案：

**核心思路**：
1. 业务操作和消息记录在**同一个本地事务**中写入（本地消息表与业务表同库）
2. 后台任务轮询消息表，将消息投递到消息队列
3. 消费者处理消息，通过 ACK 确认消费
4. 对于需要通知结果的场景，生产方提供查询接口，消费方处理后回调通知

```sql
-- 本地消息表结构
CREATE TABLE local_message (
    id          BIGINT PRIMARY KEY,
    business_id VARCHAR(64),
    topic       VARCHAR(128),
    payload     TEXT,
    status      TINYINT DEFAULT 0,  -- 0待发送 1已发送 2已消费
    retry_count INT DEFAULT 0,
    create_time DATETIME,
    update_time DATETIME
);
```

```java
// 业务操作 + 消息写入在同一事务中
@Transactional
public void placeOrder(Order order) {
    orderMapper.insert(order);                    // 业务操作
    messageMapper.insert(buildMessage(order));    // 消息记录
}
```

**最大努力通知**：投递方尽力通知，不保证 100% 到达；如果通知失败，依赖消费方的查询机制来获取最终结果。适用于对一致性要求不是 100% 的场景（如支付结果回调）。

### 1.6 AT 模式（Seata 原理）

Seata 的 AT 模式是一种无侵入的分布式事务方案，核心机制包括：

1. **全局事务**：由 TM（Transaction Manager）发起，TC（Transaction Coordinator）协调
2. **分支事务**：每个参与的 RM（Resource Manager）注册分支事务
3. **两阶段提交**：
   - **一阶段**：各 RM 执行 SQL 并提交本地事务，同时记录 **undo_log（前镜像 + 后镜像）**
   - **二阶段-提交**：TC 通知全局提交，各 RM 异步删除 undo_log
   - **二阶段-回滚**：TC 通知全局回滚，各 RM 根据 undo_log 生成反向 SQL 并执行

```sql
-- Seata undo_log 表结构
CREATE TABLE undo_log (
    id            BIGINT AUTO_INCREMENT PRIMARY KEY,
    branch_id     BIGINT,
    xid           VARCHAR(100),
    context       VARCHAR(128),
    rollback_info LONGBLOB,
    log_status    INT,
    log_created   DATETIME,
    log_modified  DATETIME
);
```

**关键设计**：
- undo_log 与业务操作在同一个本地事务中提交，保证原子性
- 使用全局锁防止二阶段回滚期间数据被修改
- 一阶段本地事务的提交释放了数据库锁，极大减少了 2PC 的锁持有时间
- 前后镜像对比生成反向 SQL，不需要业务方编写补偿逻辑

### 1.7 各方案对比表

| 方案 | 一致性 | 性能 | 侵入性 | 适用场景 |
|------|--------|------|--------|----------|
| 2PC | 强一致 | 低 | 低 | 短事务、跨库操作 |
| 3PC | 强一致 | 低 | 低 | 理论模型，生产极少使用 |
| TCC | 最终一致 | 中 | 高（需实现Try/Confirm/Cancel） | 资金扣减、库存锁定 |
| Saga 编排 | 最终一致 | 中高 | 中 | 长事务、跨服务流程 |
| Saga 协调 | 最终一致 | 中 | 中 | 流程可控的长事务 |
| 本地消息表 | 最终一致 | 高 | 低 | 异步解耦、非实时一致性 |
| 最大努力通知 | 最终一致 | 高 | 低 | 回调通知、弱依赖 |
| Seata AT | 弱一致/最终一致 | 中高 | 低（无侵入） | 希望低改造成本的场景 |

### 1.8 生产实践要点

1. **不要在所有场景使用分布式事务**——优先考虑业务重构（如合并服务减少跨库操作），实在不行再上分布式事务
2. **幂等性是分布式事务的基石**——所有 RPC 调用、消息消费、补偿操作都必须设计为幂等
3. **事务反查机制**：TCC 应提供反查接口，Saga 的协调器需要能查询分支事务状态
4. **监控与告警**：对所有分布式事务建立全链路追踪和异常告警，及时发现悬挂、空回滚、超时等问题
5. **避免长事务**：事务跨度越长，锁定资源越久，失败概率越大——长事务应拆分为 Saga
