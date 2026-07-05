---
title: OAuth2 与 SSO
date: 2026-07-05
updated: 2026-07-05
tags:
  - 安全
  - OAuth2
  - SSO
  - 认证
categories:
  - 安全
---

OAuth 2.0 解决"授权委托"——用户允许第三方访问自己在服务提供商的资源，但**不把自己的密码交给第三方**。

**核心角色**：Resource Owner（用户）、Client（第三方应用）、Authorization Server（认证服务器）、Resource Server（资源服务器）。

**四种授权模式**：

**（1）授权码模式（Authorization Code）**——最安全，有后端 Web 应用：

```
用户 → Client → 重定向到 Auth Server（带 client_id, redirect_uri）
Auth Server → 用户登录授权
Auth Server → Client 回调 redirect_uri（带 authorization code）
Client → Auth Server（用 code + client_secret 后端直连换取 access_token）
```

关键：code 走前端重定向（可能泄露于浏览器 URL），但 token 通过后端直连换取。攻击者即使拦截 code，没有 `client_secret` 也换不到 token——**code 只是一次性中间凭证**。

**（2）隐式模式**：已废弃。Auth Server 直接把 access_token 返回到前端 URL fragment（`example.com/callback#access_token=xxx`），跳过了用 client_secret 换 token 的后端步骤。问题在于 token 在 URL fragment 和浏览器历史中暴露，没有 client_secret 校验，任何拿到 token 的人都能冒充客户端访问资源。OAuth 2.1 草案已将其移除，推荐任何场景都使用 PKCE 取代隐式模式。历史面试题"四种模式的区别"现在答"三种加一个已废弃"。

**（3）密码模式**：用户把密码直接给 Client，Client 用 username/password 直接换 token。仅适用于**自家第一方应用**（如官方 App），因为用户只信任官方。第三方用此模式意味着你把密码给了陌生人，完全背离 OAuth"不共享密码"的设计初衷。OAuth 2.1 已将其移除。

**四种模式对比**：

| 模式 | 适用场景 | 安全等级 | 状态 |
|------|----------|---------|------|
| 授权码+PKCE | Web/SPA/移动端 | 最高 | 推荐 |
| 隐式 | 旧版 SPA | 低 | 已废弃 |
| 密码 | 第一方应用 | 中 | 已移除 |
| 客户端凭证 | 服务间调用 | 高 | 推荐 |

**（4）客户端凭证模式**：`Client → Auth Server（client_id + client_secret）→ access_token`，服务器到服务器，用于微服务间/定时任务认证。

**PKCE（Proof Key for Code Exchange）**——移动端/SPA 无法安全存储 `client_secret`（反编译即泄露）。攻击者可在设备上注册恶意应用并注册相同的回调 URL Scheme 拦截 code。PKCE 在授权请求时传 `code_challenge = SHA256(code_verifier)`，换 token 时传 `code_verifier`，Auth Server 验证 `SHA256(verifier) == challenge`。即使 code 被拦截，没有 verifier 也换不到 token——用非对称方式确保只有发起方才能完成令牌交换。OAuth 2.1 已将 PKCE 设为必选项，推荐所有场景都启用。
