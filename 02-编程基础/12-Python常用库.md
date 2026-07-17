# 12-Python常用库

> 课时：120 min | 难度：★★★

## 学习目标

- 使用 json 模块解析与生成 JSON 数据
- 通过 os 模块完成路径与环境操作
- 运用 datetime 处理日期与时间计算
- 使用 re 模块编写正则表达式
- 配置 logging 记录测试日志
- 利用 random 生成各类测试数据

## 核心概念

### 1. json

JSON（JavaScript Object Notation）是 API 测试中最常见的数据交换格式。

```python
import json

# JSON 字符串 → Python 对象（反序列化）
json_str = '{"name": "Alice", "age": 30, "skills": ["Python", "SQL"]}'
data = json.loads(json_str)
print(data["name"])      # Alice
print(data["skills"][0]) # Python

# Python 对象 → JSON 字符串（序列化）
config = {
    "base_url": "https://api.example.com",
    "timeout": 30,
    "headers": {"Content-Type": "application/json"},
    "retry": True
}
json_output = json.dumps(config, ensure_ascii=False, indent=2)
print(json_output)
# {
#   "base_url": "https://api.example.com",
#   "timeout": 30,
#   "headers": {"Content-Type": "application/json"},
#   "retry": true
# }

# 文件读写
# 写入 JSON 文件
with open("config.json", "w", encoding="utf-8") as f:
    json.dump(config, f, ensure_ascii=False, indent=2)

# 读取 JSON 文件
with open("config.json", "r", encoding="utf-8") as f:
    loaded = json.load(f)
    print(loaded["base_url"])

# 测试场景：解析 API 响应
api_response = '''
{
    "code": 0,
    "message": "success",
    "data": {
        "user_id": 10001,
        "username": "testuser",
        "email": "test@example.com",
        "roles": ["tester", "developer"]
    }
}
'''
resp = json.loads(api_response)
assert resp["code"] == 0
assert "tester" in resp["data"]["roles"]
print(f"用户 {resp['data']['username']} 角色验证通过")
```

### 2. os

操作系统交互：路径操作、目录管理、环境变量。

```python
import os

# 路径操作
print(os.path.exists("/tmp"))           # 判断路径是否存在
print(os.path.isfile("config.json"))    # 是否为文件
print(os.path.isdir("logs"))            # 是否为目录

# 路径拼接（跨平台安全）
log_path = os.path.join("logs", "test", "2024-01-15.log")
print(log_path)  # logs\test\2024-01-15.log（Windows）/ logs/test/2024-01-15.log（Linux）

# 路径拆分
full_path = "/home/user/project/test.py"
print(os.path.dirname(full_path))   # /home/user/project
print(os.path.basename(full_path))  # test.py
print(os.path.splitext(full_path))  # ('/home/user/project/test', '.py')

# 获取绝对路径
print(os.path.abspath("config.json"))

# 目录操作
os.makedirs("reports/2024/january", exist_ok=True)  # 递归创建，已存在不报错
# os.remove("temp.txt")           # 删除文件
# os.rmdir("empty_dir")           # 删除空目录

# 列出目录内容
for entry in os.listdir("reports"):
    full = os.path.join("reports", entry)
    print(f"{entry} -> {'目录' if os.path.isdir(full) else '文件'}")

# 遍历目录树
for root, dirs, files in os.walk("reports"):
    for file in files:
        print(os.path.join(root, file))

# 环境变量
print(os.environ.get("PATH"))
print(os.environ.get("PYTHONPATH", "未设置"))
os.environ["TEST_ENV"] = "staging"
print(os.environ["TEST_ENV"])

# 执行系统命令（推荐 subprocess 替代）
exit_code = os.system("dir")  # Windows
# exit_code = os.system("ls")  # Linux/Mac
```

**推荐使用 pathlib（Python 3.4+）**：

```python
from pathlib import Path

# 路径构建
project_dir = Path("D:/Projects/笔记/软件测试")
config_file = project_dir / "02-编程基础" / "config.json"
print(config_file)          # D:\Projects\笔记\软件测试\02-编程基础\config.json
print(config_file.exists())
print(config_file.suffix)   # .json
print(config_file.stem)     # config

# 读写文件
config_file.write_text(json.dumps({"key": "value"}, indent=2), encoding="utf-8")
content = config_file.read_text(encoding="utf-8")

# 遍历
for py_file in project_dir.rglob("*.md"):
    print(py_file.name)
```

### 3. datetime

日期时间处理：格式化、运算、时间戳。

```python
from datetime import datetime, date, timedelta, timezone

# 获取当前时间
now = datetime.now()
print(now)              # 2024-01-15 14:30:25.123456
print(date.today())     # 2024-01-15

# 创建指定时间
deadline = datetime(2024, 12, 31, 23, 59, 59)
print(deadline)

# 格式化输出
print(now.strftime("%Y-%m-%d"))             # 2024-01-15
print(now.strftime("%Y-%m-%d %H:%M:%S"))    # 2024-01-15 14:30:25
print(now.strftime("%Y年%m月%d日"))          # 2024年01月15日

# 字符串解析为 datetime
date_str = "2024-01-15 14:30:00"
parsed = datetime.strptime(date_str, "%Y-%m-%d %H:%M:%S")
print(parsed)

# 时间运算
tomorrow = now + timedelta(days=1)
last_week = now - timedelta(weeks=1)
two_hours_later = now + timedelta(hours=2, minutes=30)

# 计算时间差
start = datetime(2024, 1, 1)
end = datetime(2024, 1, 15)
delta = end - start
print(delta.days)        # 14
print(delta.total_seconds())  # 1209600.0

# 时间戳
timestamp = now.timestamp()     # datetime → 秒级时间戳
print(int(timestamp))           # 1705300225
dt_from_ts = datetime.fromtimestamp(timestamp)  # 时间戳 → datetime

# UTC 时间
utc_now = datetime.now(timezone.utc)
print(utc_now)

# 测试场景：生成测试报告文件名
report_name = f"test_report_{now.strftime('%Y%m%d_%H%M%S')}.html"
print(report_name)  # test_report_20240115_143025.html

# 测试场景：检查用例是否过期
case_create = datetime(2024, 1, 1)
expire_days = 30
is_expired = (now - case_create).days > expire_days
print(f"用例已过期: {is_expired}")
```

### 4. re

正则表达式：模式匹配、文本提取、数据清洗。

```python
import re

text = "联系邮箱: alice@example.com, 电话: 138-0013-8000, 备用: bob@test.org"

# search：查找第一个匹配
match = re.search(r'\d{3}-\d{4}-\d{4}', text)
if match:
    print(match.group())   # 138-0013-8000
    print(match.start())   # 起始位置
    print(match.end())     # 结束位置

# findall：查找所有匹配
emails = re.findall(r'[\w.+-]+@[\w-]+\.[\w.]+', text)
print(emails)  # ['alice@example.com', 'bob@test.org']

# sub：替换
cleaned = re.sub(r'\d', '*', "密码: abc123def")
print(cleaned)  # 密码: abc***

# split：按模式分割
parts = re.split(r'[,;]\s*', "apple,banana; cherry,durian")
print(parts)  # ['apple', 'banana', 'cherry', 'durian']

# compile：预编译（多次使用提升效率）
phone_pattern = re.compile(r'(\d{3})-(\d{4})-(\d{4})')
m = phone_pattern.search(text)
if m:
    print(m.group(0))  # 138-0013-8000  完整匹配
    print(m.group(1))  # 138            第一组
    print(m.group(2))  # 0013           第二组

# 常用模式速查
patterns = {
    "邮箱":     r'^[\w.+-]+@[\w-]+\.[\w.]+$',
    "手机号":   r'^1[3-9]\d{9}$',
    "IPv4":    r'\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}',
    "日期":     r'\d{4}-\d{2}-\d{2}',
    "URL":     r'https?://[^\s]+',
    "中文字符":  r'[\u4e00-\u9fff]+',
}

# 测试场景：API 响应字段校验
response_text = '{"status": "success", "code": 200, "data": {"id": 12345}}'
code_match = re.search(r'"code":\s*(\d+)', response_text)
if code_match:
    status_code = int(code_match.group(1))
    print(f"接口状态码: {status_code}")

# 测试场景：日志解析
log_line = '2024-01-15 14:30:25 [ERROR] Login failed for user "alice" from 192.168.1.100'
log_pattern = re.compile(
    r'(\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2})\s\[(\w+)\]\s(.+)'
)
m = log_pattern.match(log_line)
if m:
    timestamp, level, message = m.groups()
    print(f"时间: {timestamp}, 级别: {level}, 信息: {message}")
```

### 5. logging

分级日志输出，替代 print 进行调试与审计。

```python
import logging

# 基础配置
logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s [%(levelname)s] %(name)s - %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
    filename="test.log",
    filemode="a",
    encoding="utf-8"
)

logger = logging.getLogger("TestSuite")

# 日志级别（由低到高）
logger.debug("调试信息：变量 x = %s", 42)
logger.info("执行登录用例")
logger.warning("响应时间超过阈值：%.2fs", 3.5)
logger.error("断言失败：期望 200 实际 404")
logger.critical("数据库连接中断")

# 高级配置：多 Handler
console_handler = logging.StreamHandler()
console_handler.setLevel(logging.INFO)
console_handler.setFormatter(
    logging.Formatter("%(levelname)s - %(message)s")
)

file_handler = logging.FileHandler("error.log", encoding="utf-8")
file_handler.setLevel(logging.ERROR)
file_handler.setFormatter(
    logging.Formatter("%(asctime)s [%(levelname)s] %(filename)s:%(lineno)d - %(message)s")
)

test_logger = logging.getLogger("AdvancedTest")
test_logger.setLevel(logging.DEBUG)
test_logger.addHandler(console_handler)
test_logger.addHandler(file_handler)

test_logger.info("同时输出到控制台和文件")
test_logger.error("仅出现在 error.log 与控制台")

# 测试场景：记录用例执行
def run_case(case_name, duration, passed):
    if passed:
        logger.info("用例 [%s] 通过 | 耗时 %.2fs", case_name, duration)
    else:
        logger.error("用例 [%s] 失败 | 耗时 %.2fs", case_name, duration)

run_case("TC-001-登录", 0.85, True)
run_case("TC-002-支付", 2.30, False)
```

### 6. random

生成随机测试数据：整数、浮点、序列、字符串、抽样。

```python
import random
import string

# 整数
print(random.randint(1, 100))       # [1, 100] 闭区间整数
print(random.randrange(0, 100, 5))  # 0,5,10,...,95  5的倍数

# 浮点数
print(random.random())              # [0.0, 1.0)
print(random.uniform(1.5, 9.9))     # [1.5, 9.9] 均匀分布

# 序列操作
fruits = ["apple", "banana", "cherry", "durian"]
print(random.choice(fruits))        # 随机取一个
print(random.sample(fruits, 2))     # 随机取 2 个（不重复）
random.shuffle(fruits)              # 原地打乱
print(fruits)

# 字符串
chars = string.ascii_letters + string.digits
random_str = ''.join(random.choices(chars, k=12))
print(random_str)  # 如: aK7mP2xR9nQ1

# 中文字符
chinese_chars = [chr(random.randint(0x4e00, 0x9fff)) for _ in range(10)]
print(''.join(chinese_chars))

# 设置随机种子（保证可复现）
random.seed(42)
print(random.randint(1, 100))  # 固定输出 82
random.seed(42)
print(random.randint(1, 100))  # 仍为 82

# 测试场景：批量生成测试数据
def generate_users(n=10):
    """生成随机用户数据"""
    first_names = ["张", "李", "王", "刘", "陈", "杨", "赵", "黄"]
    last_names = ["伟", "芳", "娜", "敏", "静", "丽", "强", "磊"]
    domains = ["qq.com", "163.com", "gmail.com", "outlook.com"]
    
    users = []
    for _ in range(n):
        name = random.choice(first_names) + random.choice(last_names)
        phone = f"1{random.choice('35789')}{''.join(random.choices(string.digits, k=9))}"
        email = ''.join(random.choices(string.ascii_lowercase, k=8)) + "@" + random.choice(domains)
        age = random.randint(18, 65)
        users.append({"name": name, "phone": phone, "email": email, "age": age})
    return users

for user in generate_users(3):
    print(user)

# 测试场景：随机抽取测试集
all_cases = [f"TC-{i:04d}" for i in range(1, 501)]
sampled = random.sample(all_cases, k=50)
print(f"从 {len(all_cases)} 个用例中随机抽取 {len(sampled)} 个")
```

## 常见坑

1. **`json.dumps` 与 `json.dump`**：带 `s` 的操作对象为字符串，无 `s` 的操作对象为文件句柄，两者不可混淆
2. **datetime 时区缺失**：`datetime.now()` 返回无时区信息的 naive datetime，跨时区系统应使用 `datetime.now(timezone.utc)`
3. **正则回溯灾难**：`r'(a+)+b'` 模式在长串 `a` 后无匹配时指数级回溯，应改为 `r'a+b'` 等避免嵌套量词
4. **logging 重复配置**：`basicConfig` 在同一进程仅首次生效，后续调用被忽略；多模块应在入口统一配置
5. **`random` 非密码学安全**：涉及 token、密钥等敏感生成使用 `secrets` 模块（`secrets.token_hex(16)`）
6. **`os.path.join` 遇绝对路径重置**：`os.path.join("/a/b", "/c/d")` 结果为 `/c/d`，绝对路径会丢弃前面的部分

## 自测清单

- [ ] 能否区分 `json.loads` 与 `json.load` 的使用场景
- [ ] 能否使用 `os.path.join` 构建跨平台路径
- [ ] 能否将 datetime 对象与字符串互相转换
- [ ] 能否编写匹配邮箱、手机号的正则表达式
- [ ] 能否配置 logging 同时输出到控制台和文件
- [ ] 能否使用 `random.seed` 保证测试数据的复现性

## 延伸阅读

- 菜鸟教程 Python3 JSON：https://www.runoob.com/python3/python3-json.html
- 菜鸟教程 Python3 OS 文件/目录：https://www.runoob.com/python3/python3-os-files-methods.html
- 菜鸟教程 Python3 日期时间：https://www.runoob.com/python3/python3-date-time.html
- 菜鸟教程 Python3 正则表达式：https://www.runoob.com/python3/python3-reg-expressions.html
- 菜鸟教程 Python3 logging：https://www.runoob.com/python3/python3-logging.html
- 菜鸟教程 Python3 random：https://www.runoob.com/python3/python3-number-random.html
