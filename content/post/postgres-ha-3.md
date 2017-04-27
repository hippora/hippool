+++
categories = ["postgresql"]
date = "2015-02-27T23:49:40+08:00"
description = ""
tags = ["HA","streaming replication"]
thumbnail = ""
title = "postgresql高可用性之备库(三)"

+++

这篇主要是去掉归档,配置流复制槽

<!--more-->

## 流复制的归档相关设置 

- 流复制对于主库是不是开启归档没有必然的联系.   
- 开启归档对于做基础备份和恢复是必要的.  

如果主库开启归档,并且归档目录备库可以访问到的话,在备库启动时的工作方式为:  

1. 首先使用restore_command尽可能使用WAL归档来恢复.
2. 上面命令返回失败,则再在xlog目录下应用可以使用的
3. 上面步骤应用完之后,如果开启了流复制,尝试连接主库,并一直使用流复制方式进行准实时同步...
4. 如果流复制连接失败或者没有开启流复制,则回到第一步如此循环.

## 关闭主库归档 

> 归档在灾难恢复时用来恢复是非常重要的,这里并不是推荐关闭,还是要按照具体的灾备策咯决定.

### 主库关闭归档 

```
[postgres@fnddb data]$ vi postgresql.conf 
archive_mode = off
wal_keep_segments = 200    --需要考虑数据库负载及允许备库关闭多久因素来计算.
max_replication_slots = 2    --这个参数配置Replication Slots需要用到
[postgres@fnddb data]$ pg_ctl restart -m fast
```

> 由于没有了归档,备库直接通过主库来获取xlog.  
> 这就带来一个问题:如果备库关闭了一段时间.主库由于关闭归档,xlog文件只会保留一部分.  
> 备库再启动时获取不到需要的xlog,这时备库只能重新初始化.  
> 那如何保留足够的xlog文件呢.wal_keep_segments参数控制保留的xlog数量

## 配置流复制槽(Replication Slots) 

关闭归档并且使用参数wal_keep_segments控制xlog数量只能大致评估来解决备库不能及时获取xlog的问题.  
如果限制少了,备库有获取不到xlog的风险,限制多了,占用空间.  
Replication Slots 机制能准确控制哪些xlog已经被备库接受,以此来判断哪些xlog可以被回收利用..

### 主库上配置 

```
postgres=# select pg_create_physical_replication_slot('slot_1');
 pg_create_physical_replication_slot 
-------------------------------------
 (slot_1,)
(1 row)

postgres=# select * from pg_replication_slots;
 slot_name | plugin | slot_type | datoid | database | active | xmin | catalog_xmin | restart_lsn 
-----------+--------+-----------+--------+----------+--------+------+--------------+-------------
 slot_1    |        | physical  |        |          | f      |      |              | 
(1 row)
```

> 一旦配置好流复制槽,wal_keep_segments参数就没用了
> 没有配置max_replication_slots参数  
>    postgres=# select pg_create_physical_replication_slot('slot_1');  
>    ERROR:  replication slots can only be used if max_replication_slots > 0  

### 备库上配置 

recovery.conf 增加参数并重启

```
[postgres@vm2 data]$ vi recovery.conf 
primary_slot_name = 'slot_1'
[postgres@vm2 data]$ pg_ctl restart
```

### 检查 

*主库*

```
postgres=# select * from pg_replication_slots;
 slot_name | plugin | slot_type | datoid | database | active | xmin | catalog_xmin | restart_lsn 
-----------+--------+-----------+--------+----------+--------+------+--------------+-------------
 slot_1    |        | physical  |        |          | t      |      |              | 0/29000828
(1 row)
```

> 已经被激活使用

未完待续......

//END

