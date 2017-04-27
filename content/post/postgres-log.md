+++
categories = ["postgresql"]
date = "2015-02-11T16:48:26+08:00"
description = ""
tags = ["postgres","log"]
thumbnail = ""
title = "postgresql 修改日志输出"

+++

刚开始安装PG时,默认的服务日志是输出到stderr的, 我们通过重定向到一个文件中来保存.这种方式不太合理并且会无限增长. pg自带了日志管理.我们来简单设置一下.

<!--more-->

## 修改服务配置文件

**我准备配置成这样:**  

1. 输出格式为csv,这样可以方便使用或者导入到表中查询.  
2. 日志只保留一个月.  
3. 每天至少新生产一个日志.  
4. 一旦一个日志满100M,就生成新的日志文件,防止过大.  

修改下列参数并重启生效:  

```
[postgres@fnddb pgsql]$ cd $PGDATA
[postgres@fnddb data]$ vi postgresql.conf
......
log_destination = 'csvlog'              # Valid values are combinations of
logging_collector = on                  # Enable capturing of stderr and csvlog
log_directory = 'pg_log'                # directory where log files are written,
log_filename = 'postgresql-%d_%H%M%S.log'       # log file name pattern,
log_file_mode = 0600                    # creation mode for log files,
log_truncate_on_rotation = on           # If on, an existing log file with the
log_rotation_age = 1d                   # Automatic rotation of logfiles will
log_rotation_size = 100MB               # Automatic rotation of logfiles will
......
:wq

[postgres@fnddb data]$ pg_ctl reload
server signaled
```

找到之前重定向的日志文件查看  

```
[postgres@fnddb data]$ cd ..
[postgres@fnddb pgsql]$ pwd
/var/lib/pgsql
[postgres@fnddb pgsql]$ ls
data  pgsql.log  tsdata  tsdata01  tsdata02
[postgres@fnddb pgsql]$ tail -f pgsql.log 
......
LOG:  received SIGHUP, reloading configuration files
LOG:  parameter "log_destination" changed to "csvlog"
LOG:  parameter "logging_collector" cannot be changed without restarting the server
LOG:  parameter "log_filename" changed to "postgresql-%d_%H%M%S.log"
LOG:  parameter "log_truncate_on_rotation" changed to "on"
LOG:  parameter "log_rotation_size" changed to "100MB"
LOG:  configuration file "/var/lib/pgsql/data/postgresql.conf" contains errors; unaffected changes were applied
^C
```

logging_collector参数必须要重启才能生效.我们重启一下.

```
[postgres@fnddb pgsql]$ pg_ctl restart
waiting for server to shut down.... done
server stopped
server starting
[postgres@fnddb pgsql]$ LOG:  redirecting log output to logging collector process
HINT:  Future log output will appear in directory "pg_log".
```

控制台提示了之后的日志将会放在pg_log目录下.

```
[postgres@fnddb pg_log]$ cd $PGDATA/pg_log
[postgres@fnddb pg_log]$ ll
total 8
-rw-------. 1 postgres postgres 936 Feb 11 21:49 postgresql-11_200630.csv
-rw-------. 1 postgres postgres  96 Feb 11 20:06 postgresql-11_200630.log
```

## 把日志文件导入到table中测试

官方文档给了个默认csv格式的表,我们来测试一下

```
[postgres@fnddb pg_log]$ psql -U hippo database1
psql (9.4.1)
Type "help" for help.

database1=> CREATE TABLE postgres_log
database1-> (
database1(>   log_time timestamp(3) with time zone,
database1(>   user_name text,
database1(>   database_name text,
database1(>   process_id integer,
database1(>   connection_from text,
database1(>   session_id text,
database1(>   session_line_num bigint,
database1(>   command_tag text,
database1(>   session_start_time timestamp with time zone,
database1(>   virtual_transaction_id text,
database1(>   transaction_id bigint,
database1(>   error_severity text,
database1(>   sql_state_code text,
database1(>   message text,
database1(>   detail text,
database1(>   hint text,
database1(>   internal_query text,
database1(>   internal_query_pos integer,
database1(>   context text,
database1(>   query text,
database1(>   query_pos integer,
database1(>   location text,
database1(>   application_name text,
database1(>   PRIMARY KEY (session_id, session_line_num)
database1(> );
CREATE TABLE
database1=> COPY postgres_log FROM '/var/lib/postgres/data/pg_log/postgresql-11_200630.csv' WITH csv; --copy命令只能superuser来使用
ERROR:  must be superuser to COPY to or from a file
HINT:  Anyone can COPY to stdout or from stdin. psql's \copy command also works for anyone.
```

*使用\copy命令再执行下*

```
database1=> \copy postgres_log FROM '/var/lib/pgsql/data/pg_log/postgresql-11_200630.csv' WITH csv;
COPY 7
database1=> \x
Expanded display is on.
database1=> select * from postgres_log;
-[ RECORD 1 ]----------+------------------------------------------------------------------------------------------
log_time               | 2015-02-12 10:06:30.654+08
user_name              | 
database_name          | 
process_id             | 24167
connection_from        | 
session_id             | 54db45c6.5e67
session_line_num       | 1
command_tag            | 
session_start_time     | 2015-02-12 10:06:30+08
virtual_transaction_id | 
transaction_id         | 0
error_severity         | LOG
sql_state_code         | 00000
message                | ending log output to stderr
detail                 | 
hint                   | Future log output will go to log destination "csvlog".
internal_query         | 
internal_query_pos     | 
context                | 
query                  | 
query_pos              | 
location               | 
application_name       | 
-[ RECORD 2 ]----------+------------------------------------------------------------------------------------------
log_time               | 2015-02-12 10:06:30.657+08
......
```

## 关于log相关参数

```
[postgres@fnddb pg_log]$ grep -E '^#?log' $PGDATA/postgresql.conf 
log_destination = 'csvlog'              # Valid values are combinations of
logging_collector = on                  # Enable capturing of stderr and csvlog
log_directory = 'pg_log'                # directory where log files are written,
log_filename = 'postgresql-%d_%H%M%S.log'       # log file name pattern,
log_file_mode = 0600                    # creation mode for log files,
log_truncate_on_rotation = on           # If on, an existing log file with the
log_rotation_age = 1d                   # Automatic rotation of logfiles will
log_rotation_size = 100MB               # Automatic rotation of logfiles will
#log_min_messages = warning             # values in order of decreasing detail:
#log_min_error_statement = error        # values in order of decreasing detail:
#log_min_duration_statement = -1        # -1 is disabled, 0 logs all statements
#log_checkpoints = off
#log_connections = off
#log_disconnections = off
#log_duration = off
#log_error_verbosity = default          # terse, default, or verbose messages
#log_hostname = off
#log_line_prefix = ''                   # special values:
log_lock_waits = on                     # log lock waits >= deadlock_timeout
#log_statement = 'none'                 # none, ddl, mod, all
#log_temp_files = -1                    # log temporary files equal or larger
log_timezone = 'PRC'
#log_parser_stats = off
#log_planner_stats = off
#log_executor_stats = off
#log_statement_stats = off
#log_autovacuum_min_duration = -1       # -1 disables, 0 logs all actions and
```

//END

