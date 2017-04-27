+++
categories = ["postgresql"]
date = "2015-02-24T17:53:36+08:00"
description = ""
tags = ["inheritance","partition table"]
thumbnail = ""
title = "postgresql中表的继承及分区表(三)"

+++

这篇主要讲最简单的分区表管理

<!--more-->

## 分区表管理

按照时间来分区的分区表的典型使用场景是:

- 某一段时间之前的数据进行归档备份
- 时间要超过分区表最大限制时增添新的分区

*这样做的好处是,你的主表不管多久,数据量总是保持相同的水平.不会随着时间推移查询越来越慢*

### 添加新分区

```
db01=> create table t_partition_s200504 (check(log_date >= '2015-4-1' and log_date < '2015-5-1')) inherits (t_partition);
CREATE TABLE
db01=> create index idx_t_partition_s200504_1 on t_partition_s200504(log_date);
CREATE INDEX
```

### 修改触发器函数

```
db01=> create or replace function fnc_t_partition_insert() returns trigger as $$
db01$>     begin
db01$>     if ( new.log_date >= date'2015-2-1' and new.log_date < date'2015-3-1') then
db01$>     insert into t_partition_s200502 values (new.*);
db01$>     elsif ( new.log_date >= date'2015-3-1' and new.log_date < date'2015-4-1') then
db01$>     insert into t_partition_s200503 values (new.*);
db01$>     elsif ( new.log_date >= date'2015-4-1' and new.log_date < date'2015-5-1') then
db01$>     insert into t_partition_s200504 values (new.*);
db01$>     else
db01$>     raise exception 'Date out of range. Fix function fnc_t_partition_insert!!';
db01$>     end if;
db01$>     return null;
db01$>     end;
db01$>     $$ language plpgsql;
CREATE FUNCTION
```

> 如果应用还有可能会生产2015.1月份的数据,可以暂时不删除1月份的映射语句

### 解除旧分区的继承关系

```
db01=> alter table t_partition_s200501 no inherit t_partition;
ALTER TABLE
db01=> insert into t_partition values(4,'msg4',date'2015-4-12');
INSERT 0 0
db01=> select * from t_partition;
 id | msg  |  log_date
----+------+------------
  2 | msg2 | 2015-02-13
  3 | msg3 | 2015-03-02
  4 | msg4 | 2015-04-12
(3 rows)

db01=> \d t_partition_s200501
      Table "hippo.t_partition_s200501"
  Column  |  Type   |       Modifiers
----------+---------+------------------------
 id       | integer | not null
 msg      | text    |
 log_date | date    | not null default now()
Indexes:
    "idx_t_partition_s200501_1" btree (log_date)
Check constraints:
    "t_partition_s200501_log_date_check" CHECK (log_date >= '2015-01-01'::date AND log_date < '2015-02-01'::date)
```

> 解除继承后,查询t_partition表就不会出现t_partition_s200501的数据了.

### 普通表变成分区子表

**这种场景典型出现在数据仓库中**

 1. 每天或者每个月有一批数据需要加载进数据库,但是在处理完成前不希望用户能查询到数据
 2. 这时可以先创建一个普通表,使用批量方法加载数据,提高性能
 3. 对表添加索引,对表进行业务处理.
 4. 将表添继承关系,可以查询到正确的数据.

**示例开始**

```
db01=> create table t_partition_s200505(like t_partition including defaults including constraints);
CREATE TABLE
db01=> alter table t_partition_s200505 add check(log_date >= date'2015-5-1' and log_date < date'2015-6-1');
ALTER TABLE
```

> 为了提高批量加载性能,check约束可以加载数据后再添加,当然首先要确保数据满足条件.

......这里进行批量加载数据,进行数据加工,添加索引等等操作......

**将表添加进继承关系**

```
db01=> alter table t_partition_s200505 inherit t_partition;
ALTER TABLE
db01=> \d+ t_partition
                              Table "hippo.t_partition"
  Column  |  Type   |       Modifiers        | Storage  | Stats target | Description
----------+---------+------------------------+----------+--------------+-------------
 id       | integer | not null               | plain    |              |
 msg      | text    |                        | extended |              |
 log_date | date    | not null default now() | plain    |              |
Triggers:
    tgr_t_partition_insert BEFORE INSERT ON t_partition FOR EACH ROW EXECUTE PROCEDURE fnc_t_partition_insert()
Child tables: t_partition_s200502,
              t_partition_s200503,
              t_partition_s200504,
              t_partition_s200505
```

最后看需要对触发器函数的映射语句进行修改......

未完待续......

//END

