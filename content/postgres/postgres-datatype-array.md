+++
categories = ["postgresql"]
date = "2015-03-02T09:37:58+08:00"
description = ""
tags = ["array"]
thumbnail = ""
title = "postgresql数据类型之数组类型"

+++

数组array及相关的特性

<!--more-->

## 数组字段类型 

``` 
db01=# create table t_array(id int,array_1 int[],array_2 text[][]);
CREATE TABLE
db01=# \d t_array
      Table "hippo.t_array"
 Column  |   Type    | Modifiers
---------+-----------+-----------
 id      | integer   |
 array_1 | integer[] |
 array_2 | text[]    |
``` 

> 虽然我指定了一个是一维数组,一个是二维数组,但是表结构显示的都是一维数组
> pgadmin中看结构,array_2显示是[][].

## 数组构造插入 

我们插入数据测试一下

``` 
db01=# insert into t_array values(1,'{1,2,3}','{"aaa","bbb","ccc"}');
INSERT 0 1
db01=# insert into t_array values(1,'{{1,2,3},{4,5,6},{7,8,9}}','{{"aaa","bbb","ccc"},{"ddd","eee","fff"},{"ggg","hhh","iii"}}');
INSERT 0 1
db01=# select * from t_array;
 id |          array_1          |                   array_2
----+---------------------------+---------------------------------------------
  1 | {1,2,3}                   | {aaa,bbb,ccc}
  1 | {{1,2,3},{4,5,6},{7,8,9}} | {{aaa,bbb,ccc},{ddd,eee,fff},{ggg,hhh,iii}}
(2 rows)
``` 

> 不管一维二维几维都可以插入数据

使用array构造器语法

``` 
db01=# insert into t_array values(2,array[1,2,3,4],array[['a','b'],['c','d']]);
INSERT 0 1
``` 

*即使申明的是一维数组,也可以插入多维数组*

``` 
db01=# insert into t_array values(3,array[[[1,2],[3,4]],[[5,6],[6,7]],[[8,9],[9,10]]],array['a','b','c']);
INSERT 0 1
db01=# select * from t_array;
 id |                   array_1                    |                   array_2
----+----------------------------------------------+---------------------------------------------
  1 | {1,2,3}                                      | {aaa,bbb,ccc}
  1 | {{1,2,3},{4,5,6},{7,8,9}}                    | {{aaa,bbb,ccc},{ddd,eee,fff},{ggg,hhh,iii}}
  2 | {1,2,3,4}                                    | {{a,b},{c,d}}
  3 | {{{1,2},{3,4}},{{5,6},{6,7}},{{8,9},{9,10}}} | {a,b,c}
``` 

## 数组取值方法  

### 下标 

``` 
db01=# select id,array_1[1],array_2[2][1],array_1[3][2][2] from t_array;
 id | array_1 | array_2 | array_1
----+---------+---------+---------
  1 |       1 |         |
  1 |         | ddd     |
  2 |       1 | c       |
  3 |         |         |      10
(4 rows)
``` 

> 通过下标来取数组中的值,下标的维度跟存储的维度对应

### 数组slices

``` 
db01=# select id,array_1[2:3][1:2] from t_array;
 id |            array_1
----+--------------------------------
  1 | {}
  1 | {{4,5},{7,8}}
  2 | {}
  3 | {{{5,6},{6,7}},{{8,9},{9,10}}}
(4 rows)

db01=# select array_1,array_1[-2:2] from t_array;
                   array_1                    |            array_1
----------------------------------------------+-------------------------------
 {1,2,3}                                      | {1,2}
 {{1,2,3},{4,5,6},{7,8,9}}                    | {{1,2,3},{4,5,6}}
 {1,2,3,4}                                    | {1,2}
 {{{1,2},{3,4}},{{5,6},{6,7}},{{8,9},{9,10}}} | {{{1,2},{3,4}},{{5,6},{6,7}}}
(4 rows)
``` 

> 有点像字符串的截取,返回的还是同维度的array
> 针对的都是元素

## 数组修改

### 整个字段修改

``` 
db01=# update t_array set array_2 = '{x,y,z}' where id = 3;
UPDATE 1
db01=# select * from t_array where id = 3;
 id |                   array_1                    | array_2
----+----------------------------------------------+---------
  3 | {{{1,2},{3,4}},{{5,6},{6,7}},{{8,9},{9,10}}} | {x,y,z}
(1 row)

db01=# update t_array set array_2 = array[[['x','y','z'],['h','i','j']],[['a','b','c'],['q','w','e']]] where id =3;
UPDATE 1
db01=# select * from t_array where id = 3;
 id |                   array_1                    |                array_2
----+----------------------------------------------+---------------------------------------
  3 | {{{1,2},{3,4}},{{5,6},{6,7}},{{8,9},{9,10}}} | {{{x,y,z},{h,i,j}},{{a,b,c},{q,w,e}}}
(1 row)
``` 

> 只要符合数组类型,不管几维

### 数组元素修改

``` 
db01=# update t_array set array_2[2][2][1] = 'k',array_2[2:2][1:1][1:3]='{"o","p","q"}' where id = 3;
UPDATE 1
db01=# select array_2 from t_array where id =3;
                array_2
---------------------------------------
 {{{x,y,z},{h,i,j}},{{o,p,q},{k,w,e}}}
(1 row)
``` 

> 通过下标或者slice进行修改

## 数组中查找 

通过下标的方法已经知道,语法any是在数组中搜索匹配的

``` 
db01=# select array_1,8 = any(array_1) from t_array;
 {1,2,3}                                      | f
 {{1,2,3},{4,5,6},{7,8,9}}                    | t
 {1,2,3,4}                                    | f
 {{{1,2},{3,4}},{{5,6},{6,7}},{{8,9},{9,10}}} | t
 [-1:1]={1,2,3}                               | f
``` 

> any 要写在等号后面,写前面会报错
> any 表示只要满足有这个元素在里面

还有个all,代表所有元素都要满足

``` 
db01=# select 1 = all(array[1,2,3,4]),1 = all(array[[1,1],[1,1],[1,1]]);
 f        | t
``` 

> 写where语句的时候有用

未完待续......

//END


