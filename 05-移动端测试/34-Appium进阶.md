# 34-Appium进阶

> 课时：60 min | 难度：★★★★☆

## 学习目标

- 掌握 Hybrid App 的 WebView 上下文切换方法
- 理解 Deep Link 原理并能够测试应用内导航
- 能够通过 Appium 模拟不同网络条件
- 了解云测试平台（BrowserStack/Sauce Labs/HeadSpin）的使用
- 掌握移动端测试的 CI 集成方案
- 能够进行基础性能测试（启动时间、内存、CPU）

## 核心概念

### 34.1 Hybrid App 测试

Hybrid App 是原生与 Web 技术混合的应用，包含 Native 和 WebView 两种上下文。

```
┌─────────────────────────────────┐
│         Hybrid App              │
│  ┌───────────┐  ┌─────────────┐ │
│  │  Native   │  │   WebView   │ │
│  │  Context  │  │   Context   │ │
│  │           │  │  (H5 页面)   │ │
│  └───────────┘  └─────────────┘ │
└─────────────────────────────────┘
```

**上下文切换流程**

```python
from appium.webdriver.common.appiumby import AppiumBy

# 1. 获取所有可用上下文
contexts = driver.contexts
print(contexts)  # ['NATIVE_APP', 'WEBVIEW_com.example.app']

# 2. 切换到 WebView 上下文
driver.switch_to.context('WEBVIEW_com.example.app')

# 3. 在 WebView 中使用 Web 定位策略
element = driver.find_element(AppiumBy.CSS_SELECTOR, '#submit-btn')
element.click()

# 4. 切换回 Native 上下文
driver.switch_to.context('NATIVE_APP')

# 5. 使用原生定位策略
driver.find_element(AppiumBy.ID, "com.example.app:id/btn_back").click()
```

**WebView 调试前提**
- Android：应用需开启 WebView 调试（`WebView.setWebContentsDebuggingEnabled(true)`）
- iOS：需配置 `appium:includeSafariInSessions` Capability

**完整 Hybrid 测试示例**

```python
def test_hybrid_checkout(self):
    """Hybrid App 结算流程测试"""
    # Native 端：点击商品进入详情
    self.driver.find_element(AppiumBy.ID, "com.example.app:id/item_product").click()

    # 等待 WebView 加载
    self.wait.until(lambda d: len(d.contexts) > 1)

    # 切换到 WebView
    webview_context = [c for c in self.driver.contexts if c.startswith('WEBVIEW')][0]
    self.driver.switch_to.context(webview_context)

    # WebView 端：填写收货地址
    self.wait.until(
        lambda d: d.find_element(AppiumBy.CSS_SELECTOR, '#address-input')
    )
    self.driver.find_element(AppiumBy.CSS_SELECTOR, '#address-input').send_keys("北京市朝阳区")
    self.driver.find_element(AppiumBy.CSS_SELECTOR, '#phone-input').send_keys("13800138000")
    self.driver.find_element(AppiumBy.CSS_SELECTOR, '#save-address-btn').click()

    # 切换回 Native
    self.driver.switch_to.context('NATIVE_APP')

    # Native 端：确认订单
    self.driver.find_element(AppiumBy.ID, "com.example.app:id/btn_confirm").click()
```

### 34.2 Deep Link 测试

Deep Link 是通过 URL Scheme 或 Android App Links 直接跳转到应用内特定页面的技术。

**URL Scheme 格式**
```
myapp://product/detail?id=12345
```

**Android App Links 格式**
```
https://example.com/product/12345
```

**测试方法**

```python
# 方法1：通过 ADB 启动 Deep Link
driver.execute_script(
    'mobile: deepLink',
    {
        'url': 'myapp://product/detail?id=12345',
        'package': 'com.example.app'
    }
)

# 方法2：通过 ADB 命令
import subprocess
subprocess.run([
    'adb', 'shell', 'am', 'start',
    '-W', '-a', 'android.intent.action.VIEW',
    '-d', 'myapp://product/detail?id=12345',
    'com.example.app'
])

# 方法3：通过 mobile: startActivity
driver.execute_script('mobile: startActivity', {
    'intent': 'com.example.app/.ProductDetailActivity',
    'extras': [['id', '12345']]
})
```

**Deep Link 测试用例设计**

| 场景 | 测试点 |
|-----|-------|
| 应用已启动 | Deep Link 跳转后页面参数正确 |
| 应用未启动 | Deep Link 冷启动后直达目标页 |
| 应用后台运行 | Deep Link 恢复前台并跳转 |
| 无效 Deep Link | 应用不崩溃，显示友好提示 |
| 带特殊字符参数 | URL 编码正确解析 |
| 多层级跳转 | A → B → C 后返回栈正确 |

### 34.3 网络条件模拟

**Android 网络模拟**

```python
# 方法1：通过 ADB 限制网络（需 root 或模拟器）
import subprocess

# 模拟 2G 网络
subprocess.run(['adb', 'shell', 'svc', 'wifi', 'disable'])
subprocess.run(['adb', 'shell', 'svc', 'data', 'disable'])

# 恢复网络
subprocess.run(['adb', 'shell', 'svc', 'wifi', 'enable'])
subprocess.run(['adb', 'shell', 'svc', 'data', 'enable'])

# 方法2：通过 Appium 设置网络连接（Android）
from appium.webdriver.extensions.android.network import NetSpeed

# 设置网络为 GSM（2G）
driver.set_network_connection(1)  # 1 = Airplane mode, 2 = Wifi only, 4 = Data only, 6 = All

# 方法3：使用 Chrome DevTools Protocol（WebView 内）
driver.execute_cdp_cmd('Network.emulateNetworkConditions', {
    'offline': False,
    'downloadThroughput': 500 * 1024 / 8,  # 500 Kbps
    'uploadThroughput': 256 * 1024 / 8,     # 256 Kbps
    'latency': 200                          # 200ms
})
```

**iOS 网络模拟**

通过 Xcode 的 Network Link Conditioner 工具模拟不同网络配置文件。

**弱网测试场景**

| 场景 | 参数 | 验证点 |
|-----|------|-------|
| 2G 网络 | 250Kbps, 300ms 延迟 | 页面加载超时处理、降级展示 |
| 3G 网络 | 1Mbps, 100ms 延迟 | 图片懒加载、分页加载 |
| 网络抖动 | 丢包率 5%~10% | 请求重试机制、数据一致性 |
| 断网恢复 | 断网 30s 后恢复 | 离线缓存、自动重连 |
| 网络切换 | WiFi → 4G | 连接状态提示、数据不丢失 |

### 34.4 云测试平台

**BrowserStack**

```python
from appium import webdriver

desired_caps = {
    'platformName': 'Android',
    'platformVersion': '13.0',
    'deviceName': 'Samsung Galaxy S23',
    'app': 'bs://<app_hash>',  # BrowserStack 上传后的 app hash
    'automationName': 'UiAutomator2',
    'browserstack.user': '<username>',
    'browserstack.key': '<access_key>',
    'browserstack.networkProfile': '4g-gsm-good',  # 网络模拟
}

driver = webdriver.Remote(
    'https://hub-cloud.browserstack.com/wd/hub',
    options=UiAutomator2Options().load_capabilities(desired_caps)
)
```

**Sauce Labs**

```python
desired_caps = {
    'platformName': 'Android',
    'platformVersion': '13.0',
    'deviceName': 'Google Pixel 7',
    'app': 'storage:<file_id>',
    'automationName': 'UiAutomator2',
    'sauce:options': {
        'username': '<username>',
        'accessKey': '<access_key>',
        'name': 'Login Test',
        'build': 'v1.0.0',
    }
}

driver = webdriver.Remote(
    'https://ondemand.us-west-1.saucelabs.com/wd/hub',
    options=UiAutomator2Options().load_capabilities(desired_caps)
)
```

**HeadSpin**

```python
desired_caps = {
    'platformName': 'Android',
    'automationName': 'UiAutomator2',
    'headspin:capture': True,  # 开启视频录制
    'headspin:apiKey': '<api_key>',
}

driver = webdriver.Remote(
    'https://app.headspin.io/v0/<token>/wd/hub',
    options=UiAutomator2Options().load_capabilities(desired_caps)
)
```

**云平台对比**

| 维度 | BrowserStack | Sauce Labs | HeadSpin |
|-----|-------------|-----------|---------|
| 设备数量 | 3000+ | 2700+ | 10000+ |
| 网络模拟 | 内置 | 内置 | 真实网络 |
| 性能分析 | 基础 | 基础 | 深度（视频+指标） |
| 价格 | 中高 | 中高 | 高 |
| 适用场景 | 兼容性测试 | CI 集成 | 性能深度分析 |

### 34.5 CI 集成

**GitHub Actions 配置**

```yaml
name: Mobile Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  android-test:
    runs-on: macos-latest  # macOS 支持硬件加速模拟器

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Appium
        run: |
          npm install -g appium
          appium driver install uiautomator2

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install Python dependencies
        run: |
          pip install Appium-Python-Client pytest pytest-html

      - name: Start Appium Server
        run: appium -a 0.0.0.0 -p 4723 &

      - name: Create and start Android emulator
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 33
          arch: x86_64
          script: |
            adb wait-for-device
            adb devices

      - name: Run tests
        run: |
          pytest tests/ -v --html=report.html --self-contained-html

      - name: Upload test report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-report
          path: report.html
```

**Jenkins Pipeline 配置**

```groovy
pipeline {
    agent any

    environment {
        ANDROID_HOME = '/opt/android-sdk'
        APPIUM_PORT = '4723'
    }

    stages {
        stage('Setup') {
            steps {
                sh 'npm install -g appium'
                sh 'appium driver install uiautomator2'
                sh 'pip install -r requirements.txt'
            }
        }

        stage('Start Appium') {
            steps {
                sh "appium -a 0.0.0.0 -p ${APPIUM_PORT} &"
                sh 'sleep 5'
            }
        }

        stage('Start Emulator') {
            steps {
                sh '$ANDROID_HOME/emulator/emulator -avd test_avd -no-window -no-audio &'
                sh 'adb wait-for-device'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'pytest tests/ -v --junitxml=results.xml'
            }
        }
    }

    post {
        always {
            junit 'results.xml'
            sh 'pkill -f appium'
        }
    }
}
```

### 34.6 性能测试基础

**启动时间测试**

```python
import time
import subprocess

def measure_cold_start(driver, package, activity):
    """测量冷启动时间"""
    # 强制停止应用
    driver.terminate_app(package)

    # 记录开始时间
    start_time = time.time()

    # 启动应用
    driver.activate_app(package)

    # 等待首页关键元素出现
    wait = WebDriverWait(driver, 30)
    wait.until(
        EC.presence_of_element_located((AppiumBy.ID, f"{package}:id/home_container"))
    )

    # 计算启动耗时
    elapsed = (time.time() - start_time) * 1000
    return elapsed


def measure_start_time_via_adb(package, activity):
    """通过 ADB 测量启动时间"""
    subprocess.run(['adb', 'shell', 'am', 'force-stop', package])
    result = subprocess.run(
        ['adb', 'shell', 'am', 'start', '-W', '-n', f'{package}/{activity}'],
        capture_output=True, text=True
    )
    # 解析 TotalTime 字段
    for line in result.stdout.split('\n'):
        if 'TotalTime' in line:
            return int(line.split(':')[1].strip())
    return None
```

**内存测试**

```python
def get_memory_info(driver, package):
    """获取应用内存占用"""
    result = driver.execute_script('mobile: shell', {
        'command': 'dumpsys',
        'args': ['meminfo', package]
    })

    mem_data = {}
    for line in result.split('\n'):
        if 'TOTAL PSS' in line:
            mem_data['total_pss'] = int(line.split(':')[1].strip().split()[0])
        elif 'Native Heap' in line:
            mem_data['native_heap'] = int(line.split()[2])
        elif 'Dalvik Heap' in line:
            mem_data['dalvik_heap'] = int(line.split()[2])

    return mem_data


def test_memory_leak(driver, package):
    """内存泄漏检测：重复操作后内存是否持续增长"""
    memory_readings = []

    for i in range(10):
        # 执行目标操作（如打开/关闭页面）
        driver.find_element(AppiumBy.ID, f"{package}:id/btn_open").click()
        time.sleep(1)
        driver.find_element(AppiumBy.ID, f"{package}:id/btn_close").click()
        time.sleep(1)

        # 记录内存
        mem = get_memory_info(driver, package)
        memory_readings.append(mem['total_pss'])

    # 分析内存趋势
    first_half = sum(memory_readings[:5]) / 5
    second_half = sum(memory_readings[5:]) / 5
    growth_rate = (second_half - first_half) / first_half * 100

    assert growth_rate < 20, f"内存增长 {growth_rate:.1f}%，疑似内存泄漏"
```

**CPU 测试**

```python
def get_cpu_usage(driver, package):
    """获取应用 CPU 使用率"""
    result = driver.execute_script('mobile: shell', {
        'command': 'top',
        'args': ['-n', '1', '-b']
    })

    for line in result.split('\n'):
        if package in line:
            parts = line.split()
            # CPU% 通常在第 9 列
            cpu_percent = float(parts[8])
            return cpu_percent
    return None


def test_cpu_usage_under_load(driver, package):
    """高负载场景下 CPU 使用率测试"""
    # 触发高负载操作
    driver.find_element(AppiumBy.ID, f"{package}:id/btn_load_data").click()

    cpu_readings = []
    for _ in range(10):
        cpu = get_cpu_usage(driver, package)
        if cpu is not None:
            cpu_readings.append(cpu)
        time.sleep(0.5)

    avg_cpu = sum(cpu_readings) / len(cpu_readings)
    max_cpu = max(cpu_readings)

    assert avg_cpu < 50, f"平均 CPU 使用率 {avg_cpu:.1f}% 超过阈值"
    assert max_cpu < 80, f"峰值 CPU 使用率 {max_cpu:.1f}% 超过阈值"
```

**帧率测试（FPS）**

```python
def get_fps(driver):
    """获取当前帧率（Android）"""
    result = driver.execute_script('mobile: shell', {
        'command': 'dumpsys',
        'args': ['SurfaceFlinger', '--latency']
    })
    # 解析帧率数据（需根据实际输出格式调整）
    return result
```

## 动手实操

### 34.7 完整 Hybrid App 测试

```python
import pytest
from appium import webdriver
from appium.options.android import UiAutomator2Options
from appium.webdriver.common.appiumby import AppiumBy
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


class TestHybridApp:
    """Hybrid App 完整测试"""

    @pytest.fixture(autouse=True)
    def setup_and_teardown(self):
        options = UiAutomator2Options()
        options.platform_name = "Android"
        options.device_name = "emulator-5554"
        options.app_package = "com.example.hybrid"
        options.app_activity = ".MainActivity"
        options.automation_name = "UiAutomator2"
        options.no_reset = True
        options.new_command_timeout = 300
        # 开启 WebView 调试
        options.set_capability("appium:chromedriverExecutable", "D:/tools/chromedriver.exe")

        self.driver = webdriver.Remote("http://127.0.0.1:4723", options=options)
        self.driver.implicitly_wait(10)
        self.wait = WebDriverWait(self.driver, 15)
        yield
        self.driver.quit()

    def test_native_to_webview(self):
        """Native 页面跳转 WebView 页面"""
        # Native 端操作
        self.driver.find_element(AppiumBy.ID, "com.example.hybrid:id/btn_open_web").click()

        # 等待 WebView 上下文可用
        self.wait.until(lambda d: any(c.startswith('WEBVIEW') for c in d.contexts))

        # 切换到 WebView
        webview = [c for c in self.driver.contexts if c.startswith('WEBVIEW')][0]
        self.driver.switch_to.context(webview)

        # WebView 端操作
        self.wait.until(
            EC.presence_of_element_located((AppiumBy.CSS_SELECTOR, '#search-input'))
        )
        self.driver.find_element(AppiumBy.CSS_SELECTOR, '#search-input').send_keys("测试")
        self.driver.find_element(AppiumBy.CSS_SELECTOR, '#search-btn').click()

        # 验证搜索结果
        results = self.wait.until(
            EC.presence_of_all_elements_located((AppiumBy.CSS_SELECTOR, '.result-item'))
        )
        assert len(results) > 0

        # 切换回 Native
        self.driver.switch_to.context('NATIVE_APP')
```

### 34.8 性能测试完整流程

```python
import pytest
import time
import statistics


class TestPerformance:
    """性能测试套件"""

    @pytest.fixture(autouse=True)
    def setup_and_teardown(self):
        options = UiAutomator2Options()
        options.platform_name = "Android"
        options.device_name = "emulator-5554"
        options.app_package = "com.example.app"
        options.app_activity = ".MainActivity"
        options.automation_name = "UiAutomator2"
        options.no_reset = True

        self.driver = webdriver.Remote("http://127.0.0.1:4723", options=options)
        self.driver.implicitly_wait(10)
        yield
        self.driver.quit()

    def test_cold_start_time(self):
        """冷启动时间测试（执行 5 次取平均值）"""
        startup_times = []

        for _ in range(5):
            self.driver.terminate_app("com.example.app")
            time.sleep(1)

            start = time.time()
            self.driver.activate_app("com.example.app")
            WebDriverWait(self.driver, 30).until(
                lambda d: d.find_element(
                    AppiumBy.ID, "com.example.app:id/home_container"
                )
            )
            elapsed = (time.time() - start) * 1000
            startup_times.append(elapsed)

        avg_time = statistics.mean(startup_times)
        p95_time = sorted(startup_times)[int(len(startup_times) * 0.95)]

        print(f"平均启动时间: {avg_time:.0f}ms")
        print(f"P95 启动时间: {p95_time:.0f}ms")

        assert avg_time < 3000, f"平均启动时间 {avg_time:.0f}ms 超过 3s 阈值"
        assert p95_time < 5000, f"P95 启动时间 {p95_time:.0f}ms 超过 5s 阈值"

    def test_memory_baseline(self):
        """内存基线测试"""
        # 进入首页
        WebDriverWait(self.driver, 15).until(
            lambda d: d.find_element(AppiumBy.ID, "com.example.app:id/home_container")
        )
        time.sleep(3)  # 等待内存稳定

        # 获取内存数据
        result = self.driver.execute_script('mobile: shell', {
            'command': 'dumpsys',
            'args': ['meminfo', 'com.example.app']
        })

        total_pss = None
        for line in result.split('\n'):
            if 'TOTAL PSS' in line:
                total_pss = int(line.split(':')[1].strip().split()[0])
                break

        print(f"首页内存占用: {total_pss} KB")
        assert total_pss < 200000, f"内存占用 {total_pss}KB 超过 200MB 阈值"
```

## 常见坑

- WebView 上下文名称不固定，需通过 `driver.contexts` 动态获取
- Hybrid App 中 WebView 调试未开启时，无法切换到 WebView 上下文
- Deep Link 测试时应用未安装会导致测试失败，需前置安装验证
- 云测试平台设备排队等待时间较长，需设置合理的超时时间
- CI 环境中模拟器启动慢，需使用 `adb wait-for-device` 等待就绪
- 性能测试结果波动大，需多次采样取平均值或 P95
- `dumpsys meminfo` 输出格式因系统版本不同有差异，解析时需兼容
- 网络模拟在真机上受限，模拟器或 root 设备才能完整控制

## 自测清单

- [ ] 能够实现 Hybrid App 的 Native 与 WebView 上下文切换
- [ ] 能够测试 Deep Link 的跳转逻辑与参数传递
- [ ] 能够模拟不同网络条件并验证应用行为
- [ ] 能够在 BrowserStack/Sauce Labs 上运行测试脚本
- [ ] 能够配置 GitHub Actions 或 Jenkins 实现移动端 CI
- [ ] 能够测量应用的启动时间、内存占用、CPU 使用率

## 延伸阅读

- Appium Hybrid App 文档：<https://appium.io/docs/en/latest/guides/webviews/>
- Android Deep Links：<https://developer.android.com/training/app-links>
- BrowserStack App Automate：<https://www.browserstack.com/docs/app-automate/appium/getting-started>
- Sauce Labs Mobile Testing：<https://docs.saucelabs.com/mobile-apps/>
- HeadSpin 文档：<https://docs.headspin.io/>
- Android 性能测试指南：<https://developer.android.com/topic/performance>
- iOS Instruments 使用指南：<https://help.apple.com/instruments/mac/current/>
