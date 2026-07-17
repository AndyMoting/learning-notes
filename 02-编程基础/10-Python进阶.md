# 10-Python进阶

> 课时：120 min | 难度：★★★

## 学习目标

- 掌握类与对象的定义、属性与方法
- 理解继承、多态、封装三大特性
- 使用 try/except/else/finally 进行异常处理
- 区分模块与包的导入方式
- 理解装饰器与生成器的基本原理
- 使用 pip 管理依赖与虚拟环境

## 核心概念

### 1. 面向对象：类与对象

```python
class Animal:
    species = "Unknown"  # 类属性（所有实例共享）

    def __init__(self, name, age):
        self.name = name       # 实例属性
        self._age = age        # 约定私有属性（单下划线）

    def speak(self):
        return f"{self.name} 发出声音"

    def info(self):
        return f"{self.name} ({self.species}), {self._age} 岁"

# 实例化
dog = Animal("旺财", 3)
print(dog.info())   # 旺财 (Unknown), 3 岁
print(dog.speak())  # 旺财 发出声音

# 修改类属性
Animal.species = "哺乳动物"
print(dog.species)  # 哺乳动物
```

### 2. 三大特性

**继承**

```python
class Dog(Animal):
    def __init__(self, name, age, breed):
        super().__init__(name, age)
        self.breed = breed

    def speak(self):
        return f"{self.name} 汪汪叫"

    def fetch(self, item):
        return f"{self.name} 叼回了 {item}"

dog = Dog("旺财", 3, "金毛")
print(dog.speak())  # 旺财 汪汪叫
print(dog.fetch("球"))  # 旺财 叼回了球
```

**多态**

```python
class Cat(Animal):
    def speak(self):
        return f"{self.name} 喵喵叫"

def animal_sound(animals: list):
    for a in animals:
        print(a.speak())

animal_sound([Dog("旺财", 3, "金毛"), Cat("咪咪", 2)])
# 旺财 汪汪叫
# 咪咪 喵喵叫
```

**封装**

```python
class BankAccount:
    def __init__(self, owner, balance=0):
        self.owner = owner
        self.__balance = balance  # 双下划线触发名称改写（伪私有）

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("存入金额必须为正数")
        self.__balance += amount
        return self.__balance

    def withdraw(self, amount):
        if amount > self.__balance:
            raise ValueError("余额不足")
        self.__balance -= amount
        return self.__balance

    def get_balance(self):
        return self.__balance

account = BankAccount("Alice", 1000)
account.deposit(500)
print(account.get_balance())  # 1500
# print(account.__balance)    # 报错：AttributeError
# print(account._BankAccount__balance)  # 可强行访问，不推荐
```

### 3. 异常处理

```python
# 基础结构
result = None
try:
    result = 10 / 0
except ZeroDivisionError as e:
    print(f"除零错误: {e}")
except Exception as e:
    print(f"通用异常: {e}")
else:
    print(f"无异常，结果: {result}")
finally:
    print("无论是否异常都执行")

# 自定义异常
class ValidationError(Exception):
    """数据校验失败时抛出"""
    def __init__(self, field, message):
        self.field = field
        self.message = message
        super().__init__(f"[{field}] {message}")

def validate_age(age):
    if not isinstance(age, int):
        raise ValidationError("age", "必须为整数")
    if age < 0 or age > 150:
        raise ValidationError("age", "范围 0~150")
    return age

try:
    validate_age(-5)
except ValidationError as e:
    print(f"校验失败: {e.field} - {e.message}")
# 校验失败: age - 范围 0~150

# 主动抛出异常
def divide(a, b):
    if b == 0:
        raise ValueError("除数不能为零")
    return a / b

# raise 重新抛出
try:
    divide(10, 0)
except ValueError:
    print("捕获异常后重新抛出")
    raise
```

### 4. 模块与包

```python
# script: math_utils.py
def add(a, b):
    return a + b

PI = 3.14159

if __name__ == "__main__":
    # 仅在直接运行本文件时执行
    print("模块自测: add(2,3) =", add(2, 3))
```

```python
# 导入方式
# 1. 导入整个模块
import math_utils
print(math_utils.add(2, 3))

# 2. 导入特定成员
from math_utils import add, PI
print(add(2, 3))

# 3. 别名
from math_utils import add as math_add

# 4. 导入全部（不推荐，命名空间污染）
# from math_utils import *

# __name__ 区分运行方式
# 直接运行: __name__ == "__main__"
# 被导入:   __name__ == "math_utils"
```

**包结构**：

```
mypackage/
    __init__.py
    module_a.py
    module_b.py
```

```python
# __init__.py 控制对外暴露内容
from .module_a import func_a
from .module_b import func_b

# 使用
from mypackage import func_a, func_b
```

### 5. 装饰器

```python
import time
from functools import wraps

def timer(func):
    """计时装饰器"""
    @wraps(func)  # 保留原函数元信息
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        elapsed = time.time() - start
        print(f"{func.__name__} 耗时: {elapsed:.4f}s")
        return result
    return wrapper

def retry(max_attempts=3):
    """重试装饰器（带参数）"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    print(f"第 {attempt + 1} 次失败: {e}")
                    if attempt == max_attempts - 1:
                        raise
        return wrapper
    return decorator

@timer
def slow_function():
    time.sleep(0.5)
    return "done"

slow_function()

@retry(max_attempts=3)
def unstable_api():
    import random
    if random.random() < 0.7:
        raise ConnectionError("连接超时")
    return "success"
```

### 6. 生成器

```python
# 生成器函数
def fibonacci(n):
    a, b = 0, 1
    for _ in range(n):
        yield a
        a, b = b, a + b

# 惰性求值，节省内存
for num in fibonacci(10):
    print(num, end=" ")
# 0 1 1 2 3 5 8 13 21 34

# 生成器表达式
squares_gen = (x ** 2 for x in range(1000000))  # 不占内存
print(next(squares_gen))  # 0
print(next(squares_gen))  # 1

# 对比列表推导式
squares_list = [x ** 2 for x in range(1000000)]  # 立即生成全部
```

### 7. 虚拟环境与 pip

```bash
# 创建虚拟环境
python -m venv venv

# 激活（Windows）
venv\Scripts\activate

# 激活（macOS/Linux）
source venv/bin/activate

# 安装包
pip install requests pytest

# 安装指定版本
pip install requests==2.31.0

# 导出依赖清单
pip freeze > requirements.txt

# 从清单安装
pip install -r requirements.txt

# 卸载包
pip uninstall requests

# 退出虚拟环境
deactivate
```

## 常见坑

1. **`super().__init__()` 遗漏**：子类重写 `__init__` 未调用父类初始化，导致父类属性缺失
2. **可变类属性共享**：类属性为列表/字典时，所有实例引用同一对象，修改一处影响全部
3. **`except` 顺序错误**：子类异常必须写在父类之前，否则子类捕获永远不会执行
4. **`@wraps(func)` 遗漏**：装饰器未加 `@wraps` 会导致被装饰函数 `__name__`、`__doc__` 等元信息丢失
5. **循环导入**：模块 A 导入模块 B，B 又导入 A，应提取公共模块或延迟导入解决
6. **`yield` 与 `return` 混淆**：`yield` 不会终止函数，而是暂停并保存状态，下次从暂停处继续

## 自测清单

- [ ] 能否解释类属性与实例属性的区别
- [ ] 能否编写一个包含继承与方法重写的类体系
- [ ] 能否区分 `try/except/else/finally` 各子句的执行时机
- [ ] 能否说明 `__name__ == "__main__"` 的典型用途
- [ ] 能否编写一个带参数的装饰器
- [ ] 能否说明生成器相比列表的内存优势

## 延伸阅读

- 菜鸟教程 Python3 面向对象：https://www.runoob.com/python3/python3-class.html
- 菜鸟教程 Python3 异常处理：https://www.runoob.com/python3/python3-exceptions.html
- 菜鸟教程 Python3 模块：https://www.runoob.com/python3/python3-module.html
