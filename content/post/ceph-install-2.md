+++
#draft = true
date = "2017-03-28T17:22:25+08:00"
title = "ceph-install-2"
categories = ["Ceph"]
tags = ["ceph"]
author = "Hippo"
description = "ceph 安装工作"

+++

# 安装集群

## 创建集群及修改配置

```sh
[hpec@store01 ~]$ mkdir store-cluster;cd store-cluster
[hpec@store01 store-cluster]$ ceph-deploy new --cluster-network 192.168.210.0/24 --public-network 192.168.220.0/24 store02 store03 store04
[hpec@store01 store-cluster]$ vi ceph.conf
...
osd pool default size = 3
osd pool default min size = 1
osd pool default pg num = 256
osd pool default pgp num = 256

#Choose a reasonable crush leaf type
#0 for a 1-node cluster.
#1 for a multi node cluster in a single rack
#2 for a multi node, multi chassis cluster with multiple hosts in a chassis
#3 for a multi node cluster with hosts across racks, etc.
osd crush chooseleaf type = 1
```

> 网络相关配置在命令行上已经指定，就会在ceph.conf生成，不然可以手动添加

## 安装ceph 软件

```sh
[hpec@store01 store-cluster]$ ceph-deploy install store0{1..7} --release karken --repo-url http://mirrors.163.com/ceph/rpm-kraken/el7
...
[store07][DEBUG ]
[store07][DEBUG ] Complete!
[store07][INFO  ] Running command: sudo ceph --version
[store07][DEBUG ] ceph version 11.2.0 (f223e27eeb35991352ebc1f67423d4ebc252adb7)
```

> 网络不好可以用国内镜像站点，指定`—repo-url` 及`—gpg-url`

## 配置初始 monitor(s)、并收集所有密钥

```sh
[hpec@store01 store-cluster]$ ceph-deploy mon create-initial
```
> 当前目录会生成几个keyring

## 安装osd

```sh
[hpec@store01 ~]$ ceph-deploy osd create --zap-disk \
store01:sda:/dev/sdd5 store01:sdb:/dev/sdd6 store01:sdc:/dev/sdd7 \
store02:sda:/dev/sdd5 store02:sdb:/dev/sdd6 store02:sdc:/dev/sdd7 \
store03:sda:/dev/sdd5 store03:sdb:/dev/sdd6 store03:sdc:/dev/sdd7 \
store04:sda:/dev/sdd5 store04:sdb:/dev/sdd6 store04:sdc:/dev/sdd7 \
store05:sda:/dev/sdd5 store05:sdb:/dev/sdd6 store05:sdc:/dev/sdd7 \
store06:sda:/dev/sdd5 store06:sdb:/dev/sdd6 store06:sdc:/dev/sdd7 \
store07:sda:/dev/sdd5 store07:sdb:/dev/sdd6 store07:sdc:/dev/sdd7
```

> 指定的是整块盘做osd，会自动帮你格式化成xfs。

## 拷贝配置文件及密钥到所有主机

```sh
[hpec@store01 store-cluster]$ ceph-deploy admin store0{1..7}
```



## 安装过程中出现的问题

1. 机器重启后osd启动不了

查看日志/var/log/ceph/ceph-osd.4.log

```
2017-03-27 13:14:03.548526 7fedc1a5e940  1 filestore(/var/lib/ceph/tmp/mnt.P8UBYK) FileStore::mkfs: omap fsid is already set to 4f2f36bc-4fb6-42a8-9572-824bbc4ec128
2017-03-27 13:14:03.548546 7fedc1a5e940  1 filestore(/var/lib/ceph/tmp/mnt.P8UBYK) leveldb db exists/created
2017-03-27 13:14:03.548735 7fedc1a5e940 -1 filestore(/var/lib/ceph/tmp/mnt.P8UBYK) mkjournal error creating journal on /var/lib/ceph/tmp/mnt.P8UBYK/journal: (13) Permission denied
2017-03-27 13:14:03.548780 7fedc1a5e940 -1 OSD::mkfs: ObjectStore::mkfs failed with error -13
2017-03-27 13:14:03.548928 7fedc1a5e940 -1  ** ERROR: error creating empty object store in /var/lib/ceph/tmp/mnt.P8UBYK: (13) Permission denied
...

[root@store02 ~]# ll /dev/sdd{5..10}
brw-rw---- 1 root disk 8, 58 Mar 27 17:49 /dev/sdd10
brw-rw---- 1 root disk 8, 53 Mar 28 16:44 /dev/sdd5
brw-rw---- 1 root disk 8, 54 Mar 28 16:44 /dev/sdd6
brw-rw---- 1 root disk 8, 55 Mar 28 16:44 /dev/sdd7
brw-rw---- 1 root disk 8, 56 Mar 27 17:49 /dev/sdd8
brw-rw---- 1 root disk 8, 57 Mar 27 17:49 /dev/sdd9
```

> 由于我的journal分区是MBR的，ceph读不到gpt的一些信息，不能触发自动赋权，只能自己写个udev规则了。参考`/usr/lib/udev/rules.d/`ceph相关规则。

```sh
[root@store01 ~]# vi /etc/udev/rules.d/70-persistent-ceph-journal.rules
KERNEL=="sd?[5-9]", SUBSYSTEM=="block", SUBSYSTEMS=="scsi", ATTRS{model}=="PERC H310", OWNER="ceph", GROUP="ceph"
KERNEL=="sd?10", SUBSYSTEM=="block", SUBSYSTEMS=="scsi", ATTRS{model}=="PERC H310", OWNER="ceph", GROUP="ceph"
[root@store01 ~]# ansible all -m copy -a "src=/etc/udev/rules.d/70-persistent-ceph-journal.rules dest=/etc/udev/rules.d/ mode=0644" -k
[root@store01 ~]# ansible all -m shell -a "udevadm trigger" -k
```

> 由于我的journal分区是MBR的，ceph读不到gpt的一些信息，不能触发自动赋权，只能自己写个udev规则了。使用udevadm info -a /dev/sd?? 寻找需要的信息


2. too few PGs per OSD
```sh
[root@store01 ~]# ceph -s
    cluster a67f5548-1451-419f-acba-4f5be1f7c27e
     health HEALTH_WARN
            too few PGs per OSD (10 < min 30)
     monmap e2: 3 mons at {store02=192.168.220.102:6789/0,store03=192.168.220.103:6789/0,store04=192.168.220.104:6789/0}
            election epoch 8, quorum 0,1,2 store02,store03,store04
        mgr active: store04 standbys: store03, store02
     osdmap e98: 18 osds: 18 up, 18 in
            flags sortbitwise,require_jewel_osds,require_kraken_osds
      pgmap v293: 64 pgs, 1 pools, 0 bytes data, 0 objects
            622 MB used, 67035 GB / 67035 GB avail
                  64 active+clean
```

> ceph.conf里`osd pool default size = 3` ，也就是每个pg有3个数据副本，平均分布在18个osd中。`64*3/18=10.67`，那要达到30的话，至少要设置 `18*30/3=180`,取最接近的2的幂次方就是256
>
> 解决方法就是手动改一下

```sh
[root@store01 ~]# ceph osd lspools
0 rbd,
[root@store01 ~]# ceph osd pool get rbd pg_num
[root@store01 ~]# ceph osd pool get rbd pgp_num
[root@store01 ~]# ceph osd pool set rbd pg_num 256
[root@store01 ~]# ceph osd pool set rbd pgp_num 256
```

> 注意，pg只能改大，不能改小

3. too many PGs per OSD

```sh
[root@store01 ~]# ceph -s
    cluster a67f5548-1451-419f-acba-4f5be1f7c27e
     health HEALTH_WARN
            too many PGs per OSD (329 > max 300)
     monmap e2: 3 mons at {store02=192.168.220.102:6789/0,store03=192.168.220.103:6789/0,store04=192.168.220.104:6789/0}
            election epoch 20, quorum 0,1,2 store02,store03,store04
        mgr active: store04 standbys: store02, store03
     osdmap e654: 21 osds: 21 up, 21 in
            flags sortbitwise,require_jewel_osds,require_kraken_osds
      pgmap v2128: 2304 pgs, 2 pools, 0 bytes data, 0 objects
            1172 MB used, 78207 GB / 78208 GB avail
                2304 active+clean
```

参考上面的，我们计算下要达到每个osd占有300个pg的话，需要设置：`21*300/3=2100`，也就是21个osd，副本为3的话，不能超过2100个pg。小于2100的最大2的幂是2048。

> 官网有个计算pg的计算器：http://ceph.com/pgcalc/

 

就我的案例来说：我们计划将来磁盘还要扩展一倍，每个osd取200pg比较合适。那计算的总`pg: 21*200/3=1400` ,不能超过`2100`,如果要分配到3个池，rbd，rbd1，rbd2，空间占比25%，25%，50%，

rbd=rbd1：1400*25%=350，取256

rbd2:	1400*50%=700，取1024。	256+256+1024=1536 < 2100

4. Monitor clock skew detected

```
[root@store01 ~]# ceph -w
    cluster a67f5548-1451-419f-acba-4f5be1f7c27e
     health HEALTH_WARN
            clock skew detected on mon.store03, mon.store04
            Monitor clock skew detected
     monmap e2: 3 mons at {store02=192.168.220.102:6789/0,store03=192.168.220.103:6789/0,store04=192.168.220.104:6789/0}
            election epoch 32, quorum 0,1,2 store02,store03,store04
        mgr active: store03 standbys: store04, store02
     osdmap e1513: 21 osds: 21 up, 21 in
            flags sortbitwise,require_jewel_osds,require_kraken_osds
      pgmap v5211: 256 pgs, 1 pools, 0 bytes data, 0 objects
            1237 MB used, 78207 GB / 78208 GB avail
                 256 active+clean

2017-03-28 13:19:13.920884 mon.0 [WRN] mon.2 192.168.220.104:6789/0 clock skew 0.0563359s > max 0.05s
2017-03-28 13:21:43.922252 mon.0 [WRN] mon.1 192.168.220.103:6789/0 clock skew 0.0552689s > max 0.05s
2017-03-28 13:22:13.901971 mon.0 [INF] HEALTH_WARN; clock skew detected on mon.store03; Monitor clock skew detected
```



貌似ceph对服务器时间同步要求挺高的（max 0.05s），那就搭个本地时间服务器好了

```
[root@store01 ~]# vi /etc/ntp.conf
...
# Hosts on local network are less restricted.
restrict 192.168.220.0 mask 255.255.255.0 nomodify notrap

# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
server 0.cn.pool.ntp.org iburst
server 1.cn.pool.ntp.org iburst
server 2.cn.pool.ntp.org iburst
server 3.cn.pool.ntp.org iburst
...
[root@store01 ~]# cp /etc/ntp.conf ./
[root@store01 ~]# vi ntp.conf
...
# Hosts on local network are less restricted.
#restrict 192.168.220.0 mask 255.255.255.0 nomodify notrap

# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
server 192.168.220.101 prefer iburst
#server 0.cn.pool.ntp.org iburst
#server 1.cn.pool.ntp.org iburst
#server 2.cn.pool.ntp.org iburst
#server 3.cn.pool.ntp.org iburst
...
[root@store01 ~]# ansible all[1:] -m copy -a "src=ntp.conf dest=/etc/ mode=0644" -k
[root@store01 ~]# ansible all[1:] -m shell -a "systemctl stop ntpd" -k
[root@store01 ~]# ansible all[1:] -m shell -a "ntpdate 192.168.220.101" -k
[root@store01 ~]# ansible all[1:] -m shell -a "hwclock -w" -k
[root@store01 ~]# ansible all -m shell -a "systemctl start ntpd" -k

```

