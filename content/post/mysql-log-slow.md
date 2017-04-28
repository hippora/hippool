+++
categories = ["mysql"]
date = "2015-10-30T10:56:36+08:00"
description = ""
tags = ["General Query","Slow Query"]
thumbnail = ""
title = "mysql-普通查询(General Query)慢查询(Slow Query)相关日志配置"

+++

普通查询(General Query)慢查询(Slow Query)相关日志配置,日志的作用主要就是诊断,排错,优化.

<!--more-->

## 配置

### 配置方法一: 服务启动时

```sh
# vi /etc/my.cnf
...
log-output=TABLE,FILE
general-log=1
slow-query-log=1

# systemctl restart mysqld
```

> `log-output`默认是FILE,还有个值是NONE,就不输出日志了.我这里演示的是表和日志文件都输出.

### 配置方法二: 运行时

```sh
mysql> set global log_output="table", global general_log=on, global slow_query_log=on;
```

## 查看


查看相关变量:

```sh
mysql> show variables like 'general%';
+------------------+--------------------------+
| Variable_name    | Value                    |
+------------------+--------------------------+
| general_log      | ON                       |
| general_log_file | /var/lib/mysql/srv00.log |
+------------------+--------------------------+
2 rows in set (0.00 sec)

mysql> show variables like 'slow%';
+---------------------+-------------------------------+
| Variable_name       | Value                         |
+---------------------+-------------------------------+
| slow_launch_time    | 2                             |
| slow_query_log      | ON                            |
| slow_query_log_file | /var/lib/mysql/srv00-slow.log |
+---------------------+-------------------------------+
3 rows in set (0.00 sec)
```

相关日志文件:

`general_log_file`和`slow_query_log_file`变量指示的文件,可以按需要进行修改

日志相关的表:

```sh
mysql> select * from mysql.general_log;
mysql> select * from mysql.slow_log;
```

## 维护

日志文件

```sh
mysql> set global general_log=off;
mysql> \! mv /var/lib/mysql/srv00.log /var/lib/mysql/srv00.log.bak
mysql> set global general_log=on;
```

或者:

```sh
mv /var/lib/mysql/srv00.log /var/lib/mysql/srv00.log.bak
mv /var/lib/mysql/srv00-slow.log /var/lib/mysql/srv00-slow.log.bak
mysqladmin flush-logs
```

修改日志表原理也是一样,先暂停,维护表,再启用

## slow query log相关参数

```sh
mysql> show variables like 'long%';
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+
1 row in set (0.01 sec)
```

> 超过这个秒数的慢查询才记录

```sh
mysql> show variables like 'min%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| min_examined_row_limit | 0     |
+------------------------+-------+
1 row in set (0.00 sec)
```

> 返回记录数超过才记录

```sh
mysql> show variables like 'log_slow_admin%';
+---------------------------+-------+
| Variable_name             | Value |
+---------------------------+-------+
| log_slow_admin_statements | OFF   |
+---------------------------+-------+
1 row in set (0.00 sec)
```

> 是否记录管理类型的sql, 包括:`ALTER TABLE`, `ANALYZE TABLE`, `CHECK TABLE`, `CREATE INDEX`, `DROP INDEX`, `OPTIMIZE TABLE`, `REPAIR TABLE`.

```sh
mysql> show variables like '%not_using_indexes';
+----------------------------------------+-------+
| Variable_name                          | Value |
+----------------------------------------+-------+
| log_queries_not_using_indexes          | OFF   |
| log_throttle_queries_not_using_indexes | 0     |
+----------------------------------------+-------+
2 rows in set (0.01 sec)
```

> 没有使用索引的sql是否要记录,如果开启会产生很多记录,`log_throttle_queries_not_using_indexes`设置每分钟在此范围内只记录一次.

## slow query log 分析

使用工具`mysqldumpslow`

> 熟悉oracle的可以认为`mysqldumpslow`是oracle的`tkprof`

## 总结

当然开启都会对服务器资源消耗.只在需要的时候开启,不用的时候关掉.

//END

