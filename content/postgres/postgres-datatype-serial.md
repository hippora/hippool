+++
categories = ["postgresql"]
date = "2015-03-09T10:15:59+08:00"
description = ""
tags = ["serial"]
thumbnail = ""
title = "postgresql数据类型之Serial"

+++

一般来说,每个表都需要一个主键,主键的值一般是通过数据库的auto_increment或者使用插入sequence来保证每条记录分配唯一的值.
Oracle在12c中实现了表中序列的auto_increment,12c之前只能通过trigger来自己实现.
PG中有个数据类型serial.

<!--more-->

## 创建带serial的表 

创建和使用

```
db01=# create table t_t3(id serial,name varchar(30));
CREATE TABLE
db01=# insert into t_t3(name) values('aaa');
INSERT 0 1
db01=# insert into t_t3(name) values('aaa');
INSERT 0 1
db01=# insert into t_t3(name) select generate_series(1,10) || 'a';
INSERT 0 10
db01=# select * from t_t3;
 id | name
----+------
  1 | aaa
  2 | aaa
......
 11 | 9a
 12 | 10a
(12 rows)
```

> 只需要插入id的默认值,自动会帮你生成序列号所以下面的方式也可以

也可以通过关键字default

```
db01=# insert into t_t3 values(default,'bbb');
INSERT 0 1
db01=# select * from t_t3 where name = 'bbb';
 id | name
----+------
 13 | bbb
(1 row)
```

使用了serial类型的自动,不要主动插入,否则就会主键冲突

```
db01=# alter table t_t3 add constraint pk_t_t3 primary key(id);
ERROR:  could not create unique index "pk_t_t3"
DETAIL:  Key (id)=(1) is duplicated.
db01=# delete from t_t3 where name = 'a';
DELETE 1
db01=# alter table t_t3 add constraint pk_t_t3 primary key(id);
ALTER TABLE
```

## serial的内部

```
db01=# \ds
            List of relations
 Schema |    Name     |   Type   | Owner
--------+-------------+----------+-------
 hippo  | t_t3_id_seq | sequence | hippo
(1 row)
db01=# select currval('t_t3_id_seq');
 currval
---------
      13
(1 row)
```

serial内部其实帮助我们实现了:

1. 创建了一个sequence,名字是`表名_字段名_seq`
2. 设置id字段为NOT NULL DEFAULT nextval('此sequence')
3. 关联此sequence到相关字段.

我们自己来实现一下:

```
db01=# create table t4(id bigint,name varchar(30));
CREATE TABLE
db01=# create sequence seq_t4 increment by 2 start 1 cache 20 owned by t4.id;
CREATE SEQUENCE
db01=# select nextval('seq_t4');
       1

db01=# select nextval('seq_t4');
       3

db01=# alter table t4 alter id set not null,alter id set default nextval('seq_t4');
ALTER TABLE
db01=# insert into t4(name) values('a');
INSERT 0 1
db01=# insert into t4(name) values('a');
INSERT 0 1
db01=# select * from t4;
  5 | a
  7 | a
```

## 删除 

```
db01=# \ds
            List of relations
 Schema |    Name     |   Type   | Owner
--------+-------------+----------+-------
 hippo  | seq_t4      | sequence | hippo
 hippo  | t_t3_id_seq | sequence | hippo
(2 rows)

db01=# drop table t4;
DROP TABLE
db01=# \ds
            List of relations
 Schema |    Name     |   Type   | Owner
--------+-------------+----------+-------
 hippo  | t_t3_id_seq | sequence | hippo
(1 row)
```

> 因为这个sequence关联在表的字段上,所以删除表字段会一起删除
> 除非owner没有指定

## 补充 

***serial类型有3中,smallserial,serial,bigserial,底层分别对应smallint,integer,bigint,分别占2bit,4bit,8bit.***

查看sequence的属性

```
db01=# select * from t_t3_id_seq;
 sequence_name | last_value | start_value | increment_by |      max_value      | min_value | cache_value | log_cnt | is_cycled | is_called
---------------+------------+-------------+--------------+---------------------+-----------+-------------+---------+-----------+-----------
 t_t3_id_seq   |         13 |           1 |            1 | 9223372036854775807 |         1 |           1 |      22 | f         | t
(1 row)
```

//END


