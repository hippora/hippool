+++
categories = ["postgresql"]
date = "2015-02-22T17:21:00+08:00"
description = ""
tags = ["postgres","schema"]
thumbnail = ""
title = "postgresql中schema概念"

+++

一个数据库中可以包含多个schema

<!--more-->

schema概念有点像命名空间或者把它想像成一个文件系统中的目录,差别就是这个schema下不能再有schema嵌套.
各个对象比如表,函数等存放在各个schema下,同一个schema下不能有重复的对象名字,但在不同schema下可以重复.

**使用schema的作用**

 - 方便管理多个用户共享一个数据库,但是又可以互相独立.
 - 方便管理众多对象,更有逻辑性
 - 方便兼容某些第三方应用程序,创建对象时是有schema的

> 比如要设计一个复杂系统.由众多模块构成,有时候模块间又需要有独立性.各模块存放单独的数据库显然是不合适的.
> 这时候使用schema来分类各模块间的对象,再对用户进行适当的权限控制.这样逻辑也非常清晰.

## 创建schema

```
db01=# create schema schema01;
CREATE SCHEMA
db01=# \dn
   List of schemas
   Name   |  Owner
----------+----------
 public   | postgres
 schema01 | postgres
(2 rows)
```

### 在schema中创建对象

```
db01=# create table schema01.t1(id int);
CREATE TABLE
db01=# insert into schema01.t1 values(1);
INSERT 0 1
db01=# select * from t1;
ERROR:  relation "t1" does not exist
LINE 1: select * from t1;
                      ^
db01=# select * from schema01.t1;
 id
----
  1
(1 row)
db01=# select * from db01.schema01.t1;
 id
----
  1
(1 row)
```

*默认是在public这个schema下.因此要带上schema名称.数据库名字如果要带上,只能是当前连接的数据库!!*

## 删除schema

```
db01=# drop schema schema01;
ERROR:  cannot drop schema schema01 because other objects depend on it
DETAIL:  table schema01.t1 depends on schema schema01
HINT:  Use DROP ... CASCADE to drop the dependent objects too.
db01=# drop schema schema01 cascade;
NOTICE:  drop cascades to table schema01.t1
DROP SCHEMA
```

*schema下有对象如果需要一起删除,需要带上cascade关键字.有点像使用rmdir删除目录一样,文件夹下有东西不然删除.*

## 创建schema指定owner

*默认是谁创建的schema,owner就是谁,当然也可以指定*

```
db01=# create schema s01 authorization hippo;
CREATE SCHEMA
db01=# create schema authorization hippo;
CREATE SCHEMA
db01=# \dn
  List of schemas
  Name  |  Owner
--------+----------
 hippo  | hippo
 public | postgres
 s01    | hippo
(3 rows)
```

*指定了owner,不指定schema,则schema名字与owner一致*

**在oracle中,默认创建用户的时候,创建了一个和用户名一样的schema下,并且互相绑定.这里有点类似**

## schema 的搜索路径(Search Path)

*如果每个schema下都有同名的对象(比如表t),我查询的是哪个schema下的表呢*

**pg下通过一个搜索路径参数来控制, search_path,我们先来模拟些数据**

```
db01=# \c - hippo
You are now connected to database "db01" as user "hippo".
db01=# create table public.t1(id text);
CREATE TABLE
db01=# insert into public.t1 values('public');
INSERT 0 1
db01=# create table s01.t1(id text);
CREATE TABLE
db01=# insert into s01.t1 values('s01');
INSERT 0 1
db01=# create table hippo.t1(id text);
CREATE TABLE
db01=# insert into hippo.t1 values('hippo');
INSERT 0 1
db01=> select * from pg_tables where tablename = 't1';
 schemaname | tablename | tableowner | tablespace | hasindexes | hasrules | hastriggers
------------+-----------+------------+------------+------------+----------+-------------
 public     | t1        | hippo      |            | f          | f        | f
 s01        | t1        | hippo      |            | f          | f        | f
 hippo      | t1        | hippo      |            | f          | f        | f
(3 rows)
```

测试一下

```
db01=> show search_path;
  search_path
----------------
 "$user",public
(1 row)

db01=> select * from t1;
  id
-------
 hippo
(1 row)

db01=> set search_path = 's01';
SET
db01=> select * from t1;
 id
-----
 s01
(1 row)
```

如果不在搜索路径里面,即使存在其他schema下,也会找不到.

```
db01=> drop table t1;
DROP TABLE
db01=> select * from t1;
ERROR:  relation "t1" does not exist
LINE 1: select * from t1;
                          ^
```

**创建的表存放哪个schema也跟search_path有关**

```
db01=> set search_path=ssssss,s01;
SET
db01=> create table t2(id int);
CREATE TABLE
db01=> \dt        --ssssss这个schema不存在
       List of relations
 Schema | Name | Type  | Owner
--------+------+-------+-------
 s01    | t2   | table | hippo
(1 row)

db01=> set search_path='';
SET
db01=> create table t3(id int);
ERROR:  no schema has been selected to create in
```

*跟你是哪个用户无关,现在知道为什么之前建的表为什么都是在public下了吧*

## schema 与权限

*只有login权限的用户,对于owner不是自己的schema是没有任何权限的*

```
db01=> \c - postgres
db01=# create schema s02;
CREATE SCHEMA
db01=# create role u02 login;
CREATE ROLE
db01=# create table s02.t1(id int);
CREATE TABLE
db01=# \c - u02
You are now connected to database "db01" as user "u02".
db01=> select * from s02.t1;
ERROR:  permission denied for schema s02
LINE 1: select * from s02.t1;
                      ^
db01=> create table s02.t2(id int);
ERROR:  permission denied for schema s02
```

*赋权usage权限*

```
db01=> \c - postgres
You are now connected to database "db01" as user "postgres".
db01=# grant usage on schema s02 to u02;
GRANT
db01=# \c - u02;
You are now connected to database "db01" as user "u02".
db01=> select * from s02.t1;
ERROR:  permission denied for relation t1
```

*注意,这里提示错误已经不一样了.是没有相应的查询权限*

```
db01=# grant select on all tables in schema s02 to u02;
GRANT
db01=# \c - u02
You are now connected to database "db01" as user "u02".
db01=> select * from s02.t1;
 id
----
(0 rows)
```

**系统默认将usage,create权限授权给了PUBLIC(所有用户),所以之前新建用户就可以创建表就是这个原因**

```
db01=> create table public.t2(id int);
CREATE TABLE
db01=# revoke usage,create on schema public from public;
REVOKE
db01=# \c - u02
You are now connected to database "db01" as user "u02".
db01=> \dt
No relations found.
db01=> select * fromt t2;
ERROR:  syntax error at or near "fromt"
LINE 1: select * fromt t2;
                 ^
```

## 系统schema, pg_catalog

**pg_catalog存放了各系统表,内置函数等等,它总是在搜索路径中,需要通过current_schemas看到**

```
db01=> \c - hippo
You are now connected to database "db01" as user "hippo".
db01=> show search_path;
  search_path
----------------
 "$user",public
(1 row)

db01=> select current_schemas(true);
      current_schemas
---------------------------
 {pg_catalog,hippo,public}
(1 row)

db01=> set search_path='';
SET
db01=> select current_schemas(true);
 current_schemas
-----------------
 {pg_catalog}
(1 row)
```

## schema 使用范例

```
db02=> \c - postgres
You are now connected to database "db02" as user "postgres".
db02=# drop schema public;
DROP SCHEMA
db02=# create role jack login;
CREATE ROLE
db02=# create role joe login;
CREATE ROLE
db02=# create schema authorization jack;
CREATE SCHEMA
db02=# create schema authorization joe;
CREATE SCHEMA
db02=# grant usage on schema jack to joe;
GRANT
db02=# grant usage on schema joe;
GRANT
db02=# grant select on all tables in schema jack to joe;
GRANT
db02=# grant select on all tables in schema joe to jack;
GRANT
```

*删除public这个schema,限制用户只在自己的schema下进行活动,防止在public下占用过多的空间*

*可以创建一些有限制权限的用户,比如只有增删改查权限的用户只能访问特定的schema,记得修改它的search_path*

*grant select on all tables in schema .. to .. 语句只影响当前schema下的表,之后创建的表不受影响,这个需要注意.*

//END

