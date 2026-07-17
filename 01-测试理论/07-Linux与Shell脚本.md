# 07-Linux与Shell脚本

> 课时：60 min | 难度：★★★☆☆

## 学习目标

1. 掌握10个测试高频Linux命令的使用方法与参数
2. 能够使用命令组合进行日志查看与分析
3. 掌握Shell脚本基础语法（变量/条件/循环/函数）
4. 能够编写实用的测试辅助脚本

## 核心概念

### 10个高频命令

| 命令 | 核心功能 | 关键参数 | 典型用法 |
|------|---------|---------|---------|
| `ls` | 列出目录内容 | `-l` 长格式 / `-a` 隐藏文件 / `-h` 可读大小 | `ls -lah /var/log/` |
| `cd` | 切换目录 | `..` 上级 / `~` 家目录 / `-` 上一目录 | `cd /var/log/` |
| `mkdir` | 创建目录 | `-p` 递归创建 / `-m` 权限模式 | `mkdir -p /data/logs/app` |
| `grep` | 文本搜索 | `-i` 忽略大小写 / `-n` 显示行号 / `-r` 递归 / `-v` 反向 | `grep -n "ERROR" app.log` |
| `chown` | 修改所有者 | `-R` 递归 / `user:group` 格式 | `chown -R www:www /var/www` |
| `ps` | 查看进程状态 | `-ef` 全格式 / `-aux` BSD格式 | `ps -ef \| grep java` |
| `top` | 实时系统监控 | `M` 按内存排序 / `P` 按CPU排序 | `top -p 1234` |
| `df` | 磁盘空间查看 | `-h` 可读格式 / `-T` 文件系统类型 | `df -h /` |
| `tail` | 查看文件末尾 | `-n` 行数 / `-f` 实时追踪 | `tail -f app.log` |
| `awk` | 文本处理语言 | `-F` 字段分隔符 | `awk -F',' '{print $1}' data.csv` |

### 命令组合技巧

```bash
# 管道组合：过滤Java进程并统计数量
ps -ef | grep java | grep -v grep | wc -l

# 日志分析：统计各错误类型出现次数
grep "ERROR" app.log | awk '{print $4}' | sort | uniq -c | sort -rn

# 查找大文件（大于100MB）
find /var/log -type f -size +100M -exec ls -lh {} \;

# 实时监控多个日志文件
tail -f /var/log/app/*.log

# 按时间范围过滤日志日志
awk '/2024-01-15 10:00/,/2024-01-15 11:00/' app.log
```

### 日志查看方法

```bash
# 实时追踪日志
tail -f /var/log/app.log

# 实时追踪并过滤关键字
tail -f /var/log/app.log | grep "ERROR"

# 最新100行grep
tail -n 100 /var/log/app.log | grep "Exception"

# vi 中搜索
vim /var/log/app.log
/关键字          ← 向下搜索
?关键字          ← 向上搜索
n                ← 下一个匹配
N                ← 上一个匹配
:%s/old/new/g    ← 全文替换
```

### Shell 脚本基础语法

**变量定义：**

```bash
#!/bin/bash
echo "Shell Scripting Basics"

# 变量定义（等号两侧不能有空格）
name="tester"
age=30

# 使用变量（$变量名 或 ${变量名}）
echo "Name: $name"
echo "Age: ${age}"

# 命令赋值
current_date=$(date +%Y-%m-%d)
echo "Today: $current_date"

# 只读变量
readonly APP_NAME="ShopTest"
```

**条件判断：**

```bash
#!/bin/bash

# if-elif-else 结构
if [ "$1" == "prod" ]; then
    echo "生产环境"
elif [ "$1" == "test" ]; then
    echo "测试环境"
else
    echo "未知环境: $1"
fi

# 文件测试
if [ -f "/var/log/app.log" ]; then
    echo "文件存在"
fi

if [ -d "/app/data" ]; then
    echo "目录存在"
fi

# 数值比较：-eq -ne -gt -lt -ge -le
count=10
if [ $count -gt 5 ]; then
    echo "数量大于5"
fi
```

**循环：**

```bash
#!/bin/bash

# for 循环
for i in 1 2 3 4 5; do
    echo "迭代 $i"
done

# 范围循环
for i in {1..10}; do
    echo "Number: $i"
done

# 数组遍历
servers=("db01" "db02" "app01" "app02")
for server in "${servers[@]}"; do
    echo "检查 $server"
done

# while 循环
counter=0
while [ $counter -lt 5 ]; do
    echo "计数: $counter"
    counter=$((counter + 1))
done

# 逐行读取文件
while IFS= read -r line; do
    echo "行内容: $line"
done < /var/log/app.log
```

**函数：**

```bash
#!/bin/bash

# 函数定义
check_service() {
    local host=$1
    local port=$2
    
    if nc -z "$host" "$port" 2>/dev/null; then
        echo "[$host:$port] 服务正常"
        return 0
    else
        echo "[$host:$port] 服务异常"
        return 1
    fi
}

# 函数调用
check_service "localhost" 3306 && echo "MySQL OK" || echo "MySQL FAIL"
check_service "localhost" 6379 && echo "Redis OK" || echo "Redis FAIL"

# 带返回值的函数
get_disk_usage() {
    df -h / | awk 'NR==2{print $5}' | tr -d '%'
}

usage=$(get_disk_usage)
if [ "$usage" -gt 80 ]; then
    echo "磁盘使用率超过80%: ${usage}%"
fi
```

---

## 动手实操

### 实操1：日志分析脚本

```bash
#!/bin/bash
# 脚本名称: analyze_log.sh
# 功能: 统计日志中各级别错误数量及TOP错误
# 用法: ./analyze_log.sh /path/to/app.log

LOG_FILE="${1:-/var/log/app.log}"

if [ ! -f "$LOG_FILE" ]; then
    echo "错误: 文件不存在 $LOG_FILE"
    exit 1
fi

echo "=== 日志分析报告 ==="
echo "文件: $LOG_FILE"
echo "分析时间: $(date '+%Y-%m-%d %H:%M:%S')"
echo ""

# 各级别统计
echo "--- 日志级别分布 ---"
for level in INFO WARN ERROR FATAL; do
    count=$(grep -c "$level" "$LOG_FILE" 2>/dev/null || echo 0)
    printf "%-8s: %d 条\n" "$level" "$count"
done

echo ""
echo "--- TOP 10 错误消息 ---"
grep "ERROR" "$LOG_FILE" | awk '{$1=$2=$3=""; print $0}' | \
    sed 's/^ *//' | sort | uniq -c | sort -rn | head -n 10

echo ""
echo "--- 每小时错误分布 ---"
grep "ERROR" "$LOG_FILE" | awk '{print $2}' | cut -d: -f1 | \
    sort | uniq -c | sort -n
```

### 实操2：批量文件处理脚本

```bash
#!/bin/bash
# 脚本名称: batch_rename.sh
# 功能: 批量添加前缀并修改扩展名
# 用法: ./batch_rename.sh /path/to/dir prefix new_ext

DIR="${1:-.}"
PREFIX="${2:-backup_}"
NEW_EXT="${3:-.txt}"

if [ ! -d "$DIR" ]; then
    echo "错误: 目录不存在 $DIR"
    exit 1
fi

count=0
for file in "$DIR"/*; do
    if [ -f "$file" ]; then
        filename=$(basename "$file")
        name="${filename%.*}"
        new_name="${PREFIX}${name}.${NEW_EXT}"
        mv "$file" "$DIR/$new_name"
        echo "重命名: $filename -> $new_name"
        count=$((count + 1))
    fi
done

echo "共处理 $count 个文件"
```

### 实操3：服务健康检查脚本

```bash
#!/bin/bash
# 脚本名称: health_check.sh
# 功能: 多服务健康检查并输出报告
# 用法: ./health_check.sh

LOG_FILE="/var/log/health_check.log"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

# 服务配置列表
declare -A SERVICES=(
    ["MySQL"]="localhost:3306"
    ["Redis"]="localhost:6379"
    ["Nginx"]="localhost:80"
    ["App_Server"]="localhost:8080"
    ["RabbitMQ"]="localhost:5672"
)

check_tcp() {
    local host=$1
    local port=$2
    timeout 3 bash -c "echo >/dev/tcp/$host/$port" 2>/dev/null
}

echo "[$TIMESTAMP] ===== 服务健康检查开始 =====" | tee -a "$LOG_FILE"

for name in "${!SERVICES[@]}"; do
    IFS=':' read -r host port <<< "${SERVICES[$name]}"
    
    if check_tcp "$host" "$port"; then
        result="[✓] $name ($host:$port) - 正常"
    else
        result="[✗] $name ($host:$port) - 异常"
    fi
    
    echo "$result" | tee -a "$LOG_FILE"
done

echo "[$TIMESTAMP] ===== 服务健康检查结束 =====" | tee -a "$LOG_FILE"

# 可选：异常告警（配合 cron 定时执行）
error_count=$(grep -c "异常" "$LOG_FILE" | tail -1)
if [ "$error_count" -gt 0 ]; then
    echo "检测到 $error_count 个服务异常，请检查！"
    # 发送邮件告警（需配置 mailutils）
    # echo "服务异常详情：\n$(grep '异常' $LOG_FILE)" | mail -s "服务告警" admin@example.com
fi
```

### 实操4：配合 crontab 定时任务

```bash
# 编辑定时任务
crontab -e

# 每5分钟执行一次健康检查
*/5 * * * * /opt/scripts/health_check.sh

# 每天凌晨3点清理7天前的日志
0 3 * * * find /var/log/app -name "*.log" -mtime +7 -delete

# 每周一早上9点发送周报
0 9 * * 1 /opt/scripts/weekly_report.sh
```

---

## 常见坑

| 错误 | 说明 |
|------|------|
| 变量赋值带空格 | `name = "test"` 报错，应为 `name="test"` |
| 字符串比较用 `=` | `[ "$a" == "$b" ]` 而非 `[ "$a" = "$b" ]`（双等号更规范） |
| 忘记加 shebang | 脚本首行缺少 `#!/bin/bash`，执行时报错 |
| 文件无执行权限 | 需用 `chmod +x script.sh` 赋予执行权限 |
| tail -f 不退出 | 脚本中使用 `tail -f` 会导致进程挂起，应添加超时逻辑 |
| cron 环境变量缺失 | cron 执行时 PATH 不完整，需在脚本内 `source /etc/profile` |
| Windows 换行符问题 | Windows 编辑的脚本含 `\r`，需 `sed -i 's/\r$//' script.sh` 转换 |

---

## 自测清单

- [ ] 能否默写10个高频命令及其关键参数
- [ ] 能否使用 `tail -f + grep` 实时过滤错误日志
- [ ] 能否编写含 if/for/while 的Shell脚本
- [ ] 能否使用 awk 提取CSV指定列并统计
- [ ] 能否编写服务健康检查脚本
- [ ] 能否使用 crontab 配置定时任务
- [ ] 能否将管道符 `|` 用于命令组合实现复杂过滤

---

## 延伸阅读

- [菜鸟教程 - Shell 教程](https://www.runoob.com/linux/linux-shell.html)
- [Linux man pages - 在线文档](https://man7.org/linux/man-pages/)
- [Advanced Bash-Scripting Guide](https://tldp.org/LDP/abs/html/)
- [Linux 探索入门](https://linux.cn/article-11789-1.html)
- [crontab.guru - Cron 表达式工具](https://crontab.guru/)
