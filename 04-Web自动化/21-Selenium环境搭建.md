# 21-Selenium环境搭建

> 课时：30 min | 难度：★★

## 学习目标

- 理解 Selenium WebDriver 架构与通信协议
- 掌握 Python 虚拟环境创建与 selenium 包安装
- 区分 Selenium 4.6+ 自动驱动管理与手动驱动配置
- 编写并运行第一个完整的 Selenium 脚本
- 配置常用 Chrome 启动选项

## 核心概念

### Selenium 架构

Selenium 由三个核心组件构成：

1. **Selenium IDE** — 浏览器插件，录制回放脚本
2. **Selenium WebDriver** — 通过浏览器原生控制 API 驱动浏览器
3. **Selenium Grid** — 分布式执行，支持并行跨机器测试

本课程聚焦 Selenium WebDriver。

#### WebDriver 协议

WebDriver 采用 **W3C WebDriver 协议**（基于 HTTP 的 RESTful JSON 接口）。通信流程如下：

```
测试脚本 (Python客户端)
    ↓ JSON Wire Protocol (HTTP请求)
WebDriver客户端库 (selenium-python)
    ↓ HTTP请求
浏览器驱动 (ChromeDriver)
    ↓ 原生协议 (CDP/Chromium)
Chrome浏览器
```

每次调用 `driver.find_element()` 或 `driver.get()` 时，客户端库将操作序列化为 HTTP POST 请求，发送至浏览器驱动（ChromeDriver 默认监听 `http://localhost:9515`），驱动解析后通过浏览器原生协议执行操作，再通过 HTTP 响应返回结果。

#### Selenium 4.6+ Selenium-Manager

Selenium 4.6 起内置 **Selenium-Manager**（Go 语言编写的驱动管理工具），自动下载匹配当前浏览器版本的驱动二进制文件。手动下载 ChromeDriver 不再是必要步骤。

- Selenium 4.6 ~ 4.10：自动匹配，但失败时无明确提示
- Selenium 4.11+：自动匹配失败会抛出清晰异常

### Python 虚拟环境

在项目根目录创建并激活虚拟环境：

```bash
# Windows PowerShell
python -m venv .venv
.venv\Scripts\Activate.ps1

# macOS / Linux
python3 -m venv .venv
source .venv/bin/activate
```

虚拟环境隔离项目依赖，避免全局包冲突。

### 安装 selenium

```bash
pip install selenium>=4.11.0
```

验证安装：

```python
import selenium
print(selenium.__version__)   # 应输出 4.11.0 或更高
```

### 手动 ChromeDriver 配置（备用方案）

当 Selenium-Manager 因网络限制无法自动下载时，手动配置流程：

1. 查询当前 Chrome 版本：`chrome://settings/help`，记下主版本号（如 `120.x.x.x`）
2. 从 [Chrome for Testing](https://googlechromelabs.github.io/chrome-for-testing/) 下载对应版本 ChromeDriver
3. 解压后将 `chromedriver.exe` 所在目录加入系统 PATH，或代码中指定路径：

```python
from selenium.webdriver.chrome.service import Service

service = Service(executable_path="C:/tools/chromedriver/chromedriver.exe")
driver = webdriver.Chrome(service=service)
```

> Selenium 4.6+ 推荐使用 `Service` 类指定驱动路径，废弃 `executable_path` 直接参数传入方式。

## 动手实操

### 第一个完整脚本

```python
"""
项目结构：
first_selenium/
├── first_script.py
└── .venv/
"""

from selenium import webdriver
from selenium.webdriver.chrome.options import Options

options = Options()
options.add_argument("--headless=new")
options.add_argument("--window-size=1920,1080")
options.add_argument("--disable-gpu")

driver = webdriver.Chrome(options=options)

driver.get("https://www.python.org")

title = driver.title
print(f"页面标题: {title}")
assert "Python" in title

driver.save_screenshot("python_org.png")

driver.quit()
```

### Chrome 常用选项

| 选项 | 说明 |
|------|------|
| `--headless=new` | 无头模式（不显示浏览器窗口），headless=new 优先使用新实现 |
| `--window-size=W,H` | 设置窗口尺寸，影响响应式布局测试结果 |
| `--disable-gpu` | 禁用 GPU 加速，某些环境稳定性要求 |
| `--no-sandbox` | 以 root 权限运行时必需 |
| `--disable-dev-shm-usage` | 解决 /dev/shm 过小导致的崩溃（Docker 环境） |
| `--proxy-server=host:port` | 配置代理服务器 |
| `--user-agent="..."` | 自定义 User-Agent |
| `--incognito` | 隐身模式启动 |

```python
options.add_argument("--proxy-server=http://127.0.0.1:7890")
options.add_argument("--user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64)")
```

### 完整可运行脚本（含异常处理）

```python
import sys
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.common.exceptions import WebDriverException

def main():
    options = Options()
    options.add_argument("--headless=new")
    options.add_argument("--window-size=1920,1080")
    options.add_argument("--disable-gpu")
    options.add_argument("--no-sandbox")

    driver = None
    try:
        driver = webdriver.Chrome(options=options)
        driver.get("https://www.python.org")
        print(f"Title: {driver.title}")
        driver.save_screenshot("result.png")
        print("截图已保存")
    except WebDriverException as e:
        print(f"WebDriver 异常: {e}", file=sys.stderr)
        sys.exit(1)
    finally:
        if driver:
            driver.quit()

if __name__ == "__main__":
    main()
```

## 常见坑

1. **Chrome 与 ChromeDriver 版本不匹配** — Selenium-Manager 自动解决；手动配置时主版本号必须一致（如 Chrome 120 对应 ChromeDriver 120）
2. **驱动未加入 PATH** — 手动模式下，驱动不在 PATH 中且未指定 `Service` 路径会报 `SessionNotCreatedException`
3. **未调用 `driver.quit()`** — 进程残留，端口占用，后续运行报 `WebDriverException: unknown error: Chrome failed to start`
4. **headless 模式窗口过小** — 未设置 `--window-size` 时默认 800x600，可能导致响应式布局下元素不可见
5. **Selenium-Manager 网络受限** — 企业内网环境下自动下载失败，需手动配置驱动路径

## 自测清单

- [ ] 能描述 WebDriver 协议的四层通信流程
- [ ] 能创建并激活 Python 虚拟环境
- [ ] 能安装 selenium 4.11+ 并验证版本
- [ ] 能编写含 headless 选项的完整脚本
- [ ] 能解释 Selenium-Manager 的作用与适用版本
- [ ] 能在网络受限时手动配置 ChromeDriver 路径
- [ ] 能列举至少 5 个常用 Chrome 启动选项

## 延伸阅读

- [W3C WebDriver 协议规范](https://www.w3.org/TR/webdriver/)
- [Selenium 官方文档](https://www.selenium.dev/documentation/)
- [Chrome for Testing 下载页](https://googlechromelabs.github.io/chrome-for-testing/)
- [Selenium-Manager 源码](https://github.com/SeleniumHQ/selenium/tree/trunk/common/manager)
