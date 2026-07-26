# 72-JMeter竞赛级
> 课时：50 min | 难度：★★★★

## 学习目标
- 掌握 JMeter 竞赛级测试计划完整搭建流程
- 理解参数化、关联、场景设计三大核心技能
- 学会非 GUI 模式执行与 Dashboard 报告生成

## 核心概念

### JMeter 任务流程

```
脚本录制/手写 → 参数化 → 关联 → 断言 → 场景设计 → 执行 → 报告
```

### 核心组件配置

**事务控制器（Transaction Controller）**
- 位置：线程组 → 逻辑控制器 → 事务控制器
- 作用：将多个请求打包为一个事务，统计整体响应时间
- 配置：勾选"Generate parent sample"

**响应断言（Response Assertion）**
- 位置：请求 → 断言 → 响应断言
- 作用：验证响应内容是否包含预期字符串
- 配置：选择"Response Text"，添加模式字符串

**集合点（Synchronizing Timer）**
- 位置：线程组 → 定时器 → Synchronizing Timer
- 作用：模拟真实并发，所有虚拟用户到达后同时发起请求
- 配置：Number of Simulated Users to Group by = 并发数

### 参数化 4 种类型

| 类型 | 组件 | 适用场景 | 配置要点 |
|------|------|---------|---------|
| 唯一 ID | Counter | 自增编号 | Start/Increment/Maximum/引用名 |
| VUser ID | User Defined Variables | 每用户固定值 | 变量名=值，引用 ${变量名} |
| 随机数 | Random Variable | 随机输入 | Min/Max/引用名 |
| 文件 | CSV Data Set Config | 大量测试数据 | Filename/Variable Names/Delimiter |

**CSV Data Set Config 配置：**

```
Filename: data.csv
Variable Names: username,password
Delimiter: ,
Recycle on EOF: True
Stop thread on EOF: False
Sharing mode: All threads
```

### 关联（Correlation）

**JMeter 正则表达式提取器：**

```
位置：请求 → 后置处理器 → Regular Expression Extractor
Reference Name: token
Regular Expression: "token":"(.+?)"
Template: $1$
Match No.: 1
Default Value: NOT_FOUND
```

**后续请求引用：**

```
Header: Authorization: Bearer ${token}
```

**与 LoadRunner 对比：**

| 功能 | LoadRunner | JMeter |
|------|-----------|--------|
| 关联函数 | web_reg_save_param_ex | Regular Expression Extractor |
| 边界左边界 | LB= | 正则左匹配 |
| 边界右边界 | RB= | 正则右匹配 |
| 引用 | {变量名} | ${变量名} |

### 场景设计

**用户组配置：**

| 场景 | 线程数 | Ramp-Up | 循环次数 |
|------|--------|---------|---------|
| 负载测试 | 100 | 10s | 10 |
| 压力测试 | 200 | 5s | 无限 |
| 基准测试 | 1 | 1s | 20 |

**Run-time Settings：**
- 线程数（Number of Threads）：虚拟用户数
- Ramp-Up Period：启动全部用户所需时间（秒）
- Loop Count：每个用户执行次数

### 非 GUI 模式执行

```bash
# 执行测试
jmeter -n -t test_plan.jmx -l result.jtl

# 生成 Dashboard 报告
jmeter -g result.jtl -o report_folder

# 完整命令（含日志）
jmeter -n -t test_plan.jmx -l result.jtl -e -o report_folder -j jmeter.log
```

**参数说明：**
- `-n`：非 GUI 模式
- `-t`：测试计划文件
- `-l`：结果文件
- `-g`：从结果文件生成报告
- `-o`：报告输出目录（必须为空）
- `-e`：执行后生成报告
- `-j`：日志文件

### Dashboard 报告截图要求

| 截图项 | 内容 |
|--------|------|
| Statistics | 请求数、平均值、中位数、90%线、错误率 |
| Errors | 错误类型与数量 |
| Throughput | 每秒请求数 |
| Response Times | 响应时间分布 |

## 动手实操

1. 使用 JMeter 录制或手写一个登录-查询-退出脚本
2. 添加 CSV Data Set Config 实现用户名密码参数化
3. 添加正则表达式提取器实现 Token 关联
4. 配置事务控制器、响应断言、集合点
5. 设计负载场景（100 用户、10s Ramp-Up、循环 10 次）
6. 非 GUI 模式执行并生成 Dashboard 报告
7. 截图 Statistics、Errors、Throughput 三个面板

## 常见坑
- 参数化文件路径使用绝对路径——使用相对路径或 JMeter  bin 目录
- 正则表达式提取器放错位置——必须作为子节点放在目标请求下
- 报告输出目录非空——-o 指定的目录必须为空或不存在
- 忘记添加 HTTP Cookie Manager——需要会话保持时必须添加
- 线程组 Ramp-Up 为 0——会导致瞬时压力过大，建议设为线程数的 1/10
- 缺少监听器——至少添加"查看结果树"和"聚合报告"

## 自测清单
- [ ] 能默写 JMeter 任务流程 7 步骤
- [ ] 能配置事务控制器、响应断言、集合点
- [ ] 能列出参数化 4 种类型及适用场景
- [ ] 能编写正则表达式提取器配置
- [ ] 能写出非 GUI 执行命令
- [ ] 能列出 Dashboard 报告截图要求

## 延伸阅读
- JMeter 官方用户手册
- 性能测试场景设计方法论
- JMeter 关联技术详解
- Dashboard 报告指标解读
