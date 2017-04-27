+++
categories = ["postgresql"]
date = "2015-02-22T17:28:51+08:00"
description = ""
tags = ["postgres","DML"]
thumbnail = ""
title = "postgresql中DML操作"

+++

开始学习一些postgresql中的语法基础, 因为懂一些oracle,而且pg跟oracle也比较相像.所以只跟oracle做比较了!

<!--more-->

除了insert我感觉有些特别的写法.update和delete没啥特别的.
这里列举一些特殊的写法

## insert ##

    db01=> create table t_insert(id int,name text);
    CREATE TABLE
    db01=> insert into t_insert values(1,'hippo');
    INSERT 0 1
    db01=> insert into t_insert values(2,'jack'),(3,'joe'),(4,'alice');    --一次插入多条记录
    INSERT 0 3
    db01=> insert into t_insert default values;    --全部插入默认值
    INSERT 0 1
    db01=> alter table t_insert alter column name set default 'name';
    ALTER TABLE
    db01=> insert into t_insert default values;
    INSERT 0 1
    db01=> insert into t_insert(id,name) values(9,default);    --这种写法oracle中也有
    INSERT 0 1
    db01=> select * from t_insert;
     id | name
    ----+-------
      1 | hippo
      2 | jack
      3 | joe
      4 | alice
        |
        | name
      9 | name
    (7 rows)

