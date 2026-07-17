# 09-Python基础语法

> 课时：90 min | 难度：★★

## 学习目标

- 掌握 Python 变量定义与六大基础数据类型
- 熟练使用算术、比较、逻辑、身份、成员五类运算符
- 运用 if/for/while 构建控制流
- 完成字符串常用操作与格式化输出
- 区分函数参数形式并编写 lambda 表达式

## 核心概念

### 1. 变量与数据类型

Python 采用动态类型，赋值即声明，无需指定类型标识。

```python
# 整数
count = 100
# 浮点数
price = 19.99
# 字符串
name = "Alice"
# 布尔
passed = True
# 空值
result = None

print(type(count))   # <class 'int'>
print(type(price))   # <class 'float'>
print(type(name))    # <class 'str'>
print(type(passed))  # <class 'bool'>
print(type(result))  # <class 'NoneType'>
```

类型转换：

```python
num_str = "42"
num_int = int(num_str)       # 字符串转整数
num_float = float(num_str)   # 字符串转浮点
str_val = str(num_int)       # 数字转字符串
```

### 2. 运算符

**算术运算符**

```python
a, b = 10, 3
print(a + b)   # 13
print(a - b)   # 7
print(a * b)   # 30
print(a / b)   # 3.333...
print(a // b)  # 3  整除
print(a % b)   # 1  取余
print(a ** b)  # 1000  幂运算
```

**比较运算符**

```python
x, y = 5, 10
print(x == y)  # False
print(x != y)  # True
print(x < y)   # True
print(x > y)   # False
print(x <= 5)  # True
print(x >= 6)  # False
```

**逻辑运算符**

```python
flag_a, flag_b = True, False
print(flag_a and flag_b)  # False
print(flag_a or flag_b)   # True
print(not flag_a)         # False
```

**身份运算符（比较内存地址）**

```python
list_a = [1, 2, 3]
list_b = [1, 2, 3]
list_c = list_a

print(list_a == list_b)   # True  值相等
print(list_a is list_b)   # False  不同对象
print(list_a is list_c)   # True  同一对象
print(list_a is not list_b)  # True
```

**成员运算符**

```python
data = [1, 2, 3, 4, 5]
print(3 in data)      # True
print(6 not in data)  # True

text = "hello world"
print("world" in text)  # True
```

### 3. 控制流

**if/elif/else**

```python
score = 85

if score >= 90:
    grade = "A"
elif score >= 80:
    grade = "B"
elif score >= 70:
    grade = "C"
else:
    grade = "D"

print(f"等级: {grade}")  # 等级: B
```

**for 循环**

```python
# 遍历列表
cases = ["login", "register", "payment"]
for case in cases:
    print(case)

# range 生成数字序列
for i in range(5):        # 0~4
    print(i)

for i in range(2, 10, 2): # 2,4,6,8  步长为2
    print(i)

# enumerate 获取索引
for idx, case in enumerate(cases, start=1):
    print(f"{idx}. {case}")
```

**while 循环**

```python
count = 0
while count < 3:
    print(f"执行第 {count + 1} 次")
    count += 1
```

**break/continue/pass**

```python
# break：终止整个循环
for i in range(10):
    if i == 5:
        break
    print(i)  # 输出 0 1 2 3 4

# continue：跳过当前迭代
for i in range(5):
    if i == 2:
        continue
    print(i)  # 输出 0 1 3 4

# pass：占位空语句
for i in range(3):
    pass  # 语法需要但无操作
```

### 4. 字符串操作

```python
text = "  Hello, Python World  "

# 去空白
print(text.strip())    # "Hello, Python World"
print(text.lstrip())   # "Hello, Python World  "
print(text.rstrip())    # "  Hello, Python World"

# 大小写
print(text.upper())    # "  HELLO, PYTHON WORLD  "
print(text.lower())    # "  hello, python world  "

# 替换
print(text.replace("Python", "Testing"))  # "  Hello, Testing World  "

# 分割
words = "login,register,payment".split(",")
print(words)  # ['login', 'register', 'payment']
print(",".join(words))  # "login,register,payment"

# 索引与切片
s = "abcdefg"
print(s[0])     # a  正向索引从0开始
print(s[-1])    # g  反向索引从-1开始
print(s[1:4])   # bcd  左闭右开
print(s[:3])    # abc  从头开始
print(s[::2])   # aceg  步长为2
print(s[::-1])  # gfedcba  反转

# 查找与统计
print(s.find("c"))      # 2  找不到返回-1
print(s.index("c"))     # 2  找不到抛出 ValueError
print("abcabc".count("a"))  # 2
```

**格式化输出**

```python
name, score, rate = "Alice", 95.5, 0.875

# f-string（推荐）
print(f"姓名: {name}, 分数: {score:.1f}, 通过率: {rate:.1%}")
# 姓名: Alice, 分数: 95.5, 通过率: 87.50%

# format 方法
print("姓名: {}, 分数: {:.1f}".format(name, score))

# 旧式 % 格式化
print("姓名: %s, 分数: %.1f" % (name, score))
```

### 5. 输入/输出

```python
# 控制台输出
print("Hello", "World", sep="-", end="!\n")  # Hello-World!

# 控制台输入（返回字符串）
# user_input = input("请输入: ")
# print(f"输入内容: {user_input}")

# 写入文件
with open("output.txt", "w", encoding="utf-8") as f:
    f.write("第一行\n")
    f.write("第二行\n")

# 读取全部内容
with open("output.txt", "r", encoding="utf-8") as f:
    content = f.read()
    print(content)

# 逐行读取
with open("output.txt", "r", encoding="utf-8") as f:
    for line in f:
        print(line.strip())

# 追加写入
with open("output.txt", "a", encoding="utf-8") as f:
    f.write("第三行\n")
```

### 6. 函数

```python
# 基础函数
def add(a, b):
    """返回两数之和"""
    return a + b

print(add(3, 5))  # 8

# 默认参数
def connect(host, port=3306, timeout=30):
    return f"连接 {host}:{port}, 超时 {timeout}s"

print(connect("localhost"))                    # 连接 localhost:3306, 超时 30s
print("连接 " + connect("db.server.com", port=5432))

# 可变位置参数 *args
def sum_all(*args):
    return args, sum(args)

print(sum_all(1, 2, 3, 4))
# ((1, 2, 3, 4), 10)

# 可变关键字参数 **kwargs
def build_config(**kwargs):
    for key, value in kwargs.items():
        print(f"{key} = {value}")

build_config(browser="Chrome", headless=True, timeout=60)

# 混合参数顺序：位置参数 -> *args -> 默认参数 -> **kwargs
def func(a, b, *args, mode="default", **kwargs):
    print(a, b, args, mode, kwargs)

func(1, 2, 3, 4, mode="fast", debug=True, env="test")
# 1 2 (3, 4) fast {'debug': True, 'env': 'test'}

# 返回多个值（本质是返回元组）
def get_stats(numbers):
    return min(numbers), max(numbers), sum(numbers) / len(numbers)

minimum, maximum, average = get_stats([10, 20, 30, 40])

# lambda 表达式
square = lambda x: x ** 2
print(square(5))  # 25

# lambda 配合高阶函数
data = [{"name": "Alice", "score": 90}, {"name": "Bob", "score": 85}]
data.sort(key=lambda x: x["score"], reverse=True)
print(data)  # [{'name': 'Alice', 'score': 90}, {'name': 'Bob', 'score': 85}]

# 作用域规则
count = 100  # 全局变量

def modify():
    global count  # 声明使用全局变量
    count = 200

modify()
print(count)  # 200
```

## 常见坑

1. **可变对象默认参数**：`def func(lst=[])` 中列表被所有调用共享，应使用 `def func(lst=None)` 并在函数内判空初始化
2. **浮点数精度**：`0.1 + 0.2 != 0.3`，涉及金额计算应使用 `Decimal`
3. **字符串驻留与 `is` 误用**：`is` 比较内存地址而非值，字符串/列表比较务必使用 `==`
4. **文件未关闭**：不使用 `with` 时需手动 `close()`，否则可能丢失缓冲区数据
5. **`range` 返回范围对象而非列表**：Python 3 中 `range(5)` 不生成完整列表，需显式转换 `list(range(5))`
6. **切片越界不报错**：`s[1:100]` 在字符串长度不足时正常返回，需自行校验边界

## 自测清单

- [ ] 能否正确区分 `==` 与 `is` 的使用场景
- [ ] 能否独立编写包含 `elif` 分支的多条件判断
- [ ] 能否解释 `*args` 与 `**kwargs` 的区别与调用方式
- [ ] 能否使用 `with` 语句安全读写文件
- [ ] 能否编写至少使用三种格式化方式的输出语句
- [ ] 能否描述 `global` 关键字的适用场景与替代方案

## 延伸阅读

- 菜鸟教程 Python3 基础语法：https://www.runoob.com/python3/python3-basic-syntax.html
- 菜鸟教程 Python3 数据类型：https://www.runoob.com/python3/python3-data-type.html
- 菜鸟教程 Python3 运算符：https://www.runoob.com/python3/python3-basic-operators.html
