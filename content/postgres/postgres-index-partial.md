+++
categories = ["postgresql"]
date = "2015-03-04T09:59:06+08:00"
description = ""
tags = ["partial index"]
thumbnail = ""
title = "postgresql部分索引(Partial index)"

+++

比较特别的Partial index,主要是在满足条件的部分上建立索引,特别情况下的索引效率会很高.

<!--more-->

## 创建partial index 

```
    db01=# create table t_index(id int,name varchar(30));
    CREATE TABLE
    db01=# insert into t_index select generate_series(1,1000),'name';
    INSERT 0 1000
    db01=# insert into t_index select generate_series(1001,1010),'name1';
    INSERT 0 10
    db01=# create index idx_t_index01 on t_index(name) where name != 'name';
    CREATE INDEX
    db01=# explain select * from t_index where name != 'name';
                                      QUERY PLAN
    -------------------------------------------------------------------------------
     Index Scan using idx_t_index01 on t_index  (cost=0.14..12.31 rows=10 width=9)
    (1 row)

    db01=# explain select * from t_index where name = 'name2';
                                     QUERY PLAN
    -----------------------------------------------------------------------------
     Index Scan using idx_t_index01 on t_index  (cost=0.14..4.15 rows=1 width=9)
       Index Cond: ((name)::text = 'name2'::text)
    (2 rows)

    db01=# explain select * from t_index where name = 'name';
                            QUERY PLAN
    -----------------------------------------------------------
     Seq Scan on t_index  (cost=0.00..18.62 rows=1000 width=9)
       Filter: ((name)::text = 'name'::text)
    (2 rows)
```

索引列跟where条件可以不同

```
    db01=# drop index idx_t_index01;
    DROP INDEX
    db01=# create index idx_t_index02 on t_index(name) where id between 300 and 500;
    CREATE INDEX
    db01=# explain select * from t_index where name = 'name1' and id = 209;
                            QUERY PLAN
    -----------------------------------------------------------
     Seq Scan on t_index  (cost=0.00..21.15 rows=1 width=9)
       Filter: (((name)::text = 'name1'::text) AND (id = 209))
    (2 rows)

    db01=# explain select * from t_index where name = 'name1' and id = 300;
                                     QUERY PLAN
    -----------------------------------------------------------------------------
     Index Scan using idx_t_index02 on t_index  (cost=0.14..8.19 rows=1 width=9)
       Index Cond: ((name)::text = 'name1'::text)
       Filter: (id = 300)
    (3 rows)
```

> 从oracle 12C开始也支持这种索引.

## 索引的维护 

```
    db01=# update t_index set id = 3000 where id =300;
    UPDATE 1
    db01=# explain select * from t_index where id = 3000;
                           QUERY PLAN
    --------------------------------------------------------
     Seq Scan on t_index  (cost=0.00..18.62 rows=1 width=9)
       Filter: (id = 3000)
    (2 rows)
```

> 索引条目已经被移除索引,所以pg会自动维护相关索引

## 总结 

部分索引的使用特别适用在大部分数据访问较少,只有部分数据需要经常访问的情况
比如未审核的单子,随着时间审核完成的越来越多,未审核的永远是那么些少数


