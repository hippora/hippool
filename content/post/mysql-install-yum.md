+++
categories = ["mysql"]
date = "2015-10-22T10:39:29+08:00"
description = ""
tags = ["install"]
thumbnail = ""
title = "mysql 5.7.9 安装"

+++

centos 7 下 mysql 5.7.9 安装

<!--more-->

linux 环境: centos7

mysql : 5.7.9

## 安装yum 源

地址: http://dev.mysql.com/downloads/repo/yum/


```
# curl -L http://dev.mysql.com/get/mysql57-community-release-el7-7.noarch.rpm -o mysql57-community-release-el7-7.noarch.rpm
# yum localinstall -y mysql57-community-release-el7-7.noarch.rpm
# yum install mysql-community-server
# systemctl enable mysqld
# systemctl start mysqld
```

## 安全加固

```
## grep "password" /var/log/mysqld.log

2015-10-22T07:03:09.444960Z 1 [Note] A temporary password is generated for root@localhost: >5qrIfL)cRaE

## mysql_secure_installation

```

> 安装时默认会有一个临时密码
> 设置密码需要至少1个数字,1个特殊字符,混合大小写,长度至少8位


//END

