+++
categories = ["postgresql"]
date = "2015-03-05T10:05:29+08:00"
description = ""
tags = ["transaction isolation"]
thumbnail = ""
title = "postgresql事务隔离级别(一)"

+++

隔离级别及read committed

<!--more-->

## 隔离级别 

在讲隔离级别前,要先知道事务中会发生的三种现象

- `脏读(dirty read)`-当前事务可以读取到另一个事物未提交的数据
- `不可重复读(nonrepeatable read)`-当前事务中重复读取同样的数据,发现读取的数据已经被另一个事物修改而得到了不同的结果
- `幻读(phantom read)`-当前事务中重复执行相同的查询,返回的记录数因另一个事物插入或删除而得到不同的结果

在sql标准中定义了4种隔离级别,并且必须要满足相应的条件

```
隔离级别                脏读    不可重复读    幻读
Read uncommitted       O        O          O
Read committed         X        O          O
Repeatable read	       X        X          O
Serializable	       X        X          X
```

PG中的隔离级别遵守了sql标准,并且更加严格,具体为:

```
隔离级别                脏读    不可重复读    幻读
Read uncommitted       X        O          O
Read committed         X        O          O
Repeatable read	       X        X          X
Serializable	       X        X          X
```

> PG中实际上只有三种隔离级别
> Read uncommitted 跟 Read committed 是一样的.
> Repeatable read由于MVCC特点,幻读也不可能.
> Repeatable read与Serializable 某些特性有些差别,所以不是同一种隔离级别
> PG中默认的隔离级别是Read committed

### 设置事务隔离级别语法 

SET TRANSACTION

```
db01=# set transaction isolation level repeatable read;
WARNING:  SET TRANSACTION can only be used in transaction blocks
SET
db01=# begin;
BEGIN
db01=# set transaction isolation level repeatable read;
SET
```

> set transaction 只能在事务块中执行
> 会话相关的语句是set session.[语法][1]

## Read committed 

我们根据上表做一些测试

```
db01=# create table t1(id int ,name text);
CREATE TABLE
db01=# insert into t1 select generate_series(1,10),'aaaaaa';
INSERT 0 10
```

### 不可重复读和幻读测试

```
--session 1
db01=# begin;
BEGIN
db01=# select name from t1 where id = 1;
 aaaaaa

--session 2
db01=# update t1 set name = 'bbbb' where id = 1;
UPDATE 1
db01=# insert into t1 values(11,'aaaaa');
INSERT 0 1

--session1
db01=# select name from t1 where id = 1;    --读取到了另一个事务修改的内容
 bbbb
db01=# select count(*) from t1;    --读取到了另一个事务插入的记录
    11
db01=# end;
COMMIT
```

> 在PG中设置隔离级别为Read uncommitted效果跟Read committed一样

### 数据修改操作的影响 

DML操作或者SELECT FOR UPDATE和SELECT FOR SHARE获取数据的方式跟select一样,但是并发修改相同的数据时会先获取锁来保证数据一致性.

```
-- session 1
db01=# begin;
BEGIN

-- session 2
db01=# begin;
BEGIN
db01=# update t1 set name = name || 'x' where id = 1;
UPDATE 1

--session1
db01=# update t1 set name = name || 'b' where id = 1;
........此时id = 1的行锁被session2的事务持有,所以会在这里等待

--session2
db01=# commit;
COMMIT

--session1
UPDATE 1    --由于session2的事务提交,session1马上返回刚才update的结果
db01=# select * from t1 where id = 1;
  1 | aaaaaaxb    --由于当前的隔离级别,不可重复读发生,并且可以看到在同一个事务中之前已更新的数据.

db01=# end;
COMMIT
db01=# select * from t1 where id = 1;
  1 | aaaaaaxb    --事务完成后结果也保留下来.
```

> 增删改都类似,修改记录前需要获取符合条件的锁,如果锁被其他事务持有,则等待对方释放.
> 对方释放后,根据刚才请求锁的记录上再次判断是否符合条件,然后再持有行锁进行后续操作.

我们来测试有点混淆的情况

```
--session1
db01=# begin;
BEGIN
db01=# update t1 set id = id + 1 where id in (4,5);
UPDATE 2

--session2
db01=# update t1 set name = name || 'x' where id = 5;
...... hang住了,此时由于id=5的行锁被session1的事务持有,所以等待

--session1
db01=# commit;
COMMIT

--session2
UPDATE 0    --立即返回,并且更新了0条记录
```

> 由于不可能发生脏读,session2在执行update时只请求了id=5的一条记录的锁,但是锁正被session1持有
> session1提交后,原先id=5的被更改成6,并释放了相应的锁.
> session2得以继续执行,发现请求的这条记录已经不再满足当初请求的条件(id=5),而不会关注原先id=4的已经被session1改成了5
> 因此对于session2来说更新了0条记录.

**对于这种并发需要格外小心**

未完待续......

//END


  [1]: http://www.postgresql.org/docs/9.4/interactive/sql-set-transaction.html

