# 15-Postman操作

> 课时：90 min | 难度：★★★☆☆

## 学习目标

- 掌握 Postman 界面核心区域的功能与交互逻辑
- 能够独立创建并配置 GET/POST/PUT/DELETE 请求
- 理解变量作用域层级并正确使用环境变量
- 编写 Tests 脚本实现接口断言与自动化验证
- 使用 Pre-request Script 生成动态请求数据
- 通过 Collection Runner 执行批量测试
- 实现数据驱动测试（CSV/JSON）
- 搭建 Mock Server 模拟接口响应
- 通过 Newman CLI 在命令行执行集合

## 核心概念

### 界面结构

| 区域 | 功能 |
|------|------|
| Builder | 请求构造区，配置 URL、方法、Headers、Body、Pre-request Script、Tests |
| Console | 底部面板，查看请求/响应原始数据、网络耗时、日志输出 |
| History | 左侧面板，按时间记录所有历史请求 |
| Collection | 左侧面板，组织请求的文件夹结构，支持嵌套与标签 |

### 请求方法

| 方法 | 用途 | Body 支持 |
|------|------|-----------|
| GET | 查询资源 | 否 |
| POST | 创建资源 | 是 |
| PUT | 全量更新资源 | 是 |
| PATCH | 部分更新资源 | 是 |
| DELETE | 删除资源 | 可选 |

### Headers 管理

- **全局级别**：Manage Environments → Global Variables，所有请求生效
- **请求级别**：Builder → Headers 标签，仅当前请求生效
- **集合级别**：Collection → Headers，集合内所有请求继承
- **优先级**：请求级 > 集合级 > 全局级

### 变量作用域

| 类型 | 定义位置 | 作用范围 | 语法 |
|------|----------|----------|------|
| Global Variable | Manage Environments → Globals | 整个工作区 | `{{variableName}}` |
| Environment Variable | Manage Environments → 具体环境 | 当前激活环境 | `{{variableName}}` |
| Collection Variable | Collection → Variables | 集合内所有请求 | `{{variableName}}` |
| Local Variable | Pre-request Script / Tests 中 `pm.variables.set()` | 当前请求周期 | `{{variableName}}` |

**优先级**：Local > Environment > Collection > Global

### Body 类型

| 类型 | Content-Type | 适用场景 |
|------|--------------|----------|
| none | - | GET/DELETE 无 Body |
| form-data | multipart/form-data | 文件上传、表单 |
| x-www-form-urlencoded | application/x-www-form-urlencoded | 普通表单提交 |
| raw (JSON) | application/json | REST API 主流格式 |
| raw (XML) | application/xml | SOAP/旧系统 |
| binary | application/octet-stream | 单文件上传 |
| GraphQL | application/json | GraphQL 查询 |

## 动手实操

### 创建 GET 请求

```
GET https://api.example.com/users?page=1&limit=10
Headers:
  Authorization: Bearer {{token}}
  Accept: application/json
```

### 创建 POST 请求

```
POST https://api.example.com/users
Headers:
  Content-Type: application/json
  Authorization: Bearer {{token}}
Body (raw → JSON):
{
  "name": "张三",
  "email": "zhangsan@example.com",
  "role": "admin"
}
```

### 创建 PUT 请求

```
PUT https://api.example.com/users/123
Headers:
  Content-Type: application/json
Body (raw → JSON):
{
  "name": "李四",
  "email": "lisi@example.com"
}
```

### 创建 DELETE 请求

```
DELETE https://api.example.com/users/123
Headers:
  Authorization: Bearer {{token}}
```

### Tests 脚本

Tests 脚本在请求响应返回后执行，用于断言验证。

**基本断言结构**：

```javascript
pm.test("状态码为 200", function () {
    pm.response.to.have.status(200);
});

pm.test("响应体包含预期字段", function () {
    const jsonData = pm.response.json();
    pm.expect(jsonData.name).to.eql("张三");
    pm.expect(jsonData.id).to.be.a('number');
});

pm.test("响应时间小于 500ms", function () {
    pm.expect(pm.response.responseTime).to.be.below(500);
});
```

**常用断言方法**：

| 断言 | 说明 |
|------|------|
| `pm.response.to.have.status(code)` | 验证状态码 |
| `pm.response.to.have.statusReason(reason)` | 验证状态描述 |
| `pm.response.to.have.header(key)` | 验证响应头存在 |
| `pm.response.to.have.header(key, value)` | 验证响应头值 |
| `pm.response.to.be.ok` | 状态码 200-299 |
| `pm.response.to.be.json` | 响应体为 JSON |
| `pm.response.to.have.body(string)` | 响应体包含字符串 |
| `pm.expect(value).to.eql(expected)` | 深度相等 |
| `pm.expect(value).to.be.a(type)` | 类型检查 |
| `pm.expect(value).to.be.above(n)` | 大于 |
| `pm.expect(value).to.be.below(n)` | 小于 |
| `pm.expect(array).to.include(value)` | 包含 |
| `pm.expect(value).to.be.true/false` | 布尔断言 |

**提取响应数据到变量**：

```javascript
// 提取 JSON 字段到环境变量
const jsonData = pm.response.json();
pm.environment.set("userId", jsonData.id);
pm.environment.set("authToken", jsonData.token);

// 提取响应头
const requestId = pm.response.headers.get("X-Request-Id");
pm.environment.set("requestId", requestId);
```

### Pre-request Script

Pre-request Script 在请求发送前执行，用于动态生成数据。

**生成时间戳**：

```javascript
const timestamp = Date.now();
pm.environment.set("timestamp", timestamp);
```

**生成随机数据**：

```javascript
const randomEmail = `user${Date.now()}@test.com`;
pm.environment.set("randomEmail", randomEmail);

const randomName = pm.variables.replaceIn('{{$randomFullName}}');
pm.environment.set("randomName", randomName);
```

**生成签名**：

```javascript
const crypto = require('crypto-js');
const secret = pm.environment.get("apiSecret");
const timestamp = Date.now().toString();
const signature = crypto.HmacSHA256(timestamp, secret).toString();
pm.environment.set("signature", signature);
pm.environment.set("timestamp", timestamp);
```

**动态修改请求体**：

```javascript
const body = {
    "orderId": pm.variables.replaceIn('{{$guid}}'),
    "amount": Math.floor(Math.random() * 1000) + 1,
    "timestamp": Date.now()
};
pm.environment.set("requestBody", JSON.stringify(body));
```

### Collection Runner

Collection Runner 用于批量执行集合中的请求。

**操作步骤**：

1. 点击 Collection → Run 按钮
2. 选择要执行的请求（可勾选子集）
3. 配置参数：
   - **Iterations**：迭代次数
   - **Delay**：每次请求间隔（ms）
4. 选择环境（Environment）
5. 上传数据文件（Data）
6. 点击 Run 执行

**执行结果**：

| 列 | 说明 |
|----|------|
| Pass | 通过的断言数 |
| Fail | 失败的断言数 |
| Total | 总请求数 |
| Iteration | 当前迭代轮次 |

### 数据驱动测试

**CSV 数据文件**：

```csv
username,password,expected_status
admin,admin123,200
user1,pass404,401
locked_user,pass123,403
```

**JSON 数据文件**：

```json
[
  {
    "username": "admin",
    "password": "admin123",
    "expected_status": 200
  },
  {
    "username": "user1",
    "password": "pass404",
    "expected_status": 401
  }
]
```

**在请求中引用数据变量**：

```
POST https://api.example.com/login
Body:
{
  "username": "{{username}}",
  "password": "{{password}}"
}
```

**Tests 中验证数据驱动结果**：

```javascript
const expectedStatus = pm.iterationData.get("expected_status");
pm.test(`状态码为 ${expectedStatus}`, function () {
    pm.response.to.have.status(expectedStatus);
});
```

### Mock Server

Mock Server 用于在后端接口未就绪时模拟响应。

**创建步骤**：

1. 创建 Collection，添加请求
2. 为每个请求添加 Example（Save Response → Save as example）
3. 点击 Collection → Mock Collection → Create Mock Server
4. 获取 Mock URL
5. 客户端使用 Mock URL 替代真实接口

**Example 配置**：

```
GET /users/123
Status: 200
Headers:
  Content-Type: application/json
Body:
{
  "id": 123,
  "name": "Mock User",
  "email": "mock@example.com"
}
```

### Newman CLI 集成

Newman 是 Postman 的命令行集合运行器。

**安装**：

```bash
npm install -g newman
```

**基本运行**：

```bash
newman run collection.json
```

**常用参数**：

| 参数 | 说明 |
|------|------|
| `-e, --environment` | 指定环境文件 |
| `-g, --globals` | 指定全局变量文件 |
| `-d, --iteration-data` | 指定迭代数据文件 |
| `-n, --iteration-count` | 迭代次数 |
| `--delay-request` | 请求间隔（ms） |
| `--bail` | 遇到失败立即停止 |
| `--reporters` | 指定报告器 |
| `--reporter-htmlextra-export` | HTML 报告输出路径 |
| `--reporter-junit-export` | JUnit XML 输出路径 |

**导出集合**：

1. Postman → Collection → Export → Collection v2.1
2. 保存为 JSON 文件

**导出环境**：

1. Manage Environments → 点击环境名 → Download
2. 保存为 JSON 文件

## 常见坑

- **变量未生效**：检查环境是否激活（右上角环境选择器），变量名拼写是否一致
- **Body 格式错误**：raw 模式下必须手动设置 Content-Type 为 application/json
- **Tests 脚本执行顺序**：Tests 在响应返回后执行，无法修改当前请求，只能修改后续请求
- **Pre-request Script 中修改 Body**：需将 Body 设为 raw 并使用 `{{requestBody}}` 占位符
- **CSV 编码问题**：CSV 文件必须保存为 UTF-8 编码，否则中文乱码
- **数据文件路径**：Newman 中 `-d` 参数路径需使用绝对路径或相对于执行目录的路径
- **Mock Server 响应头**：Example 中需手动添加 Content-Type，否则默认 text/plain
- **Collection Runner 迭代变量**：迭代数据中的变量名需与请求中 `{{}}` 内的名称完全匹配
- **环境变量持久化**：通过脚本 `pm.environment.set()` 设置的变量默认不持久化，需手动保存
- **Newman 报告器缺失**：使用 HTML 报告需额外安装 `npm install -g newman-reporter-htmlextra`

## 自测清单

- [ ] 能够独立创建包含 Headers 和 Body 的 POST 请求
- [ ] 能够区分 Global/Environment/Collection 变量的作用域
- [ ] 能够编写 Tests 脚本验证状态码、响应体字段、响应时间
- [ ] 能够在 Pre-request Script 中生成随机数据和签名
- [ ] 能够通过 Collection Runner 执行批量请求并查看结果
- [ ] 能够使用 CSV 文件实现数据驱动测试
- [ ] 能够创建 Mock Server 并配置 Example 响应
- [ ] 能够通过 Newman CLI 运行集合并生成 HTML 报告

## 延伸阅读

- Postman 官方学习中心：https://learning.postman.com/
- Postman Sandbox API 参考：https://learning.postman.com/docs/writing-scripts/postman-sandbox-api-reference/
- Newman 官方文档：https://github.com/postmanlabs/newman
- Newman htmlextra 报告器：https://github.com/newman-reporter/htmlextra
