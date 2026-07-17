# 28-Playwright高级
> 课时：50 min | 难度：★★★★☆

## 学习目标

- 掌握多浏览器测试（Chromium/Firefox/WebKit）的配置与执行
- 能录制和查看 Trace 文件用于调试
- 能实现视觉对比测试（截图 diff）
- 能拦截和模拟网络请求
- 能处理文件上传/下载场景
- 能管理认证状态（storage state）
- 能使用 pytest-playwright 实现并行执行

## 核心概念

### 多浏览器测试

| 浏览器 | 引擎 | 适用场景 |
|--------|------|----------|
| Chromium | Blink | 主要测试目标（Chrome/Edge 兼容） |
| Firefox | Gecko | 跨浏览器兼容性验证 |
| WebKit | Safari 引擎 | 移动端 Safari 兼容性验证 |

### Trace 机制

Trace 记录测试执行过程中的完整信息：DOM 快照、网络请求、控制台日志、截图序列。通过 Playwright Trace Viewer 可回放整个测试过程。

### 网络拦截

Playwright 提供 `page.route()` 和 `context.route()` 拦截网络请求，支持：
- 模拟 API 响应（mock）
- 阻止资源加载（图片/字体/样式）
- 修改请求/响应
- 模拟网络错误

### Storage State

`context.storage_state()` 导出当前上下文的所有存储数据（cookies、localStorage、sessionStorage），可在后续测试中复用，实现免登录测试。

## 动手实操

### 多浏览器测试

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    # Chromium
    chromium = p.chromium.launch(headless=True)
    page = chromium.new_page()
    page.goto("https://example.com")
    print(f"Chromium: {page.title()}")
    chromium.close()

    # Firefox
    firefox = p.firefox.launch(headless=True)
    page = firefox.new_page()
    page.goto("https://example.com")
    print(f"Firefox: {page.title()}")
    firefox.close()

    # WebKit
    webkit = p.webkit.launch(headless=True)
    page = webkit.new_page()
    page.goto("https://example.com")
    print(f"WebKit: {page.title()}")
    webkit.close()
```

```python
# conftest.py - pytest-playwright 多浏览器配置
import pytest
from playwright.sync_api import sync_playwright


@pytest.fixture(params=["chromium", "firefox", "webkit"])
def browser_type(request):
    return request.param


@pytest.fixture
def browser(browser_type):
    with sync_playwright() as p:
        browser = getattr(p, browser_type).launch(headless=True)
        yield browser
        browser.close()


@pytest.fixture
def page(browser):
    context = browser.new_context()
    page = context.new_page()
    yield page
    context.close()
```

```python
# pytest.ini - 多浏览器配置
[pytest]
addopts = --browser chromium --browser firefox --browser webkit
```

```bash
# 命令行指定多个浏览器
pytest --browser chromium --browser firefox --browser webkit

# 仅运行特定浏览器
pytest --browser chromium
```

### Traces：录制与查看

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    context = browser.new_context()

    # 开始录制 Trace
    context.tracing.start(screenshots=True, snapshots=True, sources=True)

    page = context.new_page()
    page.goto("https://example.com")
    page.locator("#login").click()
    page.locator("#username").fill("admin")
    page.locator("#password").fill("password")
    page.locator("#submit").click()

    # 保存 Trace 文件
    context.tracing.stop(path="traces/test-trace.zip")

    context.close()
    browser.close()
```

```bash
# 查看 Trace 文件
playwright show-trace traces/test-trace.zip

# 查看远程 Trace（从 URL 加载）
playwright show-trace https://example.com/trace.zip
```

```python
# conftest.py - 自动录制失败用例的 Trace
import pytest


@pytest.fixture(autouse=True)
def trace_on_failure(request, page):
    """失败时自动保存 Trace"""
    yield
    if request.node.rep_call.failed:
        page.context.tracing.stop(
            path=f"traces/failure-{request.node.name}.zip"
        )
```

```ini
# pytest.ini - Trace 配置
[pytest]
addopts = --tracing=retain-on-failure
# 选项：on / off / retain-on-failure / retry-with-trace
```

### 视觉对比测试

```python
from playwright.sync_api import sync_playwright, expect

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    page.goto("https://example.com")

    # 整页截图对比
    expect(page).to_have_screenshot("baseline/homepage.png")

    # 元素截图对比
    expect(page.locator(".hero-section")).to_have_screenshot("baseline/hero.png")

    # 设置对比阈值
    expect(page).to_have_screenshot(
        "baseline/homepage.png",
        max_diff_pixels=100,        # 最大差异像素数
        max_diff_pixel_ratio=0.01,  # 最大差异比例 1%
    )

    browser.close()
```

```python
# conftest.py - 视觉对比全局配置
import pytest
from playwright.sync_api import Page


@pytest.fixture(autouse=True)
def configure_screenshot_thresholds():
    """配置截图对比阈值"""
    Page.set_default_timeout(10000)


# pytest.ini 配置
# [pytest]
# addopts = --screenshot=only-on-failure --full-page-screenshot
```

```python
# 像素级对比工具函数
from PIL import Image
import pixelmatch
import numpy as np


def compare_screenshots(baseline_path, current_path, diff_path, threshold=0.1):
    """使用 pixelmatch 进行像素级对比"""
    baseline = Image.open(baseline_path).convert("RGBA")
    current = Image.open(current_path).convert("RGBA")

    width, height = baseline.size

    diff = Image.new("RGBA", (width, height))

    num_diff_pixels = pixelmatch.pixelmatch(
        np.array(baseline),
        np.array(current),
        np.array(diff),
        width,
        height,
        {"threshold": threshold, "includeAA": True},
    )

    diff.save(diff_path)

    total_pixels = width * height
    diff_ratio = num_diff_pixels / total_pixels

    return {
        "diff_pixels": num_diff_pixels,
        "total_pixels": total_pixels,
        "diff_ratio": diff_ratio,
        "passed": diff_ratio < 0.01,  # 差异 < 1% 视为通过
    }
```

### 网络拦截与模拟

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()

    # 1. 模拟 API 响应
    page.route("**/api/users", lambda route: route.fulfill(
        status=200,
        content_type="application/json",
        body='[{"id": 1, "name": "Mock User"}]',
    ))

    # 2. 模拟延迟响应
    page.route("**/api/data", lambda route: route.fulfill(
        status=200,
        body="delayed response",
        # 延迟通过 page.route 的 delay 参数
    ))

    # 3. 阻止资源加载（加速测试）
    page.route("**/*.{png,jpg,jpeg,gif,svg}", lambda route: route.abort())
    page.route("**/analytics.js", lambda route: route.abort())
    page.route("**/fonts/**", lambda route: route.abort())

    # 4. 修改请求
    page.route("**/api/**", lambda route: route.continue_(
        headers={**route.request.headers, "X-Test-Header": "automation"},
    ))

    # 5. 修改响应
    def modify_response(route):
        response = route.fetch()
        body = response.text()
        # 修改响应体
        modified_body = body.replace('"status": "active"', '"status": "test"')
        route.fulfill(
            response=response,
            body=modified_body,
        )

    page.route("**/api/status", modify_response)

    # 6. 模拟网络错误
    page.route("**/api/critical", lambda route: route.abort("connectionfailed"))

    # 7. 条件拦截
    def conditional_mock(route):
        if "error" in route.request.url:
            route.fulfill(status=500, body="Server Error")
        else:
            route.continue_()

    page.route("**/api/**", conditional_mock)

    # 8. 获取请求/响应数据
    responses = []
    page.on("response", lambda response: responses.append({
        "url": response.url,
        "status": response.status,
    }))

    page.goto("https://example.com")
    page.locator("#load-data").click()

    # 等待特定请求完成
    with page.expect_response("**/api/data") as response_info:
        page.locator("#refresh").click()
    response = response_info.value
    print(f"Status: {response.status}")
    print(f"Body: {response.json()}")

    # 等待特定请求发出（不等待响应）
    with page.expect_request("**/api/submit") as request_info:
        page.locator("#submit").click()
    request = request_info.value
    print(f"Method: {request.method}")
    print(f"Post Data: {request.post_data}")

    browser.close()
```

### 文件下载与上传

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    context = browser.new_context(accept_downloads=True)
    page = context.new_page()

    # === 文件下载 ===
    page.goto("https://example.com/download")

    # 等待下载完成并保存
    with page.expect_download() as download_info:
        page.locator("#download-btn").click()
    download = download_info.value

    # 获取下载信息
    print(f"文件名: {download.suggested_filename}")
    print(f"URL: {download.url}")

    # 保存到指定路径
    download.save_as(f"downloads/{download.suggested_filename}")

    # 等待下载完成（大文件场景）
    download_path = download.path()  # 临时文件路径
    print(f"临时路径: {download_path}")

    # === 文件上传 ===
    page.goto("https://example.com/upload")

    # 单文件上传
    page.locator("#file-input").set_input_files("test-files/report.pdf")

    # 多文件上传
    page.locator("#file-input").set_input_files([
        "test-files/file1.pdf",
        "test-files/file2.pdf",
    ])

    # 内存文件上传（无需真实文件）
    page.locator("#file-input").set_input_files(
        name="test.txt",
        mime_type="text/plain",
        buffer=b"Hello, this is test content",
    )

    # 拖拽上传
    page.locator("#drop-zone").set_input_files("test-files/drag-file.pdf")

    # 移除已选文件
    page.locator("#file-input").set_input_files([])

    context.close()
    browser.close()
```

### 认证与 Storage State

```python
from playwright.sync_api import sync_playwright
import json

# === 第一步：登录并保存状态 ===
with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)
    context = browser.new_context()
    page = context.new_page()

    page.goto("https://example.com/login")
    page.locator("#username").fill("admin")
    page.locator("#password").fill("password123")
    page.locator("#login-btn").click()

    # 等待登录完成
    page.wait_for_url("**/dashboard")

    # 保存存储状态（cookies + localStorage + sessionStorage）
    context.storage_state(path="auth/auth-state.json")

    context.close()
    browser.close()

# === 第二步：后续测试复用状态 ===
with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)

    # 加载已保存的认证状态
    context = browser.new_context(storage_state="auth/auth-state.json")
    page = context.new_page()

    # 直接访问需要登录的页面
    page.goto("https://example.com/dashboard")
    # 无需再次登录

    page.locator("#profile").click()
    # ... 执行需要认证的测试

    context.close()
    browser.close()
```

```python
# conftest.py - 认证状态管理
import pytest
import os
from playwright.sync_api import sync_playwright


@pytest.fixture(scope="session")
def auth_state():
    """生成认证状态文件"""
    state_path = "auth/auth-state.json"

    if not os.path.exists(state_path):
        with sync_playwright() as p:
            browser = p.chromium.launch(headless=True)
            context = browser.new_context()
            page = context.new_page()

            page.goto("https://example.com/login")
            page.locator("#username").fill("admin")
            page.locator("#password").fill("password123")
            page.locator("#login-btn").click()
            page.wait_for_url("**/dashboard")

            context.storage_state(path=state_path)
            context.close()
            browser.close()

    return state_path


@pytest.fixture
def authenticated_context(browser, auth_state):
    """创建已认证的浏览器上下文"""
    context = browser.new_context(storage_state=auth_state)
    yield context
    context.close()


@pytest.fixture
def authenticated_page(authenticated_context):
    """创建已认证的页面"""
    page = authenticated_context.new_page()
    yield page
```

### pytest-playwright 并行执行

```python
# conftest.py
import pytest


@pytest.fixture(scope="session")
def browser_context_args(browser_context_args):
    """全局浏览器上下文配置"""
    return {
        **browser_context_args,
        "viewport": {"width": 1920, "height": 1080},
        "locale": "zh-CN",
        "timezone_id": "Asia/Shanghai",
    }
```

```ini
# pytest.ini
[pytest]
addopts =
    -n auto
    --browser chromium
    --browser firefox
    --tracing=retain-on-failure
    --screenshot=only-on-failure
    --full-page-screenshot
    --video=retain-on-failure
```

```bash
# 并行执行（自动检测 CPU 核心数）
pytest -n auto

# 指定进程数
pytest -n 4

# 并行 + 多浏览器
pytest -n 4 --browser chromium --browser firefox

# 并行 + 失败重跑
pytest -n 4 --reruns 2
```

### 完整高级测试套件

```python
"""
完整高级测试套件示例
功能：多浏览器 + 网络拦截 + 视觉对比 + 认证复用
"""
import pytest
from playwright.sync_api import Page, expect, Route
import json


class TestEcommerceAdvanced:
    """电商高级测试套件"""

    @pytest.fixture(autouse=True)
    def setup(self, page: Page):
        """每个测试的前置设置"""
        # 阻止分析脚本
        page.route("**/analytics/**", lambda route: route.abort())
        page.route("**/tracking/**", lambda route: route.abort())

        # 模拟支付 API
        page.route("**/api/payment", lambda route: route.fulfill(
            status=200,
            content_type="application/json",
            body=json.dumps({
                "success": True,
                "transaction_id": "mock-txn-12345",
                "amount": 99.00,
            }),
        ))

        self.page = page

    def test_homepage_visual_regression(self, page: Page):
        """首页视觉回归测试"""
        page.goto("https://demo-shop.example.com")

        # 等待关键元素加载
        expect(page.locator(".hero-banner")).to_be_visible()
        expect(page.locator(".product-grid")).to_be_visible()

        # 视觉对比
        expect(page).to_have_screenshot(
            "baseline/homepage.png",
            max_diff_pixel_ratio=0.02,
        )

    def test_product_search_with_mock(self, page: Page):
        """商品搜索（模拟 API）"""
        # 模拟搜索 API
        page.route("**/api/search*", lambda route: route.fulfill(
            status=200,
            content_type="application/json",
            body=json.dumps({
                "results": [
                    {"id": 1, "name": "Mock Product A", "price": 29.99},
                    {"id": 2, "name": "Mock Product B", "price": 49.99},
                    {"id": 3, "name": "Mock Product C", "price": 19.99},
                ],
                "total": 3,
            }),
        ))

        page.goto("https://demo-shop.example.com")
        page.locator("#search-input").fill("耳机")
        page.locator("#search-btn").click()

        # 验证搜索结果
        expect(page.locator(".search-result-item")).to_have_count(3)
        expect(page.locator(".search-result-item >> nth=0")).to_contain_text("Mock Product A")

    def test_add_to_cart_flow(self, authenticated_page: Page):
        """加入购物车流程（已认证）"""
        page = authenticated_page
        page.goto("https://demo-shop.example.com/product/1")

        # 选择规格
        page.locator("text=黑色").click()
        page.locator("text=128GB").click()

        # 设置数量
        page.locator("#quantity").fill("2")

        # 加入购物车
        page.locator("#add-to-cart").click()

        # 验证成功提示
        expect(page.locator(".toast-success")).to_be_visible()
        expect(page.locator(".cart-count")).to_have_text("2")

    def test_checkout_with_payment_mock(self, authenticated_page: Page):
        """结账流程（模拟支付）"""
        page = authenticated_page
        page.goto("https://demo-shop.example.com/cart")

        # 进入结账
        page.locator("#checkout-btn").click()

        # 填写地址
        page.locator("#address").fill("测试地址")
        page.locator("#city").fill("北京市")
        page.locator("#phone").fill("13800138000")

        # 提交订单（支付 API 已被 mock）
        page.locator("#place-order-btn").click()

        # 验证订单成功
        expect(page.locator(".order-success")).to_be_visible()
        expect(page.locator(".transaction-id")).to_contain_text("mock-txn")

    def test_file_upload(self, page: Page):
        """文件上传测试"""
        page.goto("https://demo-shop.example.com/upload")

        # 上传文件
        page.locator("#file-input").set_input_files(
            name="test-product.jpg",
            mime_type="image/jpeg",
            buffer=b"\xff\xd8\xff\xe0" + b"\x00" * 100,  # 模拟 JPEG
        )

        page.locator("#upload-btn").click()

        # 验证上传成功
        expect(page.locator(".upload-success")).to_be_visible()

    def test_network_error_handling(self, page: Page):
        """网络错误处理测试"""
        # 模拟网络断开
        page.route("**/api/critical", lambda route: route.abort("connectionfailed"))

        page.goto("https://demo-shop.example.com")
        page.locator("#load-critical-data").click()

        # 验证错误提示
        expect(page.locator(".error-message")).to_be_visible()
        expect(page.locator(".error-message")).to_contain_text("网络错误")

    def test_responsive_layout(self, page: Page):
        """响应式布局测试"""
        # 移动端视口
        page.set_viewport_size({"width": 375, "height": 812})
        page.goto("https://demo-shop.example.com")

        expect(page).to_have_screenshot("baseline/homepage-mobile.png")

        # 平板视口
        page.set_viewport_size({"width": 768, "height": 1024})
        page.goto("https://demo-shop.example.com")

        expect(page).to_have_screenshot("baseline/homepage-tablet.png")

        # 桌面视口
        page.set_viewport_size({"width": 1920, "height": 1080})
        page.goto("https://demo-shop.example.com")

        expect(page).to_have_screenshot("baseline/homepage-desktop.png")
```

## 常见坑

1. **Trace 文件过大**：长时间测试的 Trace 文件可达数百 MB。生产环境使用 `retain-on-failure` 策略，仅保留失败用例的 Trace。

2. **视觉对比的基线管理**：基线截图需纳入版本控制。不同分辨率/浏览器生成的基线不同，需为每个环境维护独立基线。

3. **网络拦截的 route 顺序**：`page.route()` 按注册顺序匹配，先注册的先匹配。通用规则应放在后面，具体规则放在前面。

4. **Storage State 过期**：认证状态有有效期。长时间运行的 CI 流程中，token 可能过期。需定期刷新 `auth-state.json` 或实现自动重新登录逻辑。

5. **文件下载路径**：`download.path()` 返回临时文件路径，测试结束后可能被清理。务必在测试中调用 `download.save_as()` 保存到持久路径。

6. **多浏览器视觉差异**：不同浏览器的渲染引擎存在像素级差异。跨浏览器视觉对比需设置更高的 `max_diff_pixel_ratio` 阈值。

7. **并行执行中的 route 隔离**：`page.route()` 仅在当前 Page 生效。并行测试中各 Page 的拦截规则互不干扰，但 `context.route()` 会影响该 Context 下所有 Page。

8. **模拟 API 的 content_type**：`route.fulfill()` 必须设置正确的 `content_type`，否则前端可能无法正确解析响应。

## 自测清单

- [ ] 能配置并执行 Chromium/Firefox/WebKit 多浏览器测试
- [ ] 能录制和查看 Trace 文件
- [ ] 能实现截图对比测试
- [ ] 能拦截和模拟网络请求
- [ ] 能处理文件上传和下载
- [ ] 能管理认证状态（storage state）
- [ ] 能使用 pytest-playwright 实现并行执行
- [ ] 能编写完整的高级测试套件
- [ ] 能配置视觉对比的容差阈值

## 延伸阅读

- [Playwright Python - 多浏览器](https://playwright.dev/python/docs/browsers)
- [Playwright Trace Viewer](https://playwright.dev/python/docs/trace-viewer)
- [Playwright 视觉对比](https://playwright.dev/python/docs/test-snapshots)
- [Playwright 网络拦截](https://playwright.dev/python/docs/network)
- [Playwright 认证](https://playwright.dev/python/docs/auth)
- [pytest-playwright 文档](https://playwright.dev/python/docs/test-runners)
