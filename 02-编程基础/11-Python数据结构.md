# 11-Python数据结构

> 课时：90 min | 难度：★★★

## 学习目标

- 掌握 List 常用操作与列表推导式
- 理解 Tuple 不可变特性与解包语法
- 熟练使用 Dict 及嵌套字典结构
- 运用 Set 完成集合运算
- 根据测试场景选择合适的数据结构

## 核心概念

### 1. List（列表）

有序可变序列，支持索引与切片。

```python
# 创建
cases = ["login", "register", "payment", "login"]
empty = list()
mixed = [1, "text", True, [2, 3]]

# 常用方法
cases.append("search")          # 末尾添加
cases.insert(1, "logout")       # 指定位置插入
cases.extend(["cart", "order"]) # 扩展可迭代对象
print(cases)
# ['login', 'logout', 'register', 'payment', 'login', 'search', 'cart', 'order']

removed = cases.pop()           # 删除并返回末尾元素
cases.pop(1)                    # 删除指定索引
cases.remove("login")           # 删除第一个匹配值
print(cases)
# ['register', 'payment', 'login', 'search', 'cart']

# 查找与统计
print(cases.index("payment"))   # 1
print(cases.count("login"))     # 1
print(len(cases))               # 5
print("search" in cases)        # True

# 排序
numbers = [3, 1, 4, 1, 5, 9, 2, 6]
numbers.sort()                  # 原地排序
print(numbers)                  # [1, 1, 2, 3, 4, 5, 6, 9]

words = ["banana", "apple", "cherry"]
words.sort(key=len)             # 按长度排序
print(words)                    # ['apple', 'banana', 'cherry']

# sorted 返回新列表
original = [3, 1, 4, 1, 5]
new_list = sorted(original, reverse=True)
print(original)  # [3, 1, 4, 1, 5]  不变
print(new_list)  # [5, 4, 3, 1, 1]
```

**列表推导式**

```python
# 基础形式
squares = [x ** 2 for x in range(10)]
print(squares)  # [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

# 带条件过滤
evens = [x for x in range(20) if x % 2 == 0]

# 多重循环（笛卡尔积）
pairs = [(x, y) for x in range(3) for y in range(3) if x != y]
print(pairs)  # [(0,1),(0,2),(1,0),(1,2),(2,0),(2,1)]

# 嵌套列表扁平化
matrix = [[1, 2, 3], [4, 5], [6, 7, 8, 9]]
flat = [num for row in matrix for num in row]
print(flat)  # [1, 2, 3, 4, 5, 6, 7, 8, 9]

# 测试场景：批量生成测试用例 ID
case_ids = [f"TC-{i:04d}" for i in range(1, 101)]
print(case_ids[:3])  # ['TC-0001', 'TC-0002', 'TC-0003']
```

### 2. Tuple（元组）

有序不可变序列，创建后不能修改。

```python
# 创建
config = ("localhost", 3306, "root", "password")
single = (42,)       # 单元素必须加逗号
empty = ()
no_parens = 1, 2, 3  # 括号可省略

# 不可变特性
# config[0] = "127.0.0.1"  # TypeError

# 解包
host, port, user, pwd = config
print(host, port)  # localhost 3306

# 扩展解包
first, *middle, last = (1, 2, 3, 4, 5)
print(middle)  # [2, 3, 4]

# 交换变量（底层使用元组）
a, b = 10, 20
a, b = b, a
print(a, b)  # 20 10

# 元组作为字典键（列表不允许）
cache = {(0, 0): "origin", (1, 0): "east"}
print(cache[(0, 0)])  # origin

# 常用方法
data = (1, 2, 3, 2, 4, 2)
print(data.count(2))  # 3
print(data.index(3))  # 2

# 测试场景：函数返回多值
def analyze_results(results):
    passed = results.count("PASS")
    failed = results.count("FAIL")
    total = len(results)
    return passed, failed, total  # 返回元组

p, f, t = analyze_results(["PASS", "FAIL", "PASS", "PASS"])
print(f"通过 {p}/{t}, 失败 {f}")
```

### 3. Dict（字典）

键值对映射结构，键必须为不可变类型。

```python
# 创建
user = {"name": "Alice", "age": 30, "city": "Beijing"}
user2 = dict(name="Bob", age=25)

# 访问
print(user["name"])        # Alice
print(user.get("salary", 0))  # 0  键不存在返回默认值

# 添加与修改
user["email"] = "alice@example.com"
user.update({"age": 31, "dept": "QA"})

# 删除
email = user.pop("email")       # 删除并返回值
del user["city"]

# 遍历
for key in user:
    print(key, user[key])

for key, value in user.items():
    print(f"{key}: {value}")

for value in user.values():
    print(value)

# 常用方法
print(user.keys())    # dict_keys(['name', 'age', 'dept'])
print(user.values())  # dict_values(['Alice', 31, 'QA'])
```

**字典推导式**

```python
# 基础形式
square_dict = {x: x ** 2 for x in range(6)}
print(square_dict)  # {0:0, 1:1, 2:4, 3:9, 4:16, 5:25}

# 过滤
evens_dict = {x: x ** 2 for x in range(10) if x % 2 == 0}

# 测试场景：状态码映射
status_map = {
    200: "OK", 301: "Redirect", 400: "Bad Request",
    401: "Unauthorized", 403: "Forbidden", 404: "Not Found",
    500: "Server Error", 502: "Bad Gateway", 503: "Service Unavailable"
}

codes = [200, 301, 404, 500, 999]
for code in codes:
    print(f"{code} -> {status_map.get(code, 'Unknown')}")
```

**嵌套字典**

```python
# 测试报告结构
test_report = {
    "project": "支付系统",
    "version": "2.1.0",
    "summary": {"total": 100, "passed": 92, "failed": 8},
    "modules": {
        "login": {"passed": 20, "failed": 1},
        "payment": {"passed": 35, "failed": 3},
        "order": {"passed": 37, "failed": 4}
    }
}

# 深层访问
print(test_report["modules"]["payment"]["failed"])  # 3

# 安全深层访问（防止 KeyError）
payment_failed = test_report.get("modules", {}).get("payment", {}).get("failed", 0)

# 遍历嵌套结构
for module, result in test_report["modules"].items():
    total = result["passed"] + result["failed"]
    rate = result["passed"] / total * 100
    print(f"{module}: 通过率 {rate:.1f}%")
```

### 4. Set（集合）

无序不重复元素集，支持集合运算。

```python
# 创建
browsers = {"Chrome", "Firefox", "Safari", "Chrome"}  # 自动去重
print(browsers)  # {'Chrome', 'Firefox', 'Safari'}

empty_set = set()      # 空集合（{} 是空字典）
from_list = set([1, 2, 2, 3, 3, 3])
print(from_list)       # {1, 2, 3}

# 增删
browsers.add("Edge")
browsers.discard("IE")     # 不存在不报错
browsers.remove("Safari")  # 不存在则 KeyError

# 集合运算
set_a = {1, 2, 3, 4, 5}
set_b = {4, 5, 6, 7, 8}

print(set_a | set_b)   # {1,2,3,4,5,6,7,8}  并集
print(set_a & set_b)   # {4, 5}            交集
print(set_a - set_b)   # {1, 2, 3}         差集
print(set_a ^ set_b)   # {1,2,3,6,7,8}     对称差集

# 子集与超集
small = {1, 2}
print(small <= set_a)   # True  子集
print(set_a >= small)   # True  超集

# 测试场景：环境差异比较
prod_env = {"Python 3.10", "Redis 7.0", "MySQL 8.0", "Nginx 1.24"}
test_env = {"Python 3.10", "Redis 7.0", "MySQL 5.7", "Nginx 1.24"}

missing_in_test = prod_env - test_env
extra_in_test = test_env - prod_env
print(f"测试环境缺失: {missing_in_test}")
print(f"测试环境多余: {extra_in_test}")

# 测试场景：去重后排列
duplicated_ids = [101, 102, 103, 101, 104, 102, 105]
unique_sorted = sorted(set(duplicated_ids))
print(unique_sorted)  # [101, 102, 103, 104, 105]
```

## 数据结构的测试场景选择

| 场景 | 推荐结构 | 理由 |
|------|---------|------|
| 有序测试用例队列 | List | 保持插入顺序，支持索引访问 |
| 数据库连接配置 | Tuple | 不可变，保证配置不被意外修改 |
| API 响应字段校验 | Dict | 键值映射，便于字段查找与嵌套验证 |
| 批量数据去重 | Set | 自动去重，支持集合运算对比 |
| 二维测试矩阵 | List[List] / Dict | 表格式/记录式数据表示 |
| 动态测试结果追加 | List + Dict | 列表存储顺序，字典存储详细信息 |

## 常见坑

1. **`dict` 键不可变**：列表、字典不可做键，需转为元组或冻结集合
2. **`set` 无序不重复**：依赖顺序的场景不可用 Set，结果无法按固定顺序取出
3. **列表浅拷贝陷阱**：`list2 = list1` 为引用拷贝，修改互相影响；应使用 `list2 = list1.copy()` 或 `list(list1)`
4. **遍历时修改字典**：`for k in d: del d[k]` 报错 `RuntimeError`，应遍历键的副本 `for k in list(d)`
5. **`dict.get(key, default)` 惰性求值**：默认值在每次调用时都会计算，昂贵操作应避免放 default

## 自测清单

- [ ] 能否区分 `list.append()` 与 `list.extend()` 的行为差异
- [ ] 能否解释为何元组可做字典键而列表不能
- [ ] 能否编写三层嵌套字典的访问与修改代码
- [ ] 能否使用集合运算找出两个环境配置的差异
- [ ] 能否描述列表推导式中多层 `for` 的执行顺序
- [ ] 能否说明 `copy()` 与深拷贝 `deepcopy` 的区别

## 延伸阅读

- 菜鸟教程 Python3 列表：https://www.runoob.com/python3/python3-list.html
- 菜鸟教程 Python3 元组：https://www.runoob.com/python3/python3-tuple.html
- 菜鸟教程 Python3 字典：https://www.runoob.com/python3/python3-dictionary.html
- 菜鸟教程 Python3 集合：https://www.runoob.com/python3/python3-set.html
