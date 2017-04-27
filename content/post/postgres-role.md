+++
categories = ["postgresql"]
date = "2015-02-09T16:17:00+08:00"
description = ""
tags = ["postgres","role"]
thumbnail = ""
title = "postgresql role(角色)"

+++

在PG中,用户是通过role来表示的.role也可以表示成一组用户.
角色与用户的概念比较模糊,可以认为带LOGIN属性的role就是用户.

<!--more-->

## 创建role

**带了login属性.就可以登录数据库.**

```
postgres=# create role role1;
CREATE ROLE
postgres=# \c - role1
FATAL:  role "role1" is not permitted to log in
Previous connection kept
postgres=# alter role role1 login;
ALTER ROLE
postgres=# \c - role1
You are now connected to database "postgres" as user "role1".
```

> create user role1 与create role role1 login 是等价的,避免混淆,我只记create role方式.

role的系统视图是pg_roles

```
postgres=> select rolname,rolsuper,rolcanlogin from pg_roles;
 rolname  | rolsuper | rolcanlogin
----------+----------+-------------
 postgres | t        | t
 hippo    | f        | t
 user2    | f        | t
 user1    | t        | t
 role1    | f        | t
```

在使用initdb初始化cluster时,默认会创建一个superuser,名字是将执行initdb命令的操作系统用户一样的用户,通常叫postgres

命令行工具如psql,pg_dump等都需要指定连接用户及连接数据库.默认用户是操作系统用户,默认数据库名字跟连接用户名保持一致.

**指定数据库名字**

```
[postgres@fnddb ~]$ psql -d database1
psql (9.4.1)
Type "help" for help.

database1=# \c
You are now connected to database "database1" as user "postgres".
```

**指定用户名**

```
[postgres@fnddb ~]$ psql -U role1 --不指定数据库名字,默认数据库跟用户名一致,所以找不到
psql: FATAL:  database "role1" does not exist
```

## role属性

可以认为是这个用户所具有的系统权限.

- LOGIN --具有登录权限
- SUPERUSER --超级用户,具有所有系统权限,除了登录验证
- CREATEDB --创建数据库权限
- CREATEROLE --创建role权限
- PASSWORD --设置密码

**修改属性**

```
postgres=# create role role2 login;
CREATE ROLE

postgres=# select * from pg_user where usename = 'role2';
 usename | usesysid | usecreatedb | usesuper | usecatupd | userepl |  passwd  | valuntil | useconfig
---------+----------+-------------+----------+-----------+---------+----------+----------+-----------
 role2   |    16494 | f           | f        | f         | f       | ******** |          |
(1 row)

postgres=# alter role role2 createdb createrole password 'rolepasswd';
ALTER ROLE
postgres=# \du role2
                 List of roles
 Role name |       Attributes       | Member of
-----------+------------------------+-----------
 role2     | Create role, Create DB | {}

postgres=# alter role role2 nocreatedb nocreaterole superuser;
ALTER ROLE
postgres=# \du role2
           List of roles
 Role name | Attributes | Member of
-----------+------------+-----------
 role2     | Superuser  | {}
```

## role的参数

可以修改用户的参数,来影响某用户操作数据库的特殊行为.这部分在讲服务器参数修改时已提及.

```
postgres=# alter role role2 set enable_indexscan = f;
ALTER ROLE
```

## role membership(role 成员)

为了管理上的方便,我们可以创建一个role group,然后可以将各用户或者有特殊权限的role组织在一起,各个role就是这个role group的membership.
role group 是不带login的role,因为pg使用role来表示所有的角色,用户,用户组,所以不要混淆,创建语句都是create role.我们来测试一下.

**我们创建一个用户,两个角色,分别有直属一个表的查询权限**

```
postgres=# create role jack login inherit;
CREATE ROLE
postgres=# create role r1;
CREATE ROLE
postgres=# create role r2;
CREATE ROLE
postgres=# \c database1
You are now connected to database "database1" as user "postgres".
database1=# create table tab1(id text);
CREATE TABLE
database1=# create table tab2(id text);
CREATE TABLE
database1=# create table tab3 (id text);
CREATE TABLE
database1=# grant select on tab1 to r1;
GRANT
database1=# grant select on tab2 to r2;
GRANT
database1=# grant select on tab3 to jack;
GRANT
```

**进行grant授权,使jack成为r1,r2的membership**

```
database1=# grant r1 to jack;
GRANT ROLE
database1=# grant r2 to jack;
GRANT ROLE
database1=# grant usage on schema public to public; --授权usage给所有用户(后一个public),否则看不到数据库中的表.
GRANT
```

**测试角色切换**

*jack继承了r1,r2的权限*

```
database1=# \c - jack
You are now connected to database "database1" as user "jack".
database1=> select * from tab3;
 id
----
(0 rows)

database1=> select * from tab1;
 id
----
(0 rows)

database1=> select * from tab2;
 id
----
(0 rows)
```

*间接继承的也可以*

```
database1=> \c - postgres
You are now connected to database "database1" as user "postgres".
database1=# revoke r2 from jack;
REVOKE ROLE
database1=# grant r2 to r1;
GRANT ROLE
database1=# \c - jack;
You are now connected to database "database1" as user "jack".
database1=> select * from tab2;
 id
----
(0 rows)
```

*关闭r1的继承*

```
database1=> \c - postgres
You are now connected to database "database1" as user "postgres".
database1=# alter role r1 noinherit;
ALTER ROLE
database1=# \c - jack;
You are now connected to database "database1" as user "jack".
database1=> select * from tab2; --已经查询不了r2的权限
ERROR:  permission denied for relation tab2
database1=> select * from tab1;
 id
----
(0 rows)
```

*直接切换到r2角色,你已经不是jack了:)*

```
database1=> set role r1;
SET
database1=> select * from tab1;
 id
----
(0 rows)

database1=> select * from tab2;
ERROR:  permission denied for relation tab2
database1=> select * from tab3;
ERROR:  permission denied for relation tab3
```

*授权不能形成回路*

```
database1=> \c - postgres
You are now connected to database "database1" as user "postgres".
database1=# \du
                             List of roles
 Role name |                   Attributes                   | Member of
-----------+------------------------------------------------+-----------
 hippo     |                                                | {}
 jack      |                                                | {r1}
 postgres  | Superuser, Create role, Create DB, Replication | {}
 r1        | No inheritance, Cannot login                   | {r2}
 r2        | Cannot login                                   | {}
 user1     | Superuser, Create role, Create DB              | {}
 user2     | Create DB                                      | {}

database1=# grant jack to r2;
ERROR:  role "jack" is a member of role "r2"
```

*系统权限任何时候都不会继承,只有主动set过去才生效*

```
database1=# alter role r1 createrole;
ALTER ROLE
database1=# \c - jack;
You are now connected to database "database1" as user "jack".
database1=> create role jacktest1;
ERROR:  permission denied to create role
database1=> set role r1;
SET
database1=> create role jacktest1;
CREATE ROLE
```

*三种方式还原到最初的jack角色*

```
database1=> set role jack;
SET
database1=> set role none;
SET
database1=> reset role;
RESET
```

## 角色删除

*在什么角色下建的对象,归属于哪个角色,而非登录者*

```
database1=> \c - postgres
You are now connected to database "database1" as user "postgres".
database1=# grant create on database database1 to r1;
GRANT
database1=# \c - jack
You are now connected to database "database1" as user "jack".
database1=> set role r1;
SET
database1=> create table tab4(id text);
CREATE TABLE
database1=> \dt tab4    --这里要注意:owner变成了r1而不是jack
       List of relations
 Schema | Name | Type  | Owner
--------+------+-------+-------
 public | tab4 | table | r1
(1 row)
```

*删除role,role下有权限或者是对象属于此role,则删除不了*

```
database1=> \c - postgres
You are now connected to database "database1" as user "postgres".
database1=# drop role r1;
ERROR:  role "r1" cannot be dropped because some objects depend on it
DETAIL:  owner of table tab4
privileges for database database1
privileges for table tab1
```

*移除掉相关权限关联后进行删除*

```
database1=# drop table tab1;
DROP TABLE
database1=# drop table tab4;
DROP TABLE
database1=# revoke create on database database1 from r1;
REVOKE
database1=# drop role r1;
DROP ROLE
```

*涉及到r1的成员或者是角色租(role group) 自动释放*

```
atabase1=# \du
                            List of roles
Role name |                   Attributes                   | Member of
----------+------------------------------------------------+-----------
hippo     |                                                | {}
jack      |                                                | {}
jacktest1 | Cannot login                                   | {}
postgres  | Superuser, Create role, Create DB, Replication | {}
r2        | Cannot login                                   | {}
user1     | Superuser, Create role, Create DB              | {}
user2     | Create DB                                      | {}
```

## ROLE总结

1. PG中的role包含了用户,角色,角色组,成员等所有含义.都使用create role来创建.
2. 一个role可以成为多个role的成员,根据role的inherit属性来决定是否集成其他role的各种权限
3. 继承关系不能形成回路.
4. role上的属性如createdb,createrole不会直接继承,需要显式通过set role切换过去.
5. 删除role需要先清理此role关联的各种权限.

//END
