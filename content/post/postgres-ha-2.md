+++
categories = ["postgresql"]
date = "2015-02-27T23:49:38+08:00"
description = ""
tags = ["streaming replication","HA"]
thumbnail = ""
title = "postgresql高可用性之备库(二)"

+++

这篇主要讲如果将刚才基于WAL文件的备库设置成Streaming Replication

<!--more-->

## 概述 

由于基于WAL归档搭建的备库,实时性不太好
而streaming replication可以将备库跟主库数据同步差异缩小至最小,实用性更好一点
这篇主要将上一篇的配置进行修改,使变成流复制的模式运行

## 主库修改配置 

### 添加一个复制用户 

```
postgres=# create role repl replication login password 'postgres';
CREATE ROLE
```

### 修改postgres.conf 

```
[postgres@fnddb data]$ vi postgresql.conf
max_wal_senders = 1
listen_addresses = '*'
......
```

> 上一篇由于用到pg_basebackup,这两个参数已经修改好了

### 修改 pg_hba.conf 

```
[postgres@fnddb data]$ vi pg_hba.conf
......
#host    replication     postgres        192.168.10.72/32        md5
host    replication     repl        192.168.10.72/32        md5
```

> 上一篇应为使用superuser运行的pg_basebackup,添加了认证,这里把它注释掉,使用我们创建的用户

### 使配置生效 

```
[postgres@fnddb data]$ pg_ctl reload
```

## 备库修改配置 

### recovery.conf修改 

```
[postgres@vm2 data]$ vi recovery.conf
standby_mode = 'on'
restore_command = 'cp /var/lib/pgsql/pg_archive/%f %p'
primary_conninfo = 'user=repl password=postgres host=192.168.10.74 port=5432'
```

> 前两个参数上一篇中已经有了

### 重启备库 

```
[postgres@vm2 data]$ pg_ctl restart
```

*查看日志*

```
[postgres@vm2 data]$ tail -f pg_log/postgresql-27_213214.csv
2015-02-27 21:32:14.545 CST,,,22002,,54f071de.55f2,2,,2015-02-27 21:32:14 CST,,0,LOG,00000,"database system is ready to accept read only connections",,,,,,,,,""
2015-02-27 21:32:14.554 CST,,,22004,,54f071de.55f4,6,,2015-02-27 21:32:14 CST,1/0,0,LOG,00000,"unexpected pageaddr 0/25000000 in log segment 000000030000000000000028, offset 0",,,,,,,,,""
2015-02-27 21:32:14.564 CST,,,22011,,54f071de.55fb,1,,2015-02-27 21:32:14 CST,,0,LOG,00000,"started streaming WAL from primary at 0/28000000 on timeline 3",,,,,,,,,""
```

## 检查测试 

*主库*

```
db01=# insert into t_standby_test select generate_series(1,1000),'name';
INSERT 0 1000
db01=# select count(*) from t_standby_test;
 count
-------
  5000
(1 row)
```

*备库*

```
db01=# select count(*) from t_standby_test;
 count
-------
  5000
(1 row)

db01=# insert into t_standby_test select generate_series(1,1000),'name';
ERROR:  cannot execute INSERT in a read-only transaction
```

> 几乎是实时同步,要看主库的压力
> 因为是只读库,备库上不允许各种DDL,DML等影响数据库的操作

## 新建表空间同步 

*表空间因为涉及到操作系统目录,我来测试下是否需要事先主备服务器都要先创建目录*

```
[postgres@fnddb pgsql]$ mkdir tsdata03
db01=# create tablespace tsdata03 location '/var/lib/pgsql/tsdata03';
CREATE TABLESPACE
```

*发现备库已经宕机了,检查备库日志*

```
......
2015-02-27 21:46:55.340 CST,,,22004,,54f071de.55f4,7,,2015-02-27 21:32:14 CST,1/0,0,FATAL,58P01,"directory ""/var/lib/pgsql/tsdata03"" does not exist",,"Create this directory for the tablespace before restarting the server.",,,"xlog redo create tablespace: 24605 ""/var/lib/pgsql/tsdata03""",,,,""
2015-02-27 21:46:55.341 CST,,,22002,,54f071de.55f2,3,,2015-02-27 21:32:14 CST,,0,LOG,00000,"startup process (PID 22004) exited with exit code 1",,,,,,,,,""
2015-02-27 21:46:55.341 CST,,,22002,,54f071de.55f2,4,,2015-02-27 21:32:14 CST,,0,LOG,00000,"terminating any other active server processes",,,,,,,,,""
2015-02-27 21:46:55.342 CST,"hippo","db01",22173,"[local]",54f0726a.569d,5,"idle",2015-02-27 21:34:34 CST,2/0,0,WARNING,57P02,"terminating connection because of crash of another server process","The postmaster has commanded this server process to roll back the current transaction and exit, because another server process exited abnormally and possibly corrupted shared memory.","In a moment you should be able to reconnect to the database and repeat your command.",,,,,,,"psql"
```

*我们在备库上建好目录再启动*

```
[postgres@vm2 ~]$ mkdir /var/lib/pgsql/tsdata03
[postgres@vm2 ~]$ pg_ctl start
[postgres@vm2 ~]$ psql -c '\db tsdata03'
            List of tablespaces
   Name   | Owner |        Location
----------+-------+-------------------------
 tsdata03 | hippo | /var/lib/pgsql/tsdata03
(1 row)
```

> 所以主库创建表空间,需要在所有的主备服务器上事先创建好目录.

## 流复制监控 

*主库上:*

```
db01=# select pg_current_xlog_location();
 pg_current_xlog_location
--------------------------
 0/28041308
(1 row)

db01=# \x
Expanded display is on.
db01=# select * from pg_stat_replication;
-[ RECORD 1 ]----+------------------------------
pid              | 11801
usesysid         | 24601
usename          | repl
application_name | walreceiver
client_addr      | 192.168.10.72
client_hostname  |
client_port      | 52252
backend_start    | 2015-02-27 21:49:49.197973+08
backend_xmin     |
state            | streaming
sent_location    | 0/28041308
write_location   | 0/28041308
flush_location   | 0/28041308
replay_location  | 0/28041308
sync_priority    | 0
sync_state       | async
```

*备库*

```
postgres=# select pg_last_xlog_receive_location();
 pg_last_xlog_receive_location
-------------------------------
 0/28041308
(1 row)
```

> pg_current_xlog_location 和pg_last_xlog_receive_location 检查主库xlog位置及备库接受到的位置
> 如果一直相差较大,说明主库的负载比较大.

未完待续......

//END

