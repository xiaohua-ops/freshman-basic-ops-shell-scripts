*****

# CPU 使用率邮件告警脚本

# 1.CPU使用率监控与告警

介绍：这是一个监控脚本，定时检查系统的 CPU 使用率，如果 CPU 使用率超过设定阈值（临界值=>90%），向指定邮箱发送警报邮件。如果 CPU 使用率恢复到正常范围内，则不发送邮件。

### 1.1提前安装小工具

```Shell
# bc 用于计算小数类型数值 例：100 - 88.8
yum install -y bc sendmail sysstat
# sudo yum install -y bc sendmail sysstat

# sendmail 邮件服务，可以向指定脚本发送邮件
yum install sendmail -y
# sudo yum install sendmail -y
```

### 1.2修改主机名规范

在启动sendmail服务之前，主机名必须修改规范，主机名称必须满足FQDN主机格式 ： 主机名称/功能 + 公司域名

```Shell
hostnamectl set-hostname node1.itops.cn && bash

hostname
```

例：node1.itops.cn

例：mysql.google.com

```Shell
# 启动邮件服务
systemctl start sendmail

# 设置开机自启
systemctl enable sendmail

# 查看状态 active（runing）
systemctl status sendmail
```

邮件编码格式 避免乱码

```Shell
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
```

### 1.3主脚本代码

```Shell
vim cpu_alert.sh
#!/bin/bash

# 配置文件路径（默认和脚本同目录）
CONFIG_FILE="./cpu_alert.conf.sh"

# 检查前置依赖是否安装
check_dependencies() {
    local dependencies=("bc" "sendmail" "mpstat")
    for dep in "${dependencies[@]}"; do
        if ! command -v "$dep" &> /dev/null; then
            echo "错误：依赖工具 $dep 未安装，请先安装后再运行脚本"
            echo "安装命令（CentOS/RHEL）：sudo dnf install -y bc sendmail sysstat"
            exit 1
        fi
    done
}

# 加载配置文件
load_config() {
    if [ ! -f "$CONFIG_FILE" ]; then
        echo "错误：配置文件 $CONFIG_FILE 不存在，请先创建配置文件"
        exit 1
    fi
    # shellcheck source=cpu_alert.conf
    source "$CONFIG_FILE"
}

# 初始化日志文件
init_log() {
    if [ ! -d "$(dirname "$LOG_FILE")" ]; then
        mkdir -p "$(dirname "$LOG_FILE")"
    fi
    touch "$LOG_FILE"
}

# 记录日志
log() {
    local level="$1"
    local message="$2"
    local timestamp
    timestamp=$(date "+%Y-%m-%d %H:%M:%S")
    echo "[$timestamp] [$level] $message" >> "$LOG_FILE"
    echo "[$timestamp] [$level] $message"
}

# 获取 CPU 空闲率
get_cpu_idle() {
# mpstat 1 2：采样2次，取第2次的结果（避免首次采样偏差）
    cpu_idle=$(mpstat 1 2 | awk '/Average:/ {print $NF}')
    echo "$cpu_idle"
}

# 发送邮件
send_email() {
    local subject="$1"
    local content="$2"
    {
        echo "From: $FROM_EMAIL"
        echo "To: $TO_EMAIL"
        echo "Subject: $subject"
        echo "Content-Type: text/plain; charset=UTF-8"
        echo "Content-Transfer-Encoding: 8bit"
        echo ""
        echo -e "$content"
    } | sendmail -t
}

# 1. 检查依赖
check_dependencies

# 2. 加载配置
load_config

# 3. 初始化日志
init_log

# 4. 初始化状态文件（记录上次是否发送过告警）
STATE_FILE="/tmp/cpu_alert_state"
touch "$STATE_FILE"
LAST_ALERT_STATE=$(cat "$STATE_FILE")

log "INFO" "CPU 使用率告警脚本启动成功"
log "INFO" "配置信息：阈值=$THRESHOLD%，检查间隔=$CHECK_INTERVAL秒，告警邮箱=$TO_EMAIL"

# 5. 循环检查
while true; do
    # 获取 CPU 空闲率
    cpu_idle=$(get_cpu_idle)

    # 验证 CPU 空闲率是否为有效数字
    if [[ -z "$cpu_idle" || ! "$cpu_idle" =~ ^[0-9]+(\.[0-9]+)?$ ]]; then
        log "ERROR" "无法提取有效的 CPU 空闲率，当前值：$cpu_idle，跳过本次检查"
        sleep "$CHECK_INTERVAL"
        continue
    fi

    # 计算 CPU 使用率
    cpu_usage=$(echo "100 - $cpu_idle" | bc -l)
    cpu_usage=$(printf "%.2f" "$cpu_usage")

    log "INFO" "当前 CPU 空闲率：${cpu_idle}%，使用率：${cpu_usage}%"

    # 判断是否超过阈值
    if [ "$(echo "$cpu_usage >= $THRESHOLD" | bc -l)" -eq 1 ]; then
        # 超过阈值，且上次未发送告警，则发送
        if [ "$LAST_ALERT_STATE" != "alert" ]; then
            log "WARN" "CPU 使用率超过阈值！当前：${cpu_usage}%，阈值：${THRESHOLD}%"
            subject="【告警】服务器 CPU 使用率超过阈值"
            content="警告：\n服务器 $(hostname) CPU 使用率已超过阈值！\n当前使用率：${cpu_usage}%\n阈值：${THRESHOLD}%\n检查时间：$(date "+%Y-%m-%d %H:%M:%S")"
            send_email "$subject" "$content"
            log "INFO" "告警邮件已发送至 $TO_EMAIL"
            # 更新状态为已告警
            echo "alert" > "$STATE_FILE"
            LAST_ALERT_STATE="alert"
        fi
    else
        # 恢复正常，且上次发送过告警，则发送恢复邮件
        if [ "$LAST_ALERT_STATE" == "alert" ]; then
            log "INFO" "CPU 使用率已恢复正常！当前：${cpu_usage}%"
            subject="【恢复】服务器 CPU 使用率已恢复正常"
            content="通知：\n服务器 $(hostname) CPU 使用率已恢复正常！\n当前使用率：${cpu_usage}%\n恢复时间：$(date "+%Y-%m-%d %H:%M:%S")"
            send_email "$subject" "$content"
            log "INFO" "恢复邮件已发送至 $TO_EMAIL"
            # 更新状态为正常
            echo "normal" > "$STATE_FILE"
            LAST_ALERT_STATE="normal"
        fi
    fi

    # 等待检查间隔
    sleep "$CHECK_INTERVAL"
done
```

### 1.4配套配置文件

```Shell
vim cpu_alert.conf.sh
# 告警阈值：CPU使用率超过这个百分比，就触发告警
THRESHOLD=90

# 邮件配置
FROM_MAIL="cpu-alert@node1.itops.cn"  # 发件人，和服务器主机名匹配
TO_MAIL="your-email@xxx.com"        # 收件人，改成你自己的真实邮箱

# 检查间隔：多久检查一次CPU，单位秒
CHECK_INTERVAL=30

# 日志文件存放路径
LOG_FILE="./logs/cpu_alert.log"
```

完成后给脚本加执行权限

```Shell
chmod +x cpu_alert.sh
chmod +x cpu_alert.conf.sh
# 脚本调试
bash -x cpu_alert.sh
bash -x cpu_alert.conf.sh

# 无问题执行
# 两个脚本需放置在同一目录下
./cpu_alert.sh
```

### 1.5压力测试

主动给CPU增加压力来模拟CPU高负载 测试脚本是否可行

```Shell
dnf install stress-ng
stress-ng --cpu 4 --timeout 60s
# cpu 4 表示启动 4 个进程来占用 CPU，模拟 4 核的压测。
# timeout 60 表示压测持续 60 秒。
# 查看cpu核数 ： lscpu
```