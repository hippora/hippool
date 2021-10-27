+++
categories = ["postgresql"]
date = "2015-03-05T10:09:49+08:00"
description = ""
tags = ["transaction isolation"]
thumbnail = ""
title = "postgresql事务隔离级别(二)"

+++

隔离级别,Repeatable read,Serializable

<!--more-->

## Repeatable read

在这个隔离级别测试一下不可重复读和幻读

```
    --session1
    db01=# begin;
    BEGIN
    db01=# set transaction isolation level repeatable read;
    SET
    db01=# select * from t1 where id = 3;    --pg会根据此查询获取相应的snapshot.
      3 | aaaaaa

    db01=# select count(*) from t1;
    10

    --session2
    db01=# update t1 set name = name || 'b' where id = 3;    进行修改id=3的记录
    UPDATE 1
    db01=# insert into t1 values(12,'aaaaaa');
    INSERT 0 1
    db01=# select count(*) from t1;    --11条记录
        11
    db01=# select * from t1 where id = 3;
      3 | aaaaaab

    --session1
    db01=# select * from t1 where id = 3;
      3 | aaaaaa

    db01=# select count(*) from t1;    --在session1事务中看到的还是10条,没有幻读
    10

    db01=# update t1 set name = name || 'x' where id = 3;
    ERROR:  could not serialize access due to concurrent update
    db01=# update t1 set name = name || 'x' where id = 3;
    ERROR:  current transaction is aborted, commands ignored until end of transaction block
    db01=# select * from t1;
    ERROR:  current transaction is aborted, commands ignored until end of transaction block
    db01=# rollback;
    ROLLBACK
```

> 由于id=3的记录在事务开始后,被另外的事务更新,如果在此隔离级别下请求更新,则会失败.
> 在此事务结束前的语句都会出错,就算查询也不行.
> 要让修改成功,只有重新再次执行
> PG中, Repeatable read下没有幻读

## Serializable

最严格的事务隔离级别,也就是说这个级别的相关事务如果之间互相影响结果的,都应该保持串行.

测试:

```
    --session1
    db01=# begin;
    BEGIN
    db01=# set transaction isolation level serializable;
    SET
    db01=# select count(*) from t1;
        15

    db01=# insert into t1 values(1,'aaa');
    INSERT 0 1
    db01=# select count(*) from t1;
        16

    --session2
    db01=# set transaction isolation level serializable;
    SET
    db01=# insert into t1 values(1,'aaa');
    INSERT 0 1
    db01=# select count(*) from t1;
        16

    db01=# commit;
    COMMIT

    --session1
    db01=# commit;
    ERROR:  could not serialize access due to read/write dependencies among transactions
    DETAIL:  Reason code: Canceled on identification as a pivot, during commit attempt.
    HINT:  The transaction might succeed if retried.
```

引用官方的说法是:

> In fact, this isolation level works exactly the same as Repeatable Read except that it monitors for conditions which could make execution of a concurrent set of serializable transactions behave in a manner inconsistent with all possible serial (one at a time) executions of those transactions.

也就是说Serializable隔离级别跟Repeatable read的工作方式一样
但是serializable更严格的地方是,所有的serializable级别的事务,互相影响结果的,都因该串行执行.否则就会报错.需要重新执行.

**对于上面的例子说明:**
如果session1先执行的话,session2得到的结果因该是17条记录.
如果session2先执行的话,session1得到的结果因该也是17条记录.
但是两个事务都是serializable,并且互相影响了结果,必然要导致一个出错回滚.

//END

