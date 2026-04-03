# 一.MySQL8数据库安装

Linux环境准备**（****CentOS** **Stream 9）**

### 1.1设置IP地址

```Shell
# 修改网卡ip 环境不一按需更改
vim /etc/NetworkManager/system-connections/ens160.nmconnection
```

### 1.2主机名称（符合FQDN规范）

```Shell
hostnamectl set-hostname node1.itops.cn && bash
#例：node1.itops.cn
```

### 1.3关闭防火墙与SELinux

安装环境之前需要关闭，安装后再启动，配置防火墙规则，释放3306端口

由于是测试环境直接全部关闭

修改规则可看iptables 和 firewalld 后续更新

```Shell
# 关闭防火墙 及开机自启
systemctl stop firewalld
systemctl disable firewalld

# 关闭iptables
iptables -F
iptables -t nat -F 
iptables -P INPUT ACCEPT 
iptables -P FORWARD ACCEPT

# 临时关闭selinux
setenforce 0
# 永久关闭selinux
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
sed -i 's/^SELINUX=permissive/SELINUX=disabled/' /etc/selinux/config
```

### 1.4设置完成后，重启服务器（测试环境）

```Shell
# 重启
reboot
或
init 6
```

### 1.5MySQL 8.0 二进制包安装主脚本

注：提前自行下载对应版本的安装包

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

1. #### 更换版本方法

只需修改脚本顶部的`MYSQL_PKG`变量值

MYSQL_PKG=

增加执行权限

```一.MySQL8数据库安装
chmod +x install_mysql8.sh

# 测试脚本
bash -x install_mysql8.sh

./install_mysql8.sh
```
