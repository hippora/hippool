+++
categories = ["postgresql"]
date = "2015-02-24T17:37:26+08:00"
description = ""
tags = ["postgres","query"]
thumbnail = ""
title = "postgresql中的query(二)"

+++

这篇主要讲with查询(Common Table Expressions)

<!--more-->

with查询有点像临时表,一般用在复杂查询中,需要多次使用到这个查询结果时.  

with还有个用处是递归查询.oracle 11gR2开始也支持了.  

PG中的DML也可以使用with语句.我们来依次看看.  

## with普通用法

```
db01=> with t as (
select * from t_insert
)
select count(*) from t;
 count 
-------
     7
(1 row)
```

*只是作为最简单的示例:)*

## with recursive 递归查询

### 用来制造测试数据

#### 使用generate_series

```
db01=> select * from generate_series(1,5,2);
 generate_series 
-----------------
               1
               3
               5
(3 rows)
```
    
#### 使用with 递归

```
db01=> with recursive t(n) as (
db01(> values(1)
db01(> union all
db01(> select n+2 from t where n<4
db01(> )select * from t;
 n 
---
 1
 3
 5
(3 rows)
```

**这里大致说一下递归的原理**

1. with子句中必须有两部分组成,非递归与递归语句.两者通过union或union all组合,非递归语句是入口.  
2. 有了入口数据,相当于是种子,反复喂给递归语句执行,递归语句执行的结果变成新的入口数据.
3. 直到最后一次产生的结果变成入口数据,但是全部不满足递归语句的谓语时,递归结束.所以不要让语句变成死循环.
4. 如果是union 需要排除重复记录,最后生成的结果集就是我们期望的.
5. 官网也说这只是迭代算不上递归.

## with recursive 树状查询

**也可以叫做层级查询**

### 制造测试数据

```
db01=> create table t_recursive(id int ,parent_id int ,name varchar(30));
CREATE TABLE
db01=> insert into t_recursive values(1,null,'boss');
INSERT 0 1
db01=> insert into t_recursive values(2,1,'jack'),(3,1,'joe');
INSERT 0 2
db01=> insert into t_recursive values(4,2,'emp1'),(5,2,'emp2'),(6,2,'emp3');
INSERT 0 3
db01=> insert into t_recursive values(7,3,'emp4'),(8,3,'emp5'),(9,3,'emp6');
INSERT 0 3
db01=> insert into t_recursive values(10,8,'emptest');
INSERT 0 1
db01=> select * from t_recursive;
 id | parent_id |  name   
----+-----------+---------
  1 |           | boss
  2 |         1 | jack
  3 |         1 | joe
  4 |         2 | emp1
  5 |         2 | emp2
  6 |         2 | emp3
  7 |         3 | emp4
  8 |         3 | emp5
  9 |         3 | emp6
 10 |         8 | emptest
(10 rows)
```

### 从节点查找到叶子节点

**查找jack及其下的所有员工**

```
db01=> with recursive t(id,parent_id,name,path) as (
select r1.*,'/'||r1.name path from t_recursive r1 where name='jack'
union all
select r.*,path||'/'||r.name from t_recursive r,t where r.parent_id = t.id
)select * from t;
 id | parent_id | name |    path    
----+-----------+------+------------
  2 |         1 | jack | /jack
  4 |         2 | emp1 | /jack/emp1
  5 |         2 | emp2 | /jack/emp2
  6 |         2 | emp3 | /jack/emp3
(4 rows)
```

*最外层对字段path排序,可以将结果从默认的广度优先变成深度优先*

### 从叶子节点往上查找节点

**查找emptest的上上级老板(他要越级告状去)**

```
db01=> with recursive t(id,parent_id,name,path,lvl) as (
select r1.*,'/'||r1.name path,1 lvl from t_recursive r1 where name='emptest'
union all
select r.*,path||'/'||r.name,lvl + 1 from t_recursive r,t where r.id = t.parent_id    --这里要注意是往上找了
)select * from t where lvl = 3;        --它自己是1级
 id | parent_id | name |       path        | lvl 
----+-----------+------+-------------------+-----
  3 |         1 | joe  | /emptest/emp5/joe |   3
(1 row)
```

## with中带DML

*日志迁移语句*

```
db01=> create table t_recursive_hist(id int,parent_id int,name varchar(30),log_when date);
CREATE TABLE
db01=> with t as (                           
delete from t_recursive where id = 10
returning *)
insert into t_recursive_hist
select t.*,current_date from t;
INSERT 0 1
db01=> select from t_recursive where id = 10;
--
(0 rows)

db01=> select * from t_recursive_hist;
 id | parent_id |  name   |  log_when  
----+-----------+---------+------------
 10 |         8 | emptest | 2015-02-24
(1 row)
```
    
> with子句中的dml语句只有带returning才能让外层的语句'可见'  
> 递归里面不能带DML语句,当然递归外层的DML可以关联递归的输出结果.  
> Oracle中只有在pl/sql里才能使用returning  

**需要注意使用returning语句返回的结果才能被外层的查询使用**  
**也就是说外层直接使用with内dml操作过的表是看不到结果的**

```
db01=> with t as (
db01(> update t_recursive_hist set name='name1'
db01(> ) select * from t_recursive_hist;
 id | parent_id |  name   |  log_when  
----+-----------+---------+------------
 10 |         8 | emptest | 2015-02-14
(1 row)

db01=> select * from t_recursive_hist;
 id | parent_id | name  |  log_when  
----+-----------+-------+------------
 10 |         8 | name1 | 2015-02-14
(1 row)

db01=> with t as (
update t_recursive_hist set name='name2' returning *
) select * from t;
 id | parent_id | name  |  log_when  
----+-----------+-------+------------
 10 |         8 | name2 | 2015-02-14
(1 row)
```

## 总结

PG中的with语句很好很强大,既可以作为一般的临时表,也可以作为递归查询,还可带上DML语句.  

oracle的with只支持查询,从11gR2开始with语句也开始支持递归查询,之前的递归都是通过start with ... connect by来实现.  

//END

