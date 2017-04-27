+++
categories = ["postgresql"]
date = "2015-02-11T16:53:32+08:00"
description = ""
tags = ["backup","restore","wal"]
thumbnail = ""
title = "postgresql在线备份与恢复(一)"

+++

PG默认是不开启归档的.数据库服务器它只维护了一些WAL日志..

<!--more-->

## WAL(write ahead log)

wal日志记录了对cluster下所有数据库的改变.默认在目录pg_xlog下,称作segment file,默认16M.
一旦数据库崩溃,只需要将数据库级别的备份恢复,并且将wal重新'回放',就可以让数据库回到最新的一致性状态中.
可以理解成ORACLE的在线日志,主要作用是用来数据库崩溃时的恢复.
未开启归档时,pg只维护一些日志为了数据库崩溃恢复使用.一旦这个segment file已经在检查点时间之前,随时都有可能被pg重复利用.

**WAL的主要作用有:**

1. 日常在线备份恢复使用.无需关闭数据库,备份整个cluster及相关wal文件,即使cluster文件备份不一致,也可以通过'回放'wal来让数据库保持一致状态
2. 非常大的数据库不可能经常做全备,但是持续备份wal日志就可以进行恢复.这对于大数据库非常有用.
3. 可以通过'回放'wal到某一个时间点,来使数据库回到某一个过去,对于对数据库做了误操作有用.
4. 可以将wal持续给另一台使用了相同备份文件的数据库服务器,来让另一台数据库服务器持续'回放',做成热备库.

**为了我们备份恢复需要,一定要开启归档.将wal日志持续的备份起来.**

## 配置归档模式(WAL Archiving)

*修改服务参数*

```
[postgres@fnddb data]$ mkdir $PGDATA/../pg_archive
[postgres@fnddb data]$ vi $PGDATA/postgresql.conf
......
wal_level = archive
archive_mode = on
archive_command = 'test ! -f /var/lib/pgsql/pg_archive/%f && cp %p /var/lib/pgsql/pg_archive/%f'
......
:wq
[postgres@fnddb data]$ pg_ctl restart
waiting for server to shut down.... done
server stopped
server starting
[postgres@fnddb data]$ LOG:  redirecting log output to logging collector process
HINT:  Future log output will appear in directory "pg_log".
```

> archive_command参数调用的可以是命令也可以是脚本.灵活性很高.
> 每次wal日志切换会执行此命令.如果命令返回失败,则认为归档失败,pg会定期地重复执行直到成功为止.

*检查一下是否可以正确归档,进行手动日志切换*

```
[postgres@fnddb data]$ psql
psql (9.4.1)
Type "help" for help.

postgres=# select pg_switch_xlog();
 pg_switch_xlog
----------------
 0/1B7CCB0
(1 row)

postgres=# \q
[postgres@fnddb data]$ ls ../pg_archive/
000000010000000000000001
```

未完待续......

//END

