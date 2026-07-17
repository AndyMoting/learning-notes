# 14-curl与接口调试

> 课时：40 min | 难度：★★☆☆☆

## 学习目标

- 掌握curl的安装方式及基本语法结构
- 能够使用curl发起GET/POST/PUT/DELETE请求并携带参数、头部、认证信息
- 掌握文件上传、重定向跟随、超时设置、verbose调试等常用选项
- 能将curl命令转换为Python requests代码，加速接口自动化脚本编写
- 能够使用curl高效完成日常接口调试与排错任务

---

## 核心概念

### 1. curl 简介与安装

curl（Client URL）是一个命令行工具，用于通过URL语法传输数据。支持HTTP、HTTPS、FTP、SMTP等协议。

#### 安装方式

| 操作系统 | 命令 |
|---------|------|
| Windows 10+ | 已内置（PowerShell中`curl`为`Invoke-WebRequest`别名，建议使用`curl.exe`） |
| macOS | 已内置，或 `brew install curl` |
| Linux (Debian/Ubuntu) | `sudo apt install curl` |
| Linux (CentOS/RHEL) | `sudo yum install curl` |

验证安装：
```bash
curl --version
```

#### 基本语法

```bash
curl [options] [URL]
```

常用选项前缀：
- 单字符选项：`-X`、`-H`、`-d`（短选项）
- 双字符选项：`--header`、`--data`、`--request`（长选项）

---

### 2. HTTP方法操作

#### GET请求

```bash
# 基础GET
curl https://httpbin.org/get

# 带查询参数
curl "https://httpbin.org/get?name=admin&page=1"

# 带请求头
curl -H "Accept: application/json" https://httpbin.org/get
```

#### POST请求

```bash
# 表单提交（application/x-www-form-urlencoded）
curl -X POST -d "username=admin&password=123456" https://httpbin.org/post

# JSON提交
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"123456"}' \
  https://httpbin.org/post

# 从文件读取请求体
curl -X POST \
  -H "Content-Type: application/json" \
  -d @request_body.json \
  https://httpbin.org/post
```

#### PUT请求

```bash
curl -X PUT \
  -H "Content-Type: application/json" \
  -d '{"name":"updated","status":"active"}' \
  https://httpbin.org/put
```

#### DELETE请求

```bash
curl -X DELETE https://httpbin.org/delete?id=123
```

#### PATCH请求

```bash
curl -X PATCH \
  -H "Content-Type: application/json" \
  -d '{"status":"inactive"}' \
  https://httpbin.org/patch
```

---

### 3. 核心选项详解

| 选项 | 长选项 | 说明 |
|------|--------|------|
| `-X` | `--request` | 指定HTTP方法 |
| `-H` | `--header` | 设置请求头（可多次使用） |
| `-d` | `--data` | 发送请求体（URL编码），`@文件`从文件读取 |
| `-F` | `--form` | multipart/form-data表单（文件上传） |
| `-u` | `--user` | 用户认证（Basic认证） |
| `-o` | `--output` | 输出到文件 |
| `-O` | `--remote-name` | 输出到远程同名文件 |
| `-L` | `--location` | 跟随重定向 |
| `-v` | `--verbose` | 显示详细通信过程 |
| `-i` | `--include` | 输出包含响应头 |
| `-s` | `--silent` | 静默模式（不显示进度条） |
| `-S` | `--show-error` | 配合`-s`使用，出错时显示错误 |
| `-w` | `--write-out` | 输出指定格式信息 |
| `-k` | `--insecure` | 跳过SSL证书验证 |
| `-x` | `--proxy` | 使用代理 |
| `-m` | `--max-time` | 最大超时时间（秒） |
| `-c` | `--cookie-jar` | 保存Cookie到文件 |
| `-b` | `--cookie` | 发送Cookie |
| `-A` | `--user-agent` | 设置User-Agent |
| `-e` | `--referer` | 设置Referer |

---

### 4. 认证操作

#### Bearer Token

```bash
curl -H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..." \
  https://httpbin.org/bearer
```

#### Basic Auth

```bash
# 方式1：-u 自动Base64编码
curl -u admin:123456 https://httpbin.org/basic-auth/admin/123456

# 方式2：手动设置Authorization头
curl -H "Authorization: Basic YWRtaW46MTIzNDU2" \
  https://httpbin.org/basic-auth/admin/123456
```

#### API Key

```bash
# 通过Header传递
curl -H "X-API-Key: your-api-key-here" https://httpbin.org/headers

# 通过URL参数传递
curl "https://httpbin.org/get?api_key=your-api-key-here"
```

---

### 5. 文件上传

```bash
# 单文件上传
curl -F "file=@/path/to/report.pdf" https://httpbin.org/post

# 多文件上传
curl -F "file1=@/path/to/a.jpg" -F "file2=@/path/to/b.png" \
  https://httpbin.org/post

# 文件+表单字段混合
curl -F "file=@/path/to/avatar.png" \
     -F "username=admin" \
     -F "description=头像" \
     https://httpbin.org/post

# 指定MIME类型
curl -F "file=@/path/to/image.png;type=image/png" \
  https://httpbin.org/post
```

---

### 6. 调试与诊断选项

#### Verbose模式

```bash
# 显示完整通信过程（请求头+响应头+TLS信息）
curl -v https://httpbin.org/get
```

输出包含：
- `*` 行：连接信息
- `>` 行：发送的请求头
- `<` 行：接收的响应头
- 额外元数据：TLS版本、证书信息、压缩方式等

#### Trace模式

```bash
# 输出原始字节级通信过程
curl --trace-ascii trace.log https://httpbin.org/get
```

#### 查看仅响应头

```bash
# 只显示响应头（HEAD请求）
curl -I https://httpbin.org/get

# 或使用 -i 包含响应头在输出中
curl -i https://httpbin.org/get
```

#### 超时设置

```bash
# 最大请求时间30秒
curl -m 30 https://httpbin.org/delay/10

# 连接超时10秒，总超时60秒
curl --connect-timeout 10 -m 60 https://httpbin.org/delay/5
```

#### 忽略SSL验证

```bash
curl -k https://self-signed.example.com/api
```

---

### 7. 输出控制

#### 保存到文件

```bash
# 指定文件名
curl -o output.json https://httpbin.org/get

# 保持远程文件名
curl -O https://httpbin.org/image/png
```

#### 静默输出+错误显示

```bash
curl -sS https://httpbin.org/get
```

#### 自定义输出格式

```bash
# 仅显示HTTP状态码
curl -s -o /dev/null -w "%{http_code}" https://httpbin.org/get

# 显示详细时间信息
curl -s -o /dev/null -w "\
DNS解析:     %{time_namelookup}s\n
TCP握手:     %{time_connect}s\n
SSL握手:     %{time_appconnect}s\n
首字节:      %{time_starttransfer}s\n
总耗时:      %{time_total}s\n
下载速度:    %{speed_download} bytes/s\n
HTTP状态码:  %{http_code}\n" \
https://httpbin.org/get
```

#### 常用`-w`变量

| 变量 | 含义 |
|------|------|
| `%{http_code}` | HTTP状态码 |
| `%{http_version}` | HTTP版本 |
| `%{time_total}` | 总耗时（秒） |
| `%{time_namelookup}` | DNS解析耗时 |
| `%{time_connect}` | TCP连接耗时 |
| `%{time_appconnect}` | SSL握手耗时 |
| `%{time_pretransfer}` | 从开始到传输前耗时 |
| `%{time_redirect}` | 重定向耗时 |
| `%{time_starttransfer}` | 首字节时间（TTFB） |
| `%{size_download}` | 下载字节数 |
| `%{size_upload}` | 上传字节数 |
| `%{speed_download}` | 下载速度 |
| `%{url_effective}` | 最终URL（跟随重定向后） |
| `%{content_type}` | Content-Type |

---

### 8. Cookie管理

```bash
# 保存Cookie到文件
curl -c cookies.txt https://httpbin.org/cookies/set?sessionid=abc123

# 发送已保存的Cookie
curl -b cookies.txt https://httpbin.org/cookies

# 同时保存和发送
curl -b cookies.txt -c cookies.txt https://httpbin.org/cookies

# 直接传递Cookie字符串
curl -b "sessionid=abc123; token=xyz789" https://httpbin.org/cookies
```

---

### 9. 代理与高级选项

```bash
# 使用HTTP代理
curl -x http://proxy.example.com:8080 https://httpbin.org/ip

# 使用SOCKS5代理
curl --socks5 127.0.0.1:1080 https://httpbin.org/ip

# 代理认证
curl -x http://proxy:8080 -U user:pass https://httpbin.org/ip

# 上传二进制文件（二进制安全）
curl --data-binary @/path/to/file.bin https://httpbin.org/post

# Gzip压缩请求体
curl -H "Content-Type: application/json" \
     --data-binary @request_body.json \
     --compressed \
     https://httpbin.org/post

# 限制下载速率
curl --limit-rate 100K -O https://httpbin.org/image/png
```

---

## 动手实操

### 10个实用curl测试示例

**示例1：基础GET请求**
```bash
curl https://httpbin.org/get
```

**示例2：带Header和认证的GET**
```bash
curl -H "Accept: application/json" \
     -H "Authorization: Bearer your_token_here" \
     "https://httpbin.org/get?page=1&limit=10"
```

**示例3：JSON POST请求**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your_token_here" \
  -d '{"title":"test","body":"content","userId":1}' \
  https://httpbin.org/posts
```

**示例4：表单数据POST**
```bash
curl -X POST \
  -d "username=admin" \
  -d "password=secret" \
  -d "remember=true" \
  https://httpbin.org/post
```

**示例5：文件上传**
```bash
curl -F "file=@./test.png;type=image/png" \
     -F "description=测试图片" \
     https://httpbin.org/post
```

**示例6：PUT更新资源**
```bash
curl -X PUT \
  -H "Content-Type: application/json" \
  -d '{"id":1,"name":"updated","status":"active"}' \
  https://httpbin.org/put
```

**示例7：DELETE请求**
```bash
curl -X DELETE https://httpbin.org/delete?id=1
```

**示例8：性能测试（获取详细耗时）**
```bash
curl -s -o /dev/null -w "\
HTTP状态码:  %{http_code}
DNS解析:     %{time_namelookup}s
TCP连接:     %{time_connect}s
SSL握手:     %{time_appconnect}s
首字节TTFB:  %{time_starttransfer}s
总耗时:      %{time_total}s
下载速度:    %{speed_download} bytes/s
" https://httpbin.org/get
```

**示例9：跟随重定向并查看完整过程**
```bash
curl -sSL -v -o output.html https://httpbin.org/redirect/3 2>&1 | head -50
```

**示例10：使用Cookie会话保持（登录后操作）**
```bash
# 第一步：登录，保存Cookie
curl -c cookies.txt -X POST \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"123456"}' \
  https://httpbin.org/post

# 第二步：使用Cookie访问需认证接口
curl -b cookies.txt https://httpbin.org/cookies
```

---

## 常见坑

1. **Windows PowerShell中`curl`是别名**：PowerShell的`curl`等同于`Invoke-WebRequest`，语法不同。使用`curl.exe`或直接写`curl.exe`避免歧义。
2. **URL未加引号**：URL含`&`、`?`等特殊字符时，不加引号会导致截断。始终对URL加引号。
3. **JSON请求体未转义**：Shell中双引号内的JSON双引号需转义，或改用单引号包裹。
4. **`-d`会覆盖Content-Type**：`-d`自动设置`application/x-www-form-urlencoded`，发送JSON时需手动`-H "Content-Type: application/json"`。
5. **文件路径的`@`符号**：`-d @file`和`-F "file=@file"`中`@`不能省略，否则会当作文本发送。
6. **`-k`跳过证书验证的风险**：仅用于测试环境，生产环境使用`-k`会遭受中间人攻击。
7. **verbose输出重定向**：`-v`输出到stderr，需`2>&1`才能重定向到文件。
8. **未跟随重定向**：返回3xx时，需加`-L`选项才能获取最终内容。

---

## curl 转 Python requests 对照表

| curl选项 | Python requests 等价写法 |
|---------|------------------------|
| `curl URL` | `requests.get(url)` |
| `curl -X POST URL` | `requests.post(url)` |
| `curl -X PUT URL` | `requests.put(url)` |
| `curl -X DELETE URL` | `requests.delete(url)` |
| `curl -H "Key: Val"` | `headers={"Key": "Val"}` |
| `curl -d "k=v"` | `data={"k": "v"}` |
| `curl -d '{"k":"v"}'` | `json={"k": "v"}` |
| `curl -F "file=@path"` | `files={"file": open("path", "rb")}` |
| `curl -u user:pass` | `auth=HTTPBasicAuth("user", "pass")` |
| `curl -H "Auth: Bearer t"` | `headers={"Authorization": "Bearer t"}` |
| `curl -L` | `allow_redirects=True` |
| `curl -k` | `verify=False` |
| `curl -b "c=v"` | `cookies={"c": "v"}` |
| `curl -m 30` | `timeout=30` |
| `curl -x proxy:port` | `proxies={"http": "proxy:port"}` |

### 转换示例

**curl命令：**
```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer token123" \
  -d '{"username":"admin","email":"admin@test.com"}' \
  -L \
  -m 30 \
  https://httpbin.org/post
```

**Python requests等价代码：**
```python
import requests

url = "https://httpbin.org/post"
headers = {
    "Content-Type": "application/json",
    "Authorization": "Bearer token123"
}
payload = {
    "username": "admin",
    "email": "admin@test.com"
}

response = requests.post(
    url,
    json=payload,
    headers=headers,
    allow_redirects=True,
    timeout=30
)

print(response.status_code)
print(response.json())
```

---

## 自测清单

- [ ] 能否使用curl发起GET请求并携带自定义Header
- [ ] 能否使用curl发送JSON格式的POST请求
- [ ] 能否使用curl上传文件（multipart/form-data）
- [ ] 能否使用curl设置Bearer Token和Basic Auth
- [ ] 能否使用`-w`选项输出HTTP状态码和各阶段耗时
- [ ] 能否使用`-v`选项分析请求/响应头
- [ ] 能否使用`-c`和`-b`管理Cookie会话
- [ ] 能否将常见curl命令转换为Python requests代码

---

## 延伸阅读

- [curl官方文档](https://curl.se/docs/)
- [curl使用指南 (中文)](https://curl.se/docs/manual.html)
- [httpbin.org - HTTP请求测试服务](https://httpbin.org)
- [httpcode - HTTP状态码速查](https://httpcode.info)
- [Requests库官方文档](https://docs.python-requests.org/)
- [curlconverter - curl转各语言代码工具](https://curlconverter.com/)
- [Everything curl - curl电子书](https://curl.se/book.html)
