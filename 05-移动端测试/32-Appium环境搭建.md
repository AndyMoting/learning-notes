# 32-Appium环境搭建

> 课时：50 min | 难度：★★★☆☆

## 学习目标

- 理解 Appium 架构与工作原理
- 完成 Appium Server、UiAutomator2 Driver、Android SDK 的安装与配置
- 掌握 Desired Capabilities 的配置方法
- 能够编写并运行第一个 Appium 测试脚本
- 使用 Appium Inspector 进行元素定位与调试

## 核心概念

### 32.1 Appium 架构

```
┌──────────────┐     HTTP/JSON      ┌──────────────┐     protocol     ┌──────────────┐
│  Appium      │ ◄───────────────► │  Appium      │ ◄──────────────► │  Device /    │
│  Client      │    (WebDriver)     │  Server      │                  │  Emulator    │
│  (Python/JS/ │                    │  (Node.js)   │                  │              │
│   Java/...)  │                    │              │                  │  UiAutomator2│
└──────────────┘                    └──────────────┘                  │  /XCUITest   │
                                                                      └──────────────┘
```

**核心组件**
- **Appium Client**：测试脚本使用的语言绑定库（Python: `Appium-Python-Client`，Java: `java-client`）
- **Appium Server**：基于 Node.js 的 HTTP 服务器，接收 WebDriver 协议请求并转换为平台特定指令
- **UiAutomator2 Driver**：Android 端驱动，通过 UiAutomator2 框架与设备交互
- **XCUITest Driver**：iOS 端驱动，通过 XCUITest 框架与设备交互

**通信流程**
1. 测试脚本调用 Client API（如 `driver.find_element()`）
2. Client 将操作封装为 HTTP POST 请求发送至 Appium Server
3. Server 根据平台选择对应 Driver
4. Driver 调用平台原生自动化框架（UiAutomator2/XCUITest）
5. 设备执行操作并返回结果

### 32.2 环境依赖

| 组件 | 用途 | 安装方式 |
|-----|------|---------|
| Node.js ≥ 16 | Appium Server 运行环境 | <https://nodejs.org/> |
| Appium Server | 核心服务 | `npm install -g appium` |
| UiAutomator2 Driver | Android 测试驱动 | `appium driver install uiautomator2` |
| Android SDK | ADB、编译工具、模拟器 | Android Studio 或 SDK Manager |
| JDK 11+ | Java 环境（部分工具依赖） | <https://adoptium.net/> |
| Appium-Python-Client | Python 语言绑定 | `pip install Appium-Python-Client` |

### 32.3 安装步骤

**Step 1：安装 Node.js**
```bash
node -v    # 验证 ≥ 16.x
npm -v
```

**Step 2：安装 Appium Server**
```bash
npm install -g appium
appium -v    # 验证安装
```

**Step 3：安装 UiAutomator2 Driver**
```bash
appium driver install uiautomator2
appium driver list --installed    # 验证已安装驱动
```

**Step 4：配置 Android SDK 环境变量**
```bash
# 系统变量
ANDROID_HOME = C:\Users\<用户名>\AppData\Local\Android\Sdk

# Path 追加
%ANDROID_HOME%\platform-tools
%ANDROID_HOME%\cmdline-tools\latest\bin
```

**Step 5：安装 Python Client**
```bash
pip install Appium-Python-Client
pip install pytest    # 可选，测试框架
```

**Step 6：启动 Appium Server**
```bash
# 默认启动（127.0.0.1:4723）
appium

# 指定地址和端口
appium -a 0.0.0.0 -p 4723

# 启动时加载 base path
appium --base-path /wd/hub
```

### 32.4 Desired Capabilities 配置

Desired Capabilities 是 JSON 格式的键值对集合，用于告知 Appium Server 测试会话的配置参数。

**Android 核心 Capabilities**

```python
from appium import webdriver
from appium.options.android import UiAutomator2Options

options = UiAutomator2Options()
options.platform_name = "Android"
options.device_name = "emulator-5554"
options.app = "D:/test/app-debug.apk"              # APK 路径（自动安装）
options.app_package = "com.example.app"             # 已安装应用的包名
options.app_activity = ".MainActivity"              # 启动 Activity
options.automation_name = "UiAutomator2"            # 自动化引擎
options.no_reset = False                            # 是否清除应用数据
options.full_reset = False                          # 是否卸载后重装
options.new_command_timeout = 300                   # 命令超时（秒）
options.auto_grant_permissions = True               # 自动授予权限

driver = webdriver.Remote("http://127.0.0.1:4723", options=options)
```

**iOS 核心 Capabilities**

```python
from appium.options.ios import XCUITestOptions

options = XCUITestOptions()
options.platform_name = "iOS"
options.device_name = "iPhone 14"
options.platform_version = "16.0"
options.app = "D:/test/app.app"                     # .app 或 .ipa 路径
options.automation_name = "XCUITest"
options.udid = "auto"                               # 设备 UDID
options.xcode_org_id = "<Team ID>"                  # 开发者 Team ID
options.xcode_signing_id = "iPhone Developer"

driver = webdriver.Remote("http://127.0.0.1:4723", options=options)
```

**常用 Capabilities 速查**

| Capability | 说明 | 示例值 |
|-----------|------|-------|
| `platform_name` | 平台名称 | `Android` / `iOS` |
| `device_name` | 设备名称 | `emulator-5554` / `iPhone 14` |
| `app` | APK/IPA 绝对路径 | `D:/test/app.apk` |
| `app_package` | 应用包名 | `com.example.app` |
| `app_activity` | 启动 Activity | `.MainActivity` |
| `automation_name` | 自动化引擎 | `UiAutomator2` / `XCUITest` |
| `no_reset` | 保留应用数据 | `True` / `False` |
| `auto_grant_permissions` | 自动授权 | `True` |
| `new_command_timeout` | 命令超时 | `300` |
| `udid` | 设备唯一标识 | `auto` |

### 32.5 第一个测试脚本

```python
import pytest
from appium import webdriver
from appium.options.android import UiAutomator2Options
from appium.webdriver.common.appiumby import AppiumBy
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


class TestLogin:
    """登录功能测试"""

    @pytest.fixture(autouse=True)
    def setup_and_teardown(self):
        """测试前置：启动应用；后置：关闭应用"""
        options = UiAutomator2Options()
        options.platform_name = "Android"
        options.device_name = "emulator-5554"
        options.app_package = "com.example.app"
        options.app_activity = ".MainActivity"
        options.automation_name = "UiAutomator2"
        options.no_reset = True
        options.new_command_timeout = 300

        self.driver = webdriver.Remote("http://127.0.0.1:4723", options=options)
        self.driver.implicitly_wait(10)
        yield
        self.driver.quit()

    def test_login_success(self):
        """正常登录流程"""
        # 定位用户名输入框并输入
        username_input = self.driver.find_element(AppiumBy.ID, "com.example.app:id/et_username")
        username_input.clear()
        username_input.send_keys("testuser")

        # 定位密码输入框并输入
        password_input = self.driver.find_element(AppiumBy.ID, "com.example.app:id/et_password")
        password_input.clear()
        password_input.send_keys("password123")

        # 点击登录按钮
        login_btn = self.driver.find_element(AppiumBy.ID, "com.example.app:id/btn_login")
        login_btn.click()

        # 验证登录成功（等待首页元素出现）
        wait = WebDriverWait(self.driver, 15)
        home_element = wait.until(
            EC.presence_of_element_located((AppiumBy.ID, "com.example.app:id/tv_welcome"))
        )
        assert "欢迎" in home_element.text

    def test_login_empty_password(self):
        """空密码登录验证"""
        username_input = self.driver.find_element(AppiumBy.ID, "com.example.app:id/et_username")
        username_input.send_keys("testuser")

        login_btn = self.driver.find_element(AppiumBy.ID, "com.example.app:id/btn_login")
        login_btn.click()

        # 验证错误提示
        error_msg = self.driver.find_element(AppiumBy.ID, "com.example.app:id/tv_error")
        assert error_msg.is_displayed()
```

### 32.6 Appium Inspector

Appium Inspector 是可视化元素检查工具，用于定位元素、生成定位表达式。

**启动步骤**
1. 启动 Appium Server
2. 启动 Appium Inspector（独立应用或 Web 版）
3. 配置 Desired Capabilities（与测试脚本一致）
4. 启动会话，Inspector 将显示当前屏幕截图与元素树

**核心功能**
- 点击截图区域选中元素，查看其属性（resource-id、content-desc、text、class）
- 生成对应定位表达式（ID、XPath、Accessibility ID）
- 录制操作并导出为测试代码
- 查看页面源码 XML（Source）

**Inspector 配置示例**
```json
{
  "platformName": "Android",
  "appium:deviceName": "emulator-5554",
  "appium:appPackage": "com.example.app",
  "appium:appActivity": ".MainActivity",
  "appium:automationName": "UiAutomator2",
  "appium:noReset": true
}
```

## 动手实操

### 32.7 从零搭建完整环境

```bash
# 1. 安装 Node.js（如未安装）
winget install OpenJS.NodeJS.LTS

# 2. 安装 Appium
npm install -g appium

# 3. 安装 UiAutomator2 驱动
appium driver install uiautomator2

# 4. 验证驱动安装
appium driver list --installed

# 5. 安装 Python 依赖
pip install Appium-Python-Client pytest

# 6. 启动 Appium Server（保持运行）
appium -a 0.0.0.0 -p 4723

# 7. 连接设备或启动模拟器
adb devices -l

# 8. 运行测试
pytest test_login.py -v
```

### 32.8 验证环境连通性

```python
from appium import webdriver
from appium.options.android import UiAutomator2Options

options = UiAutomator2Options()
options.platform_name = "Android"
options.device_name = "emulator-5554"
options.app_package = "com.example.app"
options.app_activity = ".MainActivity"
options.automation_name = "UiAutomator2"
options.no_reset = True

driver = webdriver.Remote("http://127.0.0.1:4723", options=options)

# 验证连接成功
print(f"Session ID: {driver.session_id}")
print(f"Device Time: {driver.device_time}")
print(f"Current Package: {driver.current_package}")

driver.quit()
```

## 常见坑

- Node.js 版本过低（< 16）导致 Appium 启动失败
- UiAutomator2 Driver 未安装或版本不兼容，报 `Could not find a driver`
- `ANDROID_HOME` 环境变量未配置，导致无法识别设备
- `app` 与 `app_package` + `app_activity` 同时配置时行为冲突
- 真机未开启开发者选项/USB 调试，连接后显示 `unauthorized`
- Appium Server 端口 4723 被占用，需更换端口或关闭占用进程
- Python Client 版本与 Appium Server 版本不匹配，导致 API 调用失败
- 首次启动 UiAutomator2 需安装 `io.appium.uiautomator2.server` 和 `io.appium.uiautomator2.server.test` 两个辅助 APK

## 自测清单

- [ ] 能够描述 Appium 架构中 Client、Server、Driver 的职责
- [ ] 独立完成 Appium Server + UiAutomator2 Driver 安装
- [ ] 能够配置 Android/iOS 的 Desired Capabilities
- [ ] 能够编写并运行第一个 Appium 测试脚本
- [ ] 能够使用 Appium Inspector 定位元素并生成定位表达式
- [ ] 能够排查常见环境搭建问题

## 延伸阅读

- Appium 官方文档：<https://appium.io/docs/en/latest/>
- UiAutomator2 Driver 文档：<https://github.com/appium/appium-uiautomator2-driver>
- Desired Capabilities 完整参考：<https://appium.io/docs/en/latest/guides/caps/>
- Appium Inspector：<https://github.com/appium/appium-inspector>
- UiAutomator2 官方文档：<https://developer.android.com/training/testing/other-components/ui-automator>
