+++
categories = ["postgresql"]
date = "2015-03-03T09:54:53+08:00"
description = ""
tags = ["generate_series","generate_subscripts"]
thumbnail = ""
title = "postgresql表记录返回函数"

+++

主要是两个函数generate_series, generate_subscripts 一个关键字 with ordinality

<!--more-->

数组中有一些类似的函数比如unnest也会返回表记录集,这里主要将两个函数

## generate_series 

### int,bigint 入参 

```
db01=# select generate_series(1,6);
 generate_series
-----------------
               1
               2
               3
               4
               5
               6
(6 rows)

db01=# select generate_series(1,6,2);
 generate_series
-----------------
               1
               3
               5
(3 rows)

db01=# select generate_series(0,-6,-2);
 generate_series
-----------------
               0
              -2
              -4
              -6
(4 rows)
```

### timestamp,timestamptz 

```
db01=# select generate_series('2015-1-1'::timestamp,'2015-1-4'::timestamp,interval '1 day');
   generate_series
---------------------
 2015-01-01 00:00:00
 2015-01-02 00:00:00
 2015-01-03 00:00:00
 2015-01-04 00:00:00
(4 rows)
```

## generate_subscripts 

这个函数是为了获取数组中特定维度下元素的下标

```
db01=# select array_1,generate_subscripts(array_1,1) from t_array;
                   array_1                    | generate_subscripts
----------------------------------------------+---------------------
 {1,2,3}                                      |                   1
 {1,2,3}                                      |                   2
 {1,2,3}                                      |                   3
 {{1,2,3},{4,5,6},{7,8,9}}                    |                   1
 {{1,2,3},{4,5,6},{7,8,9}}                    |                   2
 {{1,2,3},{4,5,6},{7,8,9}}                    |                   3
 {1,2,3,4}                                    |                   1
 {1,2,3,4}                                    |                   2
 {1,2,3,4}                                    |                   3
 {1,2,3,4}                                    |                   4
 {{{1,2},{3,4}},{{5,6},{6,7}},{{8,9},{9,10}}} |                   1
 {{{1,2},{3,4}},{{5,6},{6,7}},{{8,9},{9,10}}} |                   2
 {{{1,2},{3,4}},{{5,6},{6,7}},{{8,9},{9,10}}} |                   3
 [-1:1]={1,2,3}                               |                  -1
 [-1:1]={1,2,3}                               |                   0
 [-1:1]={1,2,3}                               |                   1
(16 rows)
```

> generate_series很多时候用在制作测试数据来使用

## 关键字 with ordinality 

```
db01=# select * from generate_subscripts('{{1,2,3},{4,5,6},{7,8,9}}'::int[],2) with ordinality;
 generate_subscripts | ordinality
---------------------+------------
                   1 |          1
                   2 |          2
                   3 |          3
(3 rows)
```

> 有点类似于oracle中的rownum
> 但是限制是只能放在from语句中,并且只能跟随着函数.
> 也就是说函数返回表记录集结果的,并且将此函数放在from语句中作为表来使用的可以跟with ordinality
> 参考:http://www.postgresql.org/docs/current/static/sql-select.html

//END

