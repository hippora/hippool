+++
#draft = true
date = "2017-03-28T17:22:25+08:00"
title = "ceph-install-1-prepare"
categories = ["Ceph"]
tags = ["ceph"]
author = "Hippo"
description = "ceph 安装前的一些准备工作"

+++

ceph 安装前的一些准备工作

<!--more-->

# 系统准备

## 简要说明

主机配置：
7台dell R720，16c32G。
每台2块200G ssd、3块4T硬盘，留3个硬盘位将来扩充。
2块ssd组raid1，除了装系统外，划6个10g的分区作为osd journal。
操作系统centos 7.3最小化安装。
6个千兆网卡，为了增加吞吐量和高可用做了team链路聚合。

ceph软件装kraken

> 为了节省成本，有些设置可能不太合理。

## 网络team配置

cluster network 做个team，210网段

```sh
nmcli c add type team con-name team0 ifname team0
nmcli c mod team0 ipv4.addresses 192.168.210.X/24 ipv4.gateway 192.168.210.1 ipv4.method manual 802-3-ethernet.mtu 9000
nmcli c add type team-slave con-name team0-p5p1 ifname p5p1 master team0
nmcli c add type team-slave con-name team0-p5p2 ifname p5p2 master team0
nmcli c add type team-slave con-name team0-em1 ifname em1 master team0
nmcli c add type team-slave con-name team0-em2 ifname em2 master team0
nmcli c up team0-em1
nmcli c up team0-em2
nmcli c up team0-p5p1
nmcli c up team0-p5p2
nmcli c up team0
```

public network 做个team，220网段

```sh
nmcli c add type team con-name team1 ifname team1
nmcli c mod team1 ipv4.addresses 192.168.220.X/24 ipv4.gateway 192.168.220.1 ipv4.method manual 802-3-ethernet.mtu 9000
nmcli c add type team-slave con-name team1-em3 ifname em3 master team1
nmcli c add type team-slave con-name team1-em4 ifname em4 master team1
nmcli c up team1-em3
nmcli c up team1-em4
nmcli c up team1
```

> 我这里有7台，都要设置一下

## 为了减轻工作量，装个ansible

```sh
[root@store01 ~]# yum install ansible -y
[root@store01 ~]# echo "store0[1:7]" >> /etc/ansible/hosts
```

## 更新hosts文件

```sh
vi /etc/hosts
192.168.220.101 store01
192.168.220.102 store02
...
[root@store01 ~]# ansible all -m copy -a "src=/etc/hosts dest=/etc/hosts" -k
```

## 安装epel等,并更新系统

```sh
[root@store01 ~]# ansible all -m shell -a "yum install -y epel-release deltarpm yum-plugin-priorities;yum -y update" -k
```

> 如换163源参考http://mirrors.163.com/.help/centos.html
## 关闭SELINUX

```sh
[root@store01 ~]# ansible all -m shell -a "sed -i '7s/=.*$/=disabled/' /etc/selinux/config;setenforce 0" -k
```

## 防火墙

```sh
[root@store01 ~]# ansible all -m shell -a "systemctl stop firewalld; systemctl disable firewalld" -k
```
> monitor 用6789通信，osd用6800-7300通信。ntpd使用25/udp

## 加快ssh登陆速度 

```sh
[root@store01 ~]# ansible all -m shell -a "sed -i 's/^#UseDNS.*/UseDNS no/' /etc/ssh/sshd_config;systemctl reload sshd" -k
```

## 安装ntp时间同步服务

```sh
[root@store01 ~]# ansible all -m yum -a "name=ntp,ntp-doc,ntpdate state=present" -k
[root@store01 ~]# ansible all -m shell -a "ntpdate 0.cn.pool.ntp.org;hwclock -w" -k
[root@store01 ~]# ansible all -m shell -a "date" -k
[root@store01 ~]# ansible all -m systemd -a "name=ntpd enabled=yes state=started" -k
```

## 创建部署ceph的用户

```sh
[root@store01 ~]# ansible all -m shell -a "useradd hpec;echo 'hpec:K2VKOxaXMm3D' | chpasswd" -k
[root@store01 ~]# ansible all -m shell -a "id hpec" -k
SSH password:
store01 | SUCCESS | rc=0 >>
uid=1000(hpec) gid=1000(hpec) groups=1000(hpec)
...
```
> 不要使用ceph预留的用户名`ceph` 

使`hpec`用户有sudo权限

```sh
[root@store01 ~]# echo "hpec ALL = (root) NOPASSWD:ALL" > /etc/sudoers.d/hpec
[root@store01 ~]# ansible all -m copy -a "src=/etc/sudoers.d/hpec dest=/etc/sudoers.d/hpec mode=0400" -k
```

## 终端（TTY）

```
visudo
#Defaults requiretty
```

> 注释掉这行，虽说ansible快，不过推荐使用visudo命令，那就每台机器执行一遍吧。

## osd journal分区

```sh
fdisk /dev/sdd # 每台osd机器都操作一下
[root@store01 ~]# ansible all -m shell -a "lsblk" -k
[root@store01 ~]# ansible all -m shell -a "chown ceph:ceph /dev/sdd{5,6,7,8,9,10}" -k
...
```
> 磁盘操作要检查仔细，一个osd对应一个osd journal盘
>
> ceph-deploy工具会自动格式化osd，这里就不格式化xfs了。
>
> 这里更改权限也是自己出了权限错误后加的，而且每次机器重启会还原成root权限，所以需要添加udev规则，详细查看后面的QA。


# 管理节点准备

> 我这里选用store01这台机器作为admin机器。

## 添加ceph源

```sh
[root@store01 ~]# cat << EOF >> /etc/yum.repos.d/ceph.repo
[ceph-noarch]
name=Ceph noarch packages
baseurl=http://download.ceph.com/rpm-kraken/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
EOF
```

## 安装ceph-deploy

```sh
[root@store01 ~]# yum install ceph-deploy -y
```

## 管理节点无密码登陆处理

``` sh
[root@store01 ~]# su - hpec
[hpec@store01 ~]$ ssh-keygen
[hpec@store01 ~]$ ssh-copy-id hpec@store01
[hpec@store01 ~]$ ssh-copy-id hpec@store02
[hpec@store01 ~]$ ssh-copy-id hpec@store03
...
```

> 因为 `ceph-deploy` 不支持输入密码


```sh
[hpec@store01 ~]$ cat << EOF >> ~/.ssh/config
Host store01
Hostname store01
User hpec
Host store02
Hostname store02
User hpec
Host store03
Hostname store03
User hpec
...
EOF
[hpec@store01 ~]$ chmod 400 .ssh/config
```

> 添加`~/.ssh/config`，可以简化ceph-deploy 每次都要输`--username hpec`

测试

```sh
[hpec@store01 ~]$ for i in {1..7}; do eval "ssh store0$i hostname -s";done;
store01
store02
...
```



> 机器环境准备好最好重启下

//END
