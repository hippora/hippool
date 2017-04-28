+++
categories = ["linux"]
date = "2015-07-30T10:19:21+08:00"
description = ""
tags = ["rsync"]
thumbnail = ""
title = "rsync配置"

+++

正好需要用到rsync.记录一下基本同步配置

<!--more-->

# rsync daemon 方式配置

## 实验机器

两台均为安装centos 6.6 的机器

- vm1:192.168.1.101  作为server
- vm2:192.168.1.102  作为client

## server端

### 安装

要安装最新版本的rsync,先删除已安装的

```
[root@vm1 ~]# rpm -e --nodeps rsync-3.0.6-12.el6.x86_64
[root@vm1 ~]# curl -O http://apt.sw.be/redhat/el6/en/x86_64/extras/RPMS/rsync-3.1.1-1.el6.rfx.x86_64.rpm
[root@vm1 ~]# rpm -ivh rsync-3.1.1-1.el6.rfx.x86_64.rpm
```

### 配置文件设置

```
[root@vm1 ~]# vi /etc/rsyncd.conf
uid = nobody
gid = nobody

use chroot = yes
max connections = 4
pid file = /var/run/rsyncd.pid
lock file = /var/run/rsync.lock
log file = /var/log/rsyncd.log

[media]
path = /www/media/
ignore errors
read only = false
list = false
hosts allow = 192.168.1.102/24
hosts deny = 0.0.0.0/32
auth users = rsync
secrets file = /etc/rsyncd.pwd
```

### 设置用户权限

```
[root@vm1 ~]# vi /etc/rsyncd.pwd
rsync:rsync12345
[root@vm1 ~]# chmod 600 /etc/rsyncd.pwd
```

### 设置开机自启动

```
[root@vm1 ~]# echo "/usr/bin/rsync --daemon" >> /etc/rc.d/rc.local
[root@vm1 ~]# /usr/bin/rsync --daemon
```

## client 端

### 设置访问密码文件

```
[root@vm2 ~]# vi rsync.pwd
syn12345
[root@vm2 ~]# chmod 600 rsync.pwd
```

### crontab 设置间隔

```
[root@vm2 ~]# crontab -e
*/5 * * * * rsync -azh --password-file=/root/rsync.pwd rsync@vm1::media /home/service/media/
```

## 测试

在添加crontab前可以进行测试

```
[root@vm2 ~]# rsync -azh --progress --password-file=/root/rsync.pwd rsync@10.254.191.101::media /home/service/media/
```

//END

