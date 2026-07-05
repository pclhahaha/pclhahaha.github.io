---
title: JWT 认证机制
date: 2026-07-05
updated: 2026-07-05
tags:
  - 安全
  - JWT
  - Token
  - 认证
categories:
  - 安全
---

JWT 是自包含令牌，服务端无需查询存储即可独立验证。

**结构**：`header.payload.signature`，三段 Base64URL 编码用 `.` 拼接。

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMDAxIiwicm9sZSI6ImFkbWluIn0.3Kx8L...
└──── header ────┘ └────────── payload ──────────┘ └── signature ──┘
```

- **Header**：算法（HS256/RS256）和类型（JWT）。
- **Payload**：Claims（`sub`/`iat`/`exp` + 业务自定义字段）。**仅 Base64 编码非加密，任何人都能解码，绝对不能放密码等敏感信息。**
- **Signature**：对 header+payload 用 secret（HMAC）或私钥（RSA）签名。篡改任一字节签名即不匹配。

**签发与验证**：

```java
// 签发
public String generateToken(UserDetails user) {
    return Jwts.builder()
        .setSubject(user.getUsername())
        .claim("userId", user.getId())
        .setExpiration(new Date(System.currentTimeMillis() + 7200_000))
        .signWith(SignatureAlgorithm.HS256, secretKey)
        .compact();
}
// 验证
public Claims validateToken(String token) {
    return Jwts.parser().setSigningKey(secretKey).parseClaimsJws(token).getBody();
}
```

**优点**：无状态（天然支持水平扩展，各微服务独立验证不需要共享存储）；跨域友好（`Authorization: Bearer <token>` 不受同源策略限制）；微服务调用链可直接透传 JWT。

**无状态带来的挑战——退出登录**：JWT 最大的卖点也是最大痛点——签发后在过期前无法撤销。工程实践：

1. **短过期 + Refresh Token**（最主流）：Access Token 5-15min，Refresh Token 7-30d。退出时将 Refresh Token 加入 Redis 黑名单。Access Token 最多存活 15min 自然过期，无法再刷新出新令牌。

```java
public TokenResponse refresh(String refreshToken) {
    if (redisTemplate.hasKey("blacklist:" + refreshToken))
        throw new UnauthorizedException("令牌已吊销");
    // 旧 Refresh Token 标记已用（防重放）
    redisTemplate.opsForValue().set("blacklist:" + refreshToken, "1", 30, TimeUnit.DAYS);
    return new TokenResponse(newAccessToken, newRefreshToken);
}
```

2. **JWT 黑名单**：在网关层维护 `jti` 黑名单。但让"无状态"形同虚设——又回到中心化查询状态。
3. **极短 Access Token**：5min 过期，客户端丢弃即失效。适合对即时退出要求不高的场景。

**选用建议**：单体/管理后台 → Session-Cookie；微服务/移动端/跨域 SPA → JWT + Refresh Token。

**面试常问——JWT 的 Signature 用对称（HS256）还是非对称（RS256）？** HS256 颁发和验证用同一把密钥——适合单服务场景，密钥泄露后攻击者可以自己签发有效 JWT。RS256 用私钥签名、公钥验证——适合微服务场景：认证中心持有私钥签发，各微服务只需持有公钥即可验证。即使某个微服务沦陷，公钥泄露也无法伪造 JWT。生产环境推荐 RS256/ES256（ECDSA），密钥管理更安全。
