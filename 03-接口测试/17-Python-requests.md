# 17-Python-requests

> 课时：45 min | 难度：★★☆☆☆

## 学习目标

- 理解 requests 库的设计哲学与 HTTP 协议映射关系
- 掌握 GET/POST/PUT/DELETE 四种核心方法的调用方式
- 区分 params、json、data、headers 四种参数的使用场景
- 使用 Session 对象管理会话状态
- 配置超时、重试、SSL 验证等工程化参数
- 完成文件上传与下载操作
- 独立编写完整的接口测试脚本

## 核心概念

### 17.1 安装与基本理念

requests 是 Python 生态中最广泛使用的 HTTP 库，基于 urllib3 构建，提供简洁的 API 封装。

```bash
pip install requests
```

核心设计原则：每个 HTTP 动词对应一个同名函数，URL 作为第一个参数，其余配置通过关键字参数传入。

```python
import requests

response = requests.get("https://httpbin.org/get")
print(response.status_code)  # 200
print(response.text)         # 响应体字符串
```

### 17.2 HTTP 方法映射

| HTTP 方法 | requests 函数 | 幂等性 | 典型用途 |
|-----------|--------------|--------|---------|
| GET | `requests.get()` | 是 | 查询资源 |
| POST | `requests.post()` | 否 | 创建资源 |
| PUT | `requests.put()` | 是 | 全量更新 |
| PATCH | `requests.patch()` | 否 | 局部更新 |
| DELETE | `requests.delete()` | 是 | 删除资源 |

```python
base_url = "https://httpbin.org"

requests.get(f"{base_url}/get")
requests.post(f"{base_url}/post")
requests.put(f"{base_url}/put")
requests.delete(f"{base_url}/delete")
```

### 17.3 请求参数类型

四种参数类型对应不同的 HTTP 请求部位：

**params** — 拼接至 URL 查询字符串

```python
response = requests.get(
    "https://httpbin.org/get",
    params={"page": 1, "size": 20}
)
# 实际请求 URL: https://httpbin.org/get?page=1&size=20
```

**json** — 请求体为 JSON，自动设置 `Content-Type: application/json`

```python
response = requests.post(
    "https://httpbin.org/post",
    json={"username": "alice", "password": "secret"}
)
```

**data** — 请求体为表单编码，`Content-Type: application/x-www-form-urlencoded`

```python
response = requests.post(
    "https://httpbin.org/post",
    data={"username": "alice", "password": "secret"}
)
```

**headers** — 自定义请求头

```python
response = requests.get(
    "https://httpbin.org/get",
    headers={"Authorization": "Bearer <token>", "Accept": "application/json"}
)
```

### 17.4 响应处理

```python
response = requests.get("https://httpbin.org/json")

# 状态码
response.status_code        # 200
response.ok                 # True (status_code < 400)
response.reason             # "OK"

# 响应体
response.text               # 字符串（自动解码）
response.json()             # 解析为 dict/list
response.content            # bytes（原始二进制）

# 响应头
response.headers            # CaseInsensitiveDict
response.headers["Content-Type"]

# 请求元信息
response.url                # 最终请求 URL（含重定向后）
response.encoding           # 编码方式
response.elapsed            # timedelta，请求耗时
response.request.headers    # 实际发送的请求头
```

### 17.5 Session 对象

Session 在多次请求间保持 Cookie、连接池和默认配置，避免重复设置。

```python
session = requests.Session()
session.headers.update({"User-Agent": "test-suite/1.0"})
session.verify = False  # 全局关闭 SSL 验证（仅测试环境）

# 登录后 Cookie 自动携带
session.post("https://api.example.com/login", json={"user": "admin", "pwd": "123"})
resp = session.get("https://api.example.com/profile")  # 自动携带 session cookie
```

### 17.6 超时与重试

**超时**：防止请求无限阻塞。

```python
# 连接超时 3s，读取超时 10s
requests.get("https://api.example.com/data", timeout=(3, 10))

# 统一超时 5s
requests.get("https://api.example.com/data", timeout=5)
```

**重试**：通过 `HTTPAdapter` 配置重试策略。

```python
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

session = requests.Session()
retry_strategy = Retry(
    total=3,
    backoff_factor=1,           # 重试间隔：1s, 2s, 4s
    status_forcelist=[500, 502, 503, 504],
    allowed_methods=["GET", "POST"]
)
adapter = HTTPAdapter(max_retries=retry_strategy)
session.mount("https://", adapter)
session.mount("http://", adapter)
```

### 17.7 SSL 验证

```python
# 关闭 SSL 验证（仅测试环境，生产环境禁止）
requests.get("https://self-signed.example.com", verify=False)

# 使用自定义 CA 证书
requests.get("https://internal.example.com", verify="/path/to/ca-bundle.crt")
```

关闭验证时 urllib3 会输出 `InsecureRequestWarning`，可静默处理：

```python
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
```

### 17.8 文件上传与下载

**上传**：

```python
with open("report.pdf", "rb") as f:
    response = requests.post(
        "https://api.example.com/upload",
        files={"file": ("report.pdf", f, "application/pdf")},
        data={"category": "monthly"}
    )
```

**下载**：

```python
response = requests.get("https://api.example.com/export/data.csv", stream=True)
with open("data.csv", "wb") as f:
    for chunk in response.iter_content(chunk_size=8192):
        f.write(chunk)
```

`stream=True` 避免大文件一次性载入内存，`iter_content` 按块迭代写入磁盘。

## 动手实操

### 完整示例：登录 → 创建资源 → 验证 → 清理

```python
import requests
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

BASE_URL = "https://api.example.com"

def main():
    session = requests.Session()
    session.verify = False
    session.headers.update({"Accept": "application/json"})

    # 1. 登录
    login_resp = session.post(
        f"{BASE_URL}/auth/login",
        json={"username": "tester", "password": "pass123"},
        timeout=10
    )
    assert login_resp.status_code == 200, f"登录失败: {login_resp.text}"
    token = login_resp.json()["data"]["token"]
    session.headers.update({"Authorization": f"Bearer {token}"})

    # 2. 创建资源
    create_resp = session.post(
        f"{BASE_URL}/articles",
        json={"title": "测试文章", "content": "正文内容"},
        timeout=10
    )
    assert create_resp.status_code == 201, f"创建失败: {create_resp.text}"
    article_id = create_resp.json()["data"]["id"]

    # 3. 验证资源存在
    get_resp = session.get(f"{BASE_URL}/articles/{article_id}", timeout=10)
    assert get_resp.status_code == 200
    assert get_resp.json()["data"]["title"] == "测试文章"

    # 4. 清理
    del_resp = session.delete(f"{BASE_URL}/articles/{article_id}", timeout=10)
    assert del_resp.status_code == 204

    print("全部用例通过")

if __name__ == "__main__":
    main()
```

## 常见坑

1. **json vs data 混用**：`json=` 参数自动序列化并设置 `Content-Type: application/json`；`data=` 发送表单编码。后端期望 JSON 时使用 `data=` 会导致 400 错误。
2. **忘记处理编码**：`response.text` 依赖 `response.encoding`，部分接口未声明编码时可能乱码。可手动设置 `response.encoding = "utf-8"`。
3. **未设置超时**：生产脚本必须设置 `timeout`，否则网络异常时进程永久阻塞。
4. **Session 未复用**：高频请求场景下每次新建 `Session` 无法复用 TCP 连接，性能下降显著。
5. **SSL 验证关闭进入生产**：`verify=False` 仅用于测试环境，生产环境必须验证证书。
6. **文件上传未使用 `with` 语句**：文件句柄未及时关闭会导致资源泄漏。
7. **忽略 `response.raise_for_status()`**：建议在关键请求后调用，自动抛出 4xx/5xx 异常。

## 自测清单

- [ ] 能独立安装 requests 库并发送第一个 GET 请求
- [ ] 能区分 params、json、data 的使用场景
- [ ] 能使用 Session 保持登录状态
- [ ] 能为请求配置超时和重试策略
- [ ] 能处理文件上传与下载
- [ ] 能解读响应对象的全部关键属性
- [ ] 能编写完整的 CRUD 测试流程

## 延伸阅读

- requests 官方文档：[https://docs.python-requests.org/](https://docs.python-requests.org/)
- urllib3 Retry 文档：[https://urllib3.readthedocs.io/en/stable/reference/urllib3.util.html](https://urllib3.readthedocs.io/en/stable/reference/urllib3.util.html)
- HTTP 状态码规范：[RFC 9110](https://httpwg.org/specs/rfc9110.html#status.codes)
