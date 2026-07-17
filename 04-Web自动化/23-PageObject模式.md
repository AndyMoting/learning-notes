# 23-PageObject模式

> 课时：60 min | 难度：★★★★

## 学习目标

- 理解 Page Object 模式的设计动机与三大优势
- 掌握 BasePage 基类的封装方法
- 能实现 Page Factory 模式
- 搭建完整的六文件 PO 项目结构

## 核心概念

### 为什么需要 Page Object

未使用 PO 模式的典型脚本：

```python
driver.find_element(By.ID, "username").send_keys("admin")
driver.find_element(By.ID, "password").send_keys("123456")
driver.find_element(By.ID, "submit-btn").click()
assert driver.find_element(By.CLASS_NAME, "welcome-msg").text == "欢迎"
```

问题：
- 定位符散落在测试脚本中，UI 变更需修改所有引用处
- 测试逻辑与页面操作混杂，可读性差
- 无法复用页面操作（如登录需在多个测试中重复编写）

### Page Object 三大优势

1. **可维护性** — 页面元素定位集中在 Page 类中，UI 变更只需修改一处
2. **可复用性** — 页面操作封装为方法，多个测试用例直接调用
3. **可读性** — 测试脚本语义清晰，接近自然语言描述

### 设计原则

- 每个页面对应一个 Page 类
- Page 类封装元素定位与页面操作
- 测试脚本只调用 Page 类方法，不直接操作 WebDriver
- Page 类不出现断言（断言属于测试逻辑）

## 动手实操

### 项目结构

```
po_project/
├── base/
│   └── base_page.py
├── pages/
│   ├── __init__.py
│   ├── login_page.py
│   ├── products_page.py
│   └── checkout_page.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_login.py
│   ├── test_products.py
│   └── test_checkout.py
├── pytest.ini
└── requirements.txt
```

### 文件 1：base/base_page.py

```python
from selenium.webdriver.remote.webdriver import WebDriver
from selenium.webdriver.remote.webelement import WebElement
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import TimeoutException


class BasePage:
    """所有 Page 类的基类，封装通用操作"""

    def __init__(self, driver: WebDriver, timeout: int = 10):
        self.driver = driver
        self.wait = WebDriverWait(driver, timeout)

    def find_element(self, locator: tuple[str, str]) -> WebElement:
        """显式等待并返回单个元素"""
        return self.wait.until(EC.presence_of_element_located(locator))

    def find_clickable(self, locator: tuple[str, str]) -> WebElement:
        """等待元素可点击后返回"""
        return self.wait.until(EC.element_to_be_clickable(locator))

    def click(self, locator: tuple[str, str]) -> None:
        """点击元素"""
        self.find_clickable(locator).click()

    def send_keys(self, locator: tuple[str, str], text: str) -> None:
        """清空并输入文本"""
        element = self.find_element(locator)
        element.clear()
        element.send_keys(text)

    def get_text(self, locator: tuple[str, str]) -> str:
        """获取元素文本"""
        return self.find_element(locator).text

    def is_visible(self, locator: tuple[str, str]) -> bool:
        """判断元素是否可见"""
        try:
            self.wait.until(EC.visibility_of_element_located(locator))
            return True
        except TimeoutException:
            return False

    def wait_for_url_contains(self, text: str) -> None:
        """等待 URL 包含指定文本"""
        self.wait.until(EC.url_contains(text))
```

### 文件 2：pages/login_page.py

```python
from selenium.webdriver.common.by import By
from base.base_page import BasePage


class LoginPage(BasePage):
    """登录页面"""

    # 元素定位符
    USERNAME_INPUT = (By.ID, "username")
    PASSWORD_INPUT = (By.ID, "password")
    SUBMIT_BUTTON = (By.ID, "submit-btn")
    ERROR_MESSAGE = (By.CLASS_NAME, "error-msg")
    WELCOME_MESSAGE = (By.CLASS_NAME, "welcome-msg")

    def __init__(self, driver):
        super().__init__(driver)
        self.url = "http://localhost:8080/login"

    def open(self) -> "LoginPage":
        self.driver.get(self.url)
        return self

    def login(self, username: str, password: str) -> "LoginPage":
        self.send_keys(self.USERNAME_INPUT, username)
        self.send_keys(self.PASSWORD_INPUT, password)
        self.click(self.SUBMIT_BUTTON)
        return self

    def get_error_message(self) -> str:
        return self.get_text(self.ERROR_MESSAGE)

    def get_welcome_message(self) -> str:
        return self.get_text(self.WELCOME_MESSAGE)

    def is_error_displayed(self) -> bool:
        return self.is_visible(self.ERROR_MESSAGE)
```

### 文件 3：pages/products_page.py

```python
from selenium.webdriver.common.by import By
from base.base_page import BasePage


class ProductsPage(BasePage):
    """商品列表页面"""

    PRODUCT_CARDS = (By.CSS_SELECTOR, ".product-card")
    SEARCH_INPUT = (By.NAME, "search")
    SEARCH_BUTTON = (By.CSS_SELECTOR, "button.search-btn")
    CART_LINK = (By.ID, "cart-link")
    CART_COUNT = (By.CSS_SELECTOR, ".cart-count")

    def __init__(self, driver):
        super().__init__(driver)
        self.url = "http://localhost:8080/products"

    def open(self) -> "ProductsPage":
        self.driver.get(self.url)
        return self

    def search(self, keyword: str) -> "ProductsPage":
        self.send_keys(self.SEARCH_INPUT, keyword)
        self.click(self.SEARCH_BUTTON)
        return self

    def get_product_count(self) -> int:
        cards = self.driver.find_elements(*self.PRODUCT_CARDS)
        return len(cards)

    def add_to_cart(self, index: int = 0) -> "ProductsPage":
        cards = self.driver.find_elements(*self.PRODUCT_CARDS)
        cards[index].find_element(By.CSS_SELECTOR, ".add-cart-btn").click()
        return self

    def get_cart_count(self) -> str:
        return self.get_text(self.CART_COUNT)

    def go_to_cart(self) -> None:
        self.click(self.CART_LINK)
```

### 文件 4：pages/checkout_page.py

```python
from selenium.webdriver.common.by import By
from base.base_page import BasePage


class CheckoutPage(BasePage):
    """结算页面"""

    NAME_INPUT = (By.ID, "name")
    ADDRESS_INPUT = (By.ID, "address")
    PHONE_INPUT = (By.ID, "phone")
    CONFIRM_BUTTON = (By.ID, "confirm-btn")
    SUCCESS_MESSAGE = (By.CLASS_NAME, "success-msg")
    TOTAL_PRICE = (By.CSS_SELECTOR, ".total-price")

    def __init__(self, driver):
        super().__init__(driver)
        self.url = "http://localhost:8080/checkout"

    def open(self) -> "CheckoutPage":
        self.driver.get(self.url)
        return self

    def fill_info(self, name: str, address: str, phone: str) -> "CheckoutPage":
        self.send_keys(self.NAME_INPUT, name)
        self.send_keys(self.ADDRESS_INPUT, address)
        self.send_keys(self.PHONE_INPUT, phone)
        return self

    def confirm(self) -> "CheckoutPage":
        self.click(self.CONFIRM_BUTTON)
        return self

    def get_total_price(self) -> str:
        return self.get_text(self.TOTAL_PRICE)

    def is_success_displayed(self) -> bool:
        return self.is_visible(self.SUCCESS_MESSAGE)
```

### 文件 5：tests/conftest.py

```python
import pytest
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from pages.login_page import LoginPage
from pages.products_page import ProductsPage
from pages.checkout_page import CheckoutPage


@pytest.fixture(scope="function")
def driver():
    options = Options()
    options.add_argument("--headless=new")
    options.add_argument("--window-size=1920,1080")
    options.add_argument("--disable-gpu")

    d = webdriver.Chrome(options=options)
    yield d
    d.quit()


@pytest.fixture
def login_page(driver):
    return LoginPage(driver)


@pytest.fixture
def products_page(driver):
    return ProductsPage(driver)


@pytest.fixture
def checkout_page(driver):
    return CheckoutPage(driver)


@pytest.fixture
def logged_in_driver(driver):
    """已登录状态的 driver"""
    login_page = LoginPage(driver)
    login_page.open().login("admin", "123456")
    return driver
```

### 文件 6：tests/test_login.py

```python
import pytest


class TestLogin:

    def test_login_success(self, login_page):
        login_page.open().login("admin", "123456")
        assert login_page.get_welcome_message() == "欢迎，admin"

    def test_login_wrong_password(self, login_page):
        login_page.open().login("admin", "wrong")
        assert login_page.is_error_message_displayed()
        assert "密码错误" in login_page.get_error_message()

    def test_login_empty_fields(self, login_page):
        login_page.open().login("", "")
        assert login_page.is_error_message_displayed()
```

### 文件 7：tests/test_products.py

```python
import pytest


class TestProducts:

    def test_search_product(self, products_page):
        products_page.open().search("手机")
        assert products_page.get_product_count() > 0

    def test_add_to_cart(self, products_page, logged_in_driver):
        products_page.open()
        products_page.add_to_cart(0)
        assert products_page.get_cart_count() == "1"
```

## Page Factory 模式

Page Factory 通过注解（`@FindBy`）声明元素定位，Selenium Java 版本原生支持。Python 中可模拟实现：

```python
from dataclasses import dataclass, field
from selenium.webdriver.common.by import By
from selenium.webdriver.remote.webdriver import WebDriver
from base.base_page import BasePage


@dataclass
class LoginPagePF(BasePage):
    """Page Factory 风格（Python 模拟）"""

    USERNAME: tuple = field(default=(By.ID, "username"))
    PASSWORD: tuple = field(default=(By.ID, "password"))
    SUBMIT: tuple = field(default=(By.ID, "submit-btn"))

    def __post_init__(self):
        # dataclass 不调用父类 __init__，需手动初始化
        pass

    def login(self, username: str, password: str):
        self.send_keys(self.USERNAME, username)
        self.send_keys(self.PASSWORD, password)
        self.click(self.SUBMIT)
```

> Python 生态中 Page Factory 使用较少，推荐使用类属性声明定位符（如 `USERNAME_INPUT = (By.ID, "username")`），更简洁且类型安全。

## 常见坑

1. **Page 类中出现断言** — 断言属于测试逻辑，应放在测试脚本中，Page 类只负责操作与数据获取
2. **未使用显式等待** — 直接 `find_element` 在元素未加载时立即失败，应封装 `WebDriverWait` 到 BasePage
3. **方法未返回 self** — 链式调用（`page.open().login()`）需要方法返回 `self`
4. **定位符硬编码在测试脚本** — 违反 PO 原则，所有定位符必须在 Page 类中声明
5. **BasePage 未封装等待逻辑** — 导致每个 Page 方法重复编写等待代码

## 自测清单

- [ ] 能解释 Page Object 模式的三大优势
- [ ] 能实现包含 find_element/click/send_keys/get_text 的 BasePage
- [ ] 能为三个页面编写独立的 Page 类
- [ ] 能在 conftest.py 中配置 Page 类 fixture
- [ ] 能使用链式调用编写测试脚本
- [ ] 能区分 Page 类职责与测试脚本职责
- [ ] 能搭建完整的六文件 PO 项目结构

## 延伸阅读

- [Selenium Page Object 官方指南](https://www.selenium.dev/documentation/test_practices/encouraged/page_object_models/)
- [Page Object 模式 — Martin Fowler](https://martinfowler.com/bliki/PageObject.html)
- [Python dataclass 文档](https://docs.python.org/3/library/dataclasses.html)
