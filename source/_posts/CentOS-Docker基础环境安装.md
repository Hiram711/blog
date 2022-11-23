title: CentOS Docker基础环境安装
author: 几时西风
tags:
  - docker
  - basic
categories:
  - docker
date: 2022-11-23 16:40:00
---
## 环境准备

### 切换到root账户

```
su - root
```

### 安装wget

```
yum install wget -y
```

### 备份原始Yum源

```
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
```

### 下载新的Yum源

```
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
```

### 安装EPEL源

```
yum install epel-release -y
```

### 备份原始EPEL源

```
mv /etc/yum.repos.d/epel.repo /etc/yum.repos.d/epel.repo.backup
mv /etc/yum.repos.d/epel-testing.repo /etc/yum.repos.d/epel-testing.repo.backup
```

### 下载新的EPEL源

```
wget -O /etc/yum.repos.d/epel.repo https://mirrors.aliyun.com/repo/epel-7.repo
```

### 清除Yum缓存

```
yum clean all
```

### 构建Yum缓存

```
yum makecache
```

### 安装常用工具组

```
yum groupinstall -y "Compatibility Libraries" \
                    "Console Internet Tools" \
                    "Security Tools" \
                    "Development Tools"
```

### 安装常用工具包

```
yum install -y yum-utils \
               nfs-utils \
               bind-utils \
               bridge-utils \
               httpd-tools\
               net-tools \
               device-mapper-persistent-data \
               lvm2 \
               ipvsadm \
               htop \
               tzdata \
               nscd \
               openssl \
               vim \
               neovim \
               curl \
               wget \
               zip \
               unzip \
               bzip2 \
               git \
               subversion \
               perl \
               ruby \
               python \
               python3 \
               lrzsz \
               tree \
               finger \
               lsof \
               telnet \
               nmap \
               nc
```

### 安装X11图形支持（按需）

```
yum install -y xorg-x11-xauth \
               gedit \
               gvim
```

## 禁用yum自动更新（根据实际需要选择）

### 停止yum自动更新服务

```
systemctl stop yum-cron
```

### 禁用yum自动更新服务

```
systemctl disable yum-cron
```

### 查看yum自动更新服务状态

```
systemctl status yum-cron
```

## 设置系统时区

```
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo "Asia/Shanghai" > /etc/timezone
cat /etc/timezone
echo "export TZ='Asia/Shanghai'" > /etc/profile.d/tz.sh
cat /etc/profile.d/tz.sh
```

## 设置语言（按需）

```
echo "LANG=\"zh_CN.UTF-8\"" > /etc/locale.conf
cat /etc/locale.conf
```

## NTP服务

### 安装NTP服务

```
yum install ntpdate -y
```

### 设置时间同步上游服务器

> 编辑

```
vim /etc/ntp/step-tickers
```

> 修改

```
0.centos.pool.ntp.org
=>
# 国家授时中心
ntp.ntsc.ac.cn
# 亚洲授时中心
asia.pool.ntp.org
# 阿里云授时中心
ntp.aliyun.com
```

### 启用ntpdate服务

```
systemctl enable ntpdate
```

### 启动ntpdate服务

```
systemctl restart ntpdate
```

> 有时启动不成功，多试几次，还不成功就刷新dns缓存

```
systemctl restart nscd
```

### 查看ntpdate服务状态

```
systemctl status ntpdate
```
## 防火墙

### 停止防火墙

```
systemctl stop firewalld
```

### 禁用防火墙

```
systemctl disable firewalld
```

### 查看防火墙状态

```
systemctl status firewalld
```

## 关闭SELinux

```
setenforce 0
sed -i "s/^SELINUX\=enforcing/SELINUX\=disabled/g" /etc/selinux/config
sed -i "s/^SELINUX\=permissive/SELINUX\=disabled/g" /etc/selinux/config
sed -i "s/^SELINUX\=enforcing/SELINUX\=disabled/g" /etc/sysconfig/selinux
sed -i "s/^SELINUX\=permissive/SELINUX\=disabled/g" /etc/sysconfig/selinux
getenforce
/usr/sbin/sestatus -v
```

## 内核参数设置

> 编辑

```
vim /etc/profile
```

> 追加

```
modprobe -a br_netfilter \
            ip6_udp_tunnel \
            ip_set \
            ip_set_hash_ip \
            ip_set_hash_net \
            iptable_filter \
            iptable_nat \
            iptable_mangle \
            iptable_raw \
            nf_conntrack_netlink \
            nf_conntrack \
            nf_conntrack_ipv4 \
            nf_defrag_ipv4 \
            nf_nat \
            nf_nat_ipv4 \
            nf_nat_masquerade_ipv4 \
            udp_tunnel \
            veth \
            vxlan \
            x_tables \
            xt_addrtype \
            xt_conntrack \
            xt_comment \
            xt_mark \
            xt_multiport \
            xt_nat \
            xt_recent \
            xt_set \
            xt_statistic \
            xt_tcpudp
```

> 使其生效

```
source /etc/profile
```

> 查看未加载模块（什么都不显示就对了）

```
for module in br_netfilter ip6_udp_tunnel ip_set ip_set_hash_ip ip_set_hash_net iptable_filter iptable_nat iptable_mangle iptable_raw nf_conntrack_netlink nf_conntrack nf_conntrack_ipv4   nf_defrag_ipv4 nf_nat nf_nat_ipv4 nf_nat_masquerade_ipv4 udp_tunnel veth vxlan x_tables xt_addrtype xt_conntrack xt_comment xt_mark xt_multiport xt_nat xt_recent xt_set  xt_statistic xt_tcpudp;
do
  if ! lsmod | grep -q $module; then
    if ! lsmod | grep -q $module /lib/modules/$(uname -r)/modules.builtin; then
        echo "module $module is not present";
    fi;
  fi;
done;
```

> 编辑

```
vim /etc/sysctl.conf
```

> 追加或修改内容

```
# 优化参数
net.core.netdev_max_backlog = 262144
net.core.somaxconn = 65535
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_syn_retries = 2
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_abort_on_overflow = 1
net.ipv4.tcp_rmem = 16384 1048576 12582912
net.ipv4.tcp_wmem = 16384 1048576 12582912
net.ipv4.tcp_mem = 1541646 2055528 3083292
net.ipv4.tcp_moderate_rcvbuf = 1
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_max_tw_buckets = 16384
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 0
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_tw_recycle = 0
net.ipv4.tcp_retries1 = 2
net.ipv4.tcp_retries2 = 3
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 2
net.ipv4.tcp_keepalive_probes = 3
fs.file-max = 2147483647
fs.nr_open = 1048576

# lvs必备参数
net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.lo.arp_announce = 1
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 1

# docker必备参数
net.ipv4.ip_forward = 1

# kubernetes必备参数
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1

# ab参数
net.nf_conntrack_max = 65535
net.netfilter.nf_conntrack_tcp_timeout_established = 1200
```

> 使配置立即生效

```
sysctl -p
```

## 设置文件描述符

> 编辑

```
vim /etc/security/limits.conf
```

> 追加或修改内容

```
* soft nofile  1048576
* hard nofile  1048576
* soft nproc   1048576
* hard nproc   1048576
* soft memlock unlimited
* hard memlock unlimited
```

> 编辑

```
vim /etc/security/limits.d/20-nproc.conf
```

> 追加或修改内容

```
* soft nofile  1048576
* hard nofile  1048576
* soft nproc   1048576
* hard nproc   1048576
* soft memlock unlimited
* hard memlock unlimited
```

> 编辑

```
vim /etc/pam.d/login
```

> 追加或修改内容

```
# 32位系统
session    required     /lib/security/pam_limits.so
# 64位系统
session    required     /lib64/security/pam_limits.so
```

## 关闭虚拟内存

```
swapoff -a
sed -i 's/.*swap.*/#&/' /etc/fstab
cat /etc/fstab
```

## 配置虚拟内存设置【可选】

> 物理内存小于4G（虚拟内存统一设置为4G）

```
dd if=/dev/zero of=/swap bs=1024 count=4096000
mkswap /swap
chmod 0600 /swap
swapon /swap
```
> 物理内存小于16G（虚拟内存统一设置为8G）

```
dd if=/dev/zero of=/swap bs=1024 count=8192000
mkswap /swap
chmod 0600 /swap
swapon /swap
```

> 物理内存小于64G（虚拟内存统一设置为16G）

```
dd if=/dev/zero of=/swap bs=1024 count=16384000
mkswap /swap
chmod 0600 /swap
swapon /swap
```

> 物理内存小于256G（虚拟内存统一设置为32G）

```
dd if=/dev/zero of=/swap bs=1024 count=32768000
mkswap /swap
chmod 0600 /swap
swapon /swap
```

> 设置改挂载点

```
vim /etc/fstab
```

> 编辑或新增

```
/swap swap swap defaults 0 0
```

## 关闭transparent_hugepage

### 临时关闭

```
echo "never" > /sys/kernel/mm/transparent_hugepage/enabled
echo "never" >  /sys/kernel/mm/transparent_hugepage/defrag
```

### 永久关闭

> 编辑

```
vim /etc/default/grub
```

> 在GRUB_CMDLINE_LINUX加入选项transparent_hugepage=never

```
# ext4
GRUB_CMDLINE_LINUX="crashkernel=auto spectre_v2=retpoline rhgb quiet transparent_hugepage=never"
或者
# xfs
GRUB_CMDLINE_LINUX="crashkernel=auto spectre_v2=retpoline rd.lvm.lv=centos/root rhgb quiet transparent_hugepage=never"
```

> 生成grub文件（bios）

```
grub2-mkconfig -o /boot/grub2/grub.cfg
```

> 生成文件的grub文件（efi）

```
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
```

## 添加执行路径环境变量配置

```
echo 'export PATH=.:$PATH' > /etc/profile.d/here.sh
cat /etc/profile.d/here.sh
```

## 密码策略配置

> 编辑

```
vim /etc/login.defs
```

> 修改

```
PASS_MAX_DAYS   99999
PASS_MIN_DAYS   0
=>
PASS_MAX_DAYS   180
PASS_MIN_DAYS   7
```

> 执行

```
chage --maxdays 180 root
chage --mindays 7 root
```

### 密码复杂度配置

> 编辑

```
vim /etc/security/pwquality.conf
```

> 修改

```
# minlen = 9
# minclass = 0
=>
minlen = 8
minclass = 3
```

### 密码重用限制

> 编辑

```
vim /etc/pam.d/password-auth
```

> 修改

```
password    sufficient    pam_unix.so sha512 shadow nullok try_first_pass use_authtok
=>
password    sufficient    pam_unix.so sha512 shadow nullok try_first_pass use_authtok remember=5
```

> 编辑

```
vim /etc/pam.d/system-auth
```

> 修改

```
password    sufficient    pam_unix.so sha512 shadow nullok try_first_pass use_authtok
=>
password    sufficient    pam_unix.so sha512 shadow nullok try_first_pass use_authtok remember=5
```

## SSH限制策略

> 编辑

```
vim /etc/ssh/sshd_config
```

> 修改

```
#ClientAliveInterval 0
#ClientAliveCountMax 3
=>
ClientAliveInterval 600
ClientAliveCountMax 2
```

## 创建物理卷

```
pvcreate /dev/sdb
```

### 创建卷组

```
vgcreate storage /dev/sdb
```

### 创建逻辑卷

```
lvcreate -l 100%Free -n data storage
```

### 格式化逻辑卷

```
mkfs.xfs /dev/storage/data
```

### 创建挂载点

```
mkdir -p /storage/data
```

### 添加挂载信息

> 编辑

```
vim /etc/fstab
```

> 追加

```
/dev/mapper/storage-data /storage/data          xfs     defaults        0 0
```

### 挂载

```
mount -a
```

### 查看

```
df -hT
```

### 重启

```
shutdown -r now
```

---

### 重启后切换到root账户

```
su - root
```

## 开始安装Docker

### 安装Docker

```
yum remove docker \
          docker-client \
          docker-client-latest \
          docker-common \
          docker-latest \
          docker-latest-logrotate \
          docker-logrotate \
          docker-selinux \
          docker-engine-selinux \
          docker-engine
```

### 添加Docker仓库

> 官方源

```
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

> 阿里云源

```
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

### 安装docker和docker-compose

```
yum install -y docker-ce docker-ce-cli containerd.io docker-compose
```

### 配置docker监听地址（2375按需开，因为没有密码，至少不能配置为监听公网地址）

```
vim /lib/systemd/system/docker.service
```

```
# ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
ExecStart=/usr/bin/dockerd --icc=false -H unix:///var/run/docker.sock -H tcp://127.0.0.1:2375 -H fd:// --containerd=/run/containerd/containerd.sock
```

> 或者

```
sed -i 's/^ExecStart=.*/ExecStart=\/usr\/bin\/dockerd --icc=false -H unix:\/\/\/var\/run\/docker.sock -H tcp:\/\/127.0.0.1:2375 -H fd:\/\/ --containerd=\/run\/containerd\/containerd.sock/' /lib/systemd/system/docker.service
cat /lib/systemd/system/docker.service
```

### 配置镜像加速器&私有镜像仓库的非安全访问

```
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<- 'EOF'
{
  "registry-mirrors": ["https://vax5oqx1.mirror.aliyuncs.com"],
  "insecure-registries": ["docker.parkson.net.cn:5000", "docker-registry.parkson.net.cn:5000", "docker.iparkson.net.cn:5000", "docker-registry.iparkson.net.cn:5000"]
}
EOF
```

### 重新加载服务配置

```
systemctl daemon-reload
```

### 启用docker

```
systemctl enable docker
```

### 启动docker

```
systemctl restart docker
```

### 查看docker状态

```
systemctl status docker
```

### 下载convoy

```
wget https://github.com/rancher/convoy/releases/download/v0.5.2/convoy.tar.gz
```

### 解压

```
tar xvzf convoy.tar.gz
```

### 复制

```
\cp -f convoy/convoy convoy/convoy-pdata_tools /usr/local/bin/
```

### 创建docker插件目录

```
mkdir -p /etc/docker/plugins/
```

### 写入插件信息

```
echo "unix:///var/run/convoy/convoy.sock" > /etc/docker/plugins/convoy.spec
```

### 创建convoy服务

> 创建卷存储目录

```
mkdir -p /storage/data/docker/volumes
```

> 编辑

```
vim /usr/lib/systemd/system/convoy.service
```

> 内容

```
[Unit]
Description=Convoy Service
After=network.target

[Service]
LimitNOFILE=1048576
WorkingDirectory=/storage/data/docker/volumes
ExecStart=/usr/local/bin/convoy daemon --drivers vfs --driver-opts vfs.path=/storage/data/docker/volumes
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 启动服务

> 启用服务

```
systemctl enable convoy
```

> 启动服务

```
systemctl start convoy
```

> 查看状态

```
systemctl status convoy
```

### 重启docker服务

```
systemctl restart docker
```

### 编辑crontab

```
mkdir -p /etc/cron.d/
tee /etc/cron.d/docker <<- 'EOF'
0 0 * * * root docker images | awk '$1 == "<none>" || $2 == "<none>" {print $3}' | xargs -r -t docker rmi
* * * * * root docker ps -a | grep '(unhealthy)' | awk '{print $1}' | xargs -r -t docker restart
EOF
```

### 启用crond

```
systemctl enable crond
```

### 启动crond

```
systemctl restart crond
```

### 查看crond状态

```
systemctl status crond
```

### 查看cron日志

```
cat /var/log/cron
```

### 退出root账户

```
exit
```

### 将当前用户加入到docker组

```
sudo gpasswd -a ${USER} docker
```

```
sudo newgrp docker
```
### 配置DOCKER_HOST环境变量【可选】

```
vim ~/.bashrc
```

> 追加

```
export DOCKER_CONTENT_TRUST=1
```

### 审核Docker文件和目录

> 编辑

```
vim /etc/audit/audit.rules
```

> 追加

```
-w /var/lib/docker -k docker
-w /etc/docker -k docker
-w /usr/lib/systemd/system/docker.service -k docker
-w /usr/lib/systemd/system/docker.socket -k docker
-w /usr/bin/docker-containerd -k docker
-w /usr/bin/docker-runc -k docker
```

> 编辑

```
vim /etc/audit/rules.d/audit.rules
```

> 追加

```
-w /var/lib/docker -k docker
-w /etc/docker -k docker
-w /usr/lib/systemd/system/docker.service -k docker
-w /usr/lib/systemd/system/docker.socket -k docker
-w /usr/bin/docker-containerd -k docker
-w /usr/bin/docker-runc -k docker
```

> 启用

```
systemctl enable auditd
```

> 启动

```
systemctl restart auditd
```

> 查看

```
systemctl enable auditd
```