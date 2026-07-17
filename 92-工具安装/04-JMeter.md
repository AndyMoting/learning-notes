# 04-JMeter
> 课时：45 min | 难度：★★★

## 安装步骤

### 安装 Java JDK 11+

**Windows：**

1. 访问 <https://adoptium.net/>（Eclipse Temurin）下载 JDK 17 LTS
2. 选择 `.msi` 安装包，安装时勾选 "Set JAVA_HOME" 和 "Add to PATH"
3. 安装完成后重启终端

**macOS：**

```bash
brew install openjdk@17
sudo ln -sfn /opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/openjdk-17.jdk
```

**Linux（Debian/Ubuntu）：**

```bash
sudo apt-get install openjdk-17-jdk
```

验证 Java 安装：

```bash
java -version
```

**预期输出：**

```
openjdk version "17.0.x" 2024-xx-xx
OpenJDK Runtime Environment Temurin-17.0.x
```

### 下载并解压 JMeter

1. 访问 <https://jmeter.apache.org/download.cgi>
2. 下载 `apache-jmeter-5.6.x.zip`（Binaries 列）
3. 解压至目标目录：

**Windows：** `C:\apache-jmeter-5.6`

**macOS/Linux：** `~/apache-jmeter-5.6`

4. 启动 JMeter：

**Windows：**

```powershell
cd C:\apache-jmeter-5.6\bin
jmeter.bat
```

**macOS/Linux：**

```bash
cd ~/apache-jmeter-5.6/bin
./jmeter.sh
```

### 安装插件管理器

1. 访问 <https://jmeter-plugins.org/install/Install/>
2. 下载 `jmeter-plugins-manager-1.9.jar`
3. 将 jar 文件放入 `lib/ext` 目录
4. 重启 JMeter
5. 菜单 `Options → Plugins Manager`
6. 在 `Available Plugins` 中搜索并安装：
   - **3 Basic Graphs**
   - **Custom Thread Groups**
   - **PerfMon Metrics Collector**
7. 点击 `Apply Changes and Restart JMeter`

### 创建第一个测试计划

1. 右键 `Test Plan` → `Add → Threads (Users) → Thread Group`
2. 设置：
   - Number of Threads: `10`
   - Ramp-up period: `5`
   - Loop Count: `3`
3. 右键 Thread Group → `Add → Sampler → HTTP Request`
4. 设置：
   - Server Name: `httpbin.org`
   - Path: `/get`
   - Method: `GET`
5. 右键 Thread Group → `Add → Listener → View Results Tree`
6. 右键 Thread Group → `Add → Listener → Summary Report`
7. 点击绿色运行按钮或 `Ctrl+R`
8. 在 View Results Tree 中查看请求响应

### CLI 模式运行

```bash
jmeter -n -t test_plan.jmx -l results.jtl -e -o report_folder
```

参数说明：

| 参数 | 说明           |
|------|----------------|
| `-n` | 非 GUI 模式    |
| `-t` | 测试计划路径   |
| `-l` | 结果日志路径   |
| `-e` | 生成 HTML 报告 |
| `-o` | 报告输出目录   |

## 验证安装

```bash
jmeter --version
```

**预期输出：**

```
Copyright © 1998-2023 The Apache Software Foundation
Version 5.6.x
```

CLI 模式验证：

```bash
jmeter -n -t test_plan.jmx -l results.jtl
```

**预期输出：

```
Starting the test @ Thu Jul 17 10:00:00 CST 2026
summary =     30 in 00:00:03 =   10.0/s Avg:   123 Min:    45 Max:   987 Err:     0 (0.00%)
Tidying up ...    @ Thu Jul 17 10:00:03 CST 2026
... end of run
```

## 常见坑

1. **JMeter 启动闪退** — JAVA_HOME 未设置或指向 JRE 而非 JDK，确认 `echo %JAVA_HOME%`（Windows）或 `echo $JAVA_HOME`（macOS/Linux）指向 JDK 目录
2. **GUI 模式无法用于压测** — 正式压测必须使用 CLI 模式（`-n`），GUI 仅用于调试
3. **中文乱码** — 在 `jmeter.properties` 中设置 `sampleresult.default.encoding=UTF-8`
4. **插件安装失败** — 检查网络连接，或手动下载 jar 放入 `lib/ext`
5. **Windows 下 jmeter.bat 无响应** — 以管理员身份运行 PowerShell，或检查 JDK 版本是否兼容

## 延伸阅读

- <https://jmeter.apache.org/download.cgi>
- <https://jmeter.apache.org/usermanual/get-started.html>
- <https://jmeter-plugins.org/>
- <https://jmeter.apache.org/usermanual/best-practices.html>
