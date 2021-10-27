+++
categories = ["postgresql"]
date = "2015-03-06T10:12:19+08:00"
description = ""
tags = ["composite type"]
thumbnail = ""
title = "postgresql数据类型之混合类型"

+++

Composite Type

<!--more-->

## 概述

这种有点像java中的class,c中的struct
oracle中的object类型或者在pl/sql中的record类型.

## 创建composite type 

我们测试使用一下pg中的这个数据类型

```
db01=# create type my_point as (x int,y int);
CREATE TYPE
db01=# create table my_position(id int,w my_point);
CREATE TABLE
db01=# \d my_position
   Table "hippo.my_position"
 Column |   Type   | Modifiers
--------+----------+-----------
 id     | integer  |
 w      | my_point |

db01=# \d my_point
Composite type "hippo.my_point"
 Column |  Type   | Modifiers
--------+---------+-----------
 x      | integer |
 y      | integer |
```

## 构造数据

```
db01=# insert into my_position values(1,row(10,20));
INSERT 0 1
db01=# insert into my_position values(2,(20,30));
INSERT 0 1
db01=# select * from my_position;
 id |    w
----+---------
  1 | (10,20)
  2 | (20,30)
```

> 两种方式都可以

## 访问成员

```
db01=# delete from my_position where w.y=30;
ERROR:  missing FROM-clause entry for table "w"
LINE 1: delete from my_position where w.y=30;
                                      ^
db01=# delete from my_position where (w).y=30;
DELETE 1
```

> 不加()的话,默认认为是表名,所以会失败

## 修改 

只修改其中一个成员

```
db01=# update my_position set w.x = (w).x + 1 where id = 1;
UPDATE 1
db01=# select * from my_position;
 id |    w
----+---------
  1 | (11,20)
(1 row)
```

> 虽然set后的w可以不用加(),但是强烈建议保持一致,加上().

整个记录一起修改

```
db01=# update my_position set w = row(1,2) where id = 1;
UPDATE 1
db01=# select * from my_position;
 id |   w
----+-------
  1 | (1,2)
(1 row)
```

> 不加关键字row也可以,但是推荐加上,表述更清晰

## 特别的表示

```
db01=# insert into my_position(id,w.x,w.y) values(1,100,200);
INSERT 0 1
db01=# select * from my_position where id = 1;
 id |     w
----+-----------
  1 | (1,2)
  1 | (100,200)
(2 rows)
```

> composite type 这时候每个成员可以像表字段一样列出来.

另外函数中使用composite type,访问方式也是一样.我觉得跟plsql中的record非常像.

//END


