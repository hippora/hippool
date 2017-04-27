+++
categories = ["postgresql"]
date = "2015-02-10T16:39:39+08:00"
description = ""
tags = ["pg_dump"]
thumbnail = ""
title = "postgresql备份恢复之pg_dump大数据处理"

+++

有时候数据库太大,操作系统又有文件大小限制,该如何处理

<!--more-->

## 介绍

官方文档介绍的主要有三种方式:

1. 通过unix管道,直接读取pg_dump的输出来压缩.
2. 使用pg_dump的custom-format
3. 使用pg_dump的directory-format

## 使用管道压缩和解压

**由于pg_dump工具可以输出到标准输出,可以使用unix管道来直接压缩**

*1.压缩方法*

```
[postgres@fnddb ~]$ pg_dump database1 | gzip > database1.sql.gz  
```

*还原*

```
[postgres@fnddb ~]$ createdb -T template0 testdb1
[postgres@fnddb ~]$ gunzip -c database1.sql.gz | psql testdb1
......
```

*2.切分*

*切分文件*

```
[postgres@fnddb ~]$ pg_dump database1 | split -b 1k - database1.sql
[postgres@fnddb ~]$ ll
total 16
-rw-rw-r--. 1 postgres postgres 1024 Feb 10 21:46 database1.sqlaa
-rw-rw-r--. 1 postgres postgres 1024 Feb 10 21:46 database1.sqlab
-rw-rw-r--. 1 postgres postgres   35 Feb 10 21:46 database1.sqlac
```
    
*还原*

```
[postgres@fnddb ~]$ cat database1.sql* | psql testdb1
```

## custom-format

记得编译安装时提示zlib库没有安装,custom-dump format会使用zlib库压缩输出文件  
当然这不是文本文件,恢复需要使用pg_restore命来来恢复.  

```
[postgres@fnddb ~]$ pg_dump -Fc database1 -f database1.c.dump
[postgres@fnddb ~]$ file database1.c.dump 
database1.c.dump: PostgreSQL custom database dump - v1.12-0
```

*还原*

```
[postgres@fnddb ~]$ createdb -T template0 testdb1
[postgres@fnddb ~]$ pg_restore -d testdb1 database1.c.dump 
```

**可以只恢复自己感兴趣的对象**

```
[postgres@fnddb ~]$ pg_restore -l database1.c.dump > database1.c.dump.list
[postgres@fnddb ~]$ vi database1.c.dump.list 
;
; Archive created at Tue Feb 10 22:02:23 2015
;     dbname: database1
;     TOC Entries: 16
;     Compression: -1
;     Dump Version: 1.12-0
;     Format: CUSTOM
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 9.4.1
;     Dumped by pg_dump version: 9.4.1
;
;
; Selected TOC Entries:
;
;2872; 1262 16411 DATABASE - database1 hippo
;6; 2615 2200 SCHEMA - public postgres
;2873; 0 0 COMMENT - SCHEMA public postgres
;2874; 0 0 ACL - public postgres
;175; 3079 12723 EXTENSION - plpgsql
;2875; 0 0 COMMENT - EXTENSION plpgsql
174; 1259 16544 TABLE public t3 hippo
;172; 1259 16512 TABLE public tab2 postgres
;2876; 0 0 ACL public tab2 postgres
;173; 1259 16524 TABLE public tab3 postgres
;2877; 0 0 ACL public tab3 postgres
2867; 0 16544 TABLE DATA public t3 hippo
;2865; 0 16512 TABLE DATA public tab2 postgres
;2866; 0 16524 TABLE DATA public tab3 postgres
```

*只需要将不需要恢复的对象使用;注释掉就可以了,然后还原带上此列表*

```
[postgres@fnddb ~]$ createdb -T template0 testdb2
[postgres@fnddb ~]$ pg_restore -L database1.c.dump.list -d testdb2 database1.c.dump
[postgres@fnddb ~]$ psql testdb2
psql (9.4.1)
Type "help" for help.

testdb2=# \dt
       List of relations
 Schema | Name | Type  | Owner 
--------+------+-------+-------
 public | t3   | table | hippo
(1 row)

testdb2=# select * from t3;
  id   
-------
 t3
 user3
(2 rows)
```
    
## directory-format

这种模式可以使用pg的多个工作进程.可以极大增强导出大数据库的效率.  
这种模式同一时间导出多个表.也就是并行导出,当然对服务器资源消耗也比较高,不要在高峰时刻进行.
它输出不是一个文件,需要首先创建一个目录  

```
[postgres@fnddb ~]$ mkdir db.dir
[postgres@fnddb ~]$ pg_dump -Fd -j4 -f db.dir database1
[postgres@fnddb ~]$ ls db.dir
2865.dat.gz  2866.dat.gz  2867.dat.gz  toc.dat
[postgres@fnddb ~]$ cd db.dir
[postgres@fnddb db.dir]$ cat 2866.dat.gz |gunzip 
\.


[postgres@fnddb db.dir]$ cat 2867.dat.gz |gunzip 
t3
user3
t3
user3
t3
user3
\.


[postgres@fnddb db.dir]$ file toc.dat
toc.dat: PostgreSQL custom database dump - v1.12-0
```

> 看结果因该是pg_dump导出了一个custom-format格式的数据库结构信息.  
> 同时将每一个表的数据分别导出到一个压缩文件.恢复使用copy命令可读取的格式
> -j参数指定同时几个进程来同时执行..每个进程同时只处理一个表的数据.

**恢复**

```
[postgres@fnddb ~]$ createdb -T template0 testdb3
[postgres@fnddb ~]$ pg_restore -d testdb3 -j4 db.dir
```

> -j参数指定并发的数量(job),pg_restore恢复custom-format格式也可以使用此参数,并非只适用directory-format

## 其他一些参数简单说明

**pg_dump和pg_restore很多参数都很类似.**

-n schema  |-N schema         --只导出(恢复)某个schema|排除某个schema,例子见-t table  
-t table | -T table          --导出(恢复)某个表|排除某个表

```
[postgres@fnddb ~]$ pg_dump -t t3 database1 | grep -i "create table"
CREATE TABLE t3 (
[postgres@fnddb ~]$ pg_dump -T t3 database1 | grep -i "create table"
CREATE TABLE tab2 (
CREATE TABLE tab3 (
```

--inserts         --导出成insert 语句,导入时性能很差,主要为了迁移到其他数据库使用

```
[postgres@fnddb ~]$ pg_dump database1 | grep -A3 -i "copy"
COPY t3 (id) FROM stdin;
t3
user3
t3
......
[postgres@fnddb ~]$ pg_dump --inserts database1 | grep -i "insert"
INSERT INTO t3 VALUES ('t3');
INSERT INTO t3 VALUES ('user3');
......
```
    
-a        --只导出数据,不导结构

```
[postgres@fnddb ~]$ pg_dump database1 | grep -i "create table"
CREATE TABLE t3 (
CREATE TABLE tab2 (
CREATE TABLE tab3 (
[postgres@fnddb ~]$ pg_dump -a database1 | grep -i "create table"
```

-c        --包括drop 对象的语句进去,最好附带--if-exists参数,否则导入时会报一些对象不存在的错误,虽然无关紧要

```
[postgres@fnddb ~]$ pg_dump database1 | grep -i drop 
[postgres@fnddb ~]$ pg_dump -c database1 | grep -i drop 
DROP TABLE public.tab3;
DROP TABLE public.tab2;
DROP TABLE public.t3;
DROP EXTENSION plpgsql;
DROP SCHEMA public;
```

-C        --包含创建数据库的语句

```
[postgres@fnddb ~]$ pg_dump database1 | grep -i "create database"
[postgres@fnddb ~]$ pg_dump -C database1 | grep -i "create database"
CREATE DATABASE database1 WITH TEMPLATE = template0 ENCODING = 'UTF8' LC_COLLATE = 'en_US.UTF-8' LC_CTYPE = 'en_US.UTF-8' TABLESPACE = ts01;
```

*其他参数可以通过官方文档或者pg_dump --help来查看*

//END

