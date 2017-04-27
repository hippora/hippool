+++
categories = ["postgresql"]
date = "2015-02-24T17:44:52+08:00"
description = ""
tags = ["postgres","inheritance","table","partition"]
thumbnail = ""
title = "postgresql中表的继承及分区表(一)"

+++

PG实现了表的继承(inheritance),用好了这个特性.在数据库设计时能达到事半功倍的效果.

<!--more-->

## 表的继承

### 示例

```
db01=> create table t_parent(id int,name varchar(30));
CREATE TABLE
db01=> create table t_child(flag boolean) inherits(t_parent);
CREATE TABLE
```

#### 制造数据

```
db01=> insert into t_parent values(1,'hippo1');
INSERT 0 1
db01=> insert into t_parent values(2,'hippo2',true);
ERROR:  INSERT has more expressions than target columns
LINE 1: insert into t_parent values(2,'hippo2',true);
                                               ^
db01=> insert into t_child values(2,'hippo2',true);
INSERT 0 1
```

*insert不能通过继承来映射需要存放的表.只能准确指定某个表.*

#### 查询

```
db01=> select tableoid,* from t_parent;
 tableoid | id |  name
----------+----+--------
    16552 |  1 | hippo1
    16555 |  2 | hippo2
(2 rows)

db01=> select tableoid,* from t_child;
 tableoid | id |  name  | flag
----------+----+--------+------
    16555 |  2 | hippo2 | t
(1 row)
```

*通过tableoid可以知道这记录是那个表的*

**只查询t_parent表的数据**

```
db01=> select tableoid,* from t_parent*;    --*是默认,不写也可以
 tableoid | id |  name
----------+----+--------
    16552 |  1 | hippo1
    16555 |  2 | hippo2
(2 rows)

db01=> select tableoid,* from only t_parent;
 tableoid | id |  name
----------+----+--------
    16552 |  1 | hippo1
(1 row)
```

> 父表查询能看到所有子表的内容,通过only只查询本表内容.
> 参数sql_inheritance控制这一行为,建议不要动它

**修改父表结构**

```
db01=> alter table t_parent rename column name to nm;
ALTER TABLE
db01=> \d t_child
           Table "hippo.t_child"
 Column |         Type          | Modifiers
--------+-----------------------+-----------
 id     | integer               |
 nm     | character varying(30) |
 flag   | boolean               |
Inherits: t_parent
```

## 表继承总结

表继承需要注意的几点:

- 因为是独立的物理表,所以建在父表上的索引,主键,都不会出现在子表上.
- 正因为如此,父子表如果要建立外键关联,也是需要单独指明.
- 其他表要引用父表作为外键,不能引用子表的记录,至少现在版本还不支持(最新9.4.1)

//END

