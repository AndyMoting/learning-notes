# 13-HTTP协议详解

> 课时：45 min | 难度：★★★☆☆

## 学习目标

- 掌握HTTP请求与响应的完整结构，能够在接口测试中准确构造和解析报文
- 理解常见状态码的含义，能根据状态码快速定位接口问题
- 区分GET/POST/PUT/DELETE的语义差异，在接口设计中正确选用
- 理解Cookie/Session/Token三种认证机制的原理与测试要点
- 了解HTTPS/TLS握手过程及HTTP/2、HTTP/3的核心改进

---

## 核心概念

### 1. HTTP请求结构

HTTP请求报文由四部分组成：

```
GET /api/users/123 HTTP/1.1
Host: example.com
Accept: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
Content-Type: application/json

{"name": "test", "age": 25}
```

| 组成部分 | 说明 | 示例 |
|---------|------|------|
| 请求行 | Method + URL + HTTP版本 | `GET /api/users HTTP/1.1` |
| 请求头 | 键值对，描述请求元数据 | `Content-Type: application/json` |
| 空行 | 分隔头部与正文 | CRLF |
| 请求体 | 可选，承载实际数据 | `{"username": "admin"}` |

#### 常见请求头

| 请求头 | 用途 | 示例 |
|--------|------|------|
| `Host` | 目标主机和端口 | `Host: api.example.com` |
| `Accept` | 客户端可接收的MIME类型 | `Accept: application/json` |
| `Content-Type` | 请求体的媒体类型 | `Content-Type: application/json` |
| `Authorization` | 认证凭证 | `Authorization: Bearer <token>` |
| `Cookie` | 携带的Cookie数据 | `Cookie: sessionid=abc123` |
| `User-Agent` | 客户端标识 | `User-Agent: Mozilla/5.0` |
| `Referer` | 请求来源页 | `Referer: https://example.com/login` |
| `Cache-Control` | 缓存策略 | `Cache-Control: no-cache` |
| `Content-Length` | 请求体字节数 | `Content-Length: 128` |
| `Accept-Encoding` | 支持的压缩算法 | `Accept-Encoding: gzip, deflate` |
| `Accept-Language` | 偏好语言 | `Accept-Language: zh-CN,zh;q=0.9` |
| `Origin` | 跨域请求来源 | `Origin: https://www.example.com` |
| `X-Requested-With` | AJAX标识 | `X-Requested-With: XMLHttpRequest` |
| `If-Modified-Since` | 条件请求 | `If-Modified-Since: Wed, 21 Oct 2024 07:28:00 GMT` |
| `If-None-Match` | ETag验证 | `If-None-Match: "33a64df5"` |

---

### 2. HTTP响应结构

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 42
Set-Cookie: sessionid=xyz789; Path=/; HttpOnly
Cache-Control: max-age=3600

{"id": 123, "name": "test", "status": "active"}
```

| 组成部分 | 说明 |
|---------|------|
| 状态行 | HTTP版本 + 状态码 + 原因短语 |
| 响应头 | 描述响应元数据 |
| 空行 | 分隔头部与正文 |
| 响应体 | 实际返回的数据 |

#### 常见响应头

| 响应头 | 用途 | 示例 |
|--------|------|------|
| `Content-Type` | 响应体MIME类型 | `Content-Type: application/json; charset=utf-8` |
| `Content-Length` | 响应体字节数 | `Content-Length: 1024` |
| `Set-Cookie` | 设置Cookie | `Set-Cookie: token=abc; HttpOnly; Secure` |
| `Cache-Control` | 缓存指令 | `Cache-Control: no-store` |
| `ETag` | 资源版本标识 | `ETag: "33a64df5"` |
| `Location` | 重定向目标URL | `Location: /api/users/123` |
| `Access-Control-Allow-Origin` | CORS允许源 | `Access-Control-Allow-Origin: *` |
| `X-RateLimit-Limit` | 限流上限 | `X-RateLimit-Limit: 100` |
| `X-RateLimit-Remaining` | 剩余可请求数 | `X-RateLimit-Remaining: 42` |
| `WWW-Authenticate` | 认证方式声明 | `WWW-Authenticate: Bearer` |
| `Retry-After` | 重试等待秒数 | `Retry-After: 120` |

---

### 3. 状态码完整参考

#### 1xx 信息性状态码

| 状态码 | 含义 | 测试场景 |
|--------|------|---------|
| 100 Continue | 继续发送请求体 | 大文件上传前校验 |
| 101 Switching Protocols | 协议切换 | WebSocket握手 |
| 103 Early Hints | 预加载提示 | 资源预加载场景 |

#### 2xx 成功状态码

| 状态码 | 含义 | 测试场景 |
|--------|------|---------|
| 200 OK | 请求成功 | 标准成功响应 |
| 201 Created | 资源已创建 | POST创建用户/订单 |
| 202 Accepted | 已接受，异步处理 | 批量任务、消息队列 |
| 204 No Content | 成功但无返回体 | DELETE操作 |
| 206 Partial Content | 分块下载 | 断点续传、视频流 |

#### 3xx 重定向状态码

| 状态码 | 含义 | 测试场景 |
|--------|------|---------|
| 301 Moved Permanently | 永久重定向 | 域名迁移、URL规范化 |
| 302 Found | 临时重定向 | 临时跳转 |
| 304 Not Modified | 资源未修改 | 缓存验证（ETag/Last-Modified） |
| 307 Temporary Redirect | 临时重定向（保持方法） | POST请求跳转 |
| 308 Permanent Redirect | 永久重定向（保持方法） | POST请求永久跳转 |

#### 4xx 客户端错误状态码

| 状态码 | 含义 | 测试场景 |
|--------|------|---------|
| 400 Bad Request | 请求格式错误 | 参数缺失、JSON格式错 |
| 401 Unauthorized | 未认证 | Token过期、未登录 |
| 403 Forbidden | 无权限 | 角色权限不足 |
| 404 Not Found | 资源不存在 | URL错误、资源已删除 |
| 405 Method Not Allowed | 方法不允许 | GET接口发POST请求 |
| 408 Request Timeout | 请求超时 | 网络慢、服务端阻塞 |
| 409 Conflict | 资源冲突 | 重复创建、并发编辑 |
| 413 Payload Too Large | 请求体过大 | 文件上传超限 |
| 415 Unsupported Media Type | 不支持的媒体类型 | Content-Type错误 |
| 422 Unprocessable Entity | 语义错误 | 参数校验失败（业务规则） |
| 429 Too Many Requests | 请求过于频繁 | 限流、防刷 |
| 451 Unavailable For Legal Reasons | 法律原因不可用 | 版权/合规限制 |

#### 5xx 服务端错误状态码

| 状态码 | 含义 | 测试场景 |
|--------|------|---------|
| 500 Internal Server Error | 服务端内部异常 | 代码缺陷（NullPointerException等） |
| 501 Not Implemented | 功能未实现 | 预留接口、版本差异 |
| 502 Bad Gateway | 网关错误 | 上游服务不可用 |
| 503 Service Unavailable | 服务不可用 | 维护、过载、宕机 |
| 504 Gateway Timeout | 网关超时 | 上游响应超时 |

---

### 4. HTTP方法对比

| 特性 | GET | POST | PUT | DELETE |
|------|-----|------|-----|--------|
| 用途 | 获取资源 | 创建资源 | 全量更新资源 | 删除资源 |
| 幂等性 | ✅ 是 | ❌ 否 | ✅ 是 | ✅ 是 |
| 安全性 | ✅ 安全 | ❌ 不安全 | ❌ 不安全 | ❌ 不安全 |
| 可缓存 | ✅ 是 | ⚠️ 条件缓存 | ❌ 否 | ❌ 否 |
| 请求体 | ❌ 无 | ✅ 有 | ✅ 有 | ⚠️ 可有 |
| URL长度限制 | ✅ 有（浏览器约2KB~8KB） | ❌ 无 | ❌ 无 | ❌ 无 |
| 书签/历史 | ✅ 可保存 | ❌ 不可 | ❌ 不可 | ❌ 不可 |
| 浏览器刷新行为 | 直接重发 | 提示重新提交 | — | — |

> **幂等性**：同一操作执行多次结果相同（GET、PUT、DELETE幂等，POST非幂等）。
> **安全性**：操作不改变资源状态（只有GET安全）。

---

### 5. Content-Type 类型

| Content-Type | 用途 | 示例场景 |
|--------------|------|---------|
| `application/json` | JSON数据 | REST API最常见 |
| `application/x-www-form-urlencoded` | URL编码表单 | HTML表单提交 |
| `multipart/form-data` | 多部分数据（文件上传） | 文件+表单混合提交 |
| `text/plain` | 纯文本 | 日志、简单消息 |
| `text/html` | HTML文档 | Web页面 |
| `application/xml` | XML数据 | SOAP/旧系统接口 |
| `application/octet-stream` | 二进制流 | 文件下载 |
| `application/pdf` | PDF文档 | 文件下载 |
| `image/png`, `image/jpeg` | 图片 | 图片资源 |

---

### 6. 认证机制

#### Cookie

```
服务器响应：Set-Cookie: sessionid=abc123; Path=/; HttpOnly; Secure; SameSite=Strict
客户端请求：Cookie: sessionid=abc123
```

| 属性 | 说明 |
|------|------|
| `HttpOnly` | 禁止JavaScript访问，防XSS |
| `Secure` | 仅HTTPS传输 |
| `SameSite=Strict` | 禁止跨站携带，防CSRF |
| `SameSite=Lax` | 允许GET跨站携带 |
| `Max-Age` | Cookie有效期（秒） |
| `Domain` | Cookie生效域名 |
| `Path` | Cookie生效路径 |

#### Session

```
1. 客户端登录 → 服务端创建Session，写入SessionID到Cookie
2. 后续请求携带SessionID → 服务端查Session存储验证身份
3. 服务端存储：Redis/内存/数据库，存储用户状态
```

**测试要点**：Session超时、Session固定攻击、Session劫持、集群Session共享。

#### Token (JWT)

```
Header: {"alg": "HS256", "typ": "JWT"}
Payload: {"sub": "123", "name": "admin", "exp": 1700000000}
Signature: HMACSHA256(base64(header) + "." + base64(payload), secret)
```

最终Token：`eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjMifQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c`

```
请求头：Authorization: Bearer <token>
```

**测试要点**：Token过期、Token伪造、Token泄露、刷新机制、黑名单。

#### 三种机制对比

| 特性 | Cookie | Session | Token (JWT) |
|------|--------|---------|-------------|
| 存储位置 | 客户端 | 服务端 | 客户端 |
| 状态管理 | 无状态 | 有状态 | 无状态 |
| 跨域支持 | 受限 | 受限 | 良好 |
| 扩展性 | 高 | 需共享存储 | 高 |
| 安全性 | 依赖属性配置 | 较高 | 依赖签名算法 |
| 适用场景 | Web应用 | 传统Web | SPA/移动端/微服务 |

---

### 7. HTTPS/TLS 基础

```
TLS握手简化流程：

Client                                    Server
  |                                          |
  |--- ClientHello（支持的加密套件） -------->|
  |<-- ServerHello（选定加密套件） ----------|
  |<-- Certificate（服务器证书） ------------|
  |<-- ServerKeyExchange --------------------|
  |<-- ServerHelloDone ---------------------|
  |--- ClientKeyExchange（预主密钥） ------->|
  |--- ChangeCipherSpec -------------------->|
  |--- Finished ---------------------------->|
  |<-- ChangeCipherSpec --------------------|
  |<-- Finished ----------------------------|
  |                                          |
  |========= 加密通信开始 ===============|
```

**测试要点**：证书过期、域名不匹配、自签名证书、TLS版本（建议TLS 1.2+）、HSTS头部、证书链完整性。

---

### 8. HTTP/2 vs HTTP/3

| 特性 | HTTP/1.1 | HTTP/2 | HTTP/3 |
|------|----------|--------|--------|
| 传输协议 | TCP | TCP | QUIC (UDP) |
| 多路复用 | ❌ 无（队头阻塞） | ✅ 有（Stream） | ✅ 有 |
| 头部压缩 | ❌ 无 | ✅ HPACK | ✅ QPACK |
| 服务器推送 | ❌ 无 | ✅ Server Push | ✅ 已弃用 |
| 连接建立 | TCP三次握手 | TCP+TLS | QUIC（0-RTT/1-RTT） |
| 队头阻塞 | TCP层 | TCP层（存在） | ❌ 无（QUIC解决） |
| 连接迁移 | ❌ 需重连 | ❌ 需重连 | ✅ Connection ID |

---

## 动手实操

### 使用浏览器开发者工具分析HTTP请求

1. 打开 Chrome DevTools → Network 标签
2. 访问 `https://httpbin.org/get`
3. 点击请求，查看 Headers / Preview / Response / Timing 面板
4. 记录：状态码、Content-Type、响应体大小、各阶段耗时

### 构造一个完整HTTP请求

GET请求示例：
```http
GET /api/users?page=1&size=20 HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9
Cache-Control: no-cache
```

POST请求示例：
```http
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9
Content-Length: 45

{"username": "testuser", "email": "test@example.com"}
```

### 状态码测试矩阵

| 测试用例 | 预期状态码 | 验证点 |
|---------|-----------|--------|
| 正常请求 | 200 | 返回正确数据 |
| 创建资源 | 201 | Location头指向新资源 |
| 删除资源 | 204 | 响应体为空 |
| 缺少必填参数 | 400 | 错误信息明确 |
| 未认证访问 | 401 | WWW-Authenticate头 |
| 无权限访问 | 403 | 拒绝访问描述 |
| 资源不存在 | 404 | 路径/ID验证 |
| JSON格式错误 | 400 | 解析错误详情 |
| 参数超出限制 | 422 | 字段级错误 |
| 高频请求 | 429 | Retry-After头 |

---

## 常见坑

1. **混淆401与403**：401表示"未认证"（需要登录），403表示"无权限"（登录了但不够格）
2. **误用GET传递敏感数据**：GET参数在URL中，会被日志/浏览器历史记录
3. **PUT与PATCH混淆**：PUT是全量替换，PATCH是局部更新，接口文档需明确区分
4. **忽视幂等性**：POST重复提交可能创建重复资源（如重复下单）
5. **Cookie的SameSite默认值变更**：Chrome 80+默认`SameSite=Lax`，跨站提交需注意
6. **忽略HTTP/2伪头部**：`:method`、`:path`、`:status`是HTTP/2特有的伪头部字段
7. **文件上传Content-Type错误**：必须使用`multipart/form-data`，boundary需正确设置
8. **ETag与Last-Modified同时存在时**：ETag优先级更高，缓存验证以ETag为准

---

## 自测清单

- [ ] 能否手写一个标准HTTP GET和POST请求报文
- [ ] 能否区分200/201/204在REST API中的使用场景
- [ ] 能否区分400/401/403/404的业务含义
- [ ] 能否解释GET幂等但POST非幂等的原因
- [ ] 能否描述Cookie的HttpOnly、Secure、SameSite三个属性的安全意义
- [ ] 能否描述JWT的三部分结构及签名防篡改原理
- [ ] 能否说明TLS握手的基本步骤
- [ ] 能否说明HTTP/2多路复用解决了什么问题

---

## 延伸阅读

- [RFC 7231 - HTTP/1.1 语义与规范](https://tools.ietf.org/html/rfc7231)
- [RFC 7540 - HTTP/2](https://tools.ietf.org/html/rfc7540)
- [RFC 9114 - HTTP/3](https://tools.ietf.org/html/rfc9114)
- [MDN HTTP 文档](https://developer.mozilla.org/zh-CN/docs/Web/HTTP)
- [MDN HTTP 状态码](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status)
- [OWASP - HTTP安全响应头](https://owasp.org/www-project-secure-headers/)
- [JWT规范 RFC 7519](https://tools.ietf.org/html/rfc7519)
- [MDN Cookie文档](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Cookies)
