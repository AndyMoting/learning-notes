# 01-Chrome-WebDriver
> 课时：30 min | 难度：★★

## 安装步骤

### 安装 Google Chrome

**Windows：**

1. 访问 <https://www.google.com/chrome/>
2. 下载 Windows 版安装包，双击 `ChromeSetup.exe`
3. 等待安装完成并启动 Chrome

**macOS：**

1. 下载 `.dmg` 文件
2. 将 Chrome 拖入 `Applications` 文件夹

**Linux（Debian/Ubuntu）：**

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt-get install -f
```

安装后验证 Chrome 版本：

```
chrome://settings/help
```

### 安装 ChromeDriver

#### 方式一：自动管理（Selenium 4.6+，推荐）

Selenium 4.6 起内置 `selenium-manager`，可自动下载匹配版本的 ChromeDriver，无需手动操作。

验证 selenium-manager 是否可用：

```python
from selenium import webdriver
from selenium.webdriver.chrome.service import Service

driver = webdriver.Chrome(service=Service())
driver.quit()
```

#### 方式二：手动下载

1. 查询 Chrome 版本：地址栏输入 `chrome://settings/help`
2. 访问 <https://googlechromelabs.github.io/chrome-for-testing/> 下载对应版本
3. 解压后将 `chromedriver.exe`（Windows）或 `chromedriver`（macOS/Linux）添加至 PATH

**Windows PATH 设置：**

```powershell
# 将 chromedriver.exe 放入 C:\WebDriver 后执行：
[Environment]::SetEnvironmentVariable("Path", "$env:Path;C:\WebDriver", "Machine")
```

**macOS/Linux PATH 设置：**

```bash
sudo mv chromedriver /usr/local/bin/
chmod +x /usr/local/bin/chromedriver
```

### 版本匹配规则

| Chrome 主版本 | ChromeDriver 主版本 |
|---------------|---------------------|
| 115.x         | 115.x               |
| 116.x         | 116.x               |
| 120.x         | 120.x               |

**规则：主版本号必须完全一致。** Selenium 4.6+ 通过 `selenium-manager` 自动匹配，无需手动对齐。

## 验证安装

```python
from selenium import webdriver
from selenium.webdriver.chrome.service import Service

driver = webdriver.Chrome(service=Service())
driver.get("https://www.baidu.com")
print(driver.title)
driver.quit()
```

**预期输出：**

```
百度一下，你就知道
```

或在终端执行：

```bash
chromedriver --version
```

**预期输出：**

```
ChromeDriver 120.0.6099.109 (...)
```

## 常见坑

1. **ChromeDriver 版本与 Chrome 不匹配** — 升级 Chrome 后需重新下载 ChromeDriver，或切换至 Selenium 4.6+ 自动管理
2. **chromedriver 未加入 PATH** — 手动指定路径：`Service(executable_path="/path/to/chromedriver")`，或将文件放入系统 PATH 目录
3. **Windows 缺少 Visual C++ Redistributable** — 从 <https://aka.ms/vs/17/release/vc_redist.x64.exe> 下载并安装
4. **Chrome 安装在非默认路径** — 指定 binary 位置：`options.binary_location = "C:/Program Files/Google/Chrome/Application/chrome.exe"`

## 延伸阅读

- <https://chromedriver.chromium.org/downloads>
- <https://www.selenium.dev/documentation/webdriver/getting_started/install_drivers/>
- <https://googlechromelabs.github.io/chrome-for-testing/>
