# 27-Playwright环境搭建
> 课时：35 min | 难度：★★☆☆☆

## 学习目标

- 理解 Playwright 与 Selenium 的架构差异与优劣势
- 完成 Playwright Python 环境的安装与浏览器下载
- 掌握 Browser Context 与 Page 的概念模型
- 能编写同步与异步 API 的 Playwright 脚本
- 理解 Playwright 的自动等待机制
- 能编写完整的 Playwright 入门脚本

## 核心概念

### Playwright vs Selenium

| 维度 | Playwright | Selenium |
|------|-----------|----------|
| 开发方 | Microsoft | 社区驱动 |
| 通信协议 | WebSocket（直连浏览器） | HTTP（WebDriver） |
| 自动等待 | 内置 | 需手动实现 |
| 执行速度 | 快（协议开销低） | 较慢 |
| 多浏览器 | Chromium/Firefox/WebKit | 依赖各浏览器驱动 |
| 移动端模拟 | 内置设备模拟 | 有限支持 |
| 网络拦截 | 原生支持 | 需第三方工具 |
| 录制功能 | Codegen 内置 | 需 IDE 插件 |
| 并行执行 | 原生 Browser Context 隔离 | 需手动管理 |
| 自动下载浏览器 | `playwright install` | 需手动管理驱动 |
| 社区生态 | 快速增长 | 成熟丰富 |
| 语言支持 | Python/JS/Java/C# | 多语言 |

### 核心架构

```
Playwright Script
       │
       ▼
  Playwright Server（Python 客户端）
       │  WebSocket
       ▼
  Browser Instance（浏览器进程）
       │
       ├── BrowserContext 1（独立会话，隔离 cookie/storage）
       │       ├── Page 1
       │       ├── Page 2
       │       └── ...
       │
       ├── BrowserContext 2
       │       ├── Page 3
       │       └── ...
       └── ...
```

**关键概念：**

| 概念 | 说明 |
|------|------|
| Browser | 浏览器进程实例 |
| BrowserContext | 独立的浏览器会话（类似无痕模式），隔离 cookie、localStorage、cache |
| Page | 浏览器标签页 |
| Frame | 页面中的 iframe |
| ElementHandle | DOM 元素引用（不推荐直接使用，推荐 Locator） |
| Locator | 元素定位器，Playwright 推荐的核心 API |

### 同步 API vs 异步 API

| 特性 | 同步 API (`sync_playwright`) | 异步 API (`async_playwright`) |
|------|---------------------------|------------------------------|
| 使用方式 | 直接调用 | `async/await` |
| 适用场景 | 简单脚本、pytest 集成 | 高并发、异步框架 |
| 性能 | 单线程 | 可并发（asyncio） |
| 学习曲线 | 低 | 中 |
| pytest 支持 | `pytest-playwright` | `pytest-asyncio` + `pytest-playwright` |

## 动手实操

### 安装与环境配置

```bash
# 1. 安装 Playwright Python 包
pip install playwright

# 2. 安装浏览器（Chromium + Firefox + WebKit）
playwright install

# 3. 仅安装 Chromium
playwright install chromium

# 4. 安装浏览器及系统依赖（Linux 需要）
playwright install-deps

# 5. 安装指定版本的浏览器
playwright install chromium@1000

# 6. 验证安装
playwright --version
```

```bash
# 可选：安装 pytest 插件
pip install pytest-playwright

# 完整安装（含测试框架）
pip install playwright pytest pytest-playwright
```

```ini
# requirements.txt
playwright==1.40.0
pytest==7.4.0
pytest-playwright==0.4.3
```

### 第一个脚本：同步 API

```python
from playwright.sync_api import sync_playwright

# 同步 API 入门脚本
with sync_playwright() as p:
    # 启动浏览器
    browser = p.chromium.launch(headless=False)

    # 创建新页面
    page = browser.new_page()

    # 导航到目标页面
    page.goto("https://example.com")

    # 打印页面标题
    print(f"页面标题: {page.title()}")

    # 关闭浏览器
    browser.close()
```

### 第一个脚本：异步 API

```python
import asyncio
from playwright.async_api import async_playwright


async def main():
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=False)
        page = await browser.new_page()
        await page.goto("https://example.com")
        print(f"页面标题: {await page.title()}")
        await browser.close()


asyncio.run(main())
```

### 核心 API 详解

```python
from playwright.sync_api import sync_playwright, expect

with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)
    context = browser.new_context(
        viewport={"width": 1920, "height": 1080},
        locale="zh-CN",
        timezone_id="Asia/Shanghai",
        geolocation={"longitude": 116.4074, "latitude": 39.9042},
        permissions=["geolocation"],
        user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) CustomAgent/1.0",
    )
    page = context.new_page()

    # === 导航 ===
    page.goto("https://example.com")
    page.goto("https://example.com", wait_until="networkidle")  # 等待网络空闲
    page.go_back()
    page.go_forward()
    page.reload()

    # === 元素定位（Locator）===
    # CSS 选择器
    page.locator("button.submit")
    # ID
    page.locator("#username")
    # XPath
    page.locator("//input[@name='email']")
    # 文本匹配
    page.locator("text=登录")
    # 精确文本
    page.locator("text='提交'")
    # 组合定位
    page.locator("div.form >> text=用户名")
    # 第 N 个匹配元素
    page.locator(".item >> nth=2")
    # 父元素
    page.locator("input >> ..")
    # 链式定位
    page.locator("div.card").locator("button")

    # === 元素操作 ===
    # 点击
    page.locator("#submit-btn").click()
    page.locator("#submit-btn").click(button="right")  # 右键
    page.locator("#submit-btn").dblclick()  # 双击
    page.locator("#submit-btn").click(modifiers=["Control"])  # Ctrl+点击

    # 输入
    page.locator("#username").fill("admin")  # 清空后输入
    page.locator("#search").type("hello", delay=100)  # 逐字输入

    # 复选框
    page.locator("#agree").check()
    page.locator("#agree").uncheck()
    page.locator("#agree").set_checked(True)

    # 下拉选择
    page.locator("#country").select_option("cn")
    page.locator("#country").select_option(label="中国")
    page.locator("#country").select_option(value="CN")
    page.locator("#country").select_option(index=0)
    # 多选
    page.locator("#skills").select_option(["python", "java"])

    # 文件上传
    page.locator("#file-input").set_inputs_files("test.pdf")
    page.locator("#file-input").set_inputs_files([
        "file1.pdf",
        "file2.pdf",
    ])
    page.locator("#file-input").set_inputs_files(
        name="report.pdf",
        mime_type="application/pdf",
        buffer=b"file content bytes",
    )

    # 焦点与悬停
    page.locator("#input").focus()
    page.locator(".menu-item").hover()
    page.locator("#input").press("Tab")
    page.locator("#input").press("Control+a")

    # 滚动
    page.locator("#bottom").scroll_into_view_if_needed()

    # === 获取信息 ===
    # 文本内容
    text = page.locator("#title").text_content()
    inner_text = page.locator("#title").inner_text()
    # 属性值
    href = page.locator("a.link").get_attribute("href")
    # 输入值
    value = page.locator("#username").input_value()
    # 是否可见/可用
    visible = page.locator("#popup").is_visible()
    enabled = page.locator("#submit").is_enabled()
    checked = page.locator("#agree").is_checked()

    # === 断言（推荐使用 expect）===
    expect(page.locator("#title")).to_have_text("欢迎")
    expect(page.locator("#title")).to_contain_text("欢迎")
    expect(page.locator("#username")).to_have_value("admin")
    expect(page.locator("#agree")).to_be_checked()
    expect(page.locator("#popup")).to_be_visible()
    expect(page.locator("#submit")).to_be_enabled()
    expect(page).to_have_title("首页")
    expect(page).to_have_url("https://example.com/home")

    # === 等待 ===
    page.wait_for_load_state("domcontentloaded")
    page.wait_for_load_state("networkidle")
    page.wait_for_selector(".result", state="visible", timeout=10000)
    page.wait_for_url("**/dashboard")
    page.wait_for_event("response")

    browser.close()
```

### 自动等待机制

```python
from playwright.sync_api import sync_playwright, expect

with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)
    page = browser.new_page()
    page.goto("https://example.com")

    # Playwright 自动等待以下条件满足后才执行操作：
    # 1. 元素 attached（存在于 DOM）
    # 2. 元素 visible（可见）
    # 3. 元素 stable（不再动画中）
    # 4. 元素 enabled（可交互）
    # 5. 元素 receivable（可接收事件，如未被遮罩）

    # 以下 click 调用会自动等待元素可点击
    page.locator("#dynamic-btn").click()

    # 以下 fill 调用会自动等待元素可输入
    page.locator("#async-input").fill("自动等待的值")

    # 断言也内置自动重试
    expect(page.locator("#status")).to_have_text("完成", timeout=10000)

    # 等待策略可配置
    page.locator("#btn").click(timeout=5000)  # 自定义超时
    page.locator("#btn").click(force=True)    # 跳过所有等待（强制操作）

    browser.close()
```

### Headless vs Headed 模式

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    # Headless 模式（无界面，CI/CD 推荐）
    browser = p.chromium.launch(headless=True)

    # Headed 模式（有界面，调试推荐）
    browser = p.chromium.launch(headless=False)

    # 慢动作模式（调试用，每个操作间隔指定毫秒）
    browser = p.chromium.launch(headless=False, slow_mo=500)

    # 完整启动参数
    browser = p.chromium.launch(
        headless=True,
        slow_mo=0,
        args=[
            "--disable-blink-features=AutomationControlled",
            "--no-sandbox",
            "--disable-dev-shm-usage",
            "--window-size=1920,1080",
        ],
    )
```

### 完整入门脚本

```python
"""
Playwright 完整入门脚本
功能：登录电商平台，搜索商品，加入购物车
"""
from playwright.sync_api import sync_playwright, expect
import time


def test_e2e_shopping():
    with sync_playwright() as p:
        # 启动浏览器
        browser = p.chromium.launch(
            headless=False,
            slow_mo=100,
        )

        # 创建浏览器上下文
        context = browser.new_context(
            viewport={"width": 1920, "height": 1080},
            locale="zh-CN",
        )

        # 创建页面
        page = context.new_page()

        try:
            # 1. 打开首页
            page.goto("https://demo-shop.example.com")
            expect(page).to_have_title("Demo Shop")

            # 2. 登录
            page.locator("text=登录").click()
            page.locator("#email").fill("test@example.com")
            page.locator("#password").fill("password123")
            page.locator("button[type=submit]").click()

            # 3. 验证登录成功
            expect(page.locator(".user-name")).to_have_text("测试用户")

            # 4. 搜索商品
            page.locator("#search-input").fill("无线耳机")
            page.locator("#search-btn").click()

            # 5. 等待搜索结果
            expect(page.locator(".product-list")).to_be_visible()
            expect(page.locator(".product-item")).to_have_count(10)

            # 6. 点击第一个商品
            page.locator(".product-item >> nth=0").click()

            # 7. 选择规格
            page.locator("text=黑色").click()
            page.locator("#quantity").fill("2")

            # 8. 加入购物车
            page.locator("#add-to-cart").click()

            # 9. 验证加入成功
            expect(page.locator(".cart-count")).to_have_text("2")
            expect(page.locator(".toast-success")).to_be_visible()

            # 10. 截图保存
            page.screenshot(path="screenshots/cart-success.png")

            print("测试通过：完整购物流程")

        except Exception as e:
            # 失败时截图
            page.screenshot(path="screenshots/failure.png", full_page=True)
            raise

        finally:
            context.close()
            browser.close()


if __name__ == "__main__":
    test_e2e_shopping()
```

### Codegen 录制功能

```bash
# 启动录制工具（自动生成脚本）
playwright codegen https://example.com

# 录制并指定输出文件
playwright codegen https://example.com -o test_script.py

# 指定浏览器
playwright codegen https://example.com --browser chromium

# 指定语言
playwright codegen https://example.com --target python

# 带状态录制（先登录再录制）
playwright codegen https://example.com --save-storage=auth.json

# 加载已有状态录制
playwright codegen https://example.com --load-storage=auth.json

# 指定视口大小
playwright codegen https://example.com --viewport-size 1920,1080
```

## 常见坑

1. **浏览器未安装**：`pip install playwright` 后必须执行 `playwright install` 下载浏览器二进制文件，否则报错 `BrowserType.launch: Executable doesn't exist`。

2. **Linux 系统依赖缺失**：Linux 环境需运行 `playwright install-deps` 安装系统级依赖库，否则浏览器启动失败。

3. **headless 模式被检测**：部分网站检测 headless 浏览器。可通过 `channel="chrome"` 使用系统安装的 Chrome，或添加 `--disable-blink-features=AutomationControlled` 参数。

4. **Context 与 Page 混淆**：`browser.new_page()` 在默认 context 中创建页面。需要隔离会话时应使用 `browser.new_context()` 创建独立 context，再在其中创建 page。

5. **同步/异步 API 混用**：`sync_playwright()` 和 `async_playwright()` 不能在同一脚本中混用。选择一种模式并保持一致。

6. **fill vs type**：`fill` 清空后输入，`type` 在现有内容后追加。大多数表单场景应使用 `fill`。

7. **自动等待的 force 参数**：`force=True` 跳过所有等待和可行性检查，仅在确定元素可交互时使用，否则可能引发不可预期的错误。

8. **资源未释放**：脚本异常退出时浏览器进程可能残留。使用 `with` 语句或 `try/finally` 确保 `browser.close()` 执行。

## 自测清单

- [ ] 能完成 Playwright 安装与浏览器下载
- [ ] 能解释 Browser、BrowserContext、Page 的关系
- [ ] 能使用同步 API 编写基本脚本
- [ ] 能使用异步 API 编写基本脚本
- [ ] 能使用 Locator 进行元素定位
- [ ] 能执行 click/fill/check/uncheck/select_option 操作
- [ ] 能使用 expect 进行断言
- [ ] 能理解 Playwright 的自动等待机制
- [ ] 能使用 Codegen 录制脚本
- [ ] 能配置 headless/headed 模式

## 延伸阅读

- [Playwright Python 官方文档](https://playwright.dev/python/docs/intro)
- [Playwright Locator API](https://playwright.dev/python/docs/locators)
- [Playwright Auto-waiting](https://playwright.dev/python/docs/actionability)
- [Playwright Codegen](https://playwright.dev/python/docs/codegen)
- [Playwright 设备模拟](https://playwright.dev/python/docs/emulation)
