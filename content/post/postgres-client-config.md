+++
categories = ["postgresql"]
date = "2015-02-04T15:32:14+08:00"
description = ""
tags = ["postgres","pgAdmin3"]
thumbnail = ""
title = "外部客户端连接postgresql相关设置"

+++

用pgAdmin3来连接测试. 这样配置肯定不是生产库的配置.但是我们不能跌倒在入门上,先连上去再说:)

<!--more-->


# 修改服务配置文件postgresql.conf中的监听参数

*默认配置文件在PGDATA目录下(除了你是通过服务进程中的参数config_file来指定的)*

```
[postgres@fnddb data]$ vi postgresql.conf
......
listen_addresses = '*'                  # what IP address(es) to listen on;
......
```

# 修改pg_hba.conf文件,添加一行host认证

*默认配置文件在PGDATA目录中,可以通过服务配置文件 postgresql.conf中hba_file参数来修改到其他位置*

```
[postgres@fnddb data]$ vi pg_hba.conf
......
host    all             all             192.168.10.0/24         md5
......
```

*192.168.10.0/24是我的客户机所在网段*

# 重启生效一下

```
[postgres@fnddb data]$ pg_ctl restart -l /var/lib/pgsql/pgsql.log
waiting for server to shut down...LOG:  received smart shutdown request
.LOG:  autovacuum launcher shutting down
.................................LOG:  shutting down
LOG:  database system is shut down
 done
server stopped
server starting
```

# 新建个用户

*psql默认是postgres超级用户*

```
[postgres@fnddb data]$ psql
psql (9.4.0)
Type "help" for help.

postgres=# create user hippo password 'hippo';
CREATE ROLE
postgres-# \q
```

# 测试连接

![pgadmin-test](/img/postgres/pgadmin-test.png)

//END
