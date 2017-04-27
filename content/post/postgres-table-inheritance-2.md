+++
categories = ["postgresql"]
date = "2015-02-24T17:48:39+08:00"
description = ""
tags = ["inheritance","partition table"]
thumbnail = ""
title = "postgresql中表的继承及分区表(二)"

+++

第一篇了解了表的继承.这篇主要讲pg中的分区表

<!--more-->

## 概述

在PG中分区表是通过表的继承来实现的.
父表描述表结构.多个子表继承同一个父表,通过trigger或rule来实现对这分区表的各种DML操作.
详细操作我们通过一个例子来表示

## 创建分区表

这里我准备创建一个按照日期字段来分区的分区表.每个分区只存放一个月的数据.

### 创建主表(父表)

```
db01=> create table t_partition(
db01(> id int not null,
db01(> msg text,
db01(> log_date date not null default current_timestamp
db01(> );
CREATE TABLE
```

### 创建子表(子分区)

```
db01=> create table t_partition_s200501 () inherits (t_partition);
CREATE TABLE
db01=> create table t_partition_s200502 () inherits (t_partition);
CREATE TABLE
db01=> create table t_partition_s200503 () inherits (t_partition);
CREATE TABLE
```

*通常子表不添加额外的字段.因为应用只会对主表操作.*

### 添加约束

*每个子分区只存放相应一个月数据.通过添加check约束来限制存放正确的数据*

```
db01=> alter table t_partition_s200501 add check (log_date >= date'2015-1-1' and log_date < date'2015-2-1');
ALTER TABLE
db01=> alter table t_partition_s200502 add check (log_date >= date'2015-2-1' and log_date < date'2015-3-1');
ALTER TABLE
db01=> alter table t_partition_s200503 add check (log_date >= date'2015-3-1' and log_date < date'2015-4-1');
ALTER TABLE
```

> 这个约束在创建表的时候一起添加就可以了.

### 增加索引

*在需要或者合适的字段上添加索引,上一篇文章讲过,我们要单独在每个子分区表上添加相应的索引*

```
db01=> create index idx_t_partition_s200501_1 on t_partition_s200501(log_date);
CREATE INDEX
db01=> create index idx_t_partition_s200502_1 on t_partition_s200502(log_date);
CREATE INDEX
db01=> create index idx_t_partition_s200503_1 on t_partition_s200503(log_date);
CREATE INDEX
```

> 这步骤不是必须,比如每个子分区中所有数据都是同一个日期,添加这个索引反而影响性能
> 我们现在认为这些子分区会存放每天的数据,并且平均分布.so...性能问题以后再讲

### 创建触发器

**最常用的方法是创建触发器**

```
db01=> create or replace function fnc_t_partition_insert() returns trigger as $$
begin
if ( new.log_date >= date'2015-1-1' and new.log_date < date'2015-2-1') then
insert into t_partition_s200501 values (new.*);
elsif ( new.log_date >= date'2015-2-1' and new.log_date < date'2015-3-1') then
insert into t_partition_s200502 values (new.*);
elsif ( new.log_date >= date'2015-3-1' and new.log_date < date'2015-4-1') then
insert into t_partition_s200503 values (new.*);
else
raise exception 'Date out of range. Fix function fnc_t_partition_insert!!';
end if;
return null;
end;
$$ language plpgsql;
CREATE FUNCTION
db01=> create trigger tgr_t_partition_insert before insert on t_partition
for each row execute procedure fnc_t_partition_insert();
CREATE TRIGGER
```

### 测试分区表

**当我们将数据插入主表t_partition的时候,数据应该根据相应的日期,插入到相应的子分区表中去.**

```
db01=> insert into t_partition values (1,'msg1',date'2015-1-23');
INSERT 0 0
db01=> insert into t_partition values (2,'msg2',date'2015-2-13');
INSERT 0 0
db01=> insert into t_partition values (3,'msg3',date'2015-3-2');
INSERT 0 0
db01=> select tableoid,* from t_partition;
 tableoid | id | msg  |  log_date
----------+----+------+------------
    16565 |  1 | msg1 | 2015-01-23
    16572 |  2 | msg2 | 2015-02-13
    16579 |  3 | msg3 | 2015-03-02
(3 rows)

db01=> insert into t_partition values (4,'msg3',date'2015-4-2');
ERROR:  Date out of range. Fix function fnc_t_partition_insert!!
```

> 注意,这里的提示 insert 0 0,数据根据自定义的trigger,映射到相应的分区里去了

未完待续......

//END


