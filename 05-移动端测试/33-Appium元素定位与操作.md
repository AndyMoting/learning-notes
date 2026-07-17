# 33-Appium元素定位与操作

> 课时：55 min | 难度：★★★☆☆

## 学习目标

- 掌握 5 种 Appium 元素定位策略及其适用场景
- 熟练使用元素操作方法（click、send_keys、get_text 等）
- 掌握 TouchAction 与 W3C Actions 实现手势操作
- 理解隐式等待与显式等待的区别与选择
- 能够编写完整的移动端测试流程

## 核心概念

### 33.1 元素定位策略

Appium 提供多种定位策略，按优先级排序如下：

| 策略 | 方法 | 优先级 | 适用场景 |
|-----|------|-------|---------|
| ID | `AppiumBy.ID` | 最高 | 元素有唯一 resource-id |
| Accessibility ID | `AppiumBy.ACCESSIBILITY_ID` | 高 | 元素有 content-desc |
| XPath | `AppiumBy.XPATH` | 中 | 复杂层级或属性组合 |
| Class Name | `AppiumBy.CLASS_NAME` | 低 | 定位同类元素集合 |
| Android UIAutomator | `AppiumBy.ANDROID_UIAUTOMATOR` | 高 | Android 特有，支持 UIAutomator API |

**ID 定位**
```python
from appium.webdriver.common.appiumby import AppiumBy

# 通过 resource-id 定位（推荐）
element = driver.find_element(AppiumBy.ID, "com.example.app:id/btn_submit")

# 简写形式（部分版本支持）
element = driver.find_element(AppiumBy.ID, "btn_submit")
```

**Accessibility ID 定位**
```python
# 通过 content-desc 定位（Android）
element = driver.find_element(AppiumBy.ACCESSIBILITY_ID, "登录按钮")

# 等价于 iOS 的 accessibilityIdentifier
element = driver.find_element(AppiumBy.ACCESSIBILITY_ID, "login_button")
```

**XPath 定位**
```python
# 通过 resource-id
element = driver.find_element(AppiumBy.XPATH, '//*[@resource-id="com.example.app:id/et_username"]')

# 通过 text 属性
element = driver.find_element(AppiumBy.XPATH, '//*[@text="登录"]')

# 通过 content-desc
element = driver.find_element(AppiumBy.XPATH, '//*[@content-desc="搜索"]')

# 组合条件
element = driver.find_element(
    AppiumBy.XPATH,
    '//*[@resource-id="com.example.app:id/item" and @text="设置"]'
)

# 层级定位（父级找子级）
element = driver.find_element(
    AppiumBy.XPATH,
    '//android.widget.LinearLayout[@resource-id="com.example.app:id/container"]/android.widget.Button'
)

# 模糊匹配
element = driver.find_element(AppiumBy.XPATH, '//*[contains(@text, "确定")]')
element = driver.find_element(AppiumBy.XPATH, '//*[starts-with(@text, "请输入")]')
```

**Class Name 定位**
```python
# 定位所有 EditText
inputs = driver.find_elements(AppiumBy.CLASS_NAME, "android.widget.EditText")

# 定位单个 Button
button = driver.find_element(AppiumBy.CLASS_NAME, "android.widget.Button")
```

**Android UIAutomator 定位**
```python
# 通过 resourceId
element = driver.find_element(
    AppiumBy.ANDROID_UIAUTOMATOR,
    'new UiSelector().resourceId("com.example.app:id/btn_login")'
)

# 通过 text
element = driver.find_element(
    AppiumBy.ANDROID_UIAUTOMATOR,
    'new UiSelector().text("登录")'
)

# 通过 text 模糊匹配
element = driver.find_element(
    AppiumBy.ANDROID_UIAUTOMATOR,
    'new UiSelector().textContains("登录")'
)

# 通过 description
element = driver.find_element(
    AppiumBy.ANDROID_UIAUTOMATOR,
    'new UiSelector().description("搜索图标")'
)

# 组合条件
element = driver.find_element(
    AppiumBy.ANDROID_UIAUTOMATOR,
    'new UiSelector().resourceId("com.example.app:id/item").text("设置")'
)

# 通过索引
element = driver.find_element(
    AppiumBy.ANDROID_UIAUTOMATOR,
    'new UiSelector().className("android.widget.TextView").instance(2)'
)

# 滚动查找（滚动到目标元素）
element = driver.find_element(
    AppiumBy.ANDROID_UIAUTOMATOR,
    'new UiScrollable(new UiSelector().scrollable(true)).scrollIntoView(new UiSelector().text("目标文本"))'
)
```

### 33.2 元素操作方法

```python
element = driver.find_element(AppiumBy.ID, "com.example.app:id/et_username")

# 点击
element.click()

# 输入文本
element.send_keys("testuser")

# 清除内容
element.clear()

# 获取文本
text = element.text

# 获取属性值
resource_id = element.get_attribute("resource-id")
content_desc = element.get_attribute("content-desc")
enabled = element.get_attribute("enabled")       # "true" / "false"
displayed = element.is_displayed()              # bool
selected = element.is_selected()                # bool
focused = element.get_attribute("focused")      # "true" / "false"

# 获取元素位置与尺寸
location = element.location      # {'x': 100, 'y': 200}
size = element.size              # {'width': 300, 'height': 80}
rect = element.rect              # {'x': 100, 'y': 200, 'width': 300, 'height': 80}

# 获取元素标签名
tag = element.tag_name
```

### 33.3 手势操作

**TouchAction（已废弃，部分版本仍可用）**

```python
from appium.webdriver.common.touch_action import TouchAction

# 单点点击
TouchAction(driver).tap(x=500, y=800).perform()

# 长按
TouchAction(driver).long_press(x=500, y=800, duration=2000).release().perform()

# 滑动
TouchAction(driver).press(x=500, y=1500).move_to(x=500, y=500).release().perform()
```

**W3C Actions（推荐）**

```python
from selenium.webdriver.common.action_chains import ActionChains
from selenium.webdriver.common.actions.pointer_input import PointerInput
from selenium.webdriver.common.actions import interaction

# 单点点击
actions = ActionChains(driver)
actions.w3c_actions.devices = []
finger = actions.w3c_actions.add_pointer_input(interaction.POINTER_TOUCH, "finger")
finger.create_pointer_move(x=500, y=800)
finger.create_pointer_down(button=0)
finger.create_pause(0.1)
finger.create_pointer_up(button=0)
actions.perform()

# 滑动（从下到上）
actions = ActionChains(driver)
actions.w3c_actions.devices = []
finger = actions.w3c_actions.add_pointer_input(interaction.POINTER_TOUCH, "finger")
finger.create_pointer_move(x=540, y=1800)
finger.create_pointer_down(button=0)
finger.create_pause(0.1)
finger.create_pointer_move(x=540, y=600, duration=500)
finger.create_pointer_up(button=0)
actions.perform()

# 双指缩放（pinch）
actions = ActionChains(driver)
actions.w3c_actions.devices = []

finger1 = actions.w3c_actions.add_pointer_input(interaction.POINTER_TOUCH, "finger1")
finger1.create_pointer_move(x=300, y=800)
finger1.create_pointer_down(button=0)
finger1.create_pause(0.1)
finger1.create_pointer_move(x=200, y=800, duration=500)
finger1.create_pointer_up(button=0)

finger2 = actions.w3c_actions.add_pointer_input(interaction.POINTER_TOUCH, "finger2")
finger2.create_pointer_move(x=700, y=800)
finger2.create_pointer_down(button=0)
finger2.create_pause(0.1)
finger2.create_pointer_move(x=800, y=800, duration=500)
finger2.create_pointer_up(button=0)

actions.perform()
```

**封装常用手势方法**

```python
class MobileActions:
    """移动端手势操作封装"""

    def __init__(self, driver):
        self.driver = driver
        self.size = driver.get_window_size()
        self.width = self.size['width']
        self.height = self.size['height']

    def tap(self, x, y):
        """点击指定坐标"""
        actions = ActionChains(self.driver)
        actions.w3c_actions.devices = []
        finger = actions.w3c_actions.add_pointer_input(interaction.POINTER_TOUCH, "finger")
        finger.create_pointer_move(x=x, y=y)
        finger.create_pointer_down(button=0)
        finger.create_pause(0.1)
        finger.create_pointer_up(button=0)
        actions.perform()

    def swipe(self, start_x, start_y, end_x, end_y, duration=500):
        """滑动"""
        actions = ActionChains(self.driver)
        actions.w3c_actions.devices = []
        finger = actions.w3c_actions.add_pointer_input(interaction.POINTER_TOUCH, "finger")
        finger.create_pointer_move(x=start_x, y=start_y)
        finger.create_pointer_down(button=0)
        finger.create_pause(0.1)
        finger.create_pointer_move(x=end_x, y=end_y, duration=duration)
        finger.create_pointer_up(button=0)
        actions.perform()

    def swipe_up(self):
        """上滑（屏幕下方 → 上方）"""
        self.swipe(self.width // 2, int(self.height * 0.8),
                   self.width // 2, int(self.height * 0.2))

    def swipe_down(self):
        """下滑"""
        self.swipe(self.width // 2, int(self.height * 0.2),
                   self.width // 2, int(self.height * 0.8))

    def swipe_left(self):
        """左滑"""
        self.swipe(int(self.width * 0.8), self.height // 2,
                   int(self.width * 0.2), self.height // 2)

    def swipe_right(self):
        """右滑"""
        self.swipe(int(self.width * 0.2), self.height // 2,
                   int(self.width * 0.8), self.height // 2)

    def long_press(self, x, y, duration=2000):
        """长按"""
        actions = ActionChains(self.driver)
        actions.w3c_actions.devices = []
        finger = actions.w3c_actions.add_pointer_input(interaction.POINTER_TOUCH, "finger")
        finger.create_pointer_move(x=x, y=y)
        finger.create_pointer_down(button=0)
        finger.create_pause(duration / 1000)
        finger.create_pointer_up(button=0)
        actions.perform()

    def scroll_to_element(self, locator, max_swipes=5):
        """滚动查找元素"""
        for _ in range(max_swipes):
            try:
                element = self.driver.find_element(*locator)
                if element.is_displayed():
                    return element
            except Exception:
                pass
            self.swipe_up()
        raise Exception(f"滚动 {max_swipes} 次后未找到元素: {locator}")
```

### 33.4 等待策略

**隐式等待（Implicit Wait）**

全局设置，对所有 `find_element` 调用生效。在指定时间内轮询查找元素，找到即返回。

```python
# 设置隐式等待 10 秒
driver.implicitly_wait(10)

# 后续所有 find_element 调用均等待最多 10 秒
element = driver.find_element(AppiumBy.ID, "com.example.app:id/btn_login")
```

**显式等待（Explicit Wait）**

针对特定条件设置等待，更灵活、更精确。

```python
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from appium.webdriver.common.appiumby import AppiumBy

# 等待元素出现
wait = WebDriverWait(driver, 15, poll_frequency=0.5)
element = wait.until(
    EC.presence_of_element_located((AppiumBy.ID, "com.example.app:id/tv_welcome"))
)

# 等待元素可点击
element = wait.until(
    EC.element_to_be_clickable((AppiumBy.ID, "com.example.app:id/btn_submit"))
)

# 等待元素可见
element = wait.until(
    EC.visibility_of_element_located((AppiumBy.ID, "com.example.app:id/tv_result"))
)

# 等待元素消失（如加载动画）
wait.until(
    EC.invisibility_of_element_located((AppiumBy.ID, "com.example.app:id/progress_bar"))
)

# 等待文本出现
wait.until(
    EC.text_to_be_present_in_element(
        (AppiumBy.ID, "com.example.app:id/tv_status"),
        "加载完成"
    )
)

# 自定义等待条件
from selenium.webdriver.support.wait import WebDriverWait

def element_has_text(locator, expected_text):
    def _predicate(driver):
        try:
            element = driver.find_element(*locator)
            return expected_text in element.text
        except Exception:
            return False
    return _predicate

wait.until(element_has_text(
    (AppiumBy.ID, "com.example.app:id/tv_message"),
    "成功"
))
```

**等待策略选择原则**

| 场景 | 推荐策略 |
|-----|---------|
| 全局元素查找超时 | 隐式等待 |
| 特定元素需等待特定条件 | 显式等待 |
| 页面跳转后元素加载 | 显式等待 + presence_of_element_located |
| 按钮点击前等待可交互 | 显式等待 + element_to_be_clickable |
| 加载动画消失后操作 | 显式等待 + invisibility_of_element_located |
| 动态内容文本验证 | 显式等待 + text_to_be_present_in_element |

### 33.5 完整测试流程示例

```python
import pytest
from appium import webdriver
from appium.options.android import UiAutomator2Options
from appium.webdriver.common.appiumby import AppiumBy
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


class TestECommerceFlow:
    """电商应用完整购物流程测试"""

    @pytest.fixture(autouse=True)
    def setup_and_teardown(self):
        options = UiAutomator2Options()
        options.platform_name = "Android"
        options.device_name = "emulator-5554"
        options.app_package = "com.example.shop"
        options.app_activity = ".SplashActivity"
        options.automation_name = "UiAutomator2"
        options.no_reset = False
        options.new_command_timeout = 300

        self.driver = webdriver.Remote("http://127.0.0.1:4723", options=options)
        self.driver.implicitly_wait(10)
        self.wait = WebDriverWait(self.driver, 15)
        yield
        self.driver.quit()

    def test_purchase_flow(self):
        """完整购买流程：搜索 → 选择商品 → 加购 → 下单"""
        # Step 1：等待首页加载完成
        self.wait.until(
            EC.presence_of_element_located((AppiumBy.ID, "com.example.shop:id/search_bar"))
        )

        # Step 2：点击搜索框
        search_bar = self.driver.find_element(AppiumBy.ID, "com.example.shop:id/search_bar")
        search_bar.click()

        # Step 3：输入搜索关键词
        search_input = self.wait.until(
            EC.element_to_be_clickable((AppiumBy.ID, "com.example.shop:id/et_search"))
        )
        search_input.send_keys("手机")

        # Step 4：点击搜索按钮
        self.driver.find_element(AppiumBy.ID, "com.example.shop:id/btn_search").click()

        # Step 5：等待搜索结果加载
        self.wait.until(
            EC.presence_of_element_located((AppiumBy.ID, "com.example.shop:id/rv_products"))
        )

        # Step 6：点击第一个商品
        products = self.driver.find_elements(AppiumBy.ID, "com.example.shop:id/item_product")
        assert len(products) > 0, "搜索结果为空"
        products[0].click()

        # Step 7：等待商品详情页加载
        self.wait.until(
            EC.presence_of_element_located((AppiumBy.ID, "com.example.shop:id/btn_add_cart"))
        )

        # Step 8：选择规格（如有）
        try:
            spec_option = self.driver.find_element(
                AppiumBy.ANDROID_UIAUTOMATOR,
                'new UiSelector().text("128GB")'
            )
            spec_option.click()
        except Exception:
            pass  # 无规格选择则跳过

        # Step 9：加入购物车
        self.driver.find_element(AppiumBy.ID, "com.example.shop:id/btn_add_cart").click()

        # Step 10：验证加购成功提示
        toast = self.wait.until(
            EC.presence_of_element_located(
                (AppiumBy.XPATH, '//*[contains(@text, "已加入购物车")]')
            )
        )
        assert toast.is_displayed()

        # Step 11：进入购物车
        self.driver.find_element(AppiumBy.ID, "com.example.shop:id/btn_go_cart").click()

        # Step 12：勾选商品
        checkbox = self.wait.until(
            EC.element_to_be_clickable((AppiumBy.ID, "com.example.shop:id/cb_select"))
        )
        checkbox.click()

        # Step 13：点击结算
        self.driver.find_element(AppiumBy.ID, "com.example.shop:id/btn_checkout").click()

        # Step 14：确认订单页
        self.wait.until(
            EC.presence_of_element_located((AppiumBy.ID, "com.example.shop:id/btn_submit_order"))
        )

        # Step 15：提交订单
        self.driver.find_element(AppiumBy.ID, "com.example.shop:id/btn_submit_order").click()

        # Step 16：验证跳转至支付页
        self.wait.until(
            EC.presence_of_element_located((AppiumBy.ID, "com.example.shop:id/payment_container"))
        )
        assert self.driver.find_element(
            AppiumBy.ID, "com.example.shop:id/tv_order_amount"
        ).is_displayed()
```

## 常见坑

- XPath 表达式中 `@text` 匹配空文本时返回空字符串，需用 `string-length(@text) > 0` 过滤
- `find_element` 与 `find_elements` 混淆：前者找不到抛异常，后者返回空列表
- 隐式等待与显式等待同时使用时，取两者中较大值，可能导致等待时间翻倍
- `send_keys` 输入中文失败，需切换 ADBKeyBoard 或使用 `driver.execute_script('mobile: type', {'text': '中文'})`
- 元素在屏幕外不可见时 `click()` 失败，需先滚动至可见区域
- W3C Actions 中 `create_pointer_move` 的 `duration` 单位为秒，非毫秒
- `clear()` 方法对部分自定义输入框无效，需通过 `element.send_keys(Keys.CONTROL + "a" + Keys.DELETE)` 替代

## 自测清单

- [ ] 能够区分 5 种定位策略并说明各自适用场景
- [ ] 能够使用 XPath 编写包含模糊匹配、组合条件的定位表达式
- [ ] 能够使用 Android UIAutomator 的 UiSelector API 定位元素
- [ ] 能够使用 W3C Actions 实现 tap、swipe、long_press、pinch 操作
- [ ] 能够区分隐式等待与显式等待，说明各自适用场景
- [ ] 能够编写包含完整业务流程的 Appium 测试脚本

## 延伸阅读

- Appium 定位策略文档：<https://appium.io/docs/en/latest/guides/context/>
- W3C Actions 规范：<https://www.w3.org/TR/webdriver/#actions>
- UiAutomator2 UiSelector 参考：<https://developer.android.com/reference/androidx/test/uiautomator/UiSelector>
- XPath 语法参考：<https://www.w3schools.com/xml/xpath_syntax.asp>
- Selenium Expected Conditions：<https://www.selenium.dev/documentation/webdriver/waits/>
