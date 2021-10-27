+++
categories = ["postgresql"]
date = "2015-02-24T18:01:57+08:00"
description = ""
tags = ["postgres","inheritance","table","partition"]
thumbnail = ""
title = "postgresql中表的继承及分区表(四)"

+++

这边介绍一下分区的其他方面

<!--more-->

## 分区查询优化-Constraint Exclusion  

> Oracle中这种技术叫分区裁剪(Partition Pruning),通过排除不需要的分区,只扫描特定分区来极大地优化sql查询  

**PG中的分区因为通过check约束来实现,因此查询条件中如果满足某些子分区的约束,就可以通过排除不必要的分区来提高查询效率**  

*这里我们通过执行计划来看一下pg的这个特性.*  

```
db01=> show constraint_exclusion;
 constraint_exclusion 
----------------------
 partition
(1 row)

db01=> explain select * from t_partition where log_date = date'2015-3-13';
                                          QUERY PLAN                                          
----------------------------------------------------------------------------------------------
 Append  (cost=0.00..13.67 rows=7 width=40)
   ->  Seq Scan on t_partition  (cost=0.00..0.00 rows=1 width=40)
         Filter: (log_date = '2015-03-13'::date)
   ->  Bitmap Heap Scan on t_partition_s200503  (cost=4.20..13.67 rows=6 width=40)
         Recheck Cond: (log_date = '2015-03-13'::date)
         ->  Bitmap Index Scan on idx_t_partition_s200503_1  (cost=0.00..4.20 rows=6 width=0)
               Index Cond: (log_date = '2015-03-13'::date)
(7 rows)
```

> constraint_exclusion参数控制查询优化器是否使用这种技术.  
> 这里可以看到只查询了t_partition_s200503这个分区  

*不使用此优化手段后的计划:*  

```
db01=> set constraint_exclusion=off;
SET
db01=> explain select * from t_partition where log_date = date'2015-3-13';
                                          QUERY PLAN                                          
----------------------------------------------------------------------------------------------
 Append  (cost=0.00..65.50 rows=25 width=40)
   ->  Seq Scan on t_partition  (cost=0.00..0.00 rows=1 width=40)
         Filter: (log_date = '2015-03-13'::date)
   ->  Bitmap Heap Scan on t_partition_s200502  (cost=4.20..13.67 rows=6 width=40)
         Recheck Cond: (log_date = '2015-03-13'::date)
         ->  Bitmap Index Scan on idx_t_partition_s200502_1  (cost=0.00..4.20 rows=6 width=0)
               Index Cond: (log_date = '2015-03-13'::date)
   ->  Bitmap Heap Scan on t_partition_s200503  (cost=4.20..13.67 rows=6 width=40)
         Recheck Cond: (log_date = '2015-03-13'::date)
         ->  Bitmap Index Scan on idx_t_partition_s200503_1  (cost=0.00..4.20 rows=6 width=0)
               Index Cond: (log_date = '2015-03-13'::date)
   ->  Bitmap Heap Scan on t_partition_s200504  (cost=4.20..13.67 rows=6 width=40)
         Recheck Cond: (log_date = '2015-03-13'::date)
         ->  Bitmap Index Scan on idx_t_partition_s200504_1  (cost=0.00..4.20 rows=6 width=0)
               Index Cond: (log_date = '2015-03-13'::date)
   ->  Seq Scan on t_partition_s200505  (cost=0.00..24.50 rows=6 width=40)
         Filter: (log_date = '2015-03-13'::date)
(17 rows)
```

*可以看到优化器选择的方案是每个子分区全部扫描一遍,显而易见性能差距有多大.*  

## PG支持的分区类型  

- 暂时只支持range partition 和 list partition  
- list 跟 range类似, 比如分公司表按各分公司代码分区的check 写法为 (check branch = '1010100' )

## 分区总结

总得来说pg的分区功能支持类型较少,与oracle相比较实现比较复杂.  

这里还需要注意:  

1. 各子分区建的check约束不要发生数据重叠情况.  
2. 直接更新分区关键字,迫使该记录映射到其他分区会失败,需要通过变通方法实现.

```
db01=> select * from t_partition;
 id | msg  |  log_date  
----+------+------------
  2 | msg2 | 2015-02-13
  3 | msg3 | 2015-03-02
  4 | msg4 | 2015-04-12
(3 rows)

db01=> update t_partition set log_date = date'2015-3-20' where id = 4;
ERROR:  new row for relation "t_partition_s200504" violates check constraint "t_partition_s200504_log_date_check"
DETAIL:  Failing row contains (4, msg4, 2015-03-20).
db01=> with t as (
db01(> delete from t_partition where id =4 returning * )
db01-> insert into t_partition select t.id,t.msg,date'2015-03-20' from t;
INSERT 0 0
```

> 或者可以通过触发器来实现  

**3. 约束排除(Constraint Exclusion)对于诸如now()等不确定的值不能使用此技术来优化.**  

```
db01=> show constraint_exclusion;
 constraint_exclusion 
----------------------
 partition
(1 row)

db01=> explain select * from t_partition where log_date = now();
                                          QUERY PLAN                                          
----------------------------------------------------------------------------------------------
 Append  (cost=0.00..68.45 rows=25 width=40)
   ->  Seq Scan on t_partition  (cost=0.00..0.00 rows=1 width=40)
         Filter: (log_date = now())
   ->  Bitmap Heap Scan on t_partition_s200502  (cost=4.20..13.68 rows=6 width=40)
         Recheck Cond: (log_date = now())
         ->  Bitmap Index Scan on idx_t_partition_s200502_1  (cost=0.00..4.20 rows=6 width=0)
               Index Cond: (log_date = now())
   ->  Bitmap Heap Scan on t_partition_s200503  (cost=4.20..13.68 rows=6 width=40)
         Recheck Cond: (log_date = now())
         ->  Bitmap Index Scan on idx_t_partition_s200503_1  (cost=0.00..4.20 rows=6 width=0)
               Index Cond: (log_date = now())
   ->  Bitmap Heap Scan on t_partition_s200504  (cost=4.20..13.68 rows=6 width=40)
         Recheck Cond: (log_date = now())
         ->  Bitmap Index Scan on idx_t_partition_s200504_1  (cost=0.00..4.20 rows=6 width=0)
               Index Cond: (log_date = now())
   ->  Seq Scan on t_partition_s200505  (cost=0.00..27.40 rows=6 width=40)
         Filter: (log_date = now())
(17 rows)
```

**4. 由于计划器进行constraint_exclusion时,对每个子分区的约束检查,因此子分区太多会影响解析的性能,官方所说的百八十个性能还可以我没有机会测试.不要超过千八百个..**  

//END

