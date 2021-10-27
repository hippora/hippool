+++
categories = ["mysql"]
date = "2015-10-26T10:50:38+08:00"
description = ""
tags = ["mysql-config-editor"]
thumbnail = ""
title = "mysql免密码登录"

+++

mysql_config_editor 用来设置命令行免密码登录,,并且也防止密码写入脚本或环境变量而有暴露的风险..

<!--more-->

## 设置

```
$ mysql_config_editor set -u root -p
$ mysql_config_editor set -G dev -u root -h 192.168.1.204 -p
$ mysql_config_editor print --all

[client]
user = root
password = *****
[dev]
user = root
password = *****
host = 192.168.1.204
```

> -G 用来指定login-path,默认是client

## 连接测试

```
$ mysql    # 连接默认[client]
$ mysql --login-path=dev -e "status"     # 非默认
```

## .mylogin.cnf 文件

```
$ file .mylogin.cnf
.mylogin.cnf: data

$ ll .mylogin.cnf
-rw------- 1 hippo hippo 212 Oct 26 13:51 .mylogin.cnf

$ chmod g+x .mylogin.cnf
$ mysql
mysql: [Warning] /home/hippo/.mylogin.cnf should be readable/writable only by current user.
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)
```

> .mylogin.cnf 是二进制文件,权限必须是600

//END

