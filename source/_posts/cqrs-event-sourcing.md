---
title: CQRS 与事件溯源
date: 2026-07-05
updated: 2026-07-05
tags:
  - DDD
  - CQRS
  - 事件溯源
  - Event Sourcing
categories:
  - DDD
---

读写分离——查询需求常需扁平化多表 JOIN，与聚合模型不一致。引入 CQRS 可解除这种矛盾：

```java
// 写模型——完整聚合
@Service
public class OrderWriteService {
    public void submitOrder(SubmitOrderCommand cmd) { ... }
}

// 读模型——扁平化视图，可直查 ES/Redis/物化视图
@Service
public class OrderReadService {
    public OrderDetailVO getOrderDetail(String orderId) {
        return orderReadDao.selectDetail(orderId);  // 预 JOIN 好
    }
}
```

**事件溯源 (Event Sourcing)** 是 CQRS 进阶版：存储所有事件序列而非当前状态。适用于审计要求极高的金融场景，但复杂度高，不要轻易使用。
