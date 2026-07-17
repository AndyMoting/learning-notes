# 36-JMeter基础

> 课时：60 min | 难度：★★★☆☆

## 学习目标

- 完成 JMeter 安装与基本配置
- 理解 JMeter 核心组件架构
- 掌握线程组、取样器、监听器的配置方法
- 搭建并执行第一个完整的 HTTP 性能测试计划
- 使用响应断言验证返回结果

## 核心概念

### JMeter 架构与核心组件

JMeter 基于线程组模拟并发用户，通过各类取样器发送请求，监听器收集结果，断言验证正确性。

**核心组件层次：**

```
测试计划（Test Plan）
├── 线程组（Thread Group）—— 模拟并发用户
│   ├── 配置元件（Config Element）—— 参数/Header/Cookie 等配置
│   ├── 计时器（Timer）—— 请求间隔控制
│   ├── 前置处理器（Pre-Processor）—— 请求前处理
│   ├── 取样器（Sampler）—— 发送具体请求（HTTP/JDBC/FTP等）
│   ├── 后置处理器（Post-Processor）—— 提取响应数据
│   ├── 断言（Assertion）—— 验证响应正确性
│   └── 监听器（Listener）—— 收集与展示结果
```

### 线程组（Thread Group）关键参数

| 参数 | 说明 | 典型配置 |
|------|------|----------|
| Number of Threads | 并发线程数（虚拟用户数） | 100 |
| Ramp-up Period | 线程启动时间（秒），所有线程在此时间内均匀启动 | 10（10 秒内启动 100 线程 = 每 0.1s 启动 1 个） |
| Loop Count | 循环次数，勾选 Forever 表示无限循环 | 10 或 Forever + Duration |
| Duration（s） | 测试持续总时长（勾选 Forever 后生效） | 600 |

### HTTP 请求取样器

右键线程组 → Add → Sampler → HTTP Request：

```
名称：用户登录请求
协议：https
服务器名称或 IP：api.example.com
端口号：443
方法：POST
路径：/api/v1/login
内容编码：UTF-8
请求体（Body Data）：
{
  "username": "test_user",
  "password": "test_pass"
}
```

### 响应断言（Response Assertion）

右键 HTTP Request → Add → Assertions → Response Assertion：

```
Apply to: Main sample only
Response Field to Test: Text Response
Pattern Matching Rules: Contains / Equals / Substring
Patterns to Test:
  - "code":200
  - "登录成功"
```

### 监听器

| 监听器 | 用途 |
|--------|------|
| View Results Tree | 查看每个请求的详细请求/响应数据，调试用 |
| View Results in Table | 表格形式查看汇总数据（时间/成功率等） |
| Summary Report | 统计报告（均值/中位数/90%线/错误率） |
| Aggregate Report | 更详细的聚合报告（含 95%/99% 线） |
| Graph Results | 图表可视化 |

## 动手实操：第一个完整测试计划

**步骤 1：安装 JMeter**

```bash
# 下载 JMeter（需 Java 11+）
wget https://dlcdn.apache.org//jmeter/binaries/apache-jmeter-5.6.3.tgz
tar -xzf apache-jmeter-5.6.3.tgz
cd apache-jmeter-5.6.3/bin

# 启动 GUI 模式
./jmeter.sh  # Linux/Mac
jmeter.bat   # Windows
```

**步骤 2：创建测试计划**

```
1. 新建测试计划（File → New）
2. 测试计划右键 → Add → Threads → Thread Group
```

**线程组配置：**

```
Number of Threads: 50
Ramp-up Period: 5
Loop Count: 5
```

**添加 HTTP 请求默认值（复用配置）：**

```
Test Plan → Add → Config Element → HTTP Request Defaults
协议: https
服务器: reqres.in
```

**添加 HTTP 请求：**

```
Thread Group → Add → Sampler → HTTP Request
名称: GET 用户列表
方法: GET
路径: /api/users?page=2
```

**添加断言：**

```
HTTP Request → Add → Assertions → Response Assertion
字段: Text Response
规则: Contains
模式: "first_name"
```

**添加监听器：**

```
Thread Group → Add → Listener → View Results Tree
Thread Group → Add → Listener → Summary Report
```

**步骤 3：执行与验证**

```
1. 保存测试计划（Ctrl+S）
2. 点击绿色运行按钮或 Ctrl+R
3. View Results Tree 中查看每条请求详情
4. 绿色 = 通过，红色 = 断言失败
5. Summary Report 查看聚合数据
```

**步骤 4：添加多个请求模拟完整流程**

```
Thread Group
├── HTTP Request: GET /api/users?page=1
├── HTTP Request: GET /api/users/2
├── HTTP Request: POST /api/users  (Body: {"name":"test","job":"qa"})
└── HTTP Request: DELETE /api/users/2
```

## 常见坑

- GUI 模式仅用于调试：正式压测必须使用 CLI 模式，GUI 消耗大量资源
- Ramp-up 为 0：所有线程瞬间启动，产生不真实的流量尖峰
- 未设置思考时间（Think Time）：真实用户操作间有停顿，测试应模拟
- 监听器过多：每个监听器都消耗内存，正式运行时移除 View Results Tree
- 未配置超时：网络异常时线程无限等待导致压测卡死

## 自测清单

- [ ] 能解释 JMeter 八大核心组件的作用与层级关系
- [ ] 能为 50 用户阶梯启动 10 秒配置线程组参数
- [ ] 能搭建包含默认值 + 多请求 + 断言 + 监听器的测试计划
- [ ] 能区分 View Results Tree（调试）和正式压测的监听器选择
- [ ] 能列举 3 个 JMeter GUI 模式不适合正式压测的原因

## 延伸阅读

- JMeter 官方组件参考：https://jmeter.apache.org/usermanual/component_reference.html
- JMeter 最佳实践：https://jmeter.apache.org/usermanual/best-practices.html
