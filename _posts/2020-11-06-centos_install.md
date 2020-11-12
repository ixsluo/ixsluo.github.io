---
layout: post
title: "Centos7 Installation Guildline"
author: ixsluo
date: 2020-11-06 +0800
tag:

---

<!--
# Centos7 Installation Guildline
-->

![centos](https://img.shields.io/badge/centos-installation-blue?logo=centos&logoColor=orange&labelColor=)

- [基本命令](#基本命令)
- [主机名](#主机名)
- [安全与远程登录](#安全与远程登录)
  - [SELinux](#selinux)
  - [firewall](#firewall)
  - [禁止普通用户su](#禁止普通用户su)
  - [SSH](#ssh)
- [chrony 同步时间](#chrony-同步时间)
- [软件安装](#软件安装)
  - [bash-completion](#bash-completion)
  - [Git](#git)
  - [Commitizen](#commitizen)
  - [Docker](#docker)
  - [MySQL](#mysql)

## 基本命令

管理服务

`systemctl [status|start|stop|restart|enable|disable] <servicename>`

网络

`ip [addr]`

终端提示音关闭

`setterm -blength n`

yum查找包及包含命令的包

`yum search <package>`

`yum provides <command>`

## 主机名

```bash
hostnamectl -h
set-hostname <hostname>
```

## 安全与远程登录

### SELinux

`sestatus`查看SELinux状态

- *enforcing*    强制模式，运行中，限制domian/type
- *permissive*   宽容模式，运行中，不限制但是警告
- *disabled*     关闭，没有运行

`getenforce` 查看当前模式

`setenforce [0|1]` 运行模式切换，0->宽容模式，1->强制模式

关闭SELinux需修改 */etc/selinux/config* -> `SELINUX=disabled` 并重启系统。

SELinux端口管理

```sh
semanage port -l  # 显示所有端口
semanage port [-a|-d] -t <type, [ssh_port_t]> -p tcp|udp <port number>  # 为指定策略增加或删除端口
```

`yum install [setools-console-3.3.8-4.el7.x86_64]` 安装`seinfo`、`sesearch`

### firewall

服务，`firewalld`，管理命令，`firewall-cmd`

```bash
firewall-cmd --get-active-zones  # 获取当前活动区域
firewall-cmd --get-service  # 获取所有支持的服务
firewall-cmd --reload  # 重新加载防火墙
```

部分参数
|  |  |
|:-|:-|
|--permanent     |永久加入规则，不影响当前运行状态|
|--zone=\<zone>  |对选择的区域进行操作|
|--list-services |列出当前启用的服务|
|--list-ports    |列出当前启用的端口|
|--add-service=  |启用服务|
|--add-port=[12345/tcp]  |启用端口|

添加富规则：`--add-rich-rule="rule family='ipv4' source address='192.168.0.4/24' service name='http' accept`

允许ip192.168.0.4/24访问http

移除富规则：`--remove-rich-rule="<above>"`

### 禁止普通用户su

```sh
cat "auth    required   pam_wheel.so  use_uid" >> /etc/pam.d/su
cat "SU_WHEEL_ONLY yes" >> /etc/login.defs
```

允许指定用户su
```sh
usermod -G wheel <username>
```

### SSH

配置文件 */etc/ssh/sshd.config*，[参数表](http://www.04007.cn/article/538.html)

```bash
rpm -qa | grep ssh  # 检查是否安装
yum install openssh-server  # yum安装
```

服务，`sshd`

## chrony 同步时间

```bash
rpm -qa | grep chrony
yum install chrony
```

服务，`chrony`，`start` -> `enable`

若开启防火墙，允许NTP服务

```bash
firewall-cmd --add-service=ntp --permanent
firewall-cmd --reload
```

## 软件安装

### bash-completion

命令自动补全增强

```bash
yum install bash-completion -y
```

### Git

`yum install git -y`

### Commitizen

Commitizen用于规范git提交信息。测试通过 `node=10/12`，需要 `npm>=6`

yum默认nodejs版本过低，建议手动安装
```bash
wget https://nodejs.org/dist/v14.15.0/node-v14.15.0-linux-x64.tar.xz
xz node-***.tar.xz
tar -xvf node-***.tar
mv node-*** /usr/local/node

cat >> /etc/profile.d/custom.sh << EOF
# set for nodejs
export NODE_HOME=/usr/local/node
export PATH=$NODE_HOME/bin:$PATH
EOF

source /etc/profile.d/custom.sh
node -v
npm -v
```

**本说明针对非node项目**
```
# 换淘宝源  --registry=https://registry.npm.taobao.org
npm install -g commitizen git-cz  # 全局安装
npm ls -g -depth=0  # 检查是否安装成功
```

- Angular格式
  - `npm install -g cz-conventional-changelog`
  - `echo '{ "path": "cz-conventional-changelog" }' > ~/.czrc`
- 自定义格式
  - `npm install -g cz-customizable`
  - `echo '{ "path": "cz-customization" }' > ~/.czrc`
  - `cp /usr/local/node/lib/node_modules/cz-customizable/cz-config-EXAMPLE.js /home/<user>/.cz-config.js ; chown ***`
  - 可根据模板文件修改提交提示

**使用`git cz -s`进行提交**

### Docker

`uname -a` 检查linux内核至少3.8，建议3.10以上

```bash
yum update [-y]
yum install yum-utils device-mapper-persistent-data lvm2 [-y]
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum list docker-ce --showduplicates | sort -r  # 查看所有版本
yum install [docker-ce-18.03.1.ce-1.el7.centos] [-y]
systemctl start docker ; systemctl enable docker
docker version  # 检查是否安装成功
```

```bash
docker pull <image>  # 抓取镜像
docker image ls  # 列出所有镜像
docker container ls
docker ps -a  # 列出所有容器

docker run <image> [--name <container>]  # 在容器中运行镜像
docker start|stop|restart|kill <container>  # 容器控制
docker logs <container>  # 容器日志
```

### MySQL

```sh
yum install mysql mysql-devel
```

服务端，`mariadb`或`mysql-server`
```sh
--------
yum install mariadb*  # to replace mysql-server
--------
weget http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
rpm -ivh mysql-community-release-el7-5.noarch.rpm
yum install mysql-community-server

vi /etc/systemd/system/mysql.service -> [service] Type=simple, root:root
注释/etc/my.cnf  pid-file
--------
```

编码配置
```sh
cat >> /etc/my.cnf << EOF
[mysql]
  default-character-set=utf8
EOF
```

```
semanage permissive -a mysqld_t
firewall-cmd --zone=public --permanent --add-port=3306/tcp
firewall-cmd --reload
```

启动服务
```sh
systemctl start mariadb/mysql
systemctl enable ...
mysql -u root
```

设置密码
```sh
> set password for 'root'@'localhost' =password('<password>') ;
> create DATABASE gogs ;
> insert into mysql.user(Host,User,Password) values("localhost","gogs",password("gogsmysql")) ;
```
