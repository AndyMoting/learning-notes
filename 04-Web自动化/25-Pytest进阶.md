# 25-Pytest进阶
> 课时：45 min | 难度：★★★☆☆

## 学习目标

- 理解 `conftest.py` 的作用域与 fixture 共享机制
- 掌握 `@pytest.mark` 系列标记的完整用法
- 能够配置 pytest 插件（html、cov、rerun、xdist、allure）
- 编写自定义 marker 并通过 `pytest.ini` 注册
- 使用 pytest-xdist 实现并行测试执行

## 核心概念

### conftest.py

`conftest.py` 是 pytest 的特殊文件，用于定义公共 fixture、钩子函数和插件配置。pytest 自动发现该文件，无需显式导入。

**作用域规则：**
- 根目录 `conftest.py`：全局有效
- 子目录 `conftest.py`：仅对该目录及子目录有效
- 就近原则：子目录覆盖父目录同名 fixture

**Fixture 作用域（scope）：**

| scope | 说明 |
|-------|------|
| `function` | 每个测试函数执行一次（默认） |
| `class` | 每个测试类执行一次 |
| `module` | 每个模块执行一次 |
| `package` | 每个包执行一次 |
| `session` | 整个测试会话执行一次 |

### @pytest.mark 标记体系

| 标记 | 功能 |
|------|------|
| `skip` | 无条件跳过 |
| `skipif` | 条件满足时跳过 |
| `xfail` | 预期失败 |
| `parametrize` | 参数化测试 |
| `timeout` | 超时控制 |
| `ordering` | 执行顺序控制（需插件） |

### pytest 插件生态

| 插件 | 用途 |
|------|------|
| `pytest-html` | HTML 测试报告生成 |
| `pytest-cov` | 代码覆盖率统计 |
| `pytest-rerunfailures` | 失败用例自动重跑 |
| `pytest-xdist` | 分布式/并行执行 |
| `allure-pytest` | Allure 报告集成 |

## 动手实操

### conftest.py 共享 Fixture

```python
# conftest.py
import pytest
from selenium import webdriver
from selenium.webdriver.chrome.options import Options


@pytest.fixture(scope="session")
def browser():
    """整个会话共享一个浏览器实例"""
    options = Options()
    options.add_argument("--headless")
    options.add_argument("--no-sandbox")
    options.add_argument("--disable-dev-shm-usage")
    driver = webdriver.Chrome(options=options)
    driver.implicitly_wait(10)
    driver.maximize_window()
    yield driver
    driver.quit()


@pytest.fixture(scope="function")
def login_page(browser):
    """每个测试函数重新登录"""
    browser.get("https://example.com/login")
    browser.find_element("id", "username").send_keys("admin")
    browser.find_element("id", "password").send_keys("password123")
    browser.find_element("id", "login-btn").click()
    return browser


@pytest.fixture(scope="module")
def db_connection(scope="module"):
    """模块级别数据库连接"""
    conn = create_connection("test_db")
    yield conn
    conn.close()


@pytest.fixture(autouse=True)
def setup_teardown():
    """自动执行的 fixture，无需测试函数引用"""
    print("\n[Setup] 测试开始")
    yield
    print("\n[Teardown] 测试结束")
```

**子目录覆盖父目录 fixture：**

```python
# tests/sub/conftest.py
import pytest


@pytest.fixture
def username():
    """覆盖全局的 username fixture"""
    return "subdir_user"
```

### @pytest.mark 标记完整用法

```python
import pytest
import sys
import time


class TestMarkers:
    """标记用法完整示例"""

    @pytest.mark.skip(reason="功能尚未实现")
    def test_new_feature(self):
        assert False, "不应执行到这里"

    @pytest.mark.skipif(
        sys.platform == "win32",
        reason="Windows 平台暂不支持此功能"
    )
    def test_unix_only(self):
        assert True

    @pytest.mark.skipif(
        sys.version_info < (3, 10),
        reason="需要 Python 3.10+"
    )
    def test_match_statement(self):
        value = "test"
        match value:
            case "test":
                assert True

    @pytest.mark.xfail(reason="已知 Bug: #1234，待修复")
    def test_known_bug(self):
        assert 1 == 2  # 标记为预期失败

    @pytest.mark.xfail(strict=True)
    def test_strict_xfail(self):
        """strict=True 时如果测试通过则标记为 XPASS-FAILED"""
        assert True

    @pytest.mark.parametrize("username,password,expected", [
        ("admin", "admin123", True),
        ("user", "user123", True),
        ("admin", "wrong", False),
        ("", "", False),
        ("admin' OR '1'='1", "anything", False),
    ], ids=["admin-valid", "user-valid", "wrong-password", "empty-input", "sql-injection"])
    def test_login(self, username, password, expected):
        result = login(username, password)
        assert result == expected

    @pytest.mark.parametrize("a,b,expected", [
        (1, 2, 3),
        (0, 0, 0),
        (-1, 1, 0),
    ])
    @pytest.mark.parametrize("x,y,product", [
        (2, 3, 6),
        (0, 5, 0),
    ])
    def test_parametrize_cartesian(self, a, b, expected, x, y, product):
        """笛卡尔积参数化：3*2=6 组测试"""
        assert a + b == expected
        assert x * y == product

    @pytest.mark.timeout(5)
    def test_with_timeout(self):
        """5秒超时（需 pytest-timeout 插件）"""
        time.sleep(10)
        assert False

    @pytest.mark.order(2)
    def test_second(self):
        """第二个执行（需 pytest-ordering 插件）"""
        assert True

    @pytest.mark.order(1)
    def test_first(self):
        """第一个执行"""
        assert True

    @pytest.mark.order(3)
    def test_third(self):
        """第三个执行"""
        assert True
```

### 自定义 Marker 与 pytest.ini 配置

```ini
# pytest.ini
[pytest]
minversion = 7.0
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts =
    -v
    --strict-markers
    --tb=short
    --html=reports/report.html
    --self-contained-html
    --cov=src
    --cov-report=html:reports/coverage
    --cov-report=term-missing
    --reruns=2
    --reruns-delay=1
markers =
    smoke: 冒烟测试用例
    regression: 回归测试用例
    e2e: 端到端测试用例
    slow: 执行缓慢的测试（>30s）
    flaky: 不稳定测试，允许重试
    security: 安全相关测试
    api: API 接口测试
    ui: UI 界面测试
    critical: 关键路径测试
filterwarnings =
    ignore::DeprecationWarning
```

```python
# 使用自定义 marker
import pytest


@pytest.mark.smoke
@pytest.mark.critical
def test_login_smoke():
    """冒烟测试：登录功能"""
    assert login("admin", "admin123")


@pytest.mark.regression
@pytest.mark.slow
def test_full_checkout_flow():
    """回归测试：完整下单流程"""
    pass


@pytest.mark.security
def test_xss_prevention():
    """安全测试：XSS 防护"""
    payload = '<script>alert("xss")</script>'
    result = submit_comment(payload)
    assert "<script>" not in result


# 按 marker 运行
# pytest -m smoke
# pytest -m "smoke and not slow"
# pytest -m "regression or e2e"
# pytest -m "not flaky"
```

### pytest-html 报告生成与定制

```python
# conftest.py - 定制 HTML 报告
import pytest
from datetime import datetime


def pytest_html_report_title(report):
    """自定义报告标题"""
    report.title = "Web自动化测试报告"


def pytest_configure(config):
    """添加自定义元数据"""
    config._metadata = {
        "项目": "电商平台",
        "版本": "v2.5.0",
        "环境": "Staging",
        "执行时间": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
        "执行人": "QA Team",
    }


@pytest.hookimpl(optionalhook=True)
def pytest_metadata(metadata):
    """移除不需要的默认元数据"""
    metadata.pop("Packages", None)
    metadata.pop("Plugins", None)


def pytest_html_results_summary(prefix, summary, postfix):
    """自定义报告摘要"""
    prefix.extend([
        "<p>测试范围：登录、注册、下单、支付</p>",
        f"<p>通过率：{calculate_pass_rate()}%</p>",
    ])


@pytest.hookimpl(hookimpl=True)
def pytest_runtest_makereport(item, call):
    """在报告中添加自定义列"""
    if call.when == "call":
        # 添加描述列
        item._report_sections = getattr(item, "_report_sections", [])
        item._report_sections.append(
            ("Description", item.function.__doc__ or "无描述")
        )
```

```ini
# pytest.ini - HTML 报告配置
[pytest]
addopts = --html=reports/report.html --self-contained-html
```

### pytest-xdist 并行执行

```python
# conftest.py - 并行执行安全配置
import pytest
import threading

# 线程锁，保护共享资源
_db_lock = threading.Lock()


@pytest.fixture(scope="session")
def shared_resource():
    """线程安全的共享资源"""
    with _db_lock:
        resource = create_expensive_resource()
    yield resource
    with _db_lock:
        cleanup_resource(resource)
```

```bash
# 基本并行：自动检测 CPU 核心数
pytest -n auto

# 指定进程数
pytest -n 4

# 按 CPU 核心数分配
pytest -n logical

# 分布式执行（多机器）
pytest --dist=loadscope -n 4

# 按模块分配（减少 setup 开销）
pytest --dist=loadfile -n 4

# 结合参数化并行
pytest -n 4 --maxprocessrestart=10

# 并行 + 失败重跑
pytest -n 4 --reruns 2 --reruns-delay 1
```

```ini
# pytest.ini - xdist 配置
[pytest]
addopts = -n auto --dist=loadscope
```

### pytest-cov 覆盖率配置

```ini
# .coveragerc
[run]
source = src
branch = True
omit =
    */tests/*
    */migrations/*
    */__pycache__/*
    */venv/*

[report]
precision = 2
show_missing = True
skip_covered = False
exclude_lines =
    pragma: no cover
    def __repr__
    raise NotImplementedError
    if __name__ == .__main__.:
    if TYPE_CHECKING:

[html]
html_dir = reports/coverage
```

```bash
# 运行并生成覆盖率报告
pytest --cov=src --cov-report=html --cov-report=term-missing

# 查看未覆盖的行
pytest --cov=src --cov-report=term-missing --cov-fail-under=80

# 并行 + 覆盖率
pytest -n auto --cov=src --cov-report=html
```

### allure-pytest 集成

```python
import allure
import pytest


@allure.epic("电商平台")
@allure.feature("用户管理")
class TestUserManagement:

    @allure.story("用户注册")
    @allure.severity(allure.severity_level.CRITICAL)
    @allure.tag("smoke", "regression")
    @allure.description("验证新用户注册流程")
    @allure.link("https://wiki.example.com/register", name="需求文档")
    @allure.issue("BUG-123", name="已知问题")
    def test_user_registration(self):
        with allure.step("准备测试数据"):
            email = "test@example.com"
            password = "SecurePass123!"

        with allure.step("提交注册表单"):
            result = register(email, password)

        with allure.step("验证注册结果"):
            assert result["success"] is True
            assert result["email"] == email

        allure.attach(
            str(result),
            name="注册响应",
            attachment_type=allure.attachment_type.JSON
        )

    @allure.story("用户登录")
    @allure.severity(allure.severity_level.BLOCKER)
    def test_user_login(self, browser):
        with allure.step("打开登录页"):
            browser.get("https://example.com/login")
            allure.attach(
                browser.get_screenshot_as_png(),
                name="登录页截图",
                attachment_type=allure.attachment_type.PNG
            )

        with allure.step("输入凭证"):
            browser.find_element("id", "email").send_keys("test@example.com")
            browser.find_element("id", "password").send_keys("password123")

        with allure.step("点击登录"):
            browser.find_element("id", "login-btn").click()

        with allure.step("验证登录成功"):
            assert "dashboard" in browser.current_url
```

```bash
# 运行并生成 Allure 结果
pytest --alluredir=./allure-results

# 生成并打开报告
allure serve ./allure-results

# 生成静态报告
allure generate ./allure-results -o ./allure-report --clean
```

## 常见坑

1. **conftest.py 命名错误**：文件名必须为 `conftest.py`，不能是 `conftest.txt` 或其他变体。pytest 只识别精确的文件名。

2. **Fixture 作用域混淆**：`session` 作用域的 fixture 在并行执行时需注意线程安全。多个进程共享同一资源会导致竞态条件。

3. **parametrize 的 ids 缺失**：大量参数化用例没有 `ids` 时，测试名称为 `test_foo[param0-param1]`，可读性极差。始终提供有意义的 `ids`。

4. **xfail strict 误用**：`strict=True` 时，原本预期失败的用例如果通过，会被标记为失败。这在 CI 中可能导致误报。

5. **pytest-xdist 的 fixture 作用域**：`session` 作用域的 fixture 在 `--dist=loadscope` 模式下，同一 scope 的测试会被分配到同一进程。若使用 `--dist=load`，则可能跨进程共享导致问题。

6. **HTML 报告图片嵌入**：`--self-contained-html` 选项将图片以 base64 嵌入 HTML，但报告文件会显著增大。CI 环境中建议关闭此选项。

7. **marker 未注册**：使用 `--strict-markers` 时，未在 `pytest.ini` 中注册的 marker 会报错。所有自定义 marker 必须显式声明。

8. **覆盖率合并问题**：并行执行时，各进程生成独立的 `.coverage` 文件。需使用 `coverage combine` 合并后再生成报告。

## 自测清单

- [ ] 能解释 conftest.py 的作用域层级与就近原则
- [ ] 能编写 function/class/module/session 作用域的 fixture
- [ ] 能使用 autouse fixture 实现自动 setup/teardown
- [ ] 能正确使用 skip/skipif/xfail 标记
- [ ] 能编写多维度 parametrize 测试（含笛卡尔积）
- [ ] 能在 pytest.ini 中注册自定义 marker
- [ ] 能配置 pytest-html 生成自包含报告
- [ ] 能配置 pytest-cov 并设置覆盖率阈值
- [ ] 能使用 pytest-xdist 进行并行执行
- [ ] 能集成 allure-pytest 生成结构化报告
- [ ] 能处理并行执行中的线程安全问题

## 延伸阅读

- [pytest 官方文档 - Fixtures](https://docs.pytest.org/en/stable/fixtures.html)
- [pytest 官方文档 - Mark](https://docs.pytest.org/en/stable/mark.html)
- [pytest-xdist 文档](https://pytest-xdist.readthedocs.io/en/stable/)
- [pytest-html 文档](https://pytest-html.readthedocs.io/en/latest/)
- [Allure Framework 文档](https://docs.qameta.io/allure/)
- [pytest-cov 文档](https://pytest-cov.readthedocs.io/en/latest/)
