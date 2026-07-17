# 37-JMeter进阶

> 课时：75 min | 难度：★★★★☆

## 学习目标

- 掌握参数化技术（CSV 数据文件、随机变量）
- 理解关联（Correlation）的概念与实现方式
- 使用 JSON 提取器处理 API 响应
- 使用事务控制器和吞吐量控制器组织测试逻辑
- 理解各类定时器的应用场景

## 核心概念

### 参数化（Parameterization）

同一接口使用不同数据请求，避免缓存命中和数据重复。

**CSV Data Set Config：**

```
Thread Group → Add → Config Element → CSV Data Set Config

文件名: users.csv
文件编码: UTF-8
变量名: username,password
分隔符: ,
遇到文件结束符时: EOF 时循环 / EOF 时停止 / 结束线程
```

users.csv 内容：

```
user1,pass123
user2,pass456
user3,pass789
```

HTTP 请求中引用：`${username}` / `${password}`

### 关联（Correlation）

提取接口响应中的动态数据，用于后续请求。

**正则表达式提取器：**

```
HTTP Request → Add → Post Processor → Regular Expression Extractor

引用名称: token
正则表达式: "token":"(.+?)"
模板: $1$
匹配数字: 1（第1个匹配）
缺省值: NOT_FOUND
```

**JSON 提取器（JMeter Plugins 或 5.0+ 内置）：**

```
HTTP Request → Add → Post Processor → JSON Extractor

变量名: userId
JSON Path 表达式: $.data.id
默认值: 0
```

**JSON Path Assertion：**

```
HTTP Request → Add → Assertions → JSON Assertion

JSON Path: $.code
Expected Value: 200
```

### 事务控制器（Transaction Controller）

将多个请求合并为一个事务统计，模拟完整业务流程：

```
Transaction Controller（生成样本）: 结账流程
├── HTTP Request: 浏览商品
├── HTTP Request: 加入购物车
├── HTTP Request: 提交订单
└── HTTP Request: 完成支付
```

勾选 "Generate parent sample" 后，父样本显示整体耗时，子样本显示各步骤耗时。

### 吞吐量控制器（Throughput Controller）

控制子节点执行比例（按百分比或总执行次数）：

```
通过百分比执行: 60%（占父样本总执行次数的 60%）
通过总执行次数: 10（整个测试中该分支执行 10 次）
```

### 定时器（Timers）

| 定时器 | 公式/行为 | 用途 |
|--------|----------|------|
| Constant Timer | 固定延迟 N 毫秒 | 模拟统一思考时间 |
| Uniform Random Timer | [最小值, 最大值] 均匀分布 | 模拟随机停顿 |
| Gaussian Random Timer | 正态分布，偏差值 + 延迟 | 模拟正态分布的思考时间 |
| Poisson Random Timer | 泊松分布 | 模拟真实用户到达模式 |
| Synchronizing Timer | 等待 N 个线程到达后同时释放 | 制造精确的并发尖峰 |

**Synchronizing Timer 配置：**

```
线程组: 100 用户
  Synchronizing Timer
    同步线程数: 100
    超时时间(ms): 5000
  HTTP Request: 秒杀接口
→ 100 个线程同时释放，模拟秒杀场景
```

## 动手实操：完整 API 测试计划

**场景：用户注册 → 登录 → 查看个人信息**

```bash
# 参数化数据文件: users.csv
# 内容:
# user001@mail.com,Pass123!
# user002@mail.com,Pass456!
```

**测试计划结构：**

```
Test Plan
├── Thread Group（50 用户，Ramp-up 10s，循环 3 次）
│   ├── CSV Data Set Config
│   │     文件名: users.csv
│   │     变量: email,password
│   ├── HTTP Request Defaults
│   │     服务器: reqres.in
│   ├── Transaction Controller: 注册流程
│   │   ├── HTTP Request: POST /api/register
│   │   │     Body: {"email":"${email}","password":"${password}"}
│   │   └── JSON Assertion: $.token 存在
│   ├── Transaction Controller: 登录流程
│   │   ├── Constant Timer: 1000ms
│   │   ├── HTTP Request: POST /api/login
│   │   │     Body: {"email":"${email}","password":"${password}"}
│   │   ├── JSON Extractor:
│   │   │     变量名: authToken
│   │   │     JSON Path: $.token
│   │   └── Response Assertion: "token" NotNull
│   ├── Transaction Controller: 获取用户信息
│   │   ├── Gaussian Random Timer: 偏差 300ms, 延迟 1000ms
│   │   ├── HTTP Header Manager
│   │   │     Authorization: Bearer ${authToken}
│   │   └── HTTP Request: GET /api/users/2
│   └── View Results Tree
```

**Synchronizing Timer 秒杀场景配置：**

```
Thread Group
  Number of Threads: 200
  Ramp-up: 20
  Synchronizing Timer
    同步线程数: 200
    超时: 3000
  HTTP Request: POST /api/seckill
  Response Assertion: "message" Contains "success"
→ 200 线程等待到齐后同时发出秒杀请求
```

## 常见坑

- CSV 中文乱码：文件保存为 UTF-8 无 BOM 格式，配置文件中勾选 UTF-8
- 正则提取贪婪匹配：`.+` 匹配过多内容，使用 `.+?` 非贪婪模式
- JSON Path 表达式错误：`$.data.array[0].id` 注意数组索引从 0 开始
- 定时器作用域：定时器对其下所有取样器生效，注意层级位置
- Synchronizing Timer 超时：线程数大于实际线程时永远等不到触发

## 自测清单

- [ ] 能配置 CSV Data Set Config 实现参数化并处理 EOF 策略
- [ ] 能用正则表达式提取器和 JSON 提取器完成关联
- [ ] 能用事务控制器聚合多请求的响应时间
- [ ] 能为秒杀场景配置 Synchronizing Timer
- [ ] 能区分四种定时器的数学分布特征与适用场景

## 延伸阅读

- JMeter JSON Path 插件：https://jmeter-plugins.org/wiki/JSONPathExtractor/
- JSON Path 语法指南：https://goessner.net/articles/JsonPath/
