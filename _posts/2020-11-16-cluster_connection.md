---
layout: post
title: Cluster Network
subtitle: 
author: ixsluo
date: 2020-11-16 21:14:01 +0800
categories: 
tag: 
---

记录处理集群网络连接

- [1. 主机客户机网段设置](#1-主机客户机网段设置)
	- [查看网卡](#查看网卡)
	- [登录节点设置](#登录节点设置)
		- [**主机名修改**](#主机名修改)
		- [外部网络](#外部网络)
		- [内部网络](#内部网络)
		- [ip映射](#ip映射)
	- [计算节点设置](#计算节点设置)
		- [**主机名修改**](#主机名修改-1)
		- [内部网络](#内部网络-1)
		- [ip映射](#ip映射-1)
- [2. 程序包安装](#2-程序包安装)
- [3. NIS及NFS服务](#3-nis及nfs服务)
	- [主节点](#主节点)
	- [计算节点](#计算节点)
	- [NFS目录挂载](#nfs目录挂载)
- [4. PBS安装](#4-pbs安装)
	- [Torque(6.1.1.1)](#torque6111)
		- [主节点安装](#主节点安装)
		- [计算节点安装](#计算节点安装)
		- [Torque卸载](#torque卸载)
	- [MAUI(3.3)](#maui33)

## 1. 主机客户机网段设置

- 服务`network`

### 查看网卡

```bash
ip addr
```
假设内部网卡为`eno1`，外部网卡为`eno2`

### 登录节点设置

#### **主机名修改**

修改登录节点计算机名为*main*
```bash
hostnamectl set-hostname main
```

#### 外部网络

```bash
vi /etc/sysconfig/network-scripts/ifcfg-eno2
```

修改以下配置
```txt
BOOTPROTO=static  # 静态获取地址
ONBOOT=yes  # 系统启动时自动加载此配置
IPADDR=192.168.8.14  # 外部网络的路由地址
NETMASK=255.255.255.0  # 子网掩码
GATEWAY=192.168.8.254  # 网关
```

#### 内部网络

```bash
vi /etc/sysconfig/network-scripts/ifcfg-eno1
```

修改以下配置
```txt
BOOTPROTO=static  # 静态获取地址
ONBOOT=yes  # 系统启动时自动加载此配置
IPADDR=192.168.2.1  # 内部地址，2网段
NETMASK=255.255.255.0  # 子网掩码
GATEWAY=192.168.2.254  # 网关
```

#### ip映射

```bash
cat >> /etc/hosts << EOF
# 内网ip  |  主机名
192.168.2.1   main
192.168.2.101 node01
192.168.2.102 node02
EOF
```

### 计算节点设置

#### **主机名修改**

修改计算节点计算机名为*node\*\**，以node01为例
```bash
hostnamectl set-hostname node01
```

#### 内部网络

计算节点无外部网络，故只需修改内网网卡配置
```bash
vi /etc/sysconfig/network-scripts/ifcfg-eno1
```
```txt
BOOTPROTO=static  # 静态获取地址
ONBOOT=yes  # 系统启动时自动加载此配置
IPADDR=192.168.2.101  # 内部地址，2网段，101号
NETMASK=255.255.255.0  # 子网掩码
GATEWAY=192.168.2.1  # 登录节点作为网关
```
#### ip映射

**至少**要添加主节点ip
```bash
cat >> /etc/hosts << EOF
# 内网ip  |  主机名
192.168.2.1   main
EOF
```

## 2. 程序包安装

集群无外网，且计算节点为最小化安装

从另一台最小化安装的机器上下载安装包以及依赖包，如
```bash
yum -y install ypserv ypbind yp-tool --downloadonly --downloaddir=./NIS
# 下载NIS服务及当前缺少的依赖包到NIS文件夹
scp -rP <port> NIS <remote>:<dir>
# 传输到集群机器
```

从本地安装rpm包
```bash
yum localinstall ***.rpm
```

## 3. NIS及NFS服务

### 主节点

	软件包：
	ypserv ypbind yp-tool  # NIS
	nfs-utils rpcbind  # NFS

1. 设置NIS域名
```bash
nisdomainname JohnGPU
```

2. 设置NIS网域名称，开机自动设置NIS域名
```bash
echo "NISDOMAIN=JohnGPU" >> /etc/sysconfig/network
echo "/bin/nisdomainname JohnGPU" >> /etc/rc.d/rc.local
```

3. 启动服务
```bash
systemctl restart rpcbind
systemctl restart ypserv
```

4. 建立或**更新**数据库，并再次重启
```bash
/usr/lib64/yp/ypinit -m
> Ctrl + D
> y
systemctl restart ypserv
```


### 计算节点

```bash
# 软件包
yum install ypbind yp-tool
```

1. 加入同一NIS域，以及开机自动加入
```bash
echo "NISDOMAIN=JohnGPU" >> /etc/sysconfig/network
echo "/bin/nisdomainname JohnGPU" >> /etc/rc.d/rc.local
```

2. 修改用户密码验证顺序，由`/etc/nsswitch.conf`控制，搜索顺序由左至右
```bash
sed -i 's/^passwd:.*/passwd: files nis/g' /etc/nsswitch.conf
sed -i 's/^shadow:.*/shadow: files nis/g' /etc/nsswitch.conf
sed -i 's/^group:.*/group: files nis/g' /etc/nsswitch.conf
sed -i 's/^hosts:.*/hosts: files dns nis/g' /etc/nsswitch.conf
```

3. 修改计算节点配置，增加主节点地址
```bash
echo "domain JohnGPU server <服务端内部地址>" >> /etc/yp.conf
```

4. 修改系统认证文件
```bash
sed -i 's/USENIS=no/USENIS=yes/g' /etc/sysconfig/autoconfig

sed -i 's/^password[[:space:]]*sufficient.*/\
password    sufficient    pam_unix.so sha512 shadow nis nullok try_first_pass use_authtok/g' \
/etc/pam.d/system-auth
```

5. 重启服务，若配置正确仍启动失败，则可能为防火墙与selinux权限问题
```bash
systemctl restart rpcbind
systemctl restart ypbind
```

6. 计算节点执行`yptest`命令检查是否存在用户资料信息

### NFS目录挂载

1. 主节点

- 编辑nfs挂载配置
```bash
cat >> /etc/exports << EOF
/home 192.168.2.0/255.255.255.0(insecure,rw,sync,no_root_squash)
/opt  192.168.2.0/255.255.255.0(insecure,rw,sync,no_root_squash)
EOF
# 共享家目录及共享软件目录，共享网段为192.168.2.0，权限为rw(ro仅读)
# sync开启同步
```

- 启动服务
```bash
systemctl enable rpcbind.service
systemctl enable nfs-server.service
systemctl start rpcbind.service
systemctl start nfs-server.service
```
```bash
exportfs -r  # 使配置生效
exportfs  # 查看共享目录
```

- `showmount -e`检查nfs服务，若出现’clnt_create: RPC: Program not registered‘错误，依次关闭nfs和rpcbind，再依次启动rpcbind和nfs
```bash
systemctl stop nfs
systemctl stop rpcbind
systemctl start rpcbind
systemctl start nfs
```

2. 计算节点

- `rpcbind`服务确认启动
- `nfs`服务确认安装，但不需启动

```bash
showmount -e main  # 查看可挂载的主机共享目录
mount -t nfs main:/home /home  # 挂载家目录
mount -t nfs main:/opt  /opt  # 挂载家目录
```

- 若出现`Stale File Handle`错误，先取消挂载，再挂载
```bash
umount -f /home
```

- 自动挂载-autofs

    未实现，转而使用挂载脚本
```bash
for node in 01 02 ; do
    ssh node$node > /dev/null 2>&1 << EOF
    mount -t nfs main:/home /home
    mount -t nfs main:/opt /opt
    exit
EOF
done
```

: ```bash
: # 软件包
: yum install autofs hesiod
: # 修改配置
: sed -i "s/^timeout = .*$/timeout = 0/g" /etc/autofs.conf
: echo /home  /etc/auto.nfs >> /etc/auto.master
: echo /opt  /etc/auto.nfs >> /etc/auto.master
: echo /home -rw -insecure -sync -no_root_squash main:/home >> /etc/auto.nfs
: echo /opt -rw -insecure -sync -no_root_squash main:/opt >> /etc/auto.nfs
: ```

: ```bash
: echo "main:/home /home nfs defaults 0 0" >> /etc/fstab
: echo "main:/opt  /opt  nfs defaults 0 0" >> /etc/fstab
: mount -a
: ```

## 4. PBS安装

```bash
# 软件包
yum install \
libxml2-devel openssl-devel \
gcc gcc-c++ \
boost-devel libtool
```

### Torque(6.1.1.1)

```bash
# 下载
wget http://wpfilebase.s3.amazonaws.com/torque/torque-6.1.0.tar.gz
tar -zxvf torque-*
cd torque-*
```

#### 主节点安装

1. 编译安装
```bash
./configure
# --prefix=[/usr/local]  安装目录
# --with-server-home=[/var/spool/torque]  默认配置文件目录
make
make packages
make install
echo main > /var/spool/torque/server_name  # 写入主节点主机名
```

2. 建立库文件
```bash
echo "/usr/local/lib" > /etc/ld.so.conf.d/torque.conf  # 安装库路径
ldconfig
# libtool --finish /usr/local/torque/lib
```

3. 启动服务`trqauthd`
```bash
cp contrib/init.d/trqauthd.service /usr/lib/systemd/system/
systemctl enable trqauthd.service
systemctl start trqauthd.service
```

4. 初始化设置
```bash
./torque.setup root
qterm -tquick
```

5. 设置计算节点属性
```bash
# cp ~/nodes /var/spool/torque/server_priv/
cat > $TORQUE_HOME/server_priv/nodes << EOF
# 节点名  |  cpu核数  |  属性等
node01 np=28
node02 np=28
# ...
EOF
```

6. 安装到自定义目录时，检查环境变量，若PATH中缺少相应路径，则手动添加
```bash
cat >> /etc/profile << EOF
TORQUE=/usr/local/torque  # 安装目录
TORQUE_HOME=/var/spool/torque  # 配置文件目录
export PATH=$TORQUE/bin:$TORQUE/sbin:$PATH
EOF
source /etc/profile
```

7. 启动服务`pbs_server`、`pbs_sched`
```bash
cp contrib/systemd/pbs_server.service /usr/lib/systemd/system/
systemctl enable pbs_server.service
systemctl start pbs_server.service

cp contrib/systemd/ pbs_sched.service /usr/lib/systemd/system/
systemctl enable pbs_sched.service
systemctl start pbs_sched.service
```

8. 复制计算节点安装包至各计算节点
```bash
for node in 01 02 ; do
  scp contrib/init.d/{pbs_mom,trqauthd} node$node:/etc/init.d/
  scp torque-package-{mom,clients}-linux-x86_64.sh node$node:/root/
  scp /etc/ld.so.conf.d/torque/conf node$node:/etc/ld.so.conf.d/
done
```

#### 计算节点安装

1. 安装由主节点复制过来的安装包
```bash
./torque-package-clients-linux-x86_64.sh --install
./torque-package-mom-linux-x86_64.sh --install
```

2. 链接库
```bash
/sbin/ldconfig
```

3. 设置主节点名
```bash
# 配置文件目录与主节点相同
# main为主节点主机名
echo -e "\$pbsserver main\n\$logevent 225" > /var/spool/torque/mom_priv/config
echo main > /var/spool/torque/server_name
```

4. 启动服务`pbs_mom`
```bash
systemctl enable pbs_mom
systemctl start pbs_mom
```

5. 检查是否成功
```bash
pbsnodes -a  # 检查节点状态是否为free

# 非root用户下，运行测试
qsub -I
echo "sleep 10" | qsub
qstat
```

#### Torque卸载

1. 停止服务
```bash
# 计算节点
systemctl stop pbs_mom.service
# 主节点
systemctl stop pbs_sched.service
systemctl stop pbs_server.service
systemctl stop trqauthd.service
# 计算节点
systemctl disable pbs_mom.service
# 主节点
systemctl disable pbs_sched.service
systemctl disable pbs_server.service
systemctl disable trqauthd.service
```

2. 删除添加的文件
```bash
# 计算节点
rm -f /usr/lib/systemd/system/pbs_mom.service
# 主节点
rm -f /usr/lib/systemd/system/pbs_sched.service
rm -f /usr/lib/systemd/system/pbs_server.service
rm -f /usr/lib/systemd/system/trqauthd.service
# 计算节点
rm -f /etc/ld.so.conf.d/torque.conf
# 主节点
rm -f /etc/ld.so.conf.d/torque.conf
make uninstall
```

3. 计算节点卸载
```bash
./torque-package-mom-linux-x86_64.sh -l
./torque-package-clients-linux-x86_64.sh -l
```

### MAUI(3.3)

1. 主节点编译安装
```bash
tar -zxvf maui-*.tar.gz
cd maui-*
./configure
# --with-pbs=[/usr/local]  查找pbs-config或libpbs.a的路径
# --prefix=[/usr/local/maui]
# --with-spooldir=[/usr/local/maui]
make
make install
```