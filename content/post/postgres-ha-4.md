+++
categories = ["postgresql"]
date = "2015-02-27T23:49:43+08:00"
description = ""
tags = ["HA","failover"]
thumbnail = ""
title = "postgresql高可用性之备库(四)"

+++

这篇主要讲failover

<!--more-->

## 概述 

如果一旦主库宕机,pg提供了机制,使备机变成主机提供服务.
为了使主备库可以相互切换,我们需要进行修改一些配置.

## 配置主库 

> 在主库上配置一旦变成备库的时候所需要使用的参数

### 修改服务配置文件 

```
[postgres@fnddb data]$ vi postgresql.conf
hot_standby = on
```

> 主要是变成备库后需要打开hot_standby,其他之前已经配置好

### 准备好 recovery.conf 

```
[postgres@fnddb data]$ scp 192.168.10.72:/var/lib/pgsql/data/recovery.conf recovery.failover
recovery.conf                                                                                                      100%  182     0.2KB/s   00:00
[postgres@fnddb data]$ vi recovery.failover
standby_mode = 'on'
restore_command = 'cp /var/lib/pgsql/pg_archive/%f %p'    --可以不要
primary_conninfo = 'user=repl password=postgres host=192.168.10.72 port=5432'
primary_slot_name = 'slot_2'    --备库上还没有这个slot
recovery_target_timeline = latest    --非常重要,下面详细解释
```

> 事先改名成 recovery.failover,等需要使用的时候再改成recovery.conf

## 配置备库 

> 考虑备库变成主库后需要用到的参数,包括允许流复制相关.

### 修改备库配置文件 

*主要修改的参数或影响的参数*

```
[postgres@vm2 data]$ vi postgresql.conf
......
listen_addresses = '*'
wal_level = hot_standby
archive_mode = off
max_replication_slots = 2
```

### 修改pg_hba.conf 

```
[postgres@vm2 data]$ vi pg_hba.conf
host    replication     repl        192.168.10.74/32    md5
```

> 注意如果基础备份过来已经有了,可能ip地址需要修改

## 进行failover 

### 模拟主库宕机 

```
[postgres@fnddb data]$ pg_ctl stop -m immediate
waiting for server to shut down.... done
server stopped
```

*备库上的日志记录*

```
2015-02-28 00:57:58.324 CST,"hippo","db01",312,"[local]",54f0a216.138,2,"authentication",2015-02-28 00:57:58 CST,2/1,0,LOG,00000,"connection authorized: user=hippo database=db01",,,,,,,,,""
2015-02-28 00:58:26.402 CST,,,32632,,54f0a193.7f78,2,,2015-02-28 00:55:47 CST,,0,FATAL,XX000,"could not receive data from WAL stream: server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.
",,,,,,,,,""
2015-02-28 00:58:26.412 CST,,,32626,,54f0a193.7f72,6,,2015-02-28 00:55:47 CST,1/0,0,LOG,00000,"record with zero length at 0/290139C8",,,,,,,,,""
2015-02-28 00:58:26.416 CST,,,314,,54f0a232.13a,1,,2015-02-28 00:58:26 CST,,0,FATAL,XX000,"could not connect to the primary server: could not connect to server: Connection refused
        Is the server running on host ""192.168.10.74"" and accepting
        TCP/IP connections on port 5432?
",,,,,,,,,""
```

> 由于流复制连接已经断开,查看备库进程已经看不到 receiver 进程,备库又进入了restore_command模式.

### 备库failover 

```
[postgres@vm2 data]$ pg_ctl promote
server promoting
[postgres@vm2 data]$ tail -f pg_log/postgresql-28_005547.csv
        Is the server running on host ""192.168.10.74"" and accepting
        TCP/IP connections on port 5432?
",,,,,,,,,""
2015-02-28 01:03:23.575 CST,,,32626,,54f0a193.7f72,7,,2015-02-28 00:55:47 CST,1/0,0,LOG,00000,"received promote request",,,,,,,,,""
2015-02-28 01:03:23.575 CST,,,32626,,54f0a193.7f72,8,,2015-02-28 00:55:47 CST,1/0,0,LOG,00000,"redo done at 0/29013990",,,,,,,,,""
2015-02-28 01:03:23.575 CST,,,32626,,54f0a193.7f72,9,,2015-02-28 00:55:47 CST,1/0,0,LOG,00000,"last completed transaction was at log time 2015-02-28 00:57:36.186822+08",,,,,,,,,""
2015-02-28 01:03:23.595 CST,,,32626,,54f0a193.7f72,10,,2015-02-28 00:55:47 CST,1/0,0,LOG,00000,"selected new timeline ID: 4",,,,,,,,,""
2015-02-28 01:03:23.695 CST,,,32626,,54f0a193.7f72,11,,2015-02-28 00:55:47 CST,1/0,0,LOG,00000,"archive recovery complete",,,,,,,,,""
2015-02-28 01:03:23.701 CST,,,32624,,54f0a193.7f70,3,,2015-02-28 00:55:47 CST,,0,LOG,00000,"database system is ready to accept connections",,,,,,,,,""
2015-02-28 01:03:23.702 CST,,,721,,54f0a35b.2d1,1,,2015-02-28 01:03:23 CST,,0,LOG,00000,"autovacuum launcher started",,,,,,,,,""
```

备库升级成为`新主库`,生成新的timeline ID:4

### 新主库建slot 

```
db01=# select pg_create_physical_replication_slot('slot_2');
 pg_create_physical_replication_slot
-------------------------------------
 (slot_2,)
(1 row)
```

### 原主库变备库 

改名recovery.failover

```
[postgres@fnddb data]$ mv recovery.failover recovery.conf
```

> 之前slot,流复制连接都已经配置好

启动备库(原主库)并检查启动日志

```
[postgres@fnddb data]$ pg_ctl start
[postgres@fnddb data]$ tail -f pg_log/postgresql-28_011432.csv
2015-02-28 01:14:32.830 CST,,,12824,,54f0a5f8.3218,1,,2015-02-28 01:14:32 CST,,0,LOG,00000,"ending log output to stderr",,"Future log output will go to log destination ""csvlog"".",,,,,,,""
2015-02-28 01:14:32.834 CST,,,12826,,54f0a5f8.321a,1,,2015-02-28 01:14:32 CST,,0,LOG,00000,"database system was shut down in recovery at 2015-02-28 01:14:29 CST",,,,,,,,,""
2015-02-28 01:14:32.850 CST,,,12826,,54f0a5f8.321a,2,,2015-02-28 01:14:32 CST,,0,LOG,00000,"entering standby mode",,,,,,,,,""
2015-02-28 01:14:32.881 CST,,,12826,,54f0a5f8.321a,3,,2015-02-28 01:14:32 CST,,0,LOG,00000,"restored log file ""00000003.history"" from archive",,,,,,,,,""
2015-02-28 01:14:32.885 CST,,,12826,,54f0a5f8.321a,4,,2015-02-28 01:14:32 CST,1/0,0,LOG,00000,"redo starts at 0/29001100",,,,,,,,,""
2015-02-28 01:14:32.891 CST,,,12826,,54f0a5f8.321a,5,,2015-02-28 01:14:32 CST,1/0,0,LOG,00000,"consistent recovery state reached at 0/290139C8",,,,,,,,,""
2015-02-28 01:14:32.892 CST,,,12824,,54f0a5f8.3218,2,,2015-02-28 01:14:32 CST,,0,LOG,00000,"database system is ready to accept read only connections",,,,,,,,,""
2015-02-28 01:14:32.892 CST,,,12826,,54f0a5f8.321a,6,,2015-02-28 01:14:32 CST,1/0,0,LOG,00000,"record with zero length at 0/290139C8",,,,,,,,,""
2015-02-28 01:14:32.902 CST,,,12836,,54f0a5f8.3224,1,,2015-02-28 01:14:32 CST,,0,LOG,00000,"started streaming WAL from primary at 0/29000000 on timeline 4",,,,,,,,,""
```

> 之前忘记增加 recovery_target_timeline = latest,启动备库后,只会继续找原来的timeline下的xlog,也就是Timeline ID:3

*相关报错日志*

```
[postgres@fnddb data]$ tail -f pg_log/postgresql-28_011022.csv
2015-02-28 01:14:18.527 CST,,,12753,,54f0a4fe.31d1,98,,2015-02-28 01:10:22 CST,,0,LOG,00000,"restarted WAL streaming at 0/29000000 on timeline 3",,,,,,,,,""
2015-02-28 01:14:18.528 CST,,,12753,,54f0a4fe.31d1,99,,2015-02-28 01:10:22 CST,,0,LOG,00000,"replication terminated by primary server","End of WAL reached on timeline 3 at 0/290139C8.",,,,,,,,""
2015-02-28 01:14:23.543 CST,,,12753,,54f0a4fe.31d1,100,,2015-02-28 01:10:22 CST,,0,LOG,00000,"restarted WAL streaming at 0/29000000 on timeline 3",,,,,,,,,""
2015-02-28 01:14:23.544 CST,,,12753,,54f0a4fe.31d1,101,,2015-02-28 01:10:22 CST,,0,LOG,00000,"replication terminated by primary server","End of WAL reached on timeline 3 at 0/290139C8.",,,,,,,,""
2015-02-28 01:14:28.554 CST,,,12753,,54f0a4fe.31d1,102,,2015-02-28 01:10:22 CST,,0,LOG,00000,"restarted WAL streaming at 0/29000000 on timeline 3",,,,,,,,,""
```

### 简单测试 

新主库:

```
db01=# create table t1(id int);
CREATE TABLE
db01=# insert into t1 values(1);
INSERT 0 1
db01=# \x
Expanded display is on.
db01=# select * from pg_stat_replication;
-[ RECORD 1 ]----+------------------------------
pid              | 1299
usesysid         | 24601
usename          | repl
application_name | walreceiver
client_addr      | 192.168.10.74
client_hostname  |
client_port      | 38911
backend_start    | 2015-02-28 01:14:44.802549+08
backend_xmin     |
state            | streaming
sent_location    | 0/290256B0
write_location   | 0/290256B0
flush_location   | 0/290256B0
replay_location  | 0/290256B0
sync_priority    | 0
sync_state       | async

db01=# select * from pg_replication_slots;
-[ RECORD 1 ]+-----------
slot_name    | slot_2
plugin       |
slot_type    | physical
datoid       |
database     |
active       | t
xmin         |
catalog_xmin |
restart_lsn  | 0/290256B0
```

新备库

```
db01=# select * from t1;
-[ RECORD 1 ]
id | 1

db01=# select * from pg_replication_slots;
-[ RECORD 1 ]+-----------
slot_name    | slot_1
plugin       |
slot_type    | physical
datoid       |
database     |
active       | f
xmin         |
catalog_xmin |
restart_lsn  | 0/29001098
```

> 原先的slot_1未激活

## failover 方法二 

刚才是通过pg_ctl promote进行failover,还有一种方式是通过一个文件触发failover

recovery.conf里配置trigger_file='/home/postgres/aaa.bbb'

一旦备库监控到此目录下有aaa.bbb文件(空文件也行),则触发failover

实验暂时略

//END

