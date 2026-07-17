# 24-Pytest基础

> 课时：45 min | 难度：★★★

## 学习目标

- 掌握 pytest 安装与测试发现规则
- 理解原生 assert 与 unittest 断言的区别
- 能编写不同作用域的 fixture 并实现 yield 拆解
- 使用 `@pytest.mark.parametrize` 实现数据驱动测试
- 配置 conftest.py 与 pytest.ini
- 掌握测试选择、并行执行与输出格式控制

## 核心概念

### pytest 安装

```bash
pip install pytest>=7.0.0
```

验证：

```bash
pytest --version
```

### 测试发现规则

pytest 自动发现以下模式的测试：

- 文件命名：`test_*.py` 或 `*_test.py`
- 类命名：`Test*`（不含 `__init__`）
- 函数/方法命名：`test_*`

```bash
# 运行当前目录所有测试
pytest

# 运行指定文件
pytest tests/test_login.py

# 运行指定类中的测试
pytest tests/test_login.py::TestLogin

# 运行指定方法
pytest tests/test_login.py::TestLogin::test_login_success

# 按关键字匹配
pytest -k "login and not wrong"
```

### 原生断言

pytest 使用 Python 原生 `assert`，无需 `self.assertEqual` 等方法。失败时 pytest 自动提供详细差异对比：

```python
def test_addition():
    assert 1 + 1 == 2

def test_string():
    assert "hello" in "hello world"

def test_list():
    assert [1, 2, 3] == [1, 2, 3]

def test_dict():
    result = {"status": "ok", "code": 200}
    assert result["code"] == 200
    assert result.get("message") is None
```

自定义失败消息：

```python
def test_status():
    response = {"status": "error"}
    assert response["status"] == "ok", f"期望 ok，实际 {response['status']}"
```

## Fixture

Fixture 是 pytest 的核心机制，用于测试前置准备与后置清理。

### 基本用法

```python
import pytest

@pytest.fixture
def sample_data():
    """返回测试数据"""
    return {"username": "admin", "password": "123456"}

def test_login(sample_data):
    assert sample_data["username"] == "admin"
```

### 作用域（scope）

| scope | 说明 | 适用场景 |
|-------|------|----------|
| `function` | 每个测试函数执行一次（默认） | 数据库连接、driver 实例 |
| `class` | 每个测试类执行一次 | 共享的类级别资源 |
| `module` | 每个测试模块执行一次 | 模块级配置加载 |
| `session` | 整个测试会话执行一次 | 全局配置、共享浏览器实例 |

```python
import pytest
from selenium import webdriver

@pytest.fixture(scope="session")
def driver():
    """整个会话共享一个浏览器实例"""
    d = webdriver.Chrome()
    yield d
    d.quit()

@pytest.fixture(scope="function")
def logged_in(driver):
    """每个测试函数重新登录"""
    driver.get("http://localhost:8080/login")
    driver.find_element("id", "username").send_keys("admin")
    driver.find_element("id", "password").send_keys("123456")
    driver.find_element("id", "submit-btn").click()
    return driver
```

### yield 拆解

使用 `yield` 替代 `return`，yield 之前的代码作为 setup，之后的代码作为 teardown：

```python
import pytest
import tempfile
import os

@pytest.fixture(scope="function")
def temp_file():
    """创建临时文件，测试结束后自动删除"""
    fd, path = tempfile.mkstemp()
    with os.fdopen(fd, "w") as f:
        f.write("test data")

    yield path  # 测试函数在此处执行

    # teardown：测试结束后执行
    if os.path.exists(path):
        os.remove(path)

def test_file_exists(temp_file):
    assert os.path.exists(temp_file)
    with open(temp_file) as f:
        assert f.read() == "test data"
```

### autouse fixture

`autouse=True` 的 fixture 无需显式声明参数，自动应用于作用域内所有测试：

```python
import pytest

@pytest.fixture(autouse=True, scope="function")
def reset_state():
    """每个测试前重置全局状态"""
    print("\n[setup] 重置状态")
    yield
    print("\n[teardown] 清理状态")
```

## 参数化测试

### @pytest.mark.parametrize

```python
import pytest

@pytest.mark.parametrize("username,password,expected", [
    ("admin", "123456", True),
    ("admin", "wrong", False),
    ("", "123456", False),
    ("admin", "", False),
])
def test_login(username, password, expected):
    result = do_login(username, password)
    assert result == expected
```

pytest 自动生成四个独立测试用例，命名为 `test_login[admin-123456-True]` 等。

### 组合参数化

```python
@pytest.mark.parametrize("a", [1, 2])
@pytest.mark.parametrize("b", [10, 20])
def test_multiply(a, b):
    assert a * b > 0
```

生成 2×2=4 个用例：`test_multiply[1-10]`、`test_multiply[1-20]`、`test_multiply[2-10]`、`test_multiply[2-20]`。

### 自定义 ID

```python
@pytest.mark.parametrize("input_val,expected", [
    pytest.param("admin", True, id="valid-user"),
    pytest.param("", False, id="empty-username"),
    pytest.param("a" * 100, False, id="too-long-username"),
])
def test_validate(input_val, expected):
    assert validate(input_val) == expected
```

## conftest.py

`conftest.py` 是 pytest 的共享 fixture 配置文件，自动被同目录及子目录的测试文件发现，无需 import。

```python
# tests/conftest.py
import pytest
from selenium import webdriver
from selenium.webdriver.chrome.options import Options


@pytest.fixture(scope="session")
def driver():
    options = Options()
    options.add_argument("--headless=new")
    options.add_argument("--window-size=1920,1080")
    d = webdriver.Chrome(options=options)
    yield d
    d.quit()


@pytest.fixture(scope="function")
def base_url():
    return "http://localhost:8080"
```

项目可存在多个 `conftest.py`，子目录的 conftest 覆盖父目录同名 fixture。

## pytest.ini 配置

```ini
# pytest.ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts =
    -v
    --tb=short
    --strict-markers
    -ra
markers =
    smoke: 冒烟测试
    regression: 回归测试
    slow: 耗时较长的测试
```

| 配置项 | 说明 |
|--------|------|
| `testpaths` | 测试搜索路径 |
| `addopts` | 默认命令行参数 |
| `--tb=short` | 简短的失败回溯 |
| `--strict-markers` | 未注册 marker 报错 |
| `-ra` | 显示除 passed 外所有测试的摘要 |
| `markers` | 注册自定义 marker |

### 使用 marker

```python
import pytest

@pytest.mark.smoke
def test_login():
    assert True

@pytest.mark.slow
def test_full_workflow():
    assert True
```

```bash
# 只运行冒烟测试
pytest -m smoke

# 排除慢测试
pytest -m "not slow"
```

## 运行测试

### 选择执行

```bash
# 运行所有测试
pytest

# 运行指定文件
pytest tests/test_login.py

# 运行指定类
pytest tests/test_login.py::TestLogin

# 运行指定方法
pytest tests/test_login.py::TestLogin::test_login_success

# 按关键字匹配
pytest -k "login and not wrong"

# 按 marker 筛选
pytest -m smoke

# 失败时停止
pytest -x

# 上次失败的测试优先运行
pytest --ff

# 只运行上次失败的测试
pytest --lf
```

### 并行执行

安装 `pytest-xdist`：

```bash
pip install pytest-xdist
```

```bash
# 自动检测 CPU 核心数并行
pytest -n auto

# 指定并行进程数
pytest -n 4

# 按模块分发（减少 driver 重建开销）
pytest -n auto --dist=loadscope
```

### 输出格式

```bash
# 详细输出
pytest -v

# 静默模式（只显示摘要）
pytest -q

# 显示 print 输出
pytest -s

# 失败时显示完整回溯
pytest --tb=long

# 失败时显示最短回溯
pytest --tb=line

# 生成 HTML 报告（需安装 pytest-html）
pytest --html=report.html --self-contained-html

# 生成 JUnit XML 报告
pytest --junitxml=results.xml
```

## 完整示例

```python
# tests/test_calculator.py
import pytest


class TestCalculator:

    @pytest.fixture(autouse=True)
    def setup(self):
        self.calc = Calculator()
        yield
        self.calc.reset()

    @pytest.mark.parametrize("a,b,expected", [
        (1, 2, 3),
        (-1, 1, 0),
        (0, 0, 0),
        (100, 200, 300),
    ])
    def test_add(self, a, b, expected):
        assert self.calc.add(a, b) == expected

    @pytest.mark.parametrize("a,b,expected", [
        (10, 2, 5),
        (9, 3, 3),
        pytest.param(1, 0, None, marks=pytest.mark.xfail(reason="除零未处理")),
    ])
    def test_divide(self, a, b, expected):
        if b == 0:
            with pytest.raises(ZeroDivisionError):
                self.calc.divide(a, b)
        else:
            assert self.calc.divide(a, b) == expected

    @pytest.mark.slow
    def test_performance(self):
        for i in range(10000):
            self.calc.add(i, i)
```

## 常见坑

1. **测试文件命名不符合规则** — 文件未以 `test_` 开头或 `_test` 结尾，pytest 不会发现
2. **fixture 作用域错误** — `function` 级 driver 在 `session` 级 fixture 中使用会导致 driver 提前关闭
3. **conftest.py 未被发现** — conftest.py 必须放在测试文件同级或父级目录，不能放在子目录之外
4. **parametrize 参数名不匹配** — 装饰器中的参数名必须与测试函数参数名一致
5. **未注册 marker 直接使用** — 配置 `--strict-markers` 后，未在 `pytest.ini` 注册的 marker 会报错
6. **并行测试共享可变状态** — `pytest-xdist` 多进程并行时，全局变量不共享，应使用 fixture 传递状态

## 自测清单

- [ ] 能解释 pytest 测试发现规则
- [ ] 能使用原生 assert 编写断言
- [ ] 能编写 function/class/module/session 四种作用域的 fixture
- [ ] 能使用 yield 实现 setup/teardown
- [ ] 能使用 parametrize 实现数据驱动测试
- [ ] 能配置 conftest.py 共享 fixture
- [ ] 能配置 pytest.ini 的 addopts 与 markers
- [ ] 能使用 -k/-m/-n 等参数控制测试执行

## 延伸阅读

- [pytest 官方文档](https://docs.pytest.org/en/stable/)
- [pytest fixture 参考](https://docs.pytest.org/en/stable/reference/fixtures.html)
- [pytest-xdist 文档](https://pytest-xdist.readthedocs.io/en/stable/)
- [pytest-html 文档](https://pytest-html.readthedocs.io/en/latest/)
- [pytest best practices](https://docs.pytest.org/en/stable/goodpractices.html)
