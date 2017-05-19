+++
categories = ["postgresql"]
date = "2017-04-29T22:35:00+08:00"
description = ""
tags = ["install"]
thumbnail = ""
title = "postgres 9.6安装"

+++

使用yum安装

<!--more-->

## 安装Repository RPM

https://yum.postgresql.org/repopackages.php

```
[root@srv00 ~]# curl -#O https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-7-x86_64/pgdg-centos96-9.6-3.noarch.rpm
[root@srv00 ~]# yum install epel-release -y
[root@srv00 ~]# rpm -ivh pgdg-centos96-9.6-3.noarch.rpm
warning: pgdg-centos96-9.6-3.noarch.rpm: Header V4 DSA/SHA1 Signature, key ID 442df0f8: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:pgdg-centos96-9.6-3              ################################# [100%]
[root@srv00 ~]# yum install postgresql96-server postgresql96-contrib postgresql96-docs -y
```

## 初始化cluster

```
[root@srv00 ~]# /usr/pgsql-9.6/bin/postgresql96-setup initdb
Initializing database ... OK
```

## 启动

```
[root@srv00 ~]# systemctl enable postgresql-9.6
[root@srv00 ~]# systemctl start postgresql-9.6
[root@srv00 ~]# systemctl status postgresql-9.6
● postgresql-9.6.service - PostgreSQL 9.6 database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql-9.6.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2017-04-29 23:11:10 CST; 8s ago
  Process: 16631 ExecStartPre=/usr/pgsql-9.6/bin/postgresql96-check-db-dir ${PGDATA} (code=exited, status=0/SUCCESS)
 Main PID: 16637 (postmaster)
   CGroup: /system.slice/postgresql-9.6.service
           ├─16637 /usr/pgsql-9.6/bin/postmaster -D /var/lib/pgsql/9.6/data/
           ├─16639 postgres: logger process
           ├─16641 postgres: checkpointer process
           ├─16642 postgres: writer process
           ├─16643 postgres: wal writer process
           ├─16644 postgres: autovacuum launcher process
           └─16645 postgres: stats collector process

Apr 29 23:11:10 srv00 systemd[1]: Starting PostgreSQL 9.6 database server...
Apr 29 23:11:10 srv00 postmaster[16637]: < 2017-04-29 23:11:10.765 CST > LOG:  redirecting log output to logging collector process
Apr 29 23:11:10 srv00 postmaster[16637]: < 2017-04-29 23:11:10.765 CST > HINT:  Future log output will appear in directory "pg_log".
Apr 29 23:11:10 srv00 systemd[1]: Started PostgreSQL 9.6 database server.
```

## 测试

```
-bash-4.2$ psql -c '\l'
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(3 rows)
```

//END
