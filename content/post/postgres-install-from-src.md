+++
categories = ["postgresql"]
date = "2015-02-04T15:17:34+08:00"
description = ""
tags = ["postgres"]
thumbnail = ""
title = "postgresql 在linux上的源码安装"

+++

学习postgresql 记录在centos上安装postgresql,使用源码编译.数据库版本是9.4.0 编译使用默认configure选项

<!--more-->

# 下载源码并解压

```bash
[root@fnddb ~]# wget https://ftp.postgresql.org/pub/source/v9.4.0/postgresql-9.4.0.tar.bz2
[root@fnddb ~]# tar -xjvf postgresql-9.4.0.tar.bz2
[root@fnddb ~]# cd postgresql-9.4.0
```

# 开始编译安装

```
[root@fnddb postgresql-9.4.0]# ./configure
……
checking for library containing shmget... none required
checking for library containing readline... no
configure: error: readline library not found
If you have readline already installed, see config.log for details on the
failure.  It is possible the compiler isn't looking in the proper directory.
Use --without-readline to disable readline support.
```

# 按照错误提示依次安装依赖包

```
[root@fnddb postgresql-9.4.0]# yum install readline-devel
[root@fnddb postgresql-9.4.0]# yum install zlib-devel
...
[root@fnddb postgresql-9.4.0]# ./configure
[root@fnddb postgresql-9.4.0]# make
……
All of PostgreSQL successfully made. Ready to install.
[root@fnddb postgresql-9.4.0]# make install
……
PostgreSQL installation complete.
```

# 添加用户

```
[root@fnddb postgresql-9.4.0]# useradd postgres
[root@fnddb postgresql-9.4.0]# passwd postgres
Changing password for user postgres.
New password:
BAD PASSWORD: it is based on a dictionary word
Retype new password:
passwd: all authentication tokens updated successfully.
```

# 建立好database cluster目标文件夹

```
[root@fnddb postgresql-9.4.0]# mkdir /var/lib/pgsql/data -p
[root@fnddb postgresql-9.4.0]# chown -R postgres /var/lib/pgsql
```

# 环境变量设置

```
[root@fnddb postgresql-9.4.0]# su - postgres
[postgres@fnddb ~]$ vi .bash_profile
…
# postgres
PGDATA=/var/lib/pgsql/data
PATH=/usr/local/pgsql/bin:$PATH
export PGDATA PATH

[postgres@fnddb ~]$ . .bash_profile
```

# 创建database cluster

```
[postgres@fnddb ~]$ pg_ctl initdb
......

WARNING: enabling "trust" authentication for local connections
You can change this by editing pg_hba.conf or using the option -A, or
--auth-local and --auth-host, the next time you run initdb.

Success. You can now start the database server using:

    /usr/local/pgsql/bin/postgres -D /var/lib/pgsql/data
or
    /usr/local/pgsql/bin/pg_ctl -D /var/lib/pgsql/data -l logfile start
```

# 启动数据库实例

*设置好PGDATA环境变量后,可以不带-D选项*

```
[postgres@fnddb ~]$ pg_ctl start -l /var/lib/pgsql/pgsql.log
server starting
```

# 关闭数据库实例

```
[postgres@fnddb ~]$ pg_ctl stop
waiting for server to shut down.... done
server stopped
```

# 开机自动启动设置

```
[root@fnddb postgresql-9.4.0]# vi /etc/rc.local
…
su - c '/usr/local/pgsql/bin/pg_ctl start -D /var/lib/pgsql/data -l /var/lib/pgsql/pgsql.log'
```

//END


