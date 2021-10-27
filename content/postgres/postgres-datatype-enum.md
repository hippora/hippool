+++
categories = ["postgresql"]
date = "2015-03-01T09:32:06+08:00"
description = ""
tags = ["enum"]
thumbnail = ""
title = "postgresql数据类型之枚举类型及相关函数"

+++

枚举类型及相关函数

<!--more-->

## enum的使用 

```
db01=> create type enum_clor as enum ('red','black','green','white','blue');
CREATE TYPE
db01=> create table t_pencil(id int,clor enum_clor);
CREATE TABLE
db01=> insert into t_pencil values(1,'red');
INSERT 0 1
db01=> insert into t_pencil values(1,'yellow');
ERROR:  invalid input value for enum enum_clor: "yellow"
LINE 1: insert into t_pencil values(1,'yellow');
                                      ^
db01=> select * from t_pencil;
  1 | red
```

> enum是一种有顺序的集合(set).

我们测试下排序

```
db01=> insert into t_pencil values(1,'red');
INSERT 0 1
db01=> insert into t_pencil values(2,'white');
INSERT 0 1
db01=> insert into t_pencil values(3,'blue');
INSERT 0 1
db01=> insert into t_pencil values(4,'black');
INSERT 0 1
db01=> insert into t_pencil values(5,'green');
INSERT 0 1
db01=> select * from t_pencil order by clor;
  1 | red
  4 | black
  5 | green
  2 | white
  3 | blue
```

> enum的顺序就是创建type时指定的,而非按照字符串或数字大小排

## enum 类型安全 

我们再建个enum并建一个表

```
db01=> create type enum_clor1 as enum ('red','black','green','white','blue');
CREATE TYPE
db01=> create table t_paper(id int,clor enum_clor1);
CREATE TABLE
db01=> insert into t_paper values(1,'red');
INSERT 0 1
db01=> insert into t_paper values(2,'blue');
INSERT 0 1
db01=> select * from t_paper;
  1 | red
  2 | blue
```

测试

```
db01=> select * from t_pencil c,t_paper p where c.clor = p.clor;
ERROR:  operator does not exist: enum_clor = enum_clor1
LINE 1: select * from t_pencil c,t_paper p where c.clor = p.clor;
                                                        ^
HINT:  No operator matches the given name and argument type(s). You might need to add explicit type casts.
db01=> select * from t_pencil c,t_paper p where c.clor::text = p.clor::text;
  3 | blue |  2 | blue
  1 | red  |  1 | red
```

> 不同enum类型是不能直接比较的,即使字符串一样.除非显式转换或者自定义操作符.

## enum相关函数 

```
db01=> select enum_first('black'::enum_clor),enum_first(null::enum_clor);
 red        | red

db01=> select enum_last('black'::enum_clor),enum_last(null::enum_clor);
 blue      | blue

db01=> select enum_range('black'::enum_clor),enum_range(null::enum_clor);
 {red,black,green,white,blue} | {red,black,green,white,blue}
```

两个参数的enum_range

```
db01=> select enum_range(null,'white'::enum_clor);
 {red,black,green,white}

db01=> select enum_range('black'::enum_clor,null);
 {black,green,white,blue}

db01=> select enum_range('black'::enum_clor,'white'::enum_clor);
 {black,green,white}
```

> enum_first返回enum的第一个枚举值
> enum_last返回enum的最后一个枚举值
> enum_range返回enum的所有枚举值数组按顺序排列
> enum_range 两个入参的,则返回符合条件的枚举类型数组.
> 除了两个参数enum_range,其他几个函数的入参忽略具体的枚举值,只关心枚举类型

//END


