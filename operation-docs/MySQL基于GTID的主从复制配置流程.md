## 一、GTID 概述

GTID（Global Transaction Identifier，全局事务标识符）是 MySQL 5.6 及以后版本引入的主从复制方式，相比传统基于 binlog 位置的复制，具有以下优势：

**配置更简单**：无需手动指定 binlog 文件名和位置，通过 `SOURCE_AUTO_POSITION=1` 自动追踪。

**故障切换更便捷**：主从切换时无需找位点，直接基于 GTID 同步。

**数据一致性更可靠**：每个 GTID 在一个服务器上只执行一次，避免重复执行导致数据混乱。

## 二、环境准备

### 2.1 环境说明

192.168.88.101  mysql-node1  master

192.168.88.102  mysql-node2  slave

### 2.2 基础配置（两台服务器都执行）

#### （1）安装必备软件

```Plain
dnf install vim wget rsync telnet net-tools -y
```

#### （2）设置主机名

```Plain
# master 执行
hostnamectl set-hostname mysql-node1

# slave 执行
hostnamectl set-hostname mysql-node2
```

#### （3）配置 IP 与主机映射

```Plain
vim /etc/hosts
192.168.88.101   mysql-node1   node1
192.168.88.102   mysql-node2   node2
```

#### （4）优化 Linux 系统

```Plain
# 关闭 SELinux
sed -i -r 's/SELINUX=[ep].*/SELINUX=disabled/g' /etc/selinux/config
setenforce 0# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld
iptables -F
iptables -t nat -F
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
```

#### （5）时间同步（可选）

```Plain
dnf -y install chrony
systemctl enable chronyd --now
```

## 三、安装 MySQL 8（两台服务器都执行）

### 3.1 准备安装脚本

```Plain
vim install-mysql8.sh
```

脚本内容：

```Bash
#!/bin/bash

# 格式：mysql-版本号-系统依赖-架构，无需加.tar.xz后缀
MYSQL_PKG="mysql-8.0.43-linux-glibc2.28-x86_64"

# 固定配置（无需修改）
MYSQL_INSTALL_PATH="/export/server/mysql"
MYSQL_INIT_PASSWORD="MySQL@666"

# 环境与前置检查
# 自动识别包管理器（兼容CentOS7/8/9）
if command -v dnf &> /dev/null; then
    PKG_MANAGER="dnf"
else
    PKG_MANAGER="yum"
fi

# 检查是否已安装MySQL，避免重复执行
if [ -d "${MYSQL_INSTALL_PATH}" ] && [ -f "${MYSQL_INSTALL_PATH}/bin/mysqld" ]; then
    echo "检测到MySQL已安装在 ${MYSQL_INSTALL_PATH}，跳过安装流程"
    echo "如需重新安装，请先删除 ${MYSQL_INSTALL_PATH} 目录后再执行脚本"
    exit 0
fi

# 关闭SELinux与防火墙
sed -i -r 's/SELINUX=[ep].*/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
systemctl stop firewalld &> /dev/null
systemctl disable firewalld &> /dev/null
iptables -F
iptables -t nat -F
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT

# 安装依赖
if rpm -q libaio &> /dev/null; then
    echo "libaio已安装，跳过安装"
else
    echo "开始安装依赖"
    $PKG_MANAGER -y install libaio &> /dev/null
    if [ $? -ne 0 ];then
        echo "libaio安装失败，请检查网络或yum/dnf源"
        exit 1
    fi
fi

# 解压MySQL二进制包
echo "进行解压操作，安装包：${MYSQL_PKG}.tar.xz"
if ls -l "${MYSQL_PKG}" &> /dev/null; then
    echo "已解压，跳过"
else
    if [ -f "${MYSQL_PKG}.tar.xz" ]; then
        tar -xf "${MYSQL_PKG}.tar.xz"
        ls -l "${MYSQL_PKG}"
    else
        echo "错误：未找到MySQL安装包 ${MYSQL_PKG}.tar.xz"
        echo "请将安装包上传至当前目录后再执行脚本"
        exit 1
    fi
fi

# 清理系统自带MariaDB
echo "判断是否安装过MariaDB，进行清理"
rpm -qa | grep mariadb | xargs -r $PKG_MANAGER remove -y
if [ -f /etc/my.cnf ]; then
        rm -rf /etc/my.cnf
fi

# 创建mysql运行用户
id mysql &> /dev/null
[ $? -ne 0 ] && useradd -r -s /sbin/nologin mysql

# 部署MySQL到指定目录
rm -rf "$(dirname "${MYSQL_INSTALL_PATH}")"
mkdir -p "$(dirname "${MYSQL_INSTALL_PATH}")"
cp -r "${MYSQL_PKG}" "${MYSQL_INSTALL_PATH}"
chown -R mysql:mysql "${MYSQL_INSTALL_PATH}"

# MySQL初始化
echo "正在进入mysql目录，对其进行初始化操作..."
cd "${MYSQL_INSTALL_PATH}"
# 检查data目录是否已存在，避免重复初始化
if [ -d "${MYSQL_INSTALL_PATH}/data" ] && [ "$(ls -A "${MYSQL_INSTALL_PATH}/data")" ]; then
    echo "检测到MySQL data目录已存在，跳过初始化步骤"
else
    bin/mysqld --initialize --user=mysql --basedir="${MYSQL_INSTALL_PATH}" --datadir="${MYSQL_INSTALL_PATH}/data" 2>&1 | tee /tmp/mysqld.log
    # 检查初始化是否成功
    if [ $? -ne 0 ]; then
        echo "MySQL初始化失败，请查看日志 /tmp/mysqld.log 排查问题"
        exit 1
    fi
    # 提取临时密码
    grep password /tmp/mysqld.log | awk '{print $NF}' > /tmp/mysql_temp_password.txt
fi

bin/mysql_ssl_rsa_setup --datadir="${MYSQL_INSTALL_PATH}/data" &> /dev/null

# 生成my.cnf配置文件
cat >/etc/my.cnf<<EOF
[mysqld]
port=3306
basedir=${MYSQL_INSTALL_PATH}
datadir=${MYSQL_INSTALL_PATH}/data
socket=/tmp/mysql.sock
character_set_server=utf8mb4
collation-server=utf8mb4_unicode_ci
skip-symbolic-links
explicit_defaults_for_timestamp=1
EOF
chmod 644 /etc/my.cnf
chown root:root /etc/my.cnf

# 生成systemd服务文件
cat >/etc/systemd/system/mysqld.service<<EOF
[Unit]
Description=MySQL Server
After=network.target

[Service]
User=mysql
Group=mysql
Type=forking
ExecStart=${MYSQL_INSTALL_PATH}/bin/mysqld --daemonize --pid-file=${MYSQL_INSTALL_PATH}/data/mysqld.pid
ExecStop=${MYSQL_INSTALL_PATH}/bin/mysqladmin --defaults-file=/etc/my.cnf shutdown
TimeoutSec=600
PIDFile=${MYSQL_INSTALL_PATH}/data/mysqld.pid
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
chmod 644 /etc/systemd/system/mysqld.service

# 启动MySQL服务 
echo "正在刷新后台服务，然后启动mysqld..."
systemctl daemon-reload
systemctl start mysqld
# 检查服务是否启动成功
if [ $? -ne 0 ]; then
    echo "MySQL服务启动失败，请查看日志 /tmp/mysqld.log 或执行 journalctl -u mysqld 排查问题"
    exit 1
fi
systemctl enable mysqld

# 重置MySQL管理员密码 
echo "正在重置mysql管理员密码..."
cd "${MYSQL_INSTALL_PATH}"
temp_password=`cat /tmp/mysql_temp_password.txt`
bin/mysqladmin -uroot password "${MYSQL_INIT_PASSWORD}" -p"$temp_password" &> /dev/null
if [ $? -ne 0 ]; then
    echo "密码重置可能失败，请使用临时密码手动登录重置"
    echo "临时密码已保存至：/tmp/mysql_temp_password.txt"
else
    echo "密码重置成功"
fi

# 配置环境变量 
if ! grep -q "${MYSQL_INSTALL_PATH}/bin" /etc/profile; then
    echo "export PATH=\$PATH:${MYSQL_INSTALL_PATH}/bin" >> /etc/profile
fi
source /etc/profile

echo "MySQL8安装成功！"
echo "安装包版本：${MYSQL_PKG}"
echo "安装路径：${MYSQL_INSTALL_PATH}"
echo "数据库密码：${MYSQL_INIT_PASSWORD}"

echo "1. 执行 source /etc/profile 加载最新环境变量"
echo "2. 执行 mysql -uroot -p${MYSQL_INIT_PASSWORD} 登录MySQL"
echo "3. 初始化日志已保存至：/tmp/mysqld.log"
```

### 3.2 执行安装

```Plain
# 上传 mysql-8.0.43-linux-glibc2.28-x86_64.tar.xz 到当前目录
chmod +x install-mysql8.sh
./install-mysql8.sh
source /etc/profile
```

## 四、初始化数据库（可选，建议执行）

如果之前有传统主从配置，先清理并重新初始化：

```Plain
systemctl stop mysqld
pkill mysqld
rm -rf /export/server/mysql/data/*
rm -rf /tmp/mysqld.log

# 重新初始化
/export/server/mysql/bin/mysqld --initialize --user=mysql --basedir=/export/server/mysql &>/tmp/mysqld.log
grep password /tmp/mysqld.log | awk '{print $NF}'
# 跳过权限表重置密码
mysqld --user=mysql --skip-grant-tables --skip-networking &
mysql
```

在 MySQL 命令行执行：

```Plain
FLUSH PRIVILEGES;ALTER USER 'root'@'localhost' IDENTIFIED BY 'MySQL@666';
FLUSH PRIVILEGES;EXIT;
```

重启 MySQL：

```Plain
pkill mysqld
systemctl start mysqld
```

## 五、配置主从数据库

### 5.1 同步数据（可选）

如果需要保证初始数据一致，在 master 执行：

```Plain
# 停止 master MySQL
systemctl stop mysqld

# 同步 data 目录到 slave
rsync -av --delete /export/server/mysql/data/ mysql-node2:/export/server/mysql/data/

# 分别在 master 和 slave 删除 auto.cnf（避免 UUID 冲突）
rm -f /export/server/mysql/data/auto.cnf
```

### 5.2 配置 master（mysql-node1）

#### （1）修改配置文件

```Plain
cat >/etc/my.cnf<<EOF
[mysqld]
# 基础路径
basedir=/export/server/mysql
datadir=/export/server/mysql/data
socket=/tmp/mysql.sock
port=3306

# 日志
log-error=/export/server/mysql/master.err
log-bin=/export/server/mysql/data/binlog
server-id=101

# 字符集
character_set_server=utf8mb4
collation-server=utf8mb4_unicode_ci

# GTID 核心配置
gtid_mode=ON
enforce_gtid_consistency=ON
log_slave_updates=ON
binlog_format=ROW
binlog_row_image=FULL
sync_binlog=1
expire_logs_days=7

# 复制过滤：不同步系统库
binlog-ignore-db=information_schema
binlog-ignore-db=mysql
binlog-ignore-db=performance_schema
binlog-ignore-db=sys

# InnoDB 优化
default_storage_engine=InnoDB
innodb_buffer_pool_size=1G
innodb_log_file_size=256M
innodb_log_buffer_size=64M
innodb_flush_log_at_trx_commit=1
innodb_file_per_table=1

# 连接配置
max_connections=500
max_connect_errors=1000000
table_open_cache=2000

# 慢查询日志
slow_query_log=1
slow_query_log_file=/export/server/mysql/slow.log
long_query_time=1

[client]
socket=/tmp/mysql.sock
default-character-set=utf8mb4
EOF
```

#### （2）重启 MySQL

```Plain
touch /export/server/mysql/master.err
chown -Rf mysql:mysql /export/server/mysql
systemctl restart mysqld
systemctl status mysqld --no-pager
```

### 5.3 配置 slave（mysql-node2）

#### （1）修改配置文件

```Plain
cat >/etc/my.cnf<<EOF
[mysqld]
# 基础路径
basedir=/export/server/mysql
datadir=/export/server/mysql/data
socket=/tmp/mysql.sock
port=3306

# 日志
log-error=/export/server/mysql/slave.err
log-bin=/export/server/mysql/data/binlog
relay-log=/export/server/mysql/data/relaylog
relay-log-index=/export/server/mysql/data/relaylog.index
server-id=102

# 字符集
character_set_server=utf8mb4
collation-server=utf8mb4_unicode_ci

# GTID 核心配置
gtid_mode=ON
enforce_gtid_consistency=ON
log_slave_updates=ON
binlog_format=ROW
binlog_row_image=FULL
sync_binlog=1
expire_logs_days=7

# 复制过滤：不同步系统库
replicate-ignore-db=information_schema
replicate-ignore-db=mysql
replicate-ignore-db=performance_schema
replicate-ignore-db=sys

# 从库只读
read_only=ON
super_read_only=ON
skip-slave-start=1   # 避免重启时自动启动复制线程

# 复制优化
relay_log_recovery=1
sync_relay_log=0
sync_relay_log_info=0
sync_master_info=0

# InnoDB 优化
default_storage_engine=InnoDB
innodb_buffer_pool_size=1G
innodb_log_file_size=256M
innodb_log_buffer_size=64M
innodb_flush_log_at_trx_commit=2   # 从库可设为 2 提高性能
innodb_file_per_table=1

# 连接配置
max_connections=500
max_connect_errors=1000000
table_open_cache=2000

# 慢查询日志
slow_query_log=1
slow_query_log_file=/export/server/mysql/slow.log
long_query_time=1

[client]
socket=/tmp/mysql.sock
default-character-set=utf8mb4
EOF
```

#### （2）重启 MySQL

```Plain
touch /export/server/mysql/slave.err
chown -Rf mysql:mysql /export/server/mysql
systemctl restart mysqld
systemctl status mysqld --no-pager
```

## 六、创建复制账号（在 master 执行）

```Plain
mysql -uroot -p'MySQL@666'
```

在 MySQL 命令行执行：

```Shell
-- 创建复制用户
CREATE USER 'slave'@'%' IDENTIFIED WITH mysql_native_password BY 'MySQL@666';

-- 授权
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'%';
FLUSH PRIVILEGES;
EXIT;
```

## 七、启动主从同步（在 slave 执行）

### 7.1 配置主从连接

```Plain
mysql -uroot -p'MySQL@666'
```

在 MySQL 命令行执行：

```Plain
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='192.168.88.101',
  SOURCE_PORT=3306,
  SOURCE_USER='slave',
  SOURCE_PASSWORD='MySQL@666',
  SOURCE_AUTO_POSITION=1;
```

### 7.2 启动复制

```Plain
START REPLICA;
```

### 7.3 查看同步状态

```Plain
SHOW REPLICA STATUS\G
```

![img](https://ycn1pzibbi1s.feishu.cn/space/api/box/stream/download/asynccode/?code=OWNhNzBmNGY0ZDM0MzVmMjkwZGMzN2E0MGRmZjdjNDNfU2RBR2VnaTZsZFlGZFhENkhrc0kzNHNpVDl5bWozRnRfVG9rZW46RG5jUWJWNUlCbzNzOUV4QW93SWM4VzRRbmloXzE3NzU2NTAzMDc6MTc3NTY1MzkwN19WNA)
