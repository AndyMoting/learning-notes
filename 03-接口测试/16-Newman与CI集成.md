# 16-Newman与CI集成

> 课时：75 min | 难度：★★★★☆

## 学习目标

- 掌握 Newman 安装与基本命令行操作
- 理解环境变量与全局变量的传递方式
- 能够配置迭代数据文件实现参数化测试
- 生成 HTML 与 JUnit 格式测试报告
- 在 GitHub Actions 中集成 Newman 自动化测试
- 在 Jenkins 中配置 Newman 构建任务
- 实现定时执行与通知机制

## 核心概念

### Newman 定位

Newman 是 Postman 的命令行集合运行器，将 Postman 集合从 GUI 环境迁移到 CI/CD 流水线，实现接口测试自动化。

**核心能力**：

| 能力 | 说明 |
|------|------|
| 集合执行 | 运行导出的 Postman Collection JSON |
| 环境管理 | 注入环境变量、全局变量 |
| 数据驱动 | 支持 CSV/JSON 迭代数据文件 |
| 报告生成 | 输出 HTML、JUnit XML、JSON 等格式 |
| 退出码 | 测试失败时返回非零退出码，触发 CI 失败 |

### 变量传递层级

| 来源 | 优先级 | 适用场景 |
|------|--------|----------|
| 命令行 `--env-var` | 最高 | CI 中动态注入密钥 |
| 环境文件 `-e` | 中 | 预定义环境配置 |
| 全局文件 `-g` | 中 | 跨环境共享变量 |
| 集合内变量 | 低 | 集合默认值 |

### 报告类型

| 格式 | 包名 | 用途 |
|------|------|------|
| CLI 内置 | - | 终端输出，调试用 |
| HTML | newman-reporter-htmlextra | 可视化报告，归档用 |
| JUnit XML | newman-reporter-junit | CI 系统集成，趋势分析 |
| JSON | newman-reporter-json | 程序化解析 |

## 动手实操

### 安装 Newman

```bash
# 全局安装
npm install -g newman

# 验证安装
newman --version

# 安装 HTML 报告器
npm install -g newman-reporter-htmlextra

# 安装 JUnit 报告器
npm install -g newman-reporter-junitfull
```

### 基本运行命令

```bash
# 运行集合
newman run collection.json

# 指定环境
newman run collection.json -e environment.json

# 指定全局变量
newman run collection.json -g globals.json

# 同时指定环境和全局变量
newman run collection.json -e environment.json -g globals.json
```

### 环境变量与全局变量传递

**命令行动态注入**：

```bash
newman run collection.json \
  -e environment.json \
  --env-var "baseUrl=https://api.staging.example.com" \
  --env-var "apiKey=sk-1234567890"
```

**环境文件结构**：

```json
{
  "name": "staging",
  "values": [
    {
      "key": "baseUrl",
      "value": "https://api.staging.example.com",
      "enabled": true
    },
    {
      "key": "apiKey",
      "value": "",
      "enabled": true
    }
  ]
}
```

**全局变量文件结构**：

```json
{
  "values": [
    {
      "key": "clientId",
      "value": "my-app-client-id",
      "enabled": true
    }
  ]
}
```

### 迭代数据文件

**CSV 数据文件**：

```csv
username,password,expected_status,expected_message
admin,admin123,200,登录成功
user1,wrongpass,401,密码错误
locked_user,pass123,403,账号已锁定
```

**JSON 数据文件**：

```json
[
  {
    "username": "admin",
    "password": "admin123",
    "expected_status": 200,
    "expected_message": "登录成功"
  },
  {
    "username": "user1",
    "password": "wrongpass",
    "expected_status": 401,
    "expected_message": "密码错误"
  }
]
```

**运行命令**：

```bash
# 使用 CSV 数据文件
newman run collection.json -d test_data.csv

# 使用 JSON 数据文件
newman run collection.json -d test_data.json

# 指定迭代次数（不依赖数据文件）
newman run collection.json -n 5
```

### HTML 报告生成

```bash
# 基本 HTML 报告
newman run collection.json \
  --reporters cli,htmlextra \
  --reporter-htmlextra-export ./reports/api-report.html

# 自定义报告标题
newman run collection.json \
  --reporters htmlextra \
  --reporter-htmlextra-export ./reports/api-report.html \
  --reporter-htmlextra-title "接口自动化测试报告" \
  --reporter-htmlextra-browserTitle "API Test Report" \
  --reporter-htmlextra-showEnvironmentData \
  --reporter-htmlextra-skipHeaders "Authorization" \
  --reporter-htmlextra-omitHeaders
```

**报告配置参数**：

| 参数 | 说明 |
|------|------|
| `--reporter-htmlextra-export` | 报告输出路径 |
| `--reporter-htmlextra-title` | 报告页面标题 |
| `--reporter-htmlextra-browserTitle` | 浏览器标签标题 |
| `--reporter-htmlextra-showEnvironmentData` | 显示环境变量 |
| `--reporter-htmlextra-skipHeaders` | 跳过的请求头 |
| `--reporter-htmlextra-omitHeaders` | 隐藏所有请求头 |
| `--reporter-htmlextra-logs` | 显示控制台日志 |
| `--reporter-htmlextra-showMarkdownLinks` | 显示 Markdown 链接 |

### JUnit XML 输出

```bash
# 生成 JUnit 报告
newman run collection.json \
  --reporters cli,junit \
  --reporter-junit-export ./reports/api-report.xml

# 使用 junitfull 报告器（更详细）
newman run collection.json \
  --reporters cli,junitfull \
  --reporter-junitfull-export ./reports/api-report-full.xml
```

**JUnit XML 结构**：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<testsuites name="Collection" tests="3" failures="1" errors="0">
  <testsuite name="登录接口" tests="3" failures="1" errors="0">
    <testcase name="状态码为 200" classname="登录接口"/>
    <testcase name="响应体包含 token" classname="登录接口">
      <failure message="expected undefined to deeply equal 'string'"/>
    </testcase>
    <testcase name="响应时间小于 500ms" classname="登录接口"/>
  </testsuite>
</testsuites>
```

### GitHub Actions 集成

**工作流文件**：`.github/workflows/api-test.yml`

```yaml
name: API Interface Test

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # 每天凌晨 2 点执行

jobs:
  api-test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [18.x]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}

      - name: Install Newman
        run: |
          npm install -g newman
          npm install -g newman-reporter-htmlextra
          npm install -g newman-reporter-junitfull

      - name: Run API Tests
        env:
          BASE_URL: ${{ secrets.API_BASE_URL }}
          API_KEY: ${{ secrets.API_KEY }}
        run: |
          newman run postman/collections/api-collection.json \
            -e postman/environments/staging.json \
            -d postman/data/test-data.csv \
            --env-var "baseUrl=${BASE_URL}" \
            --env-var "apiKey=${API_KEY}" \
            --reporters cli,htmlextra,junitfull \
            --reporter-htmlextra-export reports/api-report.html \
            --reporter-junitfull-export reports/api-report.xml \
            --bail \
            --delay-request 100

      - name: Upload Test Report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: api-test-report
          path: reports/
          retention-days: 30

      - name: Publish Test Results
        if: always()
        uses: dorny/test-reporter@v1
        with:
          name: API Test Results
          path: reports/api-report.xml
          reporter: java-junit
          fail-on-error: true
```

**GitHub Secrets 配置**：

1. 仓库 Settings → Secrets and variables → Actions
2. 添加 `API_BASE_URL`、`API_KEY` 等敏感变量

### Jenkins 集成

**Jenkinsfile（Declarative Pipeline）**：

```groovy
pipeline {
    agent any
    
    environment {
        NODE_HOME = tool name: 'NodeJS 18', type: 'jenkins.plugins.nodejs.tools.NodeJSInstallation'
        PATH = "${NODE_HOME}/bin:${env.PATH}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh '''
                    npm install -g newman
                    npm install -g newman-reporter-htmlextra
                    npm install -g newman-reporter-junitfull
                '''
            }
        }
        
        stage('Run API Tests') {
            steps {
                sh '''
                    newman run postman/collections/api-collection.json \
                        -e postman/environments/staging.json \
                        -d postman/data/test-data.csv \
                        --env-var "baseUrl=${API_BASE_URL}" \
                        --env-var "apiKey=${API_KEY}" \
                        --reporters cli,htmlextra,junitfull \
                        --reporter-htmlextra-export reports/api-report.html \
                        --reporter-junitfull-export reports/api-report.xml \
                        --bail
                '''
            }
        }
    }
    
    post {
        always {
            publishHTML(target: [
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'reports',
                reportFiles: 'api-report.html',
                reportName: 'API Test Report'
            ])
            
            junit 'reports/api-report.xml'
        }
        
        failure {
            emailext(
                subject: "API Test Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "测试失败，请查看报告: ${env.BUILD_URL}",
                to: "${env.CHANGE_AUTHOR_EMAIL}"
            )
        }
    }
}
```

**Jenkins 必要插件**：

| 插件 | 用途 |
|------|------|
| NodeJS Plugin | 管理 Node.js 安装 |
| HTML Publisher Plugin | 展示 HTML 报告 |
| JUnit Plugin | 解析 JUnit XML 结果 |
| Email Extension Plugin | 邮件通知 |

### 定时执行与通知

**GitHub Actions 定时触发**：

```yaml
on:
  schedule:
    - cron: '0 8 * * 1-5'  # 工作日每天 8:00
    - cron: '0 0 * * 0'    # 每周日 0:00
```

**Jenkins 定时触发**：

```groovy
triggers {
    cron('H 8 * * 1-5')  // 工作日每天 8:00
}
```

**通知配置**：

| 渠道 | 实现方式 |
|------|----------|
| 邮件 | Jenkins Email Extension / GitHub Actions 邮件步骤 |
| 企业微信 | Webhook 调用企业微信机器人 API |
| Slack | Incoming Webhook |
| 钉钉 | 自定义 Webhook |

**企业微信通知示例**：

```bash
curl 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "msgtype": "markdown",
    "markdown": {
      "content": "## 接口测试结果\n> 状态: 失败\n> 失败用例: 3\n> 报告: [查看详情](https://ci.example.com/report)"
    }
  }'
```

## 常见坑

- **Node.js 版本兼容**：Newman 要求 Node.js >= 16，CI 环境需确认版本
- **报告器未安装**：使用 `--reporters htmlextra` 前必须全局安装对应报告器
- **环境变量覆盖**：`--env-var` 会覆盖 `-e` 环境文件中的同名变量
- **数据文件编码**：CSV 文件在 Windows 上保存时需选择 UTF-8 with BOM 或纯 UTF-8
- **退出码处理**：`--bail` 参数使首次失败即退出，不添加则执行全部用例
- **GitHub Actions Secrets**：Secrets 不会自动传递给 `env` 块，需显式映射
- **Jenkins 工作空间路径**：报告输出路径建议使用绝对路径或 `${WORKSPACE}` 前缀
- **HTML 报告跨域**：GitHub Pages 托管报告时可能遇到 CORS 限制
- **定时任务时区**：GitHub Actions cron 使用 UTC 时间，需换算时区差
- **敏感信息泄露**：报告中可能包含 Token/密钥，使用 `--reporter-htmlextra-skipHeaders` 过滤

## 自测清单

- [ ] 能够通过 npm 全局安装 Newman 及报告器
- [ ] 能够使用 `newman run` 命令运行集合并指定环境文件
- [ ] 能够通过命令行动态注入环境变量
- [ ] 能够配置 CSV/JSON 数据文件实现参数化测试
- [ ] 能够生成 HTML 报告并自定义标题与内容
- [ ] 能够生成 JUnit XML 报告供 CI 系统解析
- [ ] 能够在 GitHub Actions 中配置 Newman 工作流
- [ ] 能够在 Jenkins 中配置 Pipeline 并展示报告
- [ ] 能够配置定时执行与通知机制

## 延伸阅读

- Newman 官方文档：https://github.com/postmanlabs/newman
- Newman 报告器列表：https://github.com/postmanlabs/newman#reporters
- GitHub Actions 文档：https://docs.github.com/en/actions
- Jenkins Pipeline 语法：https://www.jenkins.io/doc/book/pipeline/syntax/
- JUnit XML 格式规范：https://llg.cubic.org/docs/junit/
