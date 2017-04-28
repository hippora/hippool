+++
categories = ["mysql"]
date = "2015-10-29T10:53:12+08:00"
description = ""
tags = ["user"]
thumbnail = ""
title = "mysql用户管理"

+++

mysql 5.7 用户管理

<!--more-->

## 创建用户

```sh
mysql> create user 'hippo'@'192.168.1.100' identified by 'Mysql#1234';
Query OK, 0 rows affected (0.02 sec)

mysql> create user 'hippo'@'192.168.2.%' identified by 'Mysql#1234';
Query OK, 0 rows affected (0.00 sec)

mysql> create user 'hippo'@'192.168.10.0/255.255.255.0' identified by 'Mysql#1234';
Query OK, 0 rows affected (0.00 sec)
```

> mysql下`user@host`是个整体,省略`@host`相当于`user@'%'`,host也可以是域名
> host代表允许连接过来的地址或域名


```sh
mysql> select user,host from mysql.user where user='hippo';
+-------+----------------------------+
| user  | host                       |
+-------+----------------------------+
| hippo | %                          |
| hippo | 192.168.1.100              |
| hippo | 192.168.10.0/255.255.255.6 |
| hippo | 192.168.2.%                |
+-------+----------------------------+
4 rows in set (0.00 sec)

mysql> show create user hippo@'192.168.2.%'\G
*************************** 1. row ***************************
CREATE USER for hippo@192.168.2.%: CREATE USER 'hippo'@'192.168.2.%' IDENTIFIED WITH 'mysql_native_password' AS '*29E098A2FCFC2ED4D61C4BB6E70EE40E09331D03' REQUIRE NONE PASSWORD EXPIRE DEFAULT ACCOUNT UNLOCK
1 row in set (0.00 sec)

```

## 删除用户

```sh
mysql> drop user hippo;
Query OK, 0 rows affected (0.00 sec)

mysql> select user,host from mysql.user where user='hippo';
+-------+----------------------------+
| user  | host                       |
+-------+----------------------------+
| hippo | 192.168.1.100              |
| hippo | 192.168.10.0/255.255.255.6 |
| hippo | 192.168.2.%                |
+-------+----------------------------+
3 rows in set (0.00 sec)
```

## 修改密码

```sh
mysql> alter user hippo@'192.168.1.100' identified by 'Mysql%1234';
Query OK, 0 rows affected (0.00 sec)

mysql> set password for hippo@'192.168.1.100' = password('Mysql%1234');
Query OK, 0 rows affected, 1 warning (0.00 sec)
```

> mysql 5.6.7开始已经废弃了set password命令,`alter user`命令也保持了跟oracle命令一致.

## 密码过期

设置用户密码立即过期:

```sh
mysql> alter user hippo password expire;
Query OK, 0 rows affected (0.00 sec)
```

设置用户密码过期时间:

```sh
mysql> alter user hippo password expire interval 30 day;
mysql> alter user hippo password expire never;
mysql> alter user hippo password expire default;

```

> default 值由系统变量`default_password_lifetime`控制,默认180天,可进行修改

过期后必须进行修改后才能继续执行

```sh
mysql> select 1;
ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.
mysql> alter user user() identified by 'Mysql!234';
Query OK, 0 rows affected (0.00 sec)

mysql> select 1;
+---+
| 1 |
+---+
| 1 |
+---+
1 row in set (0.00 sec)
```

_相关信息都存放在`mysql.user`表中,当然官网建议不要直接操作`mysql.user`表._
_另外网上文章中很多有事没事`flush privileges`,其实只有直接操作`mysql.user`表时才需要将新修改的信息刷到内存中,而通过`alter user`命令是不用执行此命令滴_

## 锁定账户

```sh
mysql> alter user hippo account lock;
```

登录会有错误提示

```sh
# mysql -uhippo -p
Enter password:
ERROR 3118 (HY000): Access denied for user 'hippo'@'localhost'. Account is locked.
```

解锁:

```sh
mysql> alter user hippo account unlock;
```

> `create user`命令也适用,总之越来越像oracle了


//END

