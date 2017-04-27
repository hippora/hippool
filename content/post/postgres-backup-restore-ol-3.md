+++
categories = ["postgresql"]
date = "2015-02-13T17:09:59+08:00"
description = ""
tags = ["postgres","backup","restore","wal","pitr"]
thumbnail = ""
title = "postgresql在线备份与恢复(三)"

+++

这次使用更底层的API函数来进行备份,然后恢复到某一个时点(PITR)

<!--more-->

## 备份一个基础备份,使用底层API

### 确保你已经开启了归档

*可以通过检查参数设置,找到归档目录,手动切换wal日志来查看是否可以正确归档*

```
postgres=# select name,setting from pg_settings where name like 'archive%' or name = 'wal_level';
      name       |                                   setting
-----------------+------------------------------------------------------------------------------
 archive_command | test ! -f /var/lib/pgsql/pg_archive/%f && cp %p /var/lib/pgsql/pg_archive/%f
 archive_mode    | on
 archive_timeout | 0
 wal_level       | archive
(4 rows)
```

**进入归档目录查看当前最大字符的归档名**

```
[postgres@fnddb pg_archive]$ ll
total 311304
......
-rw-------. 1 postgres postgres 16777216 Feb 13 01:13 000000020000000000000010
-rw-------. 1 postgres postgres 16777216 Feb 13 19:13 000000020000000000000011
-rw-------. 1 postgres postgres       41 Feb 13 01:02 00000002.history
```

**切换一下日志并再次检查**

```
[postgres@fnddb pg_archive]$ ll -r && psql -c "select pg_switch_xlog()"
total 311304
-rw-------. 1 postgres postgres       41 Feb 13 01:02 00000002.history
-rw-------. 1 postgres postgres 16777216 Feb 13 19:13 000000020000000000000011
[postgres@fnddb pg_archive]$ ll -r
total 327688
-rw-------. 1 postgres postgres       41 Feb 13 01:02 00000002.history
-rw-------. 1 postgres postgres 16777216 Feb 13 19:17 000000020000000000000012
```

> 如果数据库没有任何活动,有可能切换不会进行归档

### 执行pg_start_backup命令开始备份

*使用superuser执行, 150213basebackup 是一个标签*

```
[postgres@fnddb ~]$ psql -c "select pg_start_backup('150213basebackup')"
 pg_start_backup
-----------------
 0/14000028
(1 row)
```

### 使用操作系统命令备份cluster,tablespace,归档文件.

```
[postgres@fnddb pgsql]$ tar -cjvf ~/basebk150213.tbz2 /var/lib/pgsql/data /var/lib/pgsql/tsdata /var/lib/pgsql/tsdata02 --exclude /var/lib/pgsql/data/postmaster.opts --exclude /var/lib/pgsql/data/postmaster.pid --exclude /var/lib/pgsql/data/pg_xlog --exclude /var/lib/pgsql/data/pg_replslot
```

**需要排除一些文件无需归档**

- postmaster.opts,postmaster.pid --这是pg服务器进程的信息,不需要备份
- pg_xlog --当前的wal日志,还原时可能已经过期,我们需要归档.
- pg_replslot --replication使用的,无需备份

### 执行pg_stop_backup结束备份

```
[postgres@fnddb pgsql]$ psql -c "select pg_stop_backup()"
NOTICE:  pg_stop_backup complete, all required WAL segments have been archived
 pg_stop_backup
----------------
 0/140000B8
(1 row)
```

### 制造一些测试数据来测试

*建两个表在不同的时间点各插入一批数据*

```
[postgres@fnddb pgsql]$ psql database1
psql (9.4.1)
Type "help" for help.

database1=# \! date
Fri Feb 13 19:59:28 CST 2015
database1=# create table t_19_59_28(id int);
CREATE TABLE
database1=# insert into t_19_59_28 select generate_series(1,10000);
INSERT 0 10000
database1=# select now();
              now
-------------------------------
 2015-02-13 20:04:49.63197+08
(1 row)
database1=# create table t_20_04_49(id int);
CREATE TABLE
database1=# insert into t_20_04_49 select generate_series(1,10000);
INSERT 0 10000
database1=# select now();
              now
-------------------------------
 2015-02-13 20:05:52.15099+08
(1 row)
database1=# insert into t_19_59_28 select generate_series(1,10000);
INSERT 0 10000
database1=# select now();
              now
-------------------------------
 2015-02-13 20:06:54.926986+08
(1 row)
database1=# select pg_switch_xlog();
 pg_switch_xlog
----------------
 0/151FE288
(1 row)

database1=# insert into t_20_04_49 select generate_series(1,10000);
INSERT 0 10000
database1=# select now();
              now
-------------------------------
 2015-02-13 20:08:35.060989+08
(1 row)
```

### 备份归档

```
[postgres@fnddb pgsql]$ ll -r pg_archive/
total 442380
-rw-------. 1 postgres postgres       41 Feb 13 01:02 00000002.history
-rw-------. 1 postgres postgres 16777216 Feb 13 20:08 000000020000000000000015
-rw-------. 1 postgres postgres      303 Feb 13 19:52 000000020000000000000014.00000028.backup
-rw-------. 1 postgres postgres 16777216 Feb 13 19:52 000000020000000000000014
-rw-------. 1 postgres postgres 16777216 Feb 13 19:23 000000020000000000000013
......
```

*为了方便,我备份了所有timeline= 00000002开头的归档,实际上只需要>= 000000020000000000000014 的归档即可.*
*详细可以见上一篇文章*

```
[postgres@fnddb pg_archive]$ tar -cjvf ~/basebk150213_arch.tbz2 00000002*
00000002000000000000000F
000000020000000000000010
000000020000000000000011
000000020000000000000012
000000020000000000000013
000000020000000000000014
000000020000000000000014.00000028.backup
000000020000000000000015
00000002.history
```

## 基于时间点的恢复(PITR)

### 模拟破坏数据库

模拟数据库正常运作

```
database1=# insert into t_19_59_28 select generate_series(1,10000);
INSERT 0 10000
database1=# insert into t_20_04_49 select generate_series(1,10000);
INSERT 0 10000
database1=# select count(*) from t_19_59_28;
 count
-------
 30000
(1 row)

database1=# select count(*) from t_20_04_49;
 count
-------
 30000
(1 row)

database1=# select pg_switch_xlog();
 pg_switch_xlog
----------------
 0/161DC3E8
(1 row)

[postgres@fnddb pg_archive]$ ll -r
total 458764
-rw-------. 1 postgres postgres       41 Feb 13 01:02 00000002.history
-rw-------. 1 postgres postgres 16777216 Feb 13 21:19 000000020000000000000016 --此归档未备份
```

假设数据库已经破坏,归档日志已经全部丢失.

```
[postgres@fnddb pgsql]$ rm -rf data
[postgres@fnddb pgsql]$ rm -rf pg_archive
[postgres@fnddb pgsql]$ rm -rf tsdata
[postgres@fnddb pgsql]$ rm -rf tsdata02
```

数据库操作报错,过一会数据库就崩溃关闭了

```
database1=# select count(*) from t_20_04_49;
ERROR:  could not open file "base/16385/16403": No such file or directory
database1=# insert into t_20_04_49 select generate_series(1,10000);
ERROR:  could not open file "base/16385/16403": No such file or directory
database1=# insert into t_20_04_49 select generate_series(1,10000);
server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.
The connection to the server was lost. Attempting reset: Failed.
```

### 使用基础备份与归档进行备份

#### 如果数据库还未关闭,则立即关闭

```
[postgres@fnddb pgsql]$ pg_ctl stop -m fast
```

#### 解压缩基础备份到相应目录,解压归档日志

*由于备份时指定了绝对路径目录结构一起备份了,这里指定根目录*

```
[postgres@fnddb ~]$ tar -xjvf basebk150213.tbz2 -C /
......
var/lib/pgsql/tsdata/PG_9.4_201409291/16385/16397
var/lib/pgsql/tsdata02/
var/lib/pgsql/tsdata02/PG_9.4_201409291/
[postgres@fnddb ~]$ mkdir basebk150213_arch
[postgres@fnddb ~]$ tar -xjvf basebk150213_arch.tbz2 -C basebk150213_arch
......
000000020000000000000014
000000020000000000000014.00000028.backup
000000020000000000000015
00000002.history
```

#### 恢复期间禁止用户登录

*修改pg_hba.conf,方法略*

#### 配置recovery.conf文件

*这里我想只还原到t_19_59_28表创建好,并插入了10000条记录, t_20_04_49表还未创建*

```
[postgres@fnddb ~]$ cd $PGDATA
[postgres@fnddb data]$ cp /usr/local/pgsql/share/recovery.conf.sample recovery.conf
[postgres@fnddb data]$ vi recovery.conf
......
restore_command = 'cp /home/postgres/basebk150213_arch/%f %p'
recovery_target_time = '2015-02-13 20:04:49.63197+08'
recovery_target_inclusive = true        --指示如果这个时间点上的commit是否要包括进去.true就是包括
```

> *这里需要注意: recovery_target_time 设置的时间格式,使用pg的now函数输出的格式*
> *如果使用诸如linux下date命令的格式填写的话,会转换错误*
> *之前我实验填写的是Fri Feb 13 20:04:49 CST 2015,结果恢复时转换成了2015-02-14 10:04:49+08,导致恢复点错误.*
> *不知道是不是PG的BUG*

#### 启动服务进程进行恢复

```
[postgres@fnddb data]$ pg_ctl start
[postgres@fnddb data]$ tail pg_log/postgresql-13_220103.csv
......
FATAL,XX000,"required WAL directory ""pg_xlog"" does not exist"
......
```

*检查日志发现没有pg_xlog,想起来是当初备份的时候排除了pg_xlog和pg_replslot*

```
[postgres@fnddb data]$ mkdir pg_xlog pg_replslot
[postgres@fnddb data]$ pg_ctl start
server starting
[postgres@fnddb data]$ vi pg_log/postgresql-13_232942.csv
2015-02-13 23:29:42.109 CST,,,14378,,54de1866.382a,1,,2015-02-13 23:29:42 CST,,0,LOG,00000,"ending log output to stderr",,"Future log output will go to log destination ""csvlog"".",,,,,,,""
2015-02-13 23:29:42.112 CST,,,14380,,54de1866.382c,1,,2015-02-13 23:29:42 CST,,0,LOG,00000,"database system was interrupted; last known up at 2015-02-13 19:23:06 CST",,,,,,,,,""
2015-02-13 23:29:42.112 CST,,,14380,,54de1866.382c,2,,2015-02-13 23:29:42 CST,,0,LOG,00000,"creating missing WAL directory ""pg_xlog/archive_status""",,,,,,,,,""
2015-02-13 23:29:42.113 CST,,,14380,,54de1866.382c,3,,2015-02-13 23:29:42 CST,,0,LOG,00000,"starting point-in-time recovery to 2015-02-13 20:04:49.63197+08",,,,,,,,,""
2015-02-13 23:29:42.121 CST,,,14380,,54de1866.382c,4,,2015-02-13 23:29:42 CST,,0,LOG,00000,"restored log file ""00000002.history"" from archive",,,,,,,,,""
2015-02-13 23:29:42.157 CST,,,14380,,54de1866.382c,5,,2015-02-13 23:29:42 CST,,0,LOG,00000,"restored log file ""000000020000000000000014"" from archive",,,,,,,,,""
2015-02-13 23:29:42.162 CST,,,14380,,54de1866.382c,6,,2015-02-13 23:29:42 CST,,0,LOG,00000,"redo starts at 0/14000090",,,,,,,,,""
2015-02-13 23:29:42.163 CST,,,14380,,54de1866.382c,7,,2015-02-13 23:29:42 CST,,0,LOG,00000,"consistent recovery state reached at 0/140000B8",,,,,,,,,""
2015-02-13 23:29:42.199 CST,,,14380,,54de1866.382c,8,,2015-02-13 23:29:42 CST,,0,LOG,00000,"restored log file ""000000020000000000000015"" from archive",,,,,,,,,""
2015-02-13 23:29:42.252 CST,,,14380,,54de1866.382c,9,,2015-02-13 23:29:42 CST,,0,LOG,00000,"recovery stopping before commit of transaction 1837, time 2015-02-13 20:05:04.931385+08",,,,,,,,,""
2015-02-13 23:29:42.252 CST,,,14380,,54de1866.382c,10,,2015-02-13 23:29:42 CST,,0,LOG,00000,"redo done at 0/150C2A10",,,,,,,,,""
2015-02-13 23:29:42.252 CST,,,14380,,54de1866.382c,11,,2015-02-13 23:29:42 CST,,0,LOG,00000,"last completed transaction was at log time 2015-02-13 20:03:02.823509+08",,,,,,,,,""
2015-02-13 23:29:42.265 CST,,,14380,,54de1866.382c,12,,2015-02-13 23:29:42 CST,,0,LOG,00000,"selected new timeline ID: 3",,,,,,,,,""
2015-02-13 23:29:42.272 CST,,,14380,,54de1866.382c,13,,2015-02-13 23:29:42 CST,,0,LOG,00000,"restored log file ""00000002.history"" from archive",,,,,,,,,""
2015-02-13 23:29:42.358 CST,,,14380,,54de1866.382c,14,,2015-02-13 23:29:42 CST,,0,LOG,00000,"archive recovery complete",,,,,,,,,""
2015-02-13 23:29:42.462 CST,,,14389,,54de1866.3835,1,,2015-02-13 23:29:42 CST,,0,LOG,00000,"autovacuum launcher started",,,,,,,,,""
2015-02-13 23:29:42.463 CST,,,14378,,54de1866.382a,2,,2015-02-13 23:29:42 CST,,0,LOG,00000,"database system is ready to accept connections",,,,,,,,,""
2015-02-13 23:29:42.472 CST,,,14390,,54de1866.3836,1,,2015-02-13 23:29:42 CST,,0,LOG,00000,"archive command failed with exit code 1","The failed archive command was: test ! -f /var/lib/pgsql/pg_archive/00000003.history && cp pg_xlog/00000003.history /var/lib/pgsql/pg_archive/00000003.history",,,,,,,,""
2015-02-13 23:29:43.483 CST,,,14390,,54de1866.3836,2,,2015-02-13 23:29:42 CST,,0,LOG,00000,"archive command failed with exit code 1","The failed archive command was: test ! -f /var/lib/pgsql/pg_archive/00000003.history && cp pg_xlog/00000003.history /var/lib/pgsql/pg_archive/00000003.history",,,,,,,,""
2015-02-13 23:29:44.493 CST,,,14390,,54de1866.3836,3,,2015-02-13 23:29:42 CST,,0,LOG,00000,"archive command failed with exit code 1","The failed archive command was: test ! -f /var/lib/pgsql/pg_archive/00000003.history && cp pg_xlog/00000003.history /var/lib/pgsql/pg_archive/00000003.history",,,,,,,,""
2015-02-13 23:29:44.493 CST,,,14390,,54de1866.3836,4,,2015-02-13 23:29:42 CST,,0,WARNING,01000,"archiving transaction log file ""00000003.history"" failed too many times, will try again later",,,,,,,,,""
[postgres@fnddb data]$ mkdir /var/lib/pgsql/pg_archive/
```

> 日志中可以看到是进行了PITR恢复,并生成了新的Time line ID:3
> 停止在了这里: recovery stopping before commit of transaction 1837, time 2015-02-13 20:05:04.931385+08

*报错是:归档目录之前模拟破坏被删除*

#### 进行确认检查

```
[postgres@fnddb ~]$ psql database1
psql (9.4.1)
Type "help" for help.

database1=# \dt
           List of relations
 Schema |    Name    | Type  |  Owner
--------+------------+-------+----------
 public | t3         | table | hippo
 public | t4         | table | postgres
 public | t_19_59_28 | table | postgres
(3 rows)

database1=# select count(*) from t_19_59_28;
 count
-------
 10000
(1 row)
```

#### 设置服务器可以接受外部连接

*修改pg_hba.conf配置文件*

## 注意点

- pg_xlog, pg_replslot 备份的时候最好能够只排除里面的内容,文件夹保留
- 恢复的时候最好检查一下表空间连接及实际的表空间目录是否恢复正确.如需要更改路径,需要先修改软连接
- 归档目录也需要检查一下,恢复前先确认

未完待续......

//END

