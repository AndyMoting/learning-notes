# 03-Postman
> 课时：35 min | 难度：★★

## 安装步骤

### 安装 Postman 桌面客户端

**所有平台：**

1. 访问 <https://www.postman.com/downloads/>
2. 下载对应平台安装包
3. 安装并启动 Postman
4. 创建 Postman 账号或登录（免费账号即可）

**Windows：** 双击 `Postman-Setup.exe` 安装

**macOS：** 将 Postman 拖入 `Applications`

**Linux：**

```bash
sudo snap install postman
```

或下载 `.tar.gz` 解压后运行 `./Postman/Postman`。

### 创建第一个请求

1. 点击 `+` 新建 Request
2. 方法选择 `GET`
3. URL 输入 `https://httpbin.org/get`
4. 点击 `Send`
5. 下方 Body 区域显示 JSON 响应，状态码 `200 OK`

### 创建 Collection

1. 左侧栏点击 `Collections` → `+ New Collection`
2. 命名：`API-Test-Demo`
3. 右键 Collection → `Add Request`，添加多个请求
4. 在 `Tests` 标签页编写断言脚本：

```javascript
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});

pm.test("Response time is less than 500ms", function () {
    pm.expect(pm.response.responseTime).to.be.below(500);
});
```

### 创建 Environment

1. 左侧栏点击 `Environments` → `+ New Environment`
2. 命名：`Dev-Environment`
3. 添加变量：

| Variable | Initial Value        | Current Value        |
|----------|----------------------|----------------------|
| base_url | https://httpbin.org  | https://httpbin.org  |
| token    | abc123               | abc123               |

4. 请求 URL 中使用：`{{base_url}}/get`
5. 右上角环境选择器切换 `Dev-Environment`

### 安装 Newman CLI

**前置条件：** 已安装 Node.js（<https://nodejs.org/>，推荐 LTS 版本）

```bash
npm install -g newman
```

验证安装：

```bash
newman --version
```

**预期输出：**

```
6.1.x
```

### 通过 Newman 运行 Collection

1. 在 Postman 中右键 Collection → `Export` → 选择 `Collection v2.1`，保存为 `collection.json`
2. 导出 Environment 为 `environment.json`
3. 执行：

```bash
newman run collection.json -e environment.json --reporters cli,html --reporter-html-export report.html
```

4. 终端输出测试结果摘要，`report.html` 为 HTML 报告

## 验证安装

```bash
newman run https://www.postman.com/collections/示例ID -e environment.json
```

**预期输出：**

```
Collection Runner

→ GET https://httpbin.org/get
  200 789 ms

┌─────────────────────────┬──────────┬──────────┐
│                         │ executed │   failed │
├─────────────────────────┼──────────┼──────────┤
│              iterations │        1 │        0 │
│                requests │        1 │        0 │
│            test-scripts │        1 │        0 │
│      prerequest-scripts │        0 │        0 │
│              assertions │        2 │        0 │
└─────────────────────────┴──────────┴──────────┘
```

## 常见坑

1. **Newman 未找到命令** — Node.js 未安装或 npm 全局路径未加入 PATH，执行 `npm config get prefix` 确认路径并加入系统 PATH
2. **SSL 证书错误** — 添加 `--insecure` 参数跳过证书验证（仅测试环境）
3. **Collection 导出格式不兼容** — 导出时选择 `Collection v2.1`，v1 格式 Newman 不支持
4. **环境变量未生效** — 确认右上角环境选择器已切换至目标 Environment
5. **Postman 登录后数据丢失** — 登录后 Collection 同步至云端，本地数据不受影响；未登录时数据仅存本地

## 延伸阅读

- <https://www.postman.com/downloads/>
- <https://learning.postman.com/docs/getting-started/introduction/>
- <https://www.npmjs.com/package/newman>
- <https://learning.postman.com/docs/running-collections/intro-to-collection-runs/>
