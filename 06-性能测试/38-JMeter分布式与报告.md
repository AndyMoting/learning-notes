# 38-JMeter分布式与报告

> 课时：60 min | 难度：★★★☆☆

## 学习目标

- 理解 JMeter 分布式测试架构（Master-Slave）
- 完成分布式环境的搭建与配置
- 熟练使用 CLI 模式执行压测
- 生成并解读 Dashboard 报告
- 掌握 CI 集成方法

## 核心概念

### 分布式测试架构

单节点 JMeter 受 CPU/内存/网络限制，无法产生足够高的并发。分布式测试将负载分散到多台机器。

```
              控制命令 / 测试计划
                    ↓
    ┌──────────────────────────────────┐
    │          Master 节点              │
    │  (GUI 编辑计划 + CLI 发起测试)     │
    └──────────────┬───────────────────┘
                   │ RMI 协议（端口 1099）
        ┌──────────┼──────────┐
        ↓          ↓          ↓
   Slave-1     Slave-2     Slave-3
   (jmeter-server) (jmeter-server) (jmeter-server)
   各节点独立执行相同计划，产生实际负载
```

**关键约束：**
- Master 不直接产生负载，仅负责分发计划和汇总结果
- 所有 Slave 通过网络接收 Master 发送的测试计划并执行
- 测试计划中的数据文件必须在所有节点相同路径下存在

### CLI 模式（Non-GUI Mode）

正式压测必须使用 CLI 模式，GUI 模式仅用于计划编写与调试：

```bash
jmeter -n -t test_plan.jmx -l result.jtl -e -o report/

参数说明：
-n, --nongui           # 以 non-GUI 模式运行
-t, --testfile        # 指定测试计划文件
-l, --logfile         # 指定结果输出文件（CSV/XML）
-e, --reportatendofloadtests  # 测试结束后生成 Dashboard 报告
-o, --output_folder   # 报告输出目录（必须为空或不存在）
-r, --remote          # 远程执行所有 Slave（jmeter.properties 中配置）
-R                    # 指定部分远程节点 IP，逗号分隔
-J                    # 覆盖 JMeter 属性（如 -Jusers=100）
-G                    # 覆盖全局属性（传递到所有节点）
```

### Dashboard 报告解读

JMeter 5.0+ 自动生成 HTML Dashboard 报告，包含以下关键页面：

| 页面 | 关键内容 |
|------|----------|
| Statistics | 请求数、错误率、最小/最大/平均响应时间、标准偏差 |
| Errors | 错误类型及占比分布 |
| Throughput | TPS / 每秒处理字节数随时间变化 |
| Response Times | 响应时间分布直方图 |
| Latency | 延迟与响应时间对比 |
| Response Codes | HTTP 状态码分布 |
| Connect Time | TCP 连接建立时间 |

**报告中的关键指标：**
- **Apdex（应用性能指数）**：0-1 的值，> 0.85 可接受，> 0.94 优秀
- **Percentiles**：P50/P90/P95/P99 分位线
- **First Byte Time**：首字节时间，反映服务器处理速度

### CI 集成

通过 Maven/Gradle 插件或 Jenkins Performance Plugin 集成：

```bash
# Jenkins Pipeline 示例
pipeline {
  stages {
    stage('Performance Test') {
      steps {
        sh 'jmeter -n -t perf_test.jmx -l result.jtl -e -o report/'
      }
    }
  }
  post {
    always {
      perfReport sourceDataFiles: 'result.jtl'
    }
  }
}
```

## 动手实操：分布式压测完整流程

**步骤 1：准备环境（3 台机器）**

```bash
# Slave-1: 192.168.1.101
# Slave-2: 192.168.1.102
# Master:   192.168.1.100

# 所有机器：
# 1. 安装 Java 11+
# 2. 安装 JMeter 5.6+
# 3. 关闭防火墙或开放端口 1099, 4000-4100
# 4. 确保系统时间同步（NTP）
```

**步骤 2：配置 Slave 节点**

```bash
# 编辑每个 Slave 的 jmeter.properties
vi $JMETER_HOME/bin/jmeter.properties

# 添加本机 IP
remote_hosts=127.0.0.1
server.rmi.ssl.disable=true  # 非生产环境可禁用 SSL
```

**步骤 3：配置 Master 节点**

```bash
# 编辑 Master 的 jmeter.properties
vi $JMETER_HOME/bin/jmeter.properties

# 添加所有 Slave IP（逗号分隔）
remote_hosts=192.168.1.101,192.168.1.102
server.rmi.ssl.disable=true
```

**步骤 4：启动 Slave 服务**

```bash
# 每个 Slave 上执行
cd $JMETER_HOME/bin
./jmeter-server -Djava.rmi.server.hostname=192.168.1.101

# 输出: Starting the test on host 192.168.1.101 @ ...
```

**步骤 5：Master 发起分布式测试**

```bash
cd $JMETER_HOME/bin

# 方式一：使用 -r 自动执行所有远程节点
jmeter -n -t perf_test.jmx -l result.jtl -e -o report/ -r

# 方式二：指定部分节点
jmeter -n -t perf_test.jmx -l result.jtl -e -o report/ -R 192.168.1.101,192.168.1.102
```

**步骤 6：查看报告**

```bash
# 报告目录结构
report/
├── index.html                    ← 入口文件
├── content/
│   ├── js/                       ← 图表 JS
│   └── css/                      ← 样式
├── statistics.json               ← 原始数据 JSON
└── socksum.box                   ← 统计摘要

# 浏览器打开
firefox report/index.html
```

**步骤 7：传递动态参数**

```bash
# 运行时动态覆盖线程数和持续时间
jmeter -n -t perf_test.jmx -l result.jtl -e -o report/ \
  -Jusers=200 -Jduration=600 -r
```

在测试计划中使用 `${__P(users,50)}` 引用参数。

## 常见坑

- RMI 端口不通：Slave jmeter-server 占用 1099 + 动态端口（4000-4100），防火墙需放行
- hostname 未指定：多网卡机器必须显式指定 `java.rmi.server.hostname`
- 测试计划数据文件路径不一致：所有节点 CSV 文件必须放在相同绝对路径
- 时钟不同步：分布式节点时间差影响日志时间戳，使用 NTP 同步
- 报告目录已存在：`-o` 指定的目录不能已存在，否则报错

## 自测清单

- [ ] 能画出 Master-Slave 架构图并标注通信协议和端口
- [ ] 能写出完整的 CLI 命令（含所有关键参数）
- [ ] 能配置 Slave 节点的 jmeter.properties
- [ ] 能解释 Dashboard 报告中 Apdex、Throughput、Percentiles 的含义
- [ ] 能将 JMeter 压测集成到 CI Pipeline 中

## 延伸阅读

- 官方分布式测试指南：https://jmeter.apache.org/usermanual/remote-test.html
- JMeter Dashboard 报告说明：https://jmeter.apache.org/usermanual/generating-dashboard.html
- Jenkins Performance Plugin：https://plugins.jenkins.io/performance/
