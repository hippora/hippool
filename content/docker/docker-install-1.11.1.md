+++
categories = ["docker"]
date = "2016-05-20T11:17:37+08:00"
description = ""
tags = ["install"]
thumbnail = ""
title = "docker安装"

+++

docker 1.11.1

<!--more-->

## 要求

安装在centos7上,64bit,内核至少3.10

```
[root@srv00 ~]# uname -r
3.10.0-229.14.1.el7.x86_64
```

## 安装

```
[root@srv00 ~]# curl -fsSL https://get.docker.com/ | sh
...
[root@srv00 ~]# docker -v
Docker version 1.11.1, build 5604cbe
[root@srv00 ~]# yum list installed | grep docker
docker-engine.x86_64             1.11.1-1.el7.centos          @docker-main-repo
docker-engine-selinux.noarch     1.11.1-1.el7.centos          @docker-main-repo
```

### 设置服务自启动

```
[root@srv00 ~]# systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
[root@srv00 ~]# systemctl start docker
```

测试一下

```
[root@srv00 ~]# docker run hello-world
```

## 给用户docker权限

默认只有root可以运行docker命令.要让普通用户可以执行docker命令.需要添加docker组.

```
[root@srv00 ~]# useradd hippo -G docker
```

> 如没有docker组,groupadd添加,并重启docker进程

## 卸载

```
[root@srv00 ~]# yum list installed | grep docker
[root@srv00 ~]# yum remove docker*
[root@srv00 ~]# rm -rf /var/lib/docker
```

//END

