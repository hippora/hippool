+++
categories = ["postgresql"]
date = "2015-02-06T16:07:09+08:00"
description = ""
tags = ["postgres","tablespace"]
thumbnail = ""
title = "postgresql中表空间管理"

+++

表空间作为数据库对象存放的地方,对于数据库来说是个逻辑概念,它是文件系统中一个目录下的多个数据文件组成.
目录可以不存放在PGDATA下,但是是作为cluster的一个部分,如果备份需要一起备份.

<!--more-->

## 创建表空间

*只能使用superuser来创建表空间*

```
postgres=# create tablespace ts01 location '/var/lib/pgsql/tsdata';
ERROR:  directory "/var/lib/pgsql/tsdata" does not exist
postgres=# \! mkdir -p /var/lib/pgsql/tsdata --目录需要自己创建
postgres=# create tablespace ts01 location '/var/lib/pgsql/tsdata';
CREATE TABLESPACE
postgres=# create tablespace ts02 owner hippo location '/var/lib/pgsql/tsdata'; --同一个目录只能对应一个表空间
ERROR:  directory "/var/lib/pgsql/tsdata/PG_9.4_201409291" already in use as a tablespace
postgres=# \! mkdir /var/lib/pgsql/tsdata02
postgres=# create tablespace ts02 owner hippo location '/var/lib/pgsql/tsdata02'; --可以指定用户
CREATE TABLESPACE
```

两种方式查看

```
postgres=# select * from pg_tablespace;
  spcname   | spcowner | spcacl | spcoptions
------------+----------+--------+------------
 pg_default |       10 |        |
 pg_global  |       10 |        |
 ts01       |       10 |        |
 ts02       |    16384 |        |
(4 rows)

postgres=# \db
               List of tablespaces
    Name    |  Owner   |        Location
------------+----------+-------------------------
 pg_default | postgres |
 pg_global  | postgres |
 ts01       | postgres | /var/lib/pgsql/tsdata
 ts02       | hippo    | /var/lib/pgsql/tsdata02
(4 rows)
```

## 表空间使用

```
postgres=# \c database2 user2;
You are now connected to database "database2" as user "user2".
database2=> create table t1(id text);
CREATE TABLE
```

默认对ts01没有权限

```
database2=> create table t2(id text) tablespace ts01;
ERROR:  permission denied for tablespace ts01
database2=> \c - postgres
You are now connected to database "database2" as user "postgres".
database2=# alter database database2 set tablespace ts01; --连接在上面不能进行更改
ERROR:  cannot change the tablespace of the currently open database
database2=# \c postgres postgres
You are now connected to database "postgres" as user "postgres".
```

ts01成为database2默认表空间

```
postgres=# alter database database2 set tablespace ts01;
ALTER DATABASE
postgres=# \c database2 user2
You are now connected to database "database2" as user "user2".
database2=> create table t2(id text) tablespace ts01;
CREATE TABLE
```

授权表空间权限给某用户

```
postgres=# \c database2 user2;
You are now connected to database "database2" as user "user2".
database2=> create table t3(id text) tablespace ts02; --没有ts02权限
ERROR:  permission denied for tablespace ts02
database2=> \c postgres postgres
You are now connected to database "postgres" as user "postgres".
postgres=# grant create on tablespace ts02 to user2; --授权
GRANT
postgres=# \c database2 user2;
You are now connected to database "database2" as user "user2".
database2=> create table t3(id text) tablespace ts02;
CREATE TABLE
```

## 表空间对应与文件系统

PGDATA下的pg_tblspc目录下存放的是表空间的链接

```
[postgres@fnddb pg_tblspc]$ pwd
/var/lib/pgsql/data/pg_tblspc
[postgres@fnddb pg_tblspc]$ ll
total 0
lrwxrwxrwx. 1 postgres postgres 21 Feb  6 22:52 16423 -> /var/lib/pgsql/tsdata
lrwxrwxrwx. 1 postgres postgres 23 Feb  6 23:04 16425 -> /var/lib/pgsql/tsdata02
```

## 初始化cluster时默认的表空间说明

database cluster初始化时,默认创建两个表空间

- pg_global   --系统字典表都存这里
- pg_default  --template0,template1默认表空间,因此通过create database 不带tablespace参数创建的数据库都使用此表空间

//END


