MySQL主从架构部署

## 一、主从架构概述

### 1.1 适用场景

本文档适配典型业务场景：一主多从架构，主库承担业务增删改查核心操作，从库实现数据在线热备、读写分离读请求分流，解决以下核心问题：

**数据实时****热备**：主库故障时可快速切换从库接管业务，降低数据丢失风险

**读写分离分流**：读请求分发至从库，缓解主库业务压力

**数据容灾****备份**：配合全量备份，实现数据时间点恢复能力

### 1.2 主从复制核心概念

主从复制（主从同步）：将数据从一台主数据库服务器（Master）复制到一台或多台从数据库服务器（Slave），默认采用**异步复制**模式，无需维持长连接。

表格

复制模式核心特点优势劣势异步复制主库执行完事务立即返回结果，从库异步拉取并回放 binlog不阻塞主库事务操作，业务无感知存在主从同步延迟，极端情况有数据丢失风险同步复制主库事务提交后，必须等待所有从库同步完成才返回结果主从数据高度一致，无数据丢失风险严重阻塞主库事务操作，业务性能大幅下降

核心日志说明：

**binlog** **二进制日志**：主库开启，记录所有对数据库的增删改事务操作 SQL

- **relaylog** **中继日志**：从库开启，存储从主库拉取的 binlog 内容，供从库 SQL 线程回放执行

### 2.1 环境说明

### ![image-20260408195820742](C:\Users\xiao\AppData\Roaming\Typora\typora-user-images\image-20260408195820742.png)

### 2.2 系统环境初始化

**以下操作需在 Master 和 Slave 两台节点全部执行**

1. #### 安装系统必备依赖

```Plain
dnf install vim wget rsync telnet net-tools -y
```

1. #### 配置主机名

```Plain
# Master节点执行
hostnamectl set-hostname mysql-node1

# Slave节点执行
hostnamectl set-hostname mysql-node2

# 执行后重新登录终端生效bash

或者

hostnamectl set-hostname mysql-node1 && bash
hostnamectl set-hostname mysql-node2 && bash
```

1. #### 配置 IP 与主机名映射

```Plain
vim /etc/hosts
# 尾部追加以下内容，两台节点配置完全一致
192.168.88.101   mysql-node1   node1
192.168.88.102   mysql-node2   node2
```

1. #### 关闭防火墙与 SELinux

生产环境慎重

```Plain
# 关闭SELinux
sed  -i -r 's/SELINUX=[ep].*/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
# 关闭防火墙
systemctl stop firewalld &> /dev/null
systemctl disable firewalld &> /dev/null
iptables -F
iptables -t nat -F
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
```

1. #### 时间同步（可选，生产环境必配）

```Shell
# 验证时间是否一致
date -R
# 时间同步不一致执行
dnf -y install chrony
systemctl enable chronyd --now
```

## 三、MySQL 8.0 统一安装部署

**以下操作需在 Master 和 Slave 两台节点全部执行**，采用官方二进制包安装方式。

官网自行下载mysql安装包

1. ### 准备安装脚本

```Plain
vim install-mysql8.sh
```

1. ### 写入完整安装脚本

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

1. ### 执行安装脚本

```Plain
# 给脚本添加执行权限
chmod +x install-mysql8.sh

# 执行安装
./install-mysql8.sh
```

1. ### 验证安装结果

```Plain
# 验证服务状态
systemctl status mysqld --no-pager

# 验证登录
mysql -uroot -p'MySQL@666' -e "show databases;"
```

## 四、基于 binlog 点位的主从复制

### 4.1 部署核心前提

1. **Master 和 Slave 节点 MySQL 版本完全一致**
2. **Master 节点开启 binlog 二进制日志，Slave 节点开启 relaylog 中继日志**
3. **Master 和 Slave 节点的 server-id 全局唯一，不可重复**
4. **开启主从同步前，保证 Master 和 Slave 初始数据状态一致**

### 4.2 主库（Master）配置

1. #### 编辑 my.cnf 配置文件

```Plain
vim /etc/my.cnf
```

1. #### 写入完整主库配置

```Bash
[mysqld]
# 基础路径配置
basedir=/export/server/mysql
datadir=/export/server/mysql/data
socket=/tmp/mysql.sock
port=3306

# 字符集配置
character_set_server=utf8mb4
collation-server=utf8mb4_unicode_ci

# 错误日志配置
log-error=/export/server/mysql/master.err

# 二进制日志（主从复制核心）
log-bin=/export/server/mysql/data/binlog
server-id=101
binlog_format=ROW
expire_logs_days=7
sync_binlog=1

# 跳过同步的系统库
binlog-ignore-db=information_schema
binlog-ignore-db=mysql
binlog-ignore-db=performance_schema
binlog-ignore-db=sys

# InnoDB 引擎优化
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
```

1. #### 配置生效与服务重启

```Plain
# 创建日志文件并修改权限
touch /export/server/mysql/master.err
chown -Rf mysql:mysql /export/server/mysql/

# 重启服务
systemctl restart mysqld

# 验证服务状态
systemctl status mysqld --no-pager
```

### 4.3 从库（Slave）配置

1. #### 编辑 my.cnf 配置文件

```Plain
vim /etc/my.cnf
```

1. #### 写入完整从库配置

```Markdown
[mysqld]
# 基础路径配置
basedir=/export/server/mysql
datadir=/export/server/mysql/data
socket=/tmp/mysql.sock
port=3306

# 字符集配置
character_set_server=utf8mb4
collation-server=utf8mb4_unicode_ci

# 错误日志配置
log-error=/export/server/mysql/slave.err

# 中继日志（主从复制核心）
relay-log=/export/server/mysql/data/relaylog
relay-log-index=/export/server/mysql/data/relaylog.index
read-only=1
super-read-only=1
skip-slave-start=1

# 避免同步系统库
replicate-ignore-db=information_schema
replicate-ignore-db=mysql
replicate-ignore-db=performance_schema
replicate-ignore-db=sys

# server-id 必须全局唯一
server-id=102

# InnoDB 引擎优化
default_storage_engine=InnoDB
innodb_buffer_pool_size=1G
innodb_log_file_size=256M
innodb_log_buffer_size=64M
innodb_flush_log_at_trx_commit=2
innodb_file_per_table=1

# 连接配置
max_connections=500
table_open_cache=2000
max_connect_errors=1000000

# 慢查询日志
slow_query_log=1
slow_query_log_file=/export/server/mysql/slow.log
long_query_time=1

# 复制优化
relay_log_recovery=1
sync_relay_log=0
sync_relay_log_info=0
sync_master_info=0

[client]
socket=/tmp/mysql.sock
default-character-set=utf8mb4
```

1. #### 配置生效与服务重启

```Plain
# 创建日志文件并修改权限
touch /export/server/mysql/slave.err
chown -Rf mysql:mysql /export/server/mysql/

# 重启服务
systemctl restart mysqld

# 验证服务状态
systemctl status mysqld --no-pager
```

### 4.4 主库创建复制专用账号

**在 Master 节点执行**

```Plain
# 登录数据库
mysql -uroot -p'MySQL@666'
# 创建复制账号
CREATE USER 'slave'@'%' IDENTIFIED WITH mysql_native_password BY 'MySQL@666';
# 授予复制权限
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'%';
# 刷新权限
FLUSH PRIVILEGES;
# 退出
EXIT;
```

### 4.5 主从初始数据同步（可选）

适用于主库已有业务数据的场景，保证主从初始数据一致。

```Plain
# 1. Master节点停止MySQL服务
systemctl stop mysqld

# 2. Master节点执行数据目录远程同步
rsync -av --delete /export/server/mysql/data/ mysql-node2:/export/server/mysql/data/

# 3. 分别在Master和Slave节点删除auto.cnf（避免UUID冲突）
rm -f /export/server/mysql/data/auto.cnf

# 4. 分别启动Master和Slave节点MySQL服务
systemctl start mysqld
```

### 4.6 从库开启主从同步

1. #### Master 节点获取 binlog 点位信息

**在 Master 节点执行**

```Plain
# 登录数据库
mysql -uroot -p'MySQL@666'
# 加全局读锁，禁止写操作（确保点位不变化）
FLUSH TABLES WITH READ LOCK;
# 查看binlog文件名和点位，记录下来，后续从库配置需要使用
SHOW MASTER STATUS;
# 退出
EXIT;
```

输出示例，需记录`File`和`Position`字段：

```Shell
mysql> show master status;
+---------------+----------+--------------+------------------+-------------------+
| File          | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+---------------+----------+--------------+------------------+-------------------+
| binlog.000003 |      157 |              |                  |                   |
+---------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

1. #### Slave 节点配置主从同步

**在 Slave 节点执行**

```Plain
# 登录数据库
mysql -uroot -p'MySQL@666'
# 配置主库信息（MySQL 8.0.23+ 推荐使用CHANGE REPLICATION SOURCE TO，低版本使用CHANGE MASTER TO）
CHANGE REPLICATION SOURCE TO 
  SOURCE_HOST='192.168.88.101',
  SOURCE_PORT=3306,
  SOURCE_USER='slave',
  SOURCE_PASSWORD='MySQL@666',
  SOURCE_LOG_FILE='binlog.000003',
  SOURCE_LOG_POS=157;
# 启动从库同步
START REPLICA;
# 查看同步状态
SHOW REPLICA STATUS\G
```

![img](https://ycn1pzibbi1s.feishu.cn/space/api/box/stream/download/asynccode/?code=NjM2M2EzOWJhNWM4NTRiM2RjOThkMGFkZDk3NTQyMTdfV2tudjhzRzJTQ2ltakF3ZlhGeEtRZVhVVjJPOWhhMmpfVG9rZW46VldyOGJPWEc5b1B5Tll4OGhpWmNTRTJNbk5jXzE3NzU2NDk0MDM6MTc3NTY1MzAwM19WNA)

1. #### Master 节点解锁全局读锁

**在 Master 节点执行**

```Plain
mysql -uroot -p'MySQL@666' -e "UNLOCK TABLES;"
```