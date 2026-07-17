# 02-Python-PyCharm
> 课时：40 min | 难度：★★

## 安装步骤

### 安装 Python 3.11+

**Windows：**

1. 访问 <https://www.python.org/downloads/>
2. 下载 Python 3.11 或 3.12 Windows 安装包（`python-3.11.x-amd64.exe`）
3. 安装时**勾选 "Add Python to PATH"**
4. 点击 "Install Now" 完成安装

**macOS：**

```bash
brew install python@3.11
```

或从官网下载 `.pkg` 安装包。

**Linux（Debian/Ubuntu）：**

```bash
sudo apt-get update
sudo apt-get install python3.11 python3.11-venv python3.11-pip
```

验证 Python 安装：

```bash
python --version
```

**预期输出：**

```
Python 3.11.x
```

### 安装 PyCharm Community

**所有平台：**

1. 访问 <https://www.jetbrains.com/pycharm/download/>
2. 下载 Community（免费）版本
3. 安装完成后首次启动，选择 "Do not import settings"
4. 设置主题和快捷键方案（推荐 "Windows" 或 "macOS" keymap）
5. 在 PowerShell/终端中运行 `Tools → Create Command-line Launcher`（macOS/Linux 自动配置）

### 创建虚拟环境

**通过 PyCharm：**

1. `File → New Project`
2. Location 选择项目路径
3. 选择 "Previously configured interpreter" → "Add Interpreter → Add Local Interpreter"
4. 选择 "Virtualenv Environment" → "New"
5. Base interpreter 选择 Python 3.11 路径，点击 OK

**通过命令行：**

```bash
python -m venv .venv
```

**激活虚拟环境：**

**Windows（PowerShell）：**

```powershell
.venv\Scripts\Activate.ps1

# 若遇执行策略错误，先执行：
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

**macOS/Linux：**

```bash
source .venv/bin/activate
```

激活后终端提示符前出现 `(.venv)` 标识。

### 安装测试依赖包

```bash
pip install selenium pytest requests pytest-html allure-pytest
```

验证安装：

```bash
pip list | findstr -i "selenium pytest requests"
```

**预期输出：**

```
pytest        7.4.x
requests      2.31.x
selenium      4.x.x
```

### 在 PyCharm 中运行第一个脚本

1. 右键项目根目录 → `New → Python File`，命名为 `test_demo.py`
2. 输入以下代码：

```python
import pytest

def test_addition():
    assert 1 + 1 == 2

def test_subtraction():
    assert 3 - 1 == 2
```

3. 右键编辑区 → `Run 'pytest in test_demo.py'`
4. 底部 Run 窗口显示 `2 passed in 0.01s`

## 验证安装

```bash
python -c "import selenium; print(selenium.__version__)"
pytest --version
```

**预期输出：**

```
4.15.x
pytest 7.4.x
```

## 常见坑

1. **Python 未加入 PATH** — 安装时未勾选 "Add Python to PATH"，需手动添加：`[Environment]::SetEnvironmentVariable("Path", "$env:Path;C:\Python311", "Machine")`
2. **pip 版本过旧** — 升级 pip：`python -m pip install --upgrade pip`
3. **PowerShell 执行策略阻止 venv 激活** — 执行 `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser`
4. **PyCharm 未识别 venv** — `File → Settings → Project → Python Interpreter → Add → Existing Environment`，选择 `.venv/Scripts/python.exe`
5. **pip 下载慢** — 使用国内镜像：`pip install -i https://pypi.tuna.tsinghua.edu.cn/simple selenium`

## 延伸阅读

- <https://www.python.org/downloads/>
- <https://www.jetbrains.com/pycharm/download/>
- <https://docs.pytest.org/en/stable/getting-started.html>
- <https://pypi.tuna.tsinghua.edu.cn/>
