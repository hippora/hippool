+++
categories = ["postgresql"]
date = "2015-02-24T18:09:00+08:00"
description = ""
tags = ["postgres","comparision"]
thumbnail = ""
title = "postgresql 逻辑和比较操作"

+++

主要还是记录觉得特别的地方或需要注意的地方.

<!--more-->

## 逻辑操作

需要注意跟null值的与或操作

```
db01=> select true and null,true or null;
 ?column? | ?column? 
----------+----------
          | t
(1 row)

db01=> select false and null,false or null;
 ?column? | ?column? 
----------+----------
 f        | 
(1 row)
```

## 比较操作

between 比较中,and两边的值是包括进去的.这里需要注意!

```
db01=> select 1 between 1 and 3;
 ?column? 
----------
 t
(1 row)

db01=> select 1 between 3 and 1;
 ?column? 
----------
 f
(1 row)
```

> 默认[3,1]是没有值包括在内的.可以通过关键字SYMMETRIC来忽略and两边的顺序,有时候在程序中不清楚具体值的大小时会有用.

```
db01=> select 1 between symmetric 3 and 1;
 ?column? 
----------
 t
(1 row)
```

大家知道null值是不能使用=来比较的

```
db01=> select null=null,null!=null;
 ?column? | ?column? 
----------+----------
          | 
(1 row)

db01=> with t1(id,msg) as (
values(1,'aaa'),(2,'bbb'),(null,'ccc'),(4,'ddd')
),t2(id,amount) as (
values(2,100),(null,200),(5,700)
)select * from t1,t2 where t1.id = t2.id;
 id | msg | id | amount 
----+-----+----+--------
  2 | bbb |  2 |    100
(1 row)
```

那如果正好需求需要将null相同的一并列呢??

一种方法是使用coalesce,(pg中没有nvl函数吧?!)

```
db01=> with t1(id,msg) as (
values(1,'aaa'),(2,'bbb'),(null,'ccc'),(4,'ddd')
),t2(id,amount) as (
values(2,100),(null,200),(5,700)
)select * from t1,t2 where coalesce(t1.id,-1) = coalesce(t2.id,-1);
 id | msg | id | amount 
----+-----+----+--------
  2 | bbb |  2 |    100
    | ccc |    |    200
(2 rows)
```

> 需要注意 -1 这个值不能存在id这个列中.否则结果会出错

还有一种语法: is [not] distinct from 

```
db01=> with t1(id,msg) as (
values(1,'aaa'),(2,'bbb'),(null,'ccc'),(4,'ddd')
),t2(id,amount) as (
values(2,100),(null,200),(5,700)
)select * from t1,t2 where t1.id is not distinct from t2.id;
 id | msg | id | amount 
----+-----+----+--------
  2 | bbb |  2 |    100
    | ccc |    |    200
(2 rows)
```

//END
