# 31-ADB命令

> 课时：40 min | 难度：★★☆☆☆

## 学习目标

- 掌握 ADB（Android Debug Bridge）的安装与配置
- 熟练使用 15 个高频 ADB 测试命令
- 能够通过 ADB 完成设备管理、安装日志采集等操作
- 能够通过 screencap、screenrecord、input 命令模拟用户操作
- 能够通过 dumpsys 获取系统级信息辅助性能分析
- 掌握 logcat 日志的过滤方法

## 核心概念

### 31.1 ADB 架构

ADB 采用 C/S 架构，由三部分组成：

```
┌────────┐     USB/WiFi      ┌────────┐     shell/protocol     ┌──────────┐
│ Client │ ◄───────────────► │ Daemon │ ◄─────────────────────► │  Device  │
│ (CLI)  │                   │ (ADB   │                         │(emulator │
│        │                   │Server) │                         │ or real) │
└────────┘                   └────────┘                         └──────────┘
```

- **Client**：运行在 PC 上的命令行工具，负责发送指令
- **Daemon（adbd）**：运行在 Android 设备上的后台进程
- **ADB Server**：PC 上的后台服务，管理 Client 与 Daemon 之间的通信

### 31.2 安装与配置

**安装方式**
1. 通过 Android Studio 自带 SDK Platform-Tools
2. 单独下载 Platform Tools 包：<https://developer.android.com/tools/releases/platform-tools>

**环境变量配置**
将 platform-tools 目录加入系统 PATH 后，执行 `adb version` 验证安装。

**WiFi 调试连接（Android 11+）**
```bash
# 步骤1：确保设备与 PC 在同一 WiFi
# 步骤2：开启无线调试，获取配对码和端口
adb pair <设备IP>:<配对端口>
# 输入配对码

# 步骤3：连接
adb connect <设备IP>:<调试端口>

# 步骤4：验证
adb devices -l
```

**启用开发者选项与 USB 调试**
1. 设置 → 关于手机 → 连续点击"版本号"7 次
2. 返回 → 开发者选项 → 开启 USB 调试
3. 首次连接时授权 RSA 密钥

### 31.3 15 个高频 ADB 命令

| 序号 | 命令 | 用途 |
|-----|------|------|
| 1 | `adb devices -l` | 列出已连接设备与序列号 |
| 2 | `adb install [-r] <apk>` | 安装 APK（-r 覆盖安装） |
| 3 | `adb uninstall <package>` | 卸载应用 |
| 4 | `adb shell pm clear <package>` | 清除应用数据与缓存 |
| 5 | `adb shell am start -n <package>/<activity>` | 启动指定 Activity |
| 6 | `adb shell am force-stop <package>` | 强制停止应用 |
| 7 | `adb push <local> <remote>` | PC → 设备传输文件 |
| 8 | `adb pull <remote> <local>` | 设备 → PC 传输文件 |
| 9 | `adb shell screencap -p /sdcard/screen.png` | 截屏 |
| 10 | `adb shell screenrecord /sdcard/record.mp4` | 录屏（Ctrl+C 停止） |
| 11 | `adb shell input tap <x> <y>` | 模拟触摸点击 |
| 12 | `adb shell input swipe <x1> <y1> <x2> <y2> <duration>` | 模拟滑动 |
| 13 | `adb logcat` | 实时查看系统日志 |
| 14 | `adb shell dumpsys activity activities` | 查询当前 Activity 栈 |
| 15 | `adb shell dumpsys meminfo <package>` | 查看应用内存占用 |

### 31.4 设备管理

```bash
# 列出连接设备
adb devices -l

# 查看设备状态（bootloader/recovery/device/offline）
adb get-state

# 进入设备 Shell（连续执行多条设备命令）
adb shell

# 重启设备
adb reboot

# 重启至 Bootloader/Fastboot 模式
adb reboot bootloader

# 退出 Shell
exit
```

多设备场景下，通过 `-s` 指定序列号操作目标设备：
```bash
adb -s emulator-5554 install app.apk
```

### 31.5 App 管理

```bash
# 安装 APK
adb install app.apk

# 覆盖安装（保留数据）
adb install -r app.apk

# 卸载应用
adb uninstall com.example.app

# 清除应用数据（等价于设置中"清除数据"）
adb shell pm clear com.example.app

# 启动应用（通过包名+主 Activity）
adb shell am start -n com.example.app/.MainActivity

# 通过 Action 启动（隐式 Intent）
adb shell am start -a android.intent.action.VIEW -d "https://example.com"

# 发送广播
adb shell am broadcast -a android.intent.action.PACKAGE_REPLACED

# 强制停止
adb shell am force-stop com.example.app

# 查看已安装应用包列表
adb shell pm list packages

# 查看第三方应用
adb shell pm list packages -3

# 查看 APK 安装路径
adb shell pm path com.example.app
```

### 31.6 文件操作

```bash
# PC → 设备
adb push ./local_file.txt /sdcard/

# 设备 → PC
adb pull /sdcard/screen.png ./

# 设备上执行文件操作
adb shell ls /sdcard/

# 复制（设备内部）
adb shell cp /sdcard/file1.txt /sdcard/backup/

# 删除
adb shell rm /sdcard/temp.txt
```

### 31.7 屏幕操作与模拟输入

```bash
# 截屏并拉取到本地
adb shell screencap -p /sdcard/screen.png
adb pull /sdcard/screen.png ./

# 录屏（默认 3 分钟，120秒即停止）
adb shell screenrecord --time-limit 30 /sdcard/record.mp4
adb pull /sdcard/record.mp4 ./

# 模拟点击坐标 (500, 800)
adb shell input tap 500 800

# 模拟滑动（从 500,1500 到 500,500，持续 300ms）
adb shell input swipe 500 1500 500 500 300

# 模拟输入文本
adb shell input text "HelloWorld"

# 模拟按键（HOME=3, BACK=4, POWER=26, VOLUME_UP=24, VOLUME_DOWN=25）
adb shell input keyevent 3   # Home 键
adb shell input keyevent 4   # 返回键
adb shell input keyevent 26  # 电源键（锁屏/解锁）

# 模拟输入中文（需先安装 ADBKeyBoard 输入法）
adb shell ime set io.appium.settings/.UnicodeIME
adb shell input text "测试文本"
```

### 31.8 系统信息采集

```bash
# 查看电池信息
adb shell dumpsys battery

# 查看 CPU 信息
adb shell cat /proc/cpuinfo

# 查看内存总量
adb shell cat /proc/meminfo

# 查看指定应用内存占用
adb shell dumpsys meminfo com.example.app

# 查看当前 Activity 栈（定位当前页面）
adb shell dumpsys activity activities | grep mResumedActivity

# 查看屏幕分辨率
adb shell wm size

# 查看屏幕密度
adb shell wm density

# 查看系统版本
adb shell getprop ro.build.version.release

# 查看设备型号
adb shell getprop ro.product.model

# 查看应用启动时间（冷启动）
adb shell am start -W -n com.example.app/.MainActivity
# 输出中 TotalTime 字段即为启动耗时（ms）
```

### 31.9 Logcat 日志分析

```bash
# 实时输出所有日志
adb logcat

# 清空日志缓冲区
adb logcat -c

# 按级别过滤（V/D/I/W/E/F）
adb logcat *:E          # 仅 Error 级别
adb logcat *:W          # Warning 及以上

# 按 TAG 过滤
adb logcat -s TAG_NAME

# 按 TAG + 级别组合过滤
adb logcat ActivityManager:I *:S

# 输出到文件
adb logcat > logcat.txt

# 带时间戳格式
adb logcat -v time

# 使用 grep 过滤关键字（Windows 用 findstr）
adb logcat | grep "com.example.app"
adb logcat | findstr "com.example.app"

# 过滤崩溃日志
adb logcat *:E | grep -A 50 "FATAL EXCEPTION"

# 按进程 PID 过滤
adb logcat --pid=$(adb shell pidof com.example.app)
```

**日志级别说明**

| 级别 | 含义 | 使用场景 |
|-----|------|---------|
| V | Verbose | 最详细，开发调试 |
| D | Debug | 调试信息 |
| I | Info | 关键流程节点 |
| W | Warning | 潜在问题 |
| E | Error | 错误事件 |
| F | Fatal | 致命错误 |

## 动手实操

### 31.10 完整测试场景：安装 → 启动 → 操作 → 日志采集

```bash
# 1. 确认设备连接
adb devices -l

# 2. 安装测试包
adb install -r app-debug.apk

# 3. 清除旧数据
adb shell pm clear com.example.app

# 4. 启动应用并记录启动时间
adb shell am start -W -n com.example.app/.MainActivity

# 5. 清空日志缓冲区
adb logcat -c

# 6. 执行操作（模拟点击登录按钮，坐标需根据实际 UI 调整）
adb shell input tap 540 1200

# 7. 输入账号密码
adb shell input text "testuser"
adb shell input keyevent 61  # TAB 键切换焦点
adb shell input text "password123"

# 8. 点击登录
adb shell input tap 540 1400

# 9. 等待 3 秒后采集日志
timeout /t 3 /nobreak >nul
adb logcat -d -v time > test_log.txt

# 10. 截屏保存当前状态
adb shell screencap -p /sdcard/result.png
adb pull /sdcard/result.png ./screenshots/

# 11. 查看内存占用
adb shell dumpsys meminfo com.example.app > meminfo.txt
```

### 31.11 批量操作脚本

```bash
@echo off
REM 批量安装 APK 到所有连接设备
for /f "tokens=1" %%i in ('adb devices ^| findstr /r /c:"device$"') do (
    echo Installing to %%i...
    adb -s %%i install -r app-debug.apk
)
```

## 常见坑

- 未开启开发者选项或 USB 调试，导致 `adb devices` 显示 `unauthorized`
- 多设备连接时未指定 `-s` 序列号，命令执行到错误设备
- `adb shell input text` 不支持中文输入，需切换 ADBKeyBoard 输入法
- logcat 缓冲区溢出导致早期日志丢失，测试前执行 `adb logcat -c` 清空
- 录屏文件过大未及时 pull，导致设备存储空间不足
- 坐标点击依赖固定分辨率，不同设备需动态计算坐标比例
- Windows 环境下 `grep` 不可用，需使用 `findstr` 或安装 Git Bash

## 自测清单

- [ ] 能够独立完成 ADB 安装与 WiFi 调试配置
- [ ] 熟练使用 15 个高频命令完成日常测试操作
- [ ] 能够通过 `dumpsys` 获取电池、内存、Activity 栈信息
- [ ] 能够使用 logcat 按级别、TAG、PID 过滤日志
- [ ] 能够通过 `input` 命令模拟 tap、swipe、keyevent 操作
- [ ] 能够编写批处理脚本实现多设备批量操作

## 延伸阅读

- ADB 官方文档：<https://developer.android.com/tools/adb>
- Android Logcat 指南：<https://developer.android.com/tools/logcat>
- dumpsys 可用服务列表：<https://developer.android.com/tools/dumpsys>
- ADB 命令速查表：<https://www.xda-developers.com/install-adb-windows-macos-linux/>
