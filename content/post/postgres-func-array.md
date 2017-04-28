+++
categories = ["postgresql"]
date = "2015-03-01T09:43:16+08:00"
description = ""
tags = ["array"]
thumbnail = ""
title = "postgresql数组相关函数"

+++

数组函数与操作

<!--more-->

## 数组间操作

### 数组比较和包含 

数组相等

```
db01=# select array[1.1,2.2,3.3]::int[] = '{1,2,3}'::int[];
 t
```

> 两个数组相同下标下的元素值全部相等,则认为这个数组相等.

数组大小包含

```
db01=# select array[1,2,3] < array[2,3],array[1,2,3] @> array[2,3];
 t        | t
```

> 比较有大>,<,>=,<=
> @>包含是指数组A里有数组B的所有元素,即使数组A=B,@< 则相反

奇怪的负数下标

```
db01=# select '[-1:1]={1,2,3}'::int[] < '{1,2,3}'::int[];
 t

db01=# select '[-1:1]={1,2,3}'::int[] < '{1,2}'::int[];
 f

db01=# select '[-1:1]={1,2,3}'::int[] < '{3}'::int[];
 t
```

> 下标不同,即使元素值相同,两个数组也不同.
> 负数下标数组大小如何判断????所以尽量不要使用负数下标的数组

### 数组的拼接

```
db01=# select array[1,2,3] || array[4,5,6] ;
 {1,2,3,4,5,6}

db01=# select array[1,2,3] || array[[4,5,6],[5,6,7]];
 {{1,2,3},{4,5,6},{5,6,7}}

db01=# select array[1,2] || array[[[2,3],[3,4]],[[4,5],[5,6]]];
ERROR:  cannot concatenate incompatible arrays
DETAIL:  Arrays of 1 and 3 dimensions are not compatible for concatenation.
```

> 同一维度的拼接成同一维度,低一维度的拼接成高一维度的一个元素,但不支持跨维度拼接

### 数组重叠判断 

```
db01=# select array[1,2,3] && array[2,5,6];
 t
```

> 两数组中只要有一个元素相同,则为true

## 数组函数 

### 数组合并 

array_append, array_prepend

```
db01=# select array_append(array[1,2,3],4),array_prepend(4,array[1,2,3]);
 {1,2,3,4}    | {4,1,2,3}
```

> 一个添加在前,一个添加在后,只支持一维数组

array_cat

```
db01=# select array_cat(array[[1,2],[2,3]],array[[3,4],[4,5]]);
 {{1,2},{2,3},{3,4},{4,5}}
```

> 入参都是数组

### 数组维度相关 

维度值array_ndims

```
db01=# select array_1,array_ndims(array_1) from t_array;
 {1,2,3}                                      |           1
 {{1,2,3},{4,5,6},{7,8,9}}                    |           2
 {1,2,3,4}                                    |           1
 {{{1,2},{3,4}},{{5,6},{6,7}},{{8,9},{9,10}}} |           3
```

维度结构函数 array_dims

```
db01=# select id ,array_dims(array_1),array_dims(array_2) from t_array;
 id |   array_dims    | array_dims
----+-----------------+------------
  1 | [1:3]           | [1:3]
  1 | [1:3][1:3]      | [1:3][1:3]
  2 | [1:4]           | [1:2][1:2]
  3 | [1:3][1:2][1:2] | [1:3]
(4 rows)
```

数组的默认维度是从1开始,当然也可以自己定义负数下标:

```
db01=# select ('{1,2,3}'::int[])[1];
 int4
------
    1
(1 row)

db01=# select ('[-2:0]={1,2,3}'::int[])[-1];
 int4
------
    2
(1 row)

db01=# select array_dims('[-2:0]={1,2,3}'::int[]);
 array_dims
------------
 [-2:0]
(1 row)
```

> 最好不要使用负数下标,由于比较大小出现不确定因素,有这个概念就行了,实验见上方

### array_upper, array_lower

```
db01=# insert into t_array values(5,'[-1:1]={1,2,3}'::int[],null);
INSERT 0 1
db01=# select array_1,array_upper(array_1,2),array_lower(array_1,1) from t_array;
 {1,2,3}                                      |             |           1
 {{1,2,3},{4,5,6},{7,8,9}}                    |           3 |           1
 {1,2,3,4}                                    |             |           1
 {{{1,2},{3,4}},{{5,6},{6,7}},{{8,9},{9,10}}} |           2 |           1
 [-1:1]={1,2,3}                               |             |          -1
```

> 获取数组的最低最高下标,第二个参数指的是第几个维度,如果没有相应维度返回就是null

### array_length

```
db01=# select array_1,array_length(array_1,1),array_2,array_length(array_2,1) from t_array;
 {1,2,3}                                      |            3 | {aaa,bbb,ccc}                               |            3
 {{1,2,3},{4,5,6},{7,8,9}}                    |            3 | {{aaa,bbb,ccc},{ddd,eee,fff},{ggg,hhh,iii}} |            3
 {1,2,3,4}                                    |            4 | {{a,b},{c,d}}                               |            2
 {{{1,2},{3,4}},{{5,6},{6,7}},{{8,9},{9,10}}} |            3 | {{{x,y,z},{h,i,j}},{{o,p,q},{k,w,e}}}       |            2
 [-1:1]={1,2,3}                               |            3 |                                             |
```

> 获取第二个维度中的元素个数

### array_remove, array_replace 

```
db01=# select array_remove(array[1,2,3,4],3),array_replace(array[1,2,3,3,4],3,5);
 {1,2,4}      | {1,2,5,5,4}
```

> 只支持一维数组

### 数组字符串转换 

```
db01=# select array_to_string(array[1,2,3,4,5,null,7],',','*');
 1,2,3,4,5,*,7

db01=# select string_to_array('aaa|bbb|ccc|x|x|yyy','|','x');
 {aaa,bbb,ccc,NULL,NULL,yyy}
```

### 数组中所有元素个数 

```
db01=# select array_1,cardinality(array_1) from t_array;
 {1,2,3}                                      |           3
 {{1,2,3},{4,5,6},{7,8,9}}                    |           9
 {1,2,3,4}                                    |           4
 {{{1,2},{3,4}},{{5,6},{6,7}},{{8,9},{9,10}}} |          12
 [-1:1]={1,2,3}                               |           3
```

> 不区分维度

### 数组扩展成表记录 

```
db01=# select unnest('{{{1,2},{3,4}},{{5,6},{6,7}},{{8,9},{9,10}}}'::int[][][]);
      1
      2
      3
      4
      5
      6
      6
      7
      8
      9
      9
     10
```

> unnest将数组变成表记录

也可以拼接多个数组变成表,但是只能作为'表'放在from后面

```
db01=# select unnest(array[1,2,3],array[[4,5],[5,6]]);
ERROR:  function unnest(integer[], integer[]) does not exist
LINE 1: select unnest(array[1,2,3],array[[4,5],[5,6]]);
               ^
HINT:  No function matches the given name and argument types. You might need to add explicit type casts.
db01=# select * from unnest(array[1,2,3],array[[4,5],[5,6]]);
      1 |      4
      2 |      5
      3 |      5
        |      6

db01=# select * from unnest('[-2:1]={1,2,3,4}'::int[],'{9,8,7}'::int[]);
      1 |      9
      2 |      8
      3 |      7
      4 |
```

### array_fill 

返回一个有初始化值的数组

```
db01=# select array_fill(3,array[6]);
 {3,3,3,3,3,3}

db01=# select array_fill(3,array[6],array[2]);
 [2:7]={3,3,3,3,3,3}
```

### 聚合函数 array_agg 

```
db01=# select array_agg(id),array_to_string(array_agg(id),',') from t_array;
 {1,1,2,3,5} | 1,1,2,3,5
```

> pg中实现聚合表字段拼接非常简单.
> 当然可以通过自定义聚合函数来实现,这个我们以后有空再写
> oracle中也有自定义聚合函数,还有个未公开函数wm_concat

//END

