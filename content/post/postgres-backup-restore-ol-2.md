+++
categories = ["postgresql"]
date = "2015-02-11T16:58:41+08:00"
description = ""
tags = ["postgres","backup","restore","wal"]
thumbnail = ""
title = "postgresql在线备份与恢复(二)"

+++

在线备份必须要在归档模式下.

<!--more-->

**基础备份有两种方法.**

- pg自带的命令行工具,有固定的参数设置
- 更底层的api函数来进行基础备份

## 使用pg_basebackup

> 因为pg_basebackup 使用了replication技术.需要做一些设置
> 1. pg_hba.conf 设置postgres用户replication认证
> 2. 设置max_wal_senders > 0

### 做一个基础备份

```
[postgres@fnddb data]$ pg_basebackup -D /home/postgres/basebk20150211
pg_basebackup: could not connect to server: FATAL:  no pg_hba.conf entry for replication connection from host "[local]", user "postgres"
[postgres@fnddb data]$ vi pg_hba.conf
......
local   replication     postgres                                trust
......
[postgres@fnddb data]$ pg_basebackup -D /home/postgres/basebk20150211
pg_basebackup: could not connect to server: FATAL:  number of requested standby connections exceeds max_wal_senders (currently 0)
[postgres@fnddb data]$ vi postgresql.conf
......
max_wal_senders = 1  --此参数需要重启服务器
[postgres@fnddb data]$ pg_ctl restart
```

如果在本机上备份,表空间不在PGDATA下的话,表空间目录已经被占用,需要做映射

```
[postgres@fnddb data]$ pg_basebackup -D /home/postgres/basebk20150211
pg_basebackup: directory "/var/lib/pgsql/tsdata02" exists but is not empty
[postgres@fnddb data]$ pg_basebackup -D /home/postgres/basebk20150211 -T /var/lib/pgsql/tsdata=/home/postgres/basebk20150211/tsdata -T /var/lib/pgsql/tsdata02=/home/postgres/basebk20150211/tsdata02
NOTICE:  pg_stop_backup complete, all required WAL segments have been archived
```

查看下备份目录下的东西,发现跟PGDATA目录一样,除了多出个backup_label及两个映射的表空间目录

```
[postgres@fnddb data]$ cd ~/basebk20150211/
[postgres@fnddb basebk20150211]$ ll
total 124
-rw-------. 1 postgres postgres   206 Feb 13 00:44 backup_label
drwx------. 6 postgres postgres  4096 Feb 13 00:44 base
......
-rw-------. 1 postgres postgres 21307 Feb 13 00:44 postgresql.conf
drwx------. 3 postgres postgres  4096 Feb 13 00:44 tsdata
drwx------. 3 postgres postgres  4096 Feb 13 00:44 tsdata02
[postgres@fnddb basebk20150211]$ cat backup_label
START WAL LOCATION: 0/A000028 (file 00000001000000000000000A)
CHECKPOINT LOCATION: 0/A000028
BACKUP METHOD: streamed
BACKUP FROM: master
START TIME: 2015-02-13 00:44:05 CST
LABEL: pg_basebackup base backup
```

*默认不备份所需要的归档日志,因此还需要备份一下wal归档*

```
[postgres@fnddb basebk20150211]$ ls pg_xlog --空
archive_status
```

### 备份WAL 归档

**模拟一下数据库运作**

```
[postgres@fnddb pgsql]$ psql database1
psql (9.4.1)
Type "help" for help.

database1=# \dt
       List of relations
 Schema | Name | Type  | Owner
--------+------+-------+-------
 public | t3   | table | hippo
(1 row)

database1=# insert into t3 select generate_series(1,100);
INSERT 0 100
database1=# select pg_switch_xlog();
 pg_switch_xlog
----------------
 0/9002EB0
(1 row)

database1=# insert into t3 select generate_series(1,100);
INSERT 0 100
```

**备份归档WAL日志**

```
[postgres@fnddb pg_archive]$ cd $PGDATA/../pg_archive
[postgres@fnddb pg_archive]$ ll
total 180228
-rw-------. 1 postgres postgres 16777216 Feb 13 00:41 000000010000000000000001
-rw-------. 1 postgres postgres 16777216 Feb 13 00:41 000000010000000000000002
......
-rw-------. 1 postgres postgres 16777216 Feb 13 00:44 000000010000000000000009
-rw-------. 1 postgres postgres 16777216 Feb 13 00:44 00000001000000000000000A
-rw-------. 1 postgres postgres      302 Feb 13 00:44 00000001000000000000000A.00000028.backup
-rw-------. 1 postgres postgres 16777216 Feb 13 00:47 00000001000000000000000B
[postgres@fnddb pg_archive]$ cat 00000001000000000000000A.00000028.backup
START WAL LOCATION: 0/A000028 (file 00000001000000000000000A)
STOP WAL LOCATION: 0/A0000B8 (file 00000001000000000000000A)
CHECKPOINT LOCATION: 0/A000028
BACKUP METHOD: streamed
BACKUP FROM: master
START TIME: 2015-02-13 00:44:05 CST
LABEL: pg_basebackup base backup
STOP TIME: 2015-02-13 00:44:05 CST
```

*这里有一个.backup文件,此文件是发出备份命令后自动生成的文本文件*
*记录了备份文件开始时WAL的位置*
*WAL结束位置,哪个文件,检查点的位置等等.*
*也就是说我们这个位置之前的备份文件,对于我们这次的基础备份来说是不需要的.*

```
[postgres@fnddb pg_archive]$ cd ..
[postgres@fnddb pgsql]$ tar -cjvf ~/basebk20150211_arch.tbz2 pg_archive/
pg_archive/
pg_archive/000000010000000000000004    --0A之前的对于此基础备份无用,这里简化备份我一起打包了
pg_archive/00000001000000000000000B
pg_archive/000000010000000000000003    --无需
pg_archive/000000010000000000000001    --无需
pg_archive/00000001000000000000000A.00000028.backup
pg_archive/00000001000000000000000A
......                                                          --无需
pg_archive/000000010000000000000002    --无需
```

*至此我们备份了必要的归档,我们模拟再生成一些归档*

```
[postgres@fnddb pg_archive]$ psql database1 postgres
psql (9.4.1)
Type "help" for help.

database1=# insert into t3 select generate_series(1,100);
INSERT 0 100
database1=# select pg_switch_xlog();
 pg_switch_xlog
----------------
 0/A007DB0
(1 row)

database1=# insert into t3 select generate_series(1,100);
INSERT 0 100
database1=# select pg_switch_xlog();
 pg_switch_xlog
----------------
 0/B001978
(1 row)

database1=# insert into t3 select generate_series(1,100);
INSERT 0 100
database1=# select count(*) from t3;
 count
-------
  1100
(1 row)
```

> 如果基础备份完成后又修改过参数文件postgresql.conf,pg_hba.conf,pg_ident.conf等等,可以单独再备份一份.

## 使用此备份文件来恢复

**模拟表空间在磁盘上已经损坏丢失**

```
[postgres@fnddb pgsql]$ pwd
/var/lib/pgsql
[postgres@fnddb pgsql]$ rm -rf tsdata
```

### 关闭数据库实例

```
[postgres@fnddb pgsql]$ pg_ctl stop -m fast
waiting for server to shut down.... done
server stopped
```

### 备份cluster下及表空间的文件


*虽然恢复只需要xlog下的日志,因为有相关的日志距上次备份的归档还有部分没有备份的*
*但是只要空间不是问题,强烈不要事先删除cluster及表空间的文件.待恢复完再删也来得及*

```
[postgres@fnddb pgsql]$ mv data data.pgbak
[postgres@fnddb pgsql]$ mv tsdata02 tsdata02.pgbak
```

### 将备份文件还原到相应位置,包括表空间

*我的表空间备份备份时候映射到了备份文件的root目录下*

```
[postgres@fnddb pgsql]$ cp -R ~/basebk20150211 data
```

### copy原先未归档未备份的xlog到当前cluster的pg_xlog目录下

*由于备份进程会优先读取归档目录下的WAL日志,重复没关系*

```
[postgres@fnddb pgsql]$ cp -R data.pgbak/pg_xlog/* data/pg_xlog/
[postgres@fnddb pgsql]$ ll data/pg_xlog/
total 114696
-rw-------. 1 postgres postgres      302 Feb 13 01:01 00000001000000000000000A.00000028.backup
-rw-------. 1 postgres postgres 16777216 Feb 13 01:01 00000001000000000000000E
-rw-------. 1 postgres postgres 16777216 Feb 13 01:01 00000001000000000000000F
-rw-------. 1 postgres postgres 16777216 Feb 13 01:01 000000010000000000000010
-rw-------. 1 postgres postgres 16777216 Feb 13 01:01 000000010000000000000011
-rw-------. 1 postgres postgres 16777216 Feb 13 01:01 000000010000000000000012
-rw-------. 1 postgres postgres 16777216 Feb 13 01:01 000000010000000000000013
-rw-------. 1 postgres postgres 16777216 Feb 13 01:01 000000010000000000000014
drwx------. 2 postgres postgres     4096 Feb 13 01:01 archive_status
```

### cluster目录下创建recovery.conf文件

```
[postgres@fnddb pgsql]$ cd data
[postgres@fnddb data]$ cp /usr/local/pgsql/share/recovery.conf.sample recovery.conf
```

*因为我们的归档没有丢失,所以这里不需要用到备份的归档.否则只能进行PITR恢复*

```
[postgres@fnddb data]$ vi recovery.conf
......
restore_command = 'cp /var/lib/pgsql/pg_archive/%f %p'
```

*暂时里面其他参数还用不到,可以修改pg_hba.conf暂时不让用户连接*

```
[postgres@fnddb data]$ vi pg_hba.conf --设置略
```

### 启动pg服务进程开始恢复

```
[postgres@fnddb data]$ pg_ctl start
```

**查看一下日志**

```
[postgres@fnddb data]$ tail -30 pg_log/postgresql-13_010245.csv
2015-02-13 01:02:45.326 CST,,,8752,,54dcdcb5.2230,1,,2015-02-13 01:02:45 CST,,0,LOG,00000,"ending log output to stderr",,"Future log output will go to log destination ""csvlog"".",,,,,,,""
2015-02-13 01:02:45.329 CST,,,8754,,54dcdcb5.2232,1,,2015-02-13 01:02:45 CST,,0,LOG,00000,"database system was interrupted; last known up at 2015-02-13 00:44:05 CST",,,,,,,,,""
2015-02-13 01:02:45.330 CST,,,8754,,54dcdcb5.2232,2,,2015-02-13 01:02:45 CST,,0,LOG,00000,"starting archive recovery",,,,,,,,,""
2015-02-13 01:02:45.368 CST,,,8754,,54dcdcb5.2232,3,,2015-02-13 01:02:45 CST,,0,LOG,00000,"restored log file ""00000001000000000000000A"" from archive",,,,,,,,,""
2015-02-13 01:02:45.372 CST,,,8754,,54dcdcb5.2232,4,,2015-02-13 01:02:45 CST,,0,LOG,00000,"redo starts at 0/A000090",,,,,,,,,""
2015-02-13 01:02:45.373 CST,,,8754,,54dcdcb5.2232,5,,2015-02-13 01:02:45 CST,,0,LOG,00000,"consistent recovery state reached at 0/A0000B8",,,,,,,,,""
2015-02-13 01:02:45.409 CST,,,8754,,54dcdcb5.2232,6,,2015-02-13 01:02:45 CST,,0,LOG,00000,"restored log file ""00000001000000000000000B"" from archive",,,,,,,,,""
2015-02-13 01:02:45.451 CST,,,8754,,54dcdcb5.2232,7,,2015-02-13 01:02:45 CST,,0,LOG,00000,"restored log file ""00000001000000000000000C"" from archive",,,,,,,,,""
2015-02-13 01:02:45.489 CST,,,8754,,54dcdcb5.2232,8,,2015-02-13 01:02:45 CST,,0,LOG,00000,"restored log file ""00000001000000000000000D"" from archive",,,,,,,,,""
2015-02-13 01:02:45.530 CST,,,8754,,54dcdcb5.2232,9,,2015-02-13 01:02:45 CST,,0,LOG,00000,"restored log file ""00000001000000000000000E"" from archive",,,,,,,,,""
2015-02-13 01:02:45.545 CST,,,8754,,54dcdcb5.2232,10,,2015-02-13 01:02:45 CST,,0,LOG,00000,"record with zero length at 0/F000090",,,,,,,,,""
2015-02-13 01:02:45.545 CST,,,8754,,54dcdcb5.2232,11,,2015-02-13 01:02:45 CST,,0,LOG,00000,"redo done at 0/F000028",,,,,,,,,""
2015-02-13 01:02:45.545 CST,,,8754,,54dcdcb5.2232,12,,2015-02-13 01:02:45 CST,,0,LOG,00000,"last completed transaction was at log time 2015-02-13 00:53:30.033289+08",,,,,,,,,""
2015-02-13 01:02:45.566 CST,,,8754,,54dcdcb5.2232,13,,2015-02-13 01:02:45 CST,,0,LOG,00000,"selected new timeline ID: 2",,,,,,,,,""
2015-02-13 01:02:45.665 CST,,,8754,,54dcdcb5.2232,14,,2015-02-13 01:02:45 CST,,0,LOG,00000,"archive recovery complete",,,,,,,,,""
2015-02-13 01:02:45.769 CST,,,8767,,54dcdcb5.223f,1,,2015-02-13 01:02:45 CST,,0,LOG,00000,"autovacuum launcher started",,,,,,,,,""
2015-02-13 01:02:45.770 CST,,,8752,,54dcdcb5.2230,2,,2015-02-13 01:02:45 CST,,0,LOG,00000,"database system is ready to accept connections",,,,,,,,,""
```

*检查下数据完整性*

```
[postgres@fnddb pgsql]$ psql -c "select count(*) from t3" database1
 count
-------
  1100
(1 row)
```

### 表空间位置转移(可选)

```
[postgres@fnddb pg_tblspc]$ ll
total 0
lrwxrwxrwx. 1 postgres postgres 36 Feb 13 00:55 16392 -> /home/postgres/basebk20150211/tsdata
lrwxrwxrwx. 1 postgres postgres 38 Feb 13 00:55 16393 -> /home/postgres/basebk20150211/tsdata02
```

*还是备份的目录,是个软连接,我们把它改回老地方,为了数据不损坏,我们先关闭数据库*

```
[postgres@fnddb pg_tblspc]$ pwd
/var/lib/pgsql/data/pg_tblspc
[postgres@fnddb pg_tblspc]$ pg_ctl stop -m fast
waiting for server to shut down.... done
server stopped
[postgres@fnddb pg_tblspc]$ mv /home/postgres/basebk20150211/tsdata /var/lib/pgsql/
[postgres@fnddb pg_tblspc]$ mv /home/postgres/basebk20150211/tsdata02 /var/lib/pgsql/
```

*因为目录已经被移走,我的centos下会一闪一闪,我们把它重新连接一下*

```
[postgres@fnddb pg_tblspc]$ rm 16392
[postgres@fnddb pg_tblspc]$ rm 16393
[postgres@fnddb pg_tblspc]$ ln -s /var/lib/pgsql/tsdata 16392
[postgres@fnddb pg_tblspc]$ ln -s /var/lib/pgsql/tsdata02 16393
[postgres@fnddb pg_tblspc]$ pg_ctl start
server starting
```

**测试下表空间是否可用**

```
[postgres@fnddb pg_log]$ psql database1
psql (9.4.1)
Type "help" for help.

database1=# create table t4(id text) tablespace tsdata;
CREATE TABLE
database1=# insert into t4 select generate_series(1,100);
INSERT 0 100
```

> 为了防止重复进入recovery模式,一旦恢复完成,pg会将recovery.conf改成recovery.done

### 恢复现场

**这里主要关注的是之前改过pg_hba.conf文件的话,需要改回来使用户可以正常使用数据库**

### 其他概念点

*数据库一旦恢复,就会生成一个新的时间轴,PG中叫做Timeline,oracle里叫incarnation*

```
[postgres@fnddb pgsql]$ cd pg_archive/
[postgres@fnddb pg_archive]$ ll
total 245768
......
-rw-------. 1 postgres postgres 16777216 Feb 13 01:02 00000001000000000000000F
-rw-------. 1 postgres postgres       41 Feb 13 01:02 00000002.history
[postgres@fnddb pg_archive]$ cat 00000002.history
1       0/F000090       no recovery target specified
```

**我们尝试生成一个xlog**

```
[postgres@fnddb pg_archive]$ psql -c "select pg_switch_xlog()"
 pg_switch_xlog
----------------
 0/F000118
(1 row)

[postgres@fnddb pg_archive]$ ll
total 262152
......
-rw-------. 1 postgres postgres 16777216 Feb 13 01:02 00000001000000000000000F
-rw-------. 1 postgres postgres 16777216 Feb 13 01:06 00000002000000000000000F
-rw-------. 1 postgres postgres       41 Feb 13 01:02 00000002.history
```

*timeline主要的目的一个就是生成的归档不会覆盖原来的日志文件.*
*还有一个就是基于时间点的恢复.可以有机会再次还原到上一时间轴上的某一个时间点,这个时间点在你还原的时间点之后.*
*基于Timeline的恢复等有时间再做下试验*

未完待续......

//END

