+++
categories = ["postgresql"]
date = "2015-02-10T16:29:22+08:00"
description = ""
tags = ["pg_dump"]
thumbnail = ""
title = "postgresql备份与恢复之SQL Dump"

+++

数据是很宝贵的,要时刻谨记备份的重要性.
这里讲一下通过SQL Dump方式来做备份与恢复.
pg_dump 导出某一个数据库,通过将数据库中的结构信息及数据通过sql方式输出来备份数据库.它是在执行命令那一刻时数据库一致性状态的保存.
恢复时只许将这输出在目标库上重建就可以了.

跟mysqldump是类似的

<!--more-->

## 使用pg_dump命令备份

pg_dump 默认输出到控制台,不指定参数默认是导出连接着的数据库.

```
[postgres@fnddb data]$ pg_dump | more
--
-- PostgreSQL database dump
--

SET statement_timeout = 0;
SET lock_timeout = 0;
......
--
-- PostgreSQL database dump complete
--
```

通常的做法是备份到一个文件中.

```
[postgres@fnddb ~]$ pg_dump database1 > db1.dump
```

可以导出一个schema,当然也可以只导出一个表

```
[postgres@fnddb ~]$ pg_dump database2 -n schema01

--
-- PostgreSQL database dump
--

SET statement_timeout = 0;
......
ALTER SCHEMA schema01 OWNER TO postgres;

--
-- PostgreSQL database dump complete
--
[postgres@fnddb ~]$ pg_dump -U user2 database2 -t t1 > database2.dump
```

pg_dump是一个客户端命令行工具,可以通过pg_dump --help查看

## 恢复pg_dump出来的文件.

**使用psql dbname < pgdumpfile**

> dbname是使用template0创建出来的全新数据库,否则有可能已经有对象在其中.

**先查看下要备份的数据库中有哪些表,属于哪些用户**

```
[postgres@fnddb ~]$ psql database1
psql (9.4.1)
Type "help" for help.

database1=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | user1
 public | t2   | table | user1
 public | t3   | table | user2
 public | tab2 | table | postgres
 public | tab3 | table | postgres
(5 rows)
```

**备份数据库database1**

```
[postgres@fnddb ~]$ pg_dump database1 > database1.pgdmp
```

**使用template0创建一个全新数据库**

```
[postgres@fnddb ~]$ createdb -T template0 db01
```

**进行恢复并检查**

```
[postgres@fnddb ~]$ psql db01 < database1.pgdmp
SET
SET
SET
......
[postgres@fnddb ~]$ psql db01
psql (9.4.1)
Type "help" for help.

db01=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | user1
 public | t2   | table | user1
 public | t3   | table | user2
 public | tab2 | table | postgres
 public | tab3 | table | postgres
(5 rows)

db01=# select * from t1;
  id
-------
 user1
(1 row)
```

*对象权限数据都还原了*

## 测试一下如果某个用户不存在会如何

**删除user2,需要把user2涉及的对象权限删除或者授权给其他用户**

```
database1=# drop role user1;
ERROR:  role "user1" cannot be dropped because some objects depend on it
DETAIL:  owner of database database1
2 objects in database db01
2 objects in database database2
database1=# reassign owned by user2 to hippo;
REASSIGN OWNED
database1=# \c database2
You are now connected to database "database2" as user "postgres".
database2=# reassign owned by user2 to hippo;
REASSIGN OWNED
database2=# \c db01
You are now connected to database "db01" as user "postgres".
db01=# reassign owned by user2 to hippo;
REASSIGN OWNED
db01=# revoke create on tablespace ts02 from user2;
REVOKE
db01=# alter database database1 owner to hippo;
ALTER DATABASE
db01=# drop role user2;
DROP ROLE
db01=# \q
```

**尝试恢复**

```
[postgres@fnddb ~]$ createdb -T template0 db02
[postgres@fnddb ~]$ psql db02 < database1.pgdmp
SET
......
ERROR:  role "user2" does not exist
......
[postgres@fnddb ~]$ psql db02
psql (9.4.1)
Type "help" for help.

db02=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | user1
 public | t2   | table | user1
 public | t3   | table | postgres
 public | tab2 | table | postgres
 public | tab3 | table | postgres
(5 rows)
```

*授权语句由于没有user2所以失败了,t3 owner变成了postgres,也就是使用psql连接的用户*

**可以通过参数ON_ERROR_STOP=on来停止继续执行恢复命令**

```
[postgres@fnddb ~]$ createdb -T template0 db03
[postgres@fnddb ~]$ psql --set ON_ERROR_STOP=on db03 < database1.pgdmp
SET
......
ERROR:  role "user2" does not exist
[postgres@fnddb ~]$
```

## 通过管道直接备份恢复数据库

*通过指定-1选项,可以将恢复进程放在一个事务中*

```
[postgres@fnddb ~]$ hostname -i
192.168.10.74
[postgres@fnddb ~]$ psql -h 192.168.10.72
Password:
psql (9.4.1, server 9.3.5)
Type "help" for help.

postgres=# create database db01 template template0;
CREATE DATABASE
postgres=# \q
[postgres@fnddb ~]$ pg_dump database1 | psql -1 -h 192.168.10.72 db01
Password:
SET
......
GRANT
ERROR:  role "r2" does not exist
ERROR:  current transaction is aborted, commands ignored until end of transaction block
ERROR:  current transaction is aborted, commands ignored until end of transaction block
ERROR:  current transaction is aborted, commands ignored until end of transaction block
ERROR:  current transaction is aborted, commands ignored until end of transaction block
[postgres@fnddb ~]$ psql -h 192.168.10.72 db01
Password:
psql (9.4.1, server 9.3.5)
Type "help" for help.

db01=# \dt
No relations found.
```

**把需要的role创建好再试一次**

```
db01=# create role user1;
CREATE ROLE
db01=# create role user2;
CREATE ROLE
db01=# create role r1;
CREATE ROLE
db01=# create role r2;
CREATE ROLE
db01=# create role jack;
CREATE ROLE
db01=# \q
[postgres@fnddb ~]$ pg_dump database1 | psql -1 -h 192.168.10.72 db01
Password:
SET
.......
[postgres@fnddb ~]$ psql -h 192.168.10.72 db01
Password:
psql (9.4.1, server 9.3.5)
Type "help" for help.

db01=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t3   | table | hippo
 public | tab2 | table | postgres
 public | tab3 | table | postgres
(3 rows)
```

## pg_dumpall命令

pg_dump只导出一个数据库,并且不导出role,tablespace等cluster级别的对象.
pg_dumpall导出所有包括role,tablespace及所有数据库.
pg_dumpall因为要导出role,tablespace信息,所以此命令需要superuser用户

### pg_dumpall备份全库

```
[postgres@fnddb ~]$ pg_dumpall > all.pgdmp
```

*只导出Cluster-wide数据,不包括数据库级别的对象及数据*

```
[postgres@fnddb ~]$ pg_dumpall -g > all.pgdmp
```

可以再使用pg_dump来导出单独一个数据库来使用

### 恢复pg_dumpall备份的文件

*恢复使用psql -f infile来进行,最好是使用一个全新的cluster来进行恢复*

```
[postgres@fnddb ~]$ pg_dumpall | psql -h 192.168.10.72 -f -
Password:
SET
......
psql:<stdin>:34: ERROR:  directory "/var/lib/pgsql/tsdata" does not exist
psql:<stdin>:35: ERROR:  directory "/var/lib/pgsql/tsdata02" does not exist
psql:<stdin>:36: ERROR:  tablespace "ts02" does not exist
psql:<stdin>:37: ERROR:  tablespace "ts02" does not exist
psql:<stdin>:38: ERROR:  tablespace "ts02" does not exist
psql:<stdin>:45: ERROR:  tablespace "ts01" does not exist
psql:<stdin>:46: ERROR:  database "database1" does not exist
psql:<stdin>:47: ERROR:  database "database1" does not exist
psql:<stdin>:48: ERROR:  database "database1" does not exist
psql:<stdin>:49: ERROR:  database "database1" does not exist
psql:<stdin>:50: ERROR:  tablespace "ts01" does not exist
CREATE DATABASE
CREATE DATABASE
CREATE DATABASE
REVOKE
REVOKE
GRANT
GRANT
psql:<stdin>:60: \connect: FATAL:  database "database1" does not exist
```

*在目标库上的表空间目录需要事先创建好*
然后再重建...

## 注意点

pg_dump,pg_dumpall是备份恢复某一个时刻时,数据库的一致性状态..
如果需要将数据库从崩溃或丢失数据中,恢复到运行时最新的状态,则需要使用在线备份恢复技术.

//END

