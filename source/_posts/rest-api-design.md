---
title: REST API 设计规范
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - API
  - REST
  - 设计
  - 版本管理
  - 幂等
categories:
  - API
---

REST API 是后端工程师每天都要打交道的东西。但"RESTful"这个词被过度使用——很多自称 REST 的 API 其实连基本规范都没遵守。这篇文章梳理一套切实可用的设计规范。

## 一、URL 设计

### 1.1 命名规则

```
✅ GET    /users              # 用户列表
✅ GET    /users/123          # 单个用户
✅ POST   /users              # 创建用户
✅ PUT    /users/123          # 全量更新用户
✅ PATCH  /users/123          # 部分更新用户
✅ DELETE /users/123          # 删除用户
✅ GET    /users/123/orders   # 用户的订单列表

❌ GET    /getUser?id=123     # 动词、驼峰
❌ POST   /createUser         # 动词
❌ GET    /api/users/123      # /api 前缀由网关统一添加
```

- 用名词而非动词
- 用复数而非单数
- 子资源用路径嵌套
- 避免深嵌套（最多 2 层：`/users/123/orders`，不要 `/users/123/orders/456/items`）

### 1.2 分页

```json
// 请求
GET /users?page=2&page_size=20

// 响应
{
  "data": [...],
  "pagination": {
    "page": 2,
    "page_size": 20,
    "total": 1043,
    "total_pages": 53,
    "next": "/users?page=3&page_size=20",
    "prev": "/users?page=1&page_size=20"
  }
}
```

**游标分页**（适用于实时 Feed 流）：

```json
{
  "data": [...],
  "pagination": {
    "cursor": "1625097600000",
    "has_more": true
  }
}
```

游标分页避免 offset 在数据变化时的重复/丢失问题。

### 1.3 过滤、排序、字段选择

```
GET /users?status=active&role=admin                    # 过滤
GET /users?sort=-created_at,+name                      # 排序（- 降序）
GET /users?fields=id,name,email                        # 字段选择
GET /users?q=search+keyword                            # 搜索
```

## 二、HTTP 状态码

| 码 | 含义 | 使用场景 |
|----|------|----------|
| 200 | OK | GET/PUT/PATCH 成功 |
| 201 | Created | POST 创建成功，返回 `Location` 头 |
| 204 | No Content | DELETE 成功 |
| 400 | Bad Request | 参数校验失败、格式错误 |
| 401 | Unauthorized | 未提供凭证 |
| 403 | Forbidden | 已认证但无权限 |
| 404 | Not Found | 资源不存在 |
| 409 | Conflict | 并发冲突 |
| 422 | Unprocessable | 语义错误（如重复提交） |
| 429 | Too Many Requests | 限流 |
| 500 | Internal Server Error | 未预期的服务端错误 |
| 503 | Service Unavailable | 降级/维护 |

**不要用 200 + error_code 字段替代 HTTP 状态码**——中间件（网关、CDN、负载均衡器）都依赖 HTTP 状态码做流量控制和错误统计。

## 三、响应体结构

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "id": 123,
    "name": "pcl",
    "email": "pcl@example.com"
  },
  "request_id": "a1b2c3d4"
}
```

- `code`: 业务错误码，0 表示成功；与 HTTP 状态码互补（HTTP 状态码告诉网关"发生了什么"，业务错误码告诉客户端"为什么"）
- `message`: 面向开发者的提示；面向用户的文案由客户端提供
- `request_id`: 全链路追踪 ID，排查问题时关联前后端日志

**错误响应**：

```json
{
  "code": 10001,
  "message": "email already exists",
  "data": null,
  "request_id": "a1b2c3d4"
}
```

## 四、API 版本管理

| 方案 | 做法 | 优缺点 |
|------|------|--------|
| **URL 版本** | `/v1/users` `/v2/users` | 最直观，网关路由简单；URL 变化 |
| **Header 版本** | `Accept: application/vnd.api.v2+json` | URL 不变；调试不直观 |
| **Query 参数** | `/users?version=2` | 简单；污染 URL |

**推荐**：对外 API 用 URL 版本（清晰、可缓存、网关友好），内部 API 用 Header 版本。

版本升级策略：
- 新版本上线后，旧版本至少保留 6 个月
- 通过监控和日志统计旧版本的调用量，降为 0 后才能下线
- 版本升级指南用 changelog 形式通知调用方

## 五、幂等性设计

支付、扣款、下单等写操作必须支持幂等：

```
// 客户端生成幂等键
POST /v1/orders
Idempotency-Key: 8f7d3b2a-1c4e-4a5b-9d6e-3f8a7c1b2d4e

// 服务端用 Redis + SETNX 保证幂等
SETNX idempotent:{key} TTL=24h
```

幂等键在服务端的存储需要设置 TTL（通常 24-72 小时），防止无限增长。对外告知客户端幂等键的格式要求（UUID v4 最佳），以及重试的合理间隔（指数退避）。

GET 和 DELETE 天然幂等。PUT 幂等（相同输入相同输出）。PATCH 不保证幂等（取决于 patch 操作，需额外去重逻辑）。只有 POST 需要显式的幂等设计。

## 六、鉴权

```
Authorization: Bearer eyJhbGciOi...
```

框架：JWT（无状态，适合分布式）或 Opaque Token（可撤销，需要查 Redis）。

JWT 注意事项：
- 密钥定期轮换（用 `kid` header 标识当前密钥）
- 短期 access token（15-60min）+ 长期 refresh token（7-30天）
- 不要在 JWT 中存储敏感信息（payload 仅 base64 编码，非加密）

## 七、限流头

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1625098200
```

让客户端知道自己还剩多少次调用、什么时候重置。超过限制返回 429 时，附带 `Retry-After` 头告知重试时间。

## 八、HATEOAS

HATEOAS（Hypermedia as the Engine of Application State）是最容易被忽视的 REST 约束：

```json
{
  "id": 123,
  "name": "pcl",
  "links": [
    { "rel": "self",     "href": "/users/123" },
    { "rel": "orders",   "href": "/users/123/orders" },
    { "rel": "settings", "href": "/users/123/settings" }
  ]
}
```

返回值中包含相关资源的链接，客户端可以动态发现后续操作——减少了客户端和服务端的耦合。实际开发中多数团队不实现完全 HATEOAS，但至少返回 `self` 链接，方便调试和日志。

## 九、CORS 配置

```
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET,POST,PUT,DELETE
Access-Control-Allow-Headers: Content-Type,Authorization
Access-Control-Max-Age: 86400
```

不要用 `*` 通配符，明确指定允许的域名以减少安全风险。

## 十、小结

REST API 设计规范的核心原则：URL 是资源名词、HTTP 方法表述操作、状态码表明结果、幂等键保证安全、版本号管理演进。好的 API 是团队间的契约，需要在可读性、兼容性和安全性之间找到平衡。
