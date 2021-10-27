+++
categories = ["postgresql"]
date = "2015-02-25T23:49:34+08:00"
description = ""
tags = ["standby","HA"]
thumbnail = ""
title = "postgresql高可用性之备库(一)"

+++

这篇主要记录一下搭建基于WAL归档的备库. (Log-Shipping Standby Servers)

有点类似于oracle 的dataguard.主要作用是当主库宕机后,可以立刻激活备库,使备库变成主库.

<!--more-->

## 服务器说明

- 主库fnddb: 192.168.10.74
- 备库vm2: 192.168.10.72
- 两边postgresql服务程序版本一致.

主库已配置好归档模式,参考之前的文章[在线备份与恢复](/2015/02/postgresql在线备份与恢复一/)

## 说明

由于备库是不断应用主库的WAL日志来保持备库不断地更新.所以主库的归档地址需要被备库访问的到.
为了达成这种目地,可以有多种方式方法.

- 归档目录放置在共享磁盘上
- 归档时直接传输到NAS上或者直接传输至备库服务器上.

## linux环境配置 

### 配置主机信任 

由于我需要通过scp命令直接将WAL归档到备库服务器上.

*备库*

```
[postgres@vm2 ~]$ ssh-keygen -t rsa
......
[postgres@vm2 ~]$ cd .ssh
```

*主库*

```
[postgres@fnddb ~]$ ssh-keygen -t rsa
......
[postgres@fnddb ~]$ cd .ssh
[postgres@fnddb .ssh]$ scp id_rsa.pub 192.168.10.72:/home/postgres/.ssh/authorized_keys
```

*备库*

```
[postgres@vm2 .ssh]$ cat id_rsa.pub >> authorized_keys
[postgres@vm2 .ssh]$ scp authorized_keys 192.168.10.74:/home/postgres/.ssh/
```

*测试-在两边执行无需输入密码*

```
[postgres@fnddb .ssh]$ ssh 192.168.10.72 date    --主库
Thu Feb 26 22:41:20 CST 2015
[postgres@vm2 .ssh]$ ssh 192.168.10.74 date    --备库
Thu Feb 26 22:40:55 CST 2015
```

## PG 主库配置 

### 归档目录修改 

*我们在备库上将归档目录建好*

```
[postgres@vm2 pgsql]$ pwd
/var/lib/pgsql
[postgres@vm2 pgsql]$ mkdir pg_archive
```

*修改主库归档命令*

```
[postgres@fnddb data]$ vi $PGDATA/postgresql.conf
#archive_command = 'test ! -f /var/lib/pgsql/pg_archive/%f && cp %p /var/lib/pgsql/pg_archive/%f'        # command to use to archive a logfile segment
archive_command = 'ssh 192.168.10.72 test ! -f /var/lib/pgsql/pg_archive/%f && scp %p 192.168.10.72:/var/lib/pgsql/pg_archive/%f'
:wq
[postgres@fnddb data]$ pg_ctl reload
[postgres@fnddb data]$ psql -c "select pg_switch_xlog()"
```

*备库上检查归档目录是否正确归档*

```
[postgres@vm2 pg_archive]$ ls
000000030000000000000020
```


> 先设置归档到备库上,可以防止待会做基础备份作为备库后,所需要的归档还在主库上,增加额外操作

### 允许备库机使用pg_basebackup 

```
[postgres@fnddb data]$ vi pg_hba.conf
......
host    replication     postgres        192.168.10.72/32        md5
[postgres@fnddb data]$ vi postgresql.conf
......
listen_addresses = '*'
max_wal_senders = 1
[postgres@fnddb data]$ pg_ctl reload
server signaled
```

## 对主库做一个基础备份 

**在备库上直接通过pg_basebackup操作,目录结构跟主库结构保持一致**

*主库上检查目录并在备库上建立*

```
[postgres@fnddb data]$ psql -c "\db"
               List of tablespaces
    Name    |  Owner   |        Location
------------+----------+-------------------------
 pg_default | postgres |
 pg_global  | postgres |
 tsdata     | postgres | /var/lib/pgsql/tsdata
 tsdata02   | postgres | /var/lib/pgsql/tsdata02
(4 rows)

[postgres@fnddb data]$ echo $PGDATA
/var/lib/pgsql/data
```

*备库上建立相应目录,过程略*

```
[postgres@vm2 pgsql]$ ls
data  pg_archive  tsdata  tsdata02
```

*备库上做基础备份*

```
[postgres@vm2 pgsql]$ pg_basebackup -Rx -Pv -h 192.168.10.74 -D $PGDATA
Password:
transaction log start point: 0/22000028 on timeline 3
44892/44892 kB (100%), 3/3 tablespaces
transaction log end point: 0/220000B8
pg_basebackup: base backup completed
```

> 使用任何一种方法做一个基础备份,主要用来搭建备库

## 备库配置 

### 配置recovery.conf 

由于使用pg_basebackup的-R参数,自动帮我们生成了一个recovery.conf文件,内容为:

```
standby_mode = 'on'
primary_conninfo = 'user=postgres password=postgres host=192.168.10.74 port=5432 sslmode=disable sslcompression=1'
```

我们这里先不做流复制, primary_conninfo先注释掉

```
[postgres@vm2 data]$ vi recovery.conf
restore_command = 'cp /var/lib/pgsql/pg_archive/%f %p'
standby_mode = on
:wq
```

*暂时只使用这两个参数*

### 启动备库检查日志 

```
[postgres@vm2 data]$ pg_ctl start
[postgres@vm2 data]$ tail -f pg_log/postgresql-27_193629.csv
2015-02-27 19:36:29.936 CST,,,14203,,54f056bd.377b,1,,2015-02-27 19:36:29 CST,,0,LOG,00000,"ending log output to stderr",,"Future log output will go to log destination ""csvlog"".",,,,,,,""
2015-02-27 19:36:29.941 CST,,,14205,,54f056bd.377d,1,,2015-02-27 19:36:29 CST,,0,LOG,00000,"database system was interrupted; last known up at 2015-02-27 19:26:49 CST",,,,,,,,,""
2015-02-27 19:36:29.942 CST,,,14205,,54f056bd.377d,2,,2015-02-27 19:36:29 CST,,0,LOG,00000,"entering standby mode",,,,,,,,,""
2015-02-27 19:36:29.987 CST,,,14205,,54f056bd.377d,3,,2015-02-27 19:36:29 CST,,0,LOG,00000,"restored log file ""000000030000000000000022"" from archive",,,,,,,,,""
2015-02-27 19:36:29.998 CST,,,14205,,54f056bd.377d,4,,2015-02-27 19:36:29 CST,,0,LOG,00000,"redo starts at 0/22000090",,,,,,,,,""
2015-02-27 19:36:29.999 CST,,,14205,,54f056bd.377d,5,,2015-02-27 19:36:29 CST,,0,LOG,00000,"consistent recovery state reached at 0/220000B8",,,,,,,,,""
```

### 检查备库进程 

```
[postgres@vm2 ~]$ ps -ef | grep postgres:
postgres 14204 14203  0 19:36 ?        00:00:00 postgres: logger process
postgres 14205 14203  0 19:36 ?        00:00:00 postgres: startup process   waiting for 000000030000000000000023
postgres 14208 14203  0 19:36 ?        00:00:00 postgres: checkpointer process
postgres 14209 14203  0 19:36 ?        00:00:00 postgres: writer process
postgres 14348 14309  0 19:37 pts/1    00:00:00 grep postgres:
```

### 简单测试 

*在主库上建点数据*

```
db01=> create table t_standby_test(id int ,name text);
CREATE TABLE
db01=> insert into t_standby_test select generate_series(1,1000),'name';
INSERT 0 1000
db01=# select pg_switch_xlog();
 pg_switch_xlog
----------------
 0/23017778
(1 row)
```

*在备库上检查应用WAL归档的情况*

```
[postgres@vm2 data]$ tail -f pg_log/postgresql-27_002122.csv
......
2015-02-27 19:36:29.998 CST,,,14205,,54f056bd.377d,4,,2015-02-27 19:36:29 CST,,0,LOG,00000,"redo starts at 0/22000090",,,,,,,,,""
2015-02-27 19:36:29.999 CST,,,14205,,54f056bd.377d,5,,2015-02-27 19:36:29 CST,,0,LOG,00000,"consistent recovery state reached at 0/220000B8",,,,,,,,,""
2015-02-27 19:38:55.444 CST,,,14205,,54f056bd.377d,6,,2015-02-27 19:36:29 CST,,0,LOG,00000,"restored log file ""000000030000000000000023"" from archive",,,,,,,,,""
```

### 备库设置成只读库(hot_standby) 

*没有设置的话默认在备库上不能访问*

```
[postgres@vm2 ~]$ psql
psql: FATAL:  the database system is starting up
```

#### 主库先改成hot_standby模式 

```
[postgres@fnddb data]$ vi postgresql.conf
......
wal_level = hot_standby
[postgres@fnddb data]$ pg_ctl restart
[postgres@fnddb data]$ psql -c "select pg_switch_xlog()"
```

> 需要切换生成一个WAL归档传输到备库,使备库应用
> 再接下来生成的WAL归档将包含hot_standby信息的归档

*如果不包含hot_standby的WAL归档,在备库上设置成hot_standby模式将启动失败,会报如下错误*

```
2015-02-27 19:47:30.635 CST,,,15111,,54f05952.3b07,4,,2015-02-27 19:47:30 CST,,0,FATAL,XX000,"hot standby is not possible because wal_level was not set to ""hot_standby"" or higher on the master server",,"Either set wal_level to ""hot_standby"" on the master, or turn off hot_standby here.",,,,,,"CheckRequiredParameterValues, xlog.c:5949",""
2015-02-27 19:47:30.636 CST,,,15109,,54f05952.3b05,2,,2015-02-27 19:47:30 CST,,0,LOG,00000,"startup process (PID 15111) exited with exit code 1",,,,,,,,"LogChildExit, postmaster.c:3325",""
2015-02-27 19:47:30.636 CST,,,15109,,54f05952.3b05,3,,2015-02-27 19:47:30 CST,,0,LOG,00000,"aborting startup due to startup process failure",,,,,,,,"reaper, postmaster.c:2604",""
```

#### 备库设置成hot_standby 模式 

```
[postgres@vm2 data]$ vi postgresql.conf
......
hot_standby = on
[postgres@fnddb data]$ pg_ctl restart
```

#### 检查两边数据一致性 

*主库测试*

```
db01=# insert into t_standby_test select generate_series(1,1000),'name';
INSERT 0 1000
db01=# select count(*) from t_standby_test;
 count
-------
  3000
(1 row)

db01=# select pg_switch_xlog();
 pg_switch_xlog
----------------
 0/270175A8
(1 row)
```

> 为啥要切换生成日志?
> 因为架设的是基于文件的备库,只有归档传输到备库后才会应用,否则记录还在主库的xlog中.
> 下篇流复制实时性就很高了!

*备库测试*

```
[postgres@vm2 ~]$ psql db01 hippo
psql (9.4.1)
Type "help" for help.

db01=# select count(*) from t_standby_test;
 count
-------
  3000
(1 row)
```

## 总结

备库的原理跟之前文章里的在线备份与恢复非常相似,只是recovery.conf配置有些许差异.
这文章中的备库模式有点像oracle中dataguard使用arch进程进行传输standby redolog,实时性不好.
可以通过配置archive_timeout参数来减少主备之间数据差异的窗口.

未完待续......

//END
