+++
categories = ["postgresql"]
date = "2015-02-24T17:35:26+08:00"
description = ""
tags = ["postgres","query"]
thumbnail = ""
title = "postgresql中的query(一)"

+++

主要还是学习一下一些特殊的语法,这里讲最普通的select

<!--more-->

查询命令格式

`[WITH with_queries] SELECT select_list FROM table_expression [sort_specification]`

## select

### 特殊语法1:字段别名

```
db01=> \d t_insert
         Table "hippo.t_insert"
 Column |  Type   |      Modifiers
--------+---------+----------------------
 id     | integer |
 name   | text    | default 'name'::text

db01=> select id as no,name from t_insert;
 no | name
----+-------
    ......
  9 | name
(7 rows)

db01=> select * from t_insert a(no);    --按照字段顺序,未重命名的按原名显示
 no | name
----+-------
    ......
  9 | name
(7 rows)
```

### 特殊语法2:可以不列任何字段名

```
db01=> select from t_insert;
--
(7 rows)
```

*只提示有7行数据,这种结果可能只在需要判断有没有符合条件的记录的时候才有用吧.*

### 特殊语法3:直接查询一个列表

```
db01=> select * from (values('jack',1),('joe',2),('hippo',3)) as t;    --必须要有一个别名,否则出错,默认字段名叫column1......
 column1 | column2
---------+---------
 jack    |       1
 joe     |       2
 hippo   |       3
(3 rows)

db01=> select * from (values('jack',1),('joe',2),('hippo',3)) as t(name,id);
 name  | id
-------+----
 jack  |  1
 joe   |  2
 hippo |  3
(3 rows)
```

### 特殊语法4:查询列表无需select

```
db01=> values(1,2),(2,3);
 column1 | column2
---------+---------
       1 |       2
       2 |       3
(2 rows)
```

### 特殊语法5:整行空的表达

```
db01=> select * from t_insert where t_insert is null;
 id | name
----+------
    |
(1 row)
```

*不必像oracle中一个字段一个字段列出来.*

### 特殊语法6:limit和offset

```
db01=> select * from t_insert limit 3;
 id | name
----+-------
  1 | hippo
  2 | jack
  3 | joe
(3 rows)

db01=> select * from t_insert limit 3 offset 2;    --offset优先
 id | name
----+-------
  3 | joe
  4 | alice
    |
(3 rows)
```

*在oracle 12c中开始支持这种语法,pg很多功能都很先进啊*
*offset跳过的行,pg内部还是计算的,所以优化是算不上的*

//END

