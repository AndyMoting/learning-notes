# 39-Locust入门

> 课时：60 min | 难度：★★★☆☆

## 学习目标

- 理解 Locust 与 JMeter 的差异与选型依据
- 掌握 Locust 核心概念（User/TaskSet/HttpUser）
- 编写 locustfile.py 定义测试场景
- 使用 Web UI 和 Headless 模式运行测试
- 配置分布式模式

## 核心概念

### Locust vs JMeter 对比

| 维度 | Locust | JMeter |
|------|--------|--------|
| 语言 | Python | Java |
| 界面 | Web UI | GUI |
| 脚本编写 | Python 代码 | XML/可视化编辑 |
| 并发模型 | 协程（gevent），单进程可支持数千用户 | 线程模型，资源消耗大 |
| 扩展性 | 极灵活，可调用任意 Python 库 | 相对固定，通过插件扩展 |
| 分布式 | 天生支持，命令启动 Master/Worker | 需要 RMI 配置 |
| 报告 | Web 实时图表，可选 HTML/CSV | Dashboard HTML 报告 |
| 学习曲线 | 需 Python 基础 | 无需编程但配置复杂 |

### Locust 核心概念

```
Locust
├── User（用户类）        → 定义用户行为的基础类
│   ├── HttpUser         → 基于 requests 的 HTTP 用户
│   ├── wait_time        → 用户思考时间（between/constant）
│   └── weight           → 用户类型的权重比例
├── Task（任务）          → 单个用户行为方法
│   ├── @task            → 标记权重，数字越大执行频率越高
│   └── @task(10)        → 执行频率为不带权重任务的 10 倍
├── Environment          → 运行时上下文
│   ├── runner           → 运行控制器
│   └── web_ui           → Web 界面
└── locustfile.py        → 测试脚本入口文件
```

### HttpUser 与 Task 定义

```python
from locust import HttpUser, task, between

class WebsiteUser(HttpUser):
    wait_time = between(1, 3)  # 每个任务间隔 1-3 秒

    @task(3)
    def view_items(self):
        self.client.get("/api/items")

    @task(1)
    def view_detail(self):
        self.client.get("/api/items/1")
```

### 运行模式

| 模式 | 命令 | 特点 |
|------|------|------|
| Web UI | `locust -f locustfile.py` | 浏览器实时监控，可视化操作 |
| Headless | `locust -f locustfile.py --headless -u 100 -r 10 -t 60s` | 命令行直接运行，适合 CI |
| Distributed | `locust --master` + `locust --worker --master-host=x.x.x.x` | 多进程并行，突破单机限制 |

## 动手实操：第一个 Locust 测试

**步骤 1：安装 Locust**

```bash
pip install locust

# 验证安装
locust --version
# 输出: locust 2.x.x
```

**步骤 2：编写 locustfile.py**

```python
from locust import HttpUser, task, between, SequentialTaskSet
from locust.exception import RescheduleTask
import json

class UserLoginAndBrowse(HttpUser):
    """
    模拟用户登录后浏览商品的行为
    """
    wait_time = between(1, 5)
    weight = 3  # 该类型用户占比
    host = "https://reqres.in"

    def on_start(self):
        """每个用户启动时执行一次"""
        self.token = None
        self.login()

    def on_stop(self):
        """每个用户结束时执行一次"""
        pass

    def login(self):
        """登录获取 Token"""
        response = self.client.post(
            "/api/login",
            json={
                "email": "eve.holt@reqres.in",
                "password": "cityslicka"
            },
            name="用户登录"
        )

        if response.status_code == 200:
            data = response.json()
            self.token = data.get("token", "")
            response.success()
        else:
            response.failure(f"登录失败: {response.status_code}")

    @task(5)
    def browse_products(self):
        """浏览商品列表"""
        self.client.get(
            "/api/unknown",
            name="浏览商品"
        )

    @task(2)
    def view_product_detail(self):
        """查看单个商品详情"""
        headers = {}
        if self.token:
            headers["Authorization"] = f"Bearer {self.token}"

        self.client.get(
            "/api/users/2",
            headers=headers,
            name="查看用户详情"
        )

    @task(1)
    def search_products(self):
        """搜索商品"""
        self.client.get(
            "/api/users?page=2",
            name="搜索用户"
        )

class QuickCheckUser(HttpUser):
    """
    仅执行快速健康检查的用户类型
    """
    wait_time = between(0.5, 1)
    weight = 1
    host = "https://reqres.in"

    @task
    def health_check(self):
        self.client.get("/api/users?page=1", name="健康检查")
```

**步骤 3：Web UI 运行**

```bash
# 启动 Locust Web UI
locust -f locustfile.py

# 浏览器访问 http://localhost:8089
```

Web UI 操作：
1. Number of users：总用户数（如 100）
2. Ramp up：每秒启动用户数（如 10）
3. Host：目标地址（locustfile 中已配置则不填）
4. 点击 Start swarming 开始测试
5. 实时查看图表：RPS、响应时间、用户数变化

**步骤 4：Headless 模式运行**

```bash
# 基本 Headless 模式
locust -f locustfile.py --headless \
  --users 100 \
  --spawn-rate 10 \
  --run-time 60s \
  --html report.html \
  --csv result

# 输出文件：
# report.html  - HTML 报告
# result_stats.csv       - 统计数据
# result_stats_history.csv - 历史数据
# result_failures.csv    - 失败数据
# result_exceptions.csv  - 异常数据
```

**步骤 5：分布式模式运行**

```bash
# 机器 1（Master）
locust -f locustfile.py --master

# 机器 2-N（Worker）
locust -f locustfile.py --worker --master-host=192.168.1.100

# Master 默认通过 5557 端口与 Worker 通信
# 可指定多个 Worker 进程：
# locust -f locustfile.py --worker --master-host=192.168.1.100 &
# locust -f locustfile.py --worker --master-host=192.168.1.100 &
```

**步骤 6：自定义报告与指标**

```python
from locust import events
import time

@events.request.add_listener
def on_request(request_type, name, response_time, response_length,
               response, context, exception, **kwargs):
    """每个请求完成后触发，可接入 ELK/Prometheus"""
    if exception:
        print(f"[FAIL] {name}: {exception}")
    elif response_time > 2000:
        # 记录慢请求
        with open("slow_requests.log", "a") as f:
            f.write(f"{time.time()}|{name}|{response_time}\n")

@events.test_stop.add_listener
def on_test_stop(environment, **kwargs):
    """测试结束时生成自定义报告"""
    stats = environment.runner.stats

    with open("summary_report.md", "w") as f:
        f.write("# Locust 测试总结\n\n")
        f.write(f"总请求数: {stats.total.num_requests}\n")
        f.write(f"总失败数: {stats.total.num_failures}\n")
        f.write(f"平均响应时间: {stats.total.avg_response_time:.2f}ms\n")
        f.write(f"P95 响应时间: {stats.total.get_response_time_percentile(0.95):.2f}ms\n")
        f.write(f"RPS: {stats.total.total_rps:.2f}\n")
```

## 常见坑

- on_start 中未异常捕获：登录失败导致后续请求全部异常，影响统计
- wait_time 过短：Locust 用户没有思考时间，TPS 虚高不真实
- 单进程 CPU 瓶颈：gevent 受 GIL 限制，单机可启动 Worker 进程数 = CPU 核心数
- host 未配置：User 类未设置 `host` 属性，且启动时未传 `--host`
- 资源文件路径：Worker 模式下所有节点必须有 locustfile 和依赖文件

## 自测清单

- [ ] 能列出 Locust 与 JMeter 的 5 个关键差异
- [ ] 能编写包含 HttpUser、@task、wait_time 的完整 locustfile
- [ ] 能用 Headless 模式运行测试并指定用户数/生成速率/持续时间
- [ ] 能配置 Master-Worker 分布式模式
- [ ] 能通过 events 钩子扩展自定义指标和报告

## 延伸阅读

- Locust 官方文档：https://docs.locust.io/en/stable/
- Locust 编写指南：https://docs.locust.io/en/stable/writing-a-locustfile.html
- Locust 分布式模式：https://docs.locust.io/en/stable/running-distributed.html
