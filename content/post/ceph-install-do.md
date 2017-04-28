+++
date = "2017-03-28T17:22:25+08:00"
categories = ["ceph"]
tags = ["install"]
author = "Hippo"
title = "ceph 安装"

+++

ceph 集群安装开始

<!--more-->

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
[hpec@store01 store-cluster]$ ceph -s 
```

未完待续……

//END
