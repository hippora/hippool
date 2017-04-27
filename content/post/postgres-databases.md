+++
categories = ["postgresql"]
date = "2015-02-06T15:55:01+08:00"
description = ""
tags = ["database"]
thumbnail = ""
title = "postgresql中database的管理"

+++

学习postgresql中数据库的概念,什么是template databases,如何创建和删除database,database相关的配置及其他

<!--more-->

## PG中database的概念

不像oracle中一个或多个实例只管理一个DB,在PG中一个实例下有一个database cluster,可以管理多个DB,一个DB下有多个schema,schema下有各种数据库对象如table,index等等.

*有些系统视图比如pg_database等属于cluster层级*

## 创建DB

先检查下目前的数据库,2种方法

```
postgres=# select datname from pg_database;
  datname
-----------
 template1
 template0
 postgres
(3 rows)

postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(3 rows)
```

创建一个DB,不指定owner,默认的owner就是当前用户.

```
postgres=# create database db01;
CREATE DATABASE
postgres=# create database db02 owner hippo;
CREATE DATABASE
postgres=# \q
[postgres@fnddb ~]$ createdb -O hippo db03
[postgres@fnddb ~]$ psql
psql (9.4.0)
Type "help" for help.

postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 db01      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 db02      | hippo    | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 db03      | hippo    | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(6 rows)
```


## Template databases

### 建一点测试数据(已把之前的数据库删除)

```
postgres=# \c
You are now connected to database "postgres" as user "postgres".
postgres=# create role user1 login createdb;
CREATE ROLE
postgres=# create role user2 login createdb;
CREATE ROLE
postgres=# create database database1 owner user1;
CREATE DATABASE
postgres=# create database database2 owner user2;
CREATE DATABASE
```

### 测试下普通数据库作为模板

```
postgres=# \c postgres user1;
You are now connected to database "postgres" as user "user1".
postgres=> create database db01 template database1;
CREATE DATABASE
postgres=> create database db02 template database2;
ERROR:  permission denied to copy database "database2"
postgres=> \c database1 user1 --当前会话连接在上面也可以复制
You are now connected to database "database1" as user "user1".
database1=> create database db03 template database1;
CREATE DATABASE
```

只有owner和superuser可以复制普通数据库

### 测试有session连接着复制

新开个新session2:

```
[postgres@fnddb ~]$ psql -U hippo database1
```

在session1中执行:

```
database1=> create database db04 template database1;
ERROR:  source database "database1" is being accessed by other users
DETAIL:  There is 1 other session using the database.
```

有session连接着的数据库作为数据源不能复制

### 把数据库设置为模板

```
database1=> \c postgres postgres
You are now connected to database "postgres" as user "postgres".
postgres=# update pg_database set datistemplate='t' where datname='database2';
UPDATE 1
postgres=# \c database1 user1
You are now connected to database "database1" as user "user1".
database1=> create database db05 template database2;
CREATE DATABASE
```

设置成模板后,非owner用户也可以复制.

### 设置不让session登录

```
postgres=# select datname,datistemplate,datallowconn from pg_database;
  datname  | datistemplate | datallowconn
-----------+---------------+--------------
 template1 | t             | t
 template0 | t             | f
 postgres  | f             | t
 db01      | f             | t
 db03      | f             | t
 database1 | f             | t
 database2 | t             | t
 db05      | f             | t
(8 rows)

postgres=# update pg_database set datallowconn='f' where datname='database2';
UPDATE 1
postgres=# \c database2
FATAL:  database "database2" is not currently accepting connections
Previous connection kept
```

### 操作template1

```
postgres-# \c template1 postgres
You are now connected to database "template1" as user "postgres".
template1-# \d test_table
Did not find any relation named "test_table".
template1=# create table test_table(id varchar(10));
CREATE TABLE
template1=# insert into test_table values('test data');
INSERT 0 1
template1=# create database db06 ;
CREATE DATABASE
template1=# \c db06
You are now connected to database "db06" as user "postgres".
db06=# select * from test_table;
    id
-----------
 test data
(1 row)
```

模板中的对象及数据一起带过来了.

### 模板数据库总结

初始数据库有三个.postgres,template0,template1.(见上查询结果)
这是创建database cluster时创建的.

 - `create database` 默认使用template1来克隆新数据库,因此如果对template1进行了修改,修改就会出现在新建的数据库中.

 - template0与template1是一模一样的,只是官方建议不要动这个,原因是恢复pg_dump备份时,会基于此数据库来还原,并且如果template1被不小心弄乱了.还可以通过template0来再创建一个.
 - postgres是默认登录的数据库,也是初始时通过template1创建的.
 - 模板可以被任何有权限的用户复制,非模板数据库只能由owner和superuser复制.
 - 有用户连接的数据库不能作为数据源来复制
 - 可以通过修改pg_database中相应标志,来设置模板与否,能否连接,字符集等.

## 删除DB

```
db06=# \c postgres hippo;
You are now connected to database "postgres" as user "hippo".
postgres=> drop database db01;
ERROR:  must be owner of database db01
postgres=> \c postgres user1
You are now connected to database "postgres" as user "user1".
postgres=> drop database db01;
DROP DATABASE
```

数据库只能由owner或superuser删除,删除后所有此数据库下的对象一并删除不能撤销.

shell命令删除:

```
[postgres@fnddb ~]$ dropdb db06
```

//END


