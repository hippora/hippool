+++
date = "2017-03-29T17:22:25+08:00"
categories = ["ceph"]
tags = ["osd"]
author = "Hippo"
title = "ceph手动添加osd"

+++

ceph手动添加osd

<!--more-->

```sh
[root@store01 ~]# osdid=`ceph osd create`;echo ${osdid}
[root@store01 ~]# mkfs.xfs /dev/sdd3 #osd journal,前面事先分好了
[root@store01 ~]# mkdir /var/lib/ceph/osd/ceph-${osdid}
[root@store01 ~]# echo "UUID=`blkid -s UUID /dev/sdd3 -o value` /var/lib/ceph/osd/ceph-${osdid} noatime 0 0" >> /etc/fstab
[root@store01 ~]# mount -a
## xfs 文件系统没有user_xattr选项
```

初始化osd数据目录

```sh
[root@store01 ~]# ceph-osd -i ${osdid} --mkfs --mkkey
2017-03-29 17:37:59.762940 7f74b20af940 -1 journal FileJournal::_open: disabling aio for non-block journal.  Use journal_force_aio to force use of aio anyway
2017-03-29 17:37:59.777044 7f74b20af940 -1 journal FileJournal::_open: disabling aio for non-block journal.  Use journal_force_aio to force use of aio anyway
2017-03-29 17:37:59.780391 7f74b20af940 -1 filestore(/var/lib/ceph/osd/ceph-21) could not find #-1:7b3f43c4:::osd_superblock:0# in index: (2) No such file or directory
2017-03-29 17:37:59.797523 7f74b20af940 -1 created object store /var/lib/ceph/osd/ceph-21 for osd.21 fsid a67f5548-1451-419f-acba-4f5be1f7c27e
2017-03-29 17:37:59.797589 7f74b20af940 -1 auth: error reading file: /var/lib/ceph/osd/ceph-21/keyring: can't open /var/lib/ceph/osd/ceph-21/keyring: (2) No such file or directory
2017-03-29 17:37:59.797785 7f74b20af940 -1 created new key in keyring /var/lib/ceph/osd/ceph-21/keyring
## 添加认证
[root@store01 ~]# ceph auth add osd.${osdid} osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-${osdid}/keyring
added key for osd.21
## 加入crush图
[root@store01 ~]# ceph osd crush add ${osdid} 0.08 host=`hostname -s`
add item id 21 name 'osd.21' weight 0.08 at location {host=store01} to crush map
```

> 权重设置暂时以1T=1来计算，80G～=0.08

开机自启动启动

```sh
[root@store01 ~]# chown ceph:ceph -R /var/lib/ceph/osd/ceph-${osdid}/
[root@store01 ~]# systemctl start ceph-osd@${osdid}
[root@store01 ~]# systemctl status ceph-osd@${osdid}
[root@store01 ~]# systemctl enable ceph-osd@${osdid}
Created symlink from /run/systemd/system/ceph-osd.target.wants/ceph-osd@21.service to /usr/lib/systemd/system/ceph-osd@.service.
[root@store01 ~]# ceph osd stat
     osdmap e2729: 22 osds: 22 up, 22 in
            flags sortbitwise,require_jewel_osds,require_kraken_osds
```

> 一开始启动还是失败，原因还是目录的权限问题
>
> 看了下ceph-deploy部署的osd启动状态是：enabled-runtime，我们也搞成一样的。

其他几台机器也这样操作一遍。貌似可以写成脚本……



## 删除osd

按照官网文档来：

1. 进入osd所在主机，检查下osd是up且in的话，

```
[root@store06 ~]# ceph -s
[root@store06 ~]# ceph osd tree
[root@store06 ~]# ceph osd out 27
## 等待集群active+clean
[root@store06 ~]# ceph -w
```

2. 停止osd进程

```
[root@store06 ~]# systemctl stop ceph-osd@27
[root@store06 ~]# systemctl disable ceph-osd@27
[root@store06 ~]# ceph osd tree
...
27  1.00000         osd.27      down        0          1.00000
```

3. 删除crush图对应条目

```
[root@store06 ~]# ceph osd crush remove osd.27
removed item id 27 name 'osd.27' from crush map
```

4. 删除osd认证密钥

```
[root@store06 ~]# ceph auth del osd.27
updated
```

5. 删除osd

```
[root@store06 ~]# ceph osd rm 27
removed osd.27
[root@store06 ~]# ll /var/lib/ceph/osd/ceph-27
total 0
[root@store07 ~]# umount /var/lib/ceph/osd/ceph-27
[root@store07 ~]# rm -rf /var/lib/ceph/osd/ceph-27
```

6. ceph.conf中删除[osd.27]相关的设置

```
# 这个需要登录到admin机器，修改后用ceph-deploy再分发一下
```

7. 删除/etc/fstab中的条目

```
略
```



附 create_osd_manual.sh

```
#!/usr/bin/bash
osdid=`ceph osd create`;echo ${osdid}
mkfs.xfs /dev/sdd3 -f
mkdir /var/lib/ceph/osd/ceph-${osdid}
echo "UUID=`blkid -s UUID /dev/sdd3 -o value` /var/lib/ceph/osd/ceph-${osdid} xfs noatime 0 0" >> /etc/fstab
mount -a
ceph-osd -i ${osdid} --mkfs --mkkey
ceph auth add osd.${osdid} osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-${osdid}/keyring
ceph osd crush add ${osdid} 1 host=`hostname -s`
chown ceph:ceph -R /var/lib/ceph/osd/ceph-${osdid}/
systemctl start ceph-osd@${osdid}
systemctl status ceph-osd@${osdid}
systemctl enable ceph-osd@${osdid} --runtime
ceph -s
```

//END
