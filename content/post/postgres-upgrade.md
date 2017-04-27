+++
categories = ["postgresql"]
date = "2015-02-06T16:13:36+08:00"
description = ""
tags = ["upgrade"]
thumbnail = ""
title = "postgresql升级-小版本号升级"

+++

今天发现更新了9.4.1,正好更新一下
根据官方文档,小版本号不需要做复杂操作,只需要用新的二进制文件替换掉原来的.
数据等不需要操作.
大版本号升级要复杂一点,等新版本出来后再记录吧.

<!--more-->

## 使用postgres用户停止数据库

```
[postgres@fnddb data]$ pg_ctl stop
waiting for server to shut down.... done
server stopped
```

## 下载源码并编译安装

```
[root@fnddb ~]# wget https://ftp.postgresql.org/pub/source/v9.4.1/postgresql-9.4.1.tar.bz2
[root@fnddb ~]# tar -xjvf postgresql-9.4.1.tar.bz2
[root@fnddb ~]# cd postgresql-9.4.1
[root@fnddb postgresql-9.4.1]# ./configure
[root@fnddb postgresql-9.4.1]# make
[root@fnddb postgresql-9.4.1]# make install
...
```

## 使用postgres用户启动数据库

```
[postgres@fnddb ~]$ pg_ctl start -l /var/lib/pgsql/pgsql.log
```

检查一下版本

```
[postgres@fnddb ~]$ postgres -V
postgres (PostgreSQL) 9.4.1
```

东西还在 ;)

```
[postgres@fnddb ~]$ psql -U user1 database1
psql (9.4.1)
Type "help" for help.

database1=> select * from t1;
 id
----
(0 rows)

database1=> \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 database1 | user1    | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 database2 | user2    | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(5 rows)
```

//END

