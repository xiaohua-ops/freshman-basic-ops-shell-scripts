# Nginx 安装

# 一、Nginx 安装（Linux 环境准备，CentOS Stream 9 通用）

### 1.1 设置 IP 地址

```Shell
# 修改网卡ip 环境不一按需更改
vim /etc/NetworkManager/system-connections/ens160.nmconnection
```

### 1.2 主机名称（符合 FQDN 规范）

```Shell
hostnamectl set-hostname web01.itops.cn && bash
#例：web01.itops.cn
```

### 1.3 关闭防火墙与 SELinux

安装环境之前需要关闭，安装后再启动，配置防火墙规则，释放 80/443 端口由于是测试环境直接全部关闭修改规则可看 iptables 和 firewalld 后续更新

```Markdown
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

### 1.4 设置完成后，重启服务器（测试环境）

```Shell
# 重启
reboot
或
init 6
```

### Nginx 1.28.0 源码编译安装主脚本

注：提前自行下载对应版本的安装包，或让脚本自动从官网下载

```Bash
#!/bin/bash

#  File  : install_nginx.sh
#  Desc  : 源码编译安装 Nginx（本地包优先，官网下载兜底）
#  Test  : CentOS Stream 9 / Rocky 9 / CentOS 7

# 可自行修改的变量
NGX_VER=1.28.0
NGX_TAR="nginx-${NGX_VER}.tar.gz"
NGX_DIR="/usr/local/nginx"          # 安装路径
NGX_USER="nginx"                    # 运行用户
# -----------------------------------------------------

# 环境与前置检查
# 自动识别包管理器（兼容CentOS7/8/9、Rocky 9）
if command -v dnf &> /dev/null; then
    PKG_MANAGER="dnf"
else
    PKG_MANAGER="yum"
fi

# 检查是否已安装Nginx，避免重复执行
if [ -d "${NGX_DIR}" ] && [ -f "${NGX_DIR}/sbin/nginx" ]; then
    echo -e "\033[0;32m[INFO]\033[0m 检测到Nginx已安装在 ${NGX_DIR}，跳过安装流程"
    echo -e "\033[0;32m[INFO]\033[0m 如需重新安装，请先删除 ${NGX_DIR} 目录后再执行脚本"
    exit 0
fi

# 颜色输出
RED='\033[0;31m'; GREEN='\033[0;32m'; NC='\033[0m'
log_info(){ echo -e "${GREEN}[INFO]${NC} $*"; }
log_err(){ echo -e "${RED}[ERR]${NC} $*" >&2; }

# 必须以 root 运行
[[ $EUID -eq 0 ]] || { log_err "请用 root 执行脚本"; exit 1; }

# 优化Linux系统
log_info "优化Linux系统（关闭SELinux与防火墙）..."
sed -i -r 's/SELINUX=[ep].*/SELINUX=disabled/g' /etc/selinux/config
setenforce 0 2>/dev/null || log_info "SELinux已处于关闭状态"
systemctl stop firewalld &> /dev/null
systemctl disable firewalld &> /dev/null
iptables -F 2>/dev/null
iptables -t nat -F 2>/dev/null
iptables -P INPUT ACCEPT 2>/dev/null
iptables -P FORWARD ACCEPT 2>/dev/null

# 卸载旧 nginx（如有）
if rpm -q nginx &>/dev/null; then
    log_info "检测到系统已安装 rpm 版 nginx，正在卸载..."
    $PKG_MANAGER remove -y nginx
fi

# 安装基础编译依赖
log_info "安装依赖包..."
$PKG_MANAGER -y install gcc make pcre-devel zlib-devel openssl-devel
if [ $? -ne 0 ]; then
    log_err "依赖包安装失败，请检查网络或yum/dnf源"
    exit 1
fi

# 创建运行用户
id -u $NGX_USER &>/dev/null || {
    log_info "创建Nginx运行用户：${NGX_USER}"
    useradd -r -s /sbin/nologin -d /var/cache/nginx -c "Nginx user" $NGX_USER
}

# 找源码包（本地优先 → 官网下载）
if [[ -f $NGX_TAR ]]; then
    log_info "使用本地源码包：$NGX_TAR"
else
    log_info "本地未找到 $NGX_TAR，正在从官网下载..."
    wget -O "$NGX_TAR" "https://nginx.org/download/$NGX_TAR"
    if [ $? -ne 0 ]; then
        log_err "源码包下载失败，请检查网络连接或手动下载 ${NGX_TAR} 到当前目录"
        exit 1
    fi
fi

# 解压（优化：清理旧解压目录）
log_info "解压源码..."
if [ -d "nginx-${NGX_VER}" ]; then
    rm -rf "nginx-${NGX_VER}"
fi
tar -xf "$NGX_TAR"
cd "nginx-${NGX_VER}" || { log_err "进入源码目录失败"; exit 1; }

# 预编译
log_info "配置编译参数..."
./configure \
  --prefix=$NGX_DIR \
  --user=$NGX_USER \
  --group=$NGX_USER \
  --with-http_ssl_module \
  --with-http_stub_status_module \
  --with-http_realip_module
if [ $? -ne 0 ]; then
    log_err "预编译配置失败，请检查依赖是否完整"
    exit 1
fi

# 编译并安装（优化：增加失败检查） 
log_info "开始编译（并行 $(nproc) 任务）..."
make -j$(nproc)
if [ $? -ne 0 ]; then
    log_err "编译失败，请查看上面的报错信息排查问题"
    exit 1
fi
log_info "编译完成，正在安装..."
make install
if [ $? -ne 0 ]; then
    log_err "安装失败，请查看上面的报错信息排查问题"
    exit 1
fi

# 环境变量 & 权限（优化：避免重复添加）
log_info "配置环境变量与目录权限..."
if [ ! -f /etc/profile.d/nginx.sh ]; then
    echo "export PATH=\$PATH:$NGX_DIR/sbin" > /etc/profile.d/nginx.sh
fi
source /etc/profile.d/nginx.sh
chown -R $NGX_USER:$NGX_USER $NGX_DIR

# systemd 服务（优化：增加权限设置）
log_info "创建 systemd 单元文件..."
cat >/usr/lib/systemd/system/nginx.service <<EOF
[Unit]
Description=Nginx Web Server
After=network.target

[Service]
Type=forking

ExecStartPre=/usr/bin/rm -f /run/nginx.pid
ExecStart=${NGX_DIR}/sbin/nginx -c ${NGX_DIR}/conf/nginx.conf
ExecReload=${NGX_DIR}/sbin/nginx -s reload
ExecStop=${NGX_DIR}/sbin/nginx -s quit
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
chmod 644 /usr/lib/systemd/system/nginx.service

# 启服务 & 开机自启 
log_info "加载 systemd 并启动 nginx..."
systemctl daemon-reload
systemctl enable --now nginx
if [ $? -ne 0 ]; then
    log_err "Nginx服务启动失败，请执行 journalctl -u nginx 查看日志排查问题"
    exit 1
fi

# 验证（优化：增加端口监听检查）
log_info "查看版本与运行状态..."
nginx -v
systemctl status nginx --no-pager -l
if netstat -tulnp 2>/dev/null | grep -q ':80 ' || ss -tulnp 2>/dev/null | grep -q ':80 '; then
    log_info "80端口已正常监听"
else
    log_info "80端口未监听，请检查防火墙或Nginx配置"
fi

log_info "Nginx 源码编译安装完成！"
log_info "安装包版本：nginx-${NGX_VER}"
log_info "安装路径：${NGX_DIR}"
log_info "访问 http://<本机IP> 即可验证！"
log_info "执行 source /etc/profile.d/nginx.sh 获取最新环境变量！"
```

### 更换版本方法

```Shell
NGX_VER=1.26.2  # 例如更换为1.26.2稳定版
```

### 增加执行权限

```Shell
chmod +x install_nginx.sh
```

### 测试脚本

```Shell
# 调试模式执行（查看每一步执行过程）
bash -x install_nginx.sh

# 直接执行
./install_nginx.sh
```