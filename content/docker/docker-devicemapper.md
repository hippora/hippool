+++
categories = ["docker"]
date = "2016-05-26T11:39:14+08:00"
description = ""
tags = ["devicemapper","driect-lvm"]
thumbnail = ""
title = "docker的devicemapper 最佳实践"

+++

devicemapper生产环境要使用driect-lvm模式

<!--more-->

# devicemapper 最佳实践

docker 最先是跑在ubuntu和debian上的,使用aufs存储器.
由于docker越来越流行,许多公司希望在RHEL上使用,但是上游内核中没有包括aufs,所以rhel不能使用aufs.
最终开发者们开发了一个新的后端存储引擎`devicemapper`,基于已有的`Device Mapper`技术,并且使docker支持可插拔,现在全世界有很多真实案例在生产环境使用`devicemapper`.

## 镜像层与共享

`devicemapper`存储每个镜像和容器在自己的虚拟设备上,也就是说这些设备是按需分配(copy-on-write snapshot devices),`Device Mapper`技术是工作在block级别的而不是文件级别的.

`devicemapper`创建镜像的方式是:

1. `devicemapper`基于块设备或`loop mounted sparse files`来创建一个虚拟池.
2. 然后在上面创建一个有文件系统的基础设备(base device).
3. 每个镜像层就是基于这个基础设备的COW快照(snapshot),也就是说这些快照初始化时是空的,只有数据写入时才会占用池中的空间.

图示:

![base_device](https://docs.docker.com/engine/userguide/storagedriver/images/two_dm_container.jpg)

> 容器就是在镜像上再创建一个快照

## devicemapper的读操作

例图:

![dm_container](https://docs.docker.com/engine/userguide/storagedriver/images/dm_container.jpg)

1. 容器中的应用请求块`0x44f`,因为容器层也是个虚拟的快照,不存数据,只有指针.通过指针找到存放数据的镜像层.
2. 根据指针找到`a005e`层的`0xf33`.
3. `devicemapper` copy其中的数据到容器内存中.
4. 存储驱动器返回数据给请求的应用.

## devicemapper的写操作

`devicemapper`写操作是通过`allocate-on-demand`(按需分配)的方式.更新数据是通过COW方式.
因为`devicemapper`是一种基于块的技术,所以就算修改一个大文件,也不会copy整个文件过来.只会copy对应要修改的块.

### 写新数据

写56k的数据到容器中:

1. 应用请求写56k的数据到容器
2. 按需分配一个新的64k大小的块在容器层.(超过64k,则分配多个)
3. 数据写入新分配的块中

### 修改数据

1. 应用请求修改容器中数据
2. cow操作定位到要更新的块
3. 分配新的块在容器层并将数据copy过来
4. 对数据进行修改

## 配置docker使用`devicemapper`

基于rhel的分支默认使用的`devicemapper`,并且默认配置成`loop-lvm`模式运行.这种模式使用文件来作为虚拟池(thin pool)构建镜像和容器的层.
但是生产环境不因该使用这种模式.

通过docker info命令来检查

```
[root@srv00 ~]# docker info 
 WARNING: Usage of loopback devices is strongly discouraged for production use. Either use `--storage-opt dm.thinpooldev` or use `--storage-opt dm.no_warn_on_loop_devices=true` to suppress this warning.
...
 Data loop file: /var/lib/docker/devicemapper/devicemapper/data
 Metadata loop file: /var/lib/docker/devicemapper/devicemapper/metadata
 Library Version: 1.02.107-RHEL7 (2015-12-01)
 ...
```

> `Data loop file`,`Metadata loop file`指示了docker运行在`loop-lvm`模式下,并且还有个WARNING.

## 为生产环境配置`direct-lvm`模式

生产环境下因该使用`direct-lvm`,如果之前有镜像在`loop-lvm`模式下创建,需要切换,则因该将镜像做备份.(push到hub或私有registry)

我给虚拟机分配个30G磁盘

1.停止docker daemon

```
[root@srv00 ~]# systemctl stop docker
```

2.创建相关的逻辑卷和thinpool

创建pv

```
[root@srv00 ~]# fdisk -l <==检查下磁盘
...
Disk /dev/xvdb: 32.2 GB, 32212254720 bytes, 62914560 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

[root@srv00 ~]# pvcreate /dev/xvdb
  Physical volume "/dev/xvdb" successfully created
```

创建vg

```
[root@srv00 ~]# vgcreate vgdocker /dev/xvdb
  Volume group "vgdocker" successfully created
```

创建一个thin pool,名字叫`thinpool`,先来创建逻辑卷

```
[root@srv00 ~]# lvcreate --wipesignatures y -n thinpool -l 95%VG vgdocker
  Logical volume "thinpool" created.
[root@srv00 ~]# lvcreate --wipesignatures y -n thinpoolmeta -l 1%VG vgdocker  
  Logical volume "thinpoolmeta" created.
[root@srv00 ~]# lvscan
  ACTIVE            '/dev/centos/swap' [4.00 GiB] inherit
  ACTIVE            '/dev/centos/root' [35.47 GiB] inherit
  ACTIVE            '/dev/vgdocker/thinpool' [28.50 GiB] inherit
  ACTIVE            '/dev/vgdocker/thinpoolmeta' [304.00 MiB] inherit
```

> 剩余的4%留给它们自动扩展

转换成thin pool

```
[root@srv00 ~]# lvconvert -y --zero n -c 512K --thinpool vgdocker/thinpool --poolmetadata vgdocker/thinpoolmeta
  WARNING: Converting logical volume vgdocker/thinpool and vgdocker/thinpoolmeta to pool's data and metadata volumes.
  THIS WILL DESTROY CONTENT OF LOGICAL VOLUME (filesystem etc.)
  Converted vgdocker/thinpool to thin pool.
```

设置thinpool的自动扩展参数,并应用此profile

```
[root@srv00 ~]# vi /etc/lvm/profile/docker-thinpool.profile
activation {
    thin_pool_autoextend_threshold=80
    thin_pool_autoextend_percent=20
}
[root@srv00 ~]# lvchange --metadataprofile docker-thinpool vgdocker/thinpool
  Logical volume "thinpool" changed.
```

> 当空间大于80%时进行扩展.扩展的大小是空闲空间的20%

查看thinpool是否是已监视状态

```
[root@srv00 ~]# lvs -o+seg_monitor
  LV       VG        Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert Monitor  
  root     centos    -wi-ao---- 35.47g                                                              
  swap     centos    -wi-ao----  4.00g                                                              
  thinpool vgdocker twi-a-t--- 28.50g             0.00   0.02                             monitored
  ```

3.删除docker存储目录

```
[root@srv00 ~]# rm -rf /var/lib/docker/*
```

> 注意备份重要镜像等

4.修改启动参数并启动

我们通过systemd的drop-in方式修改,也是官方推荐的

```
[root@srv00 ~]# mkdir /etc/systemd/system/docker.service.d
[root@srv00 ~]# vi /etc/systemd/system/docker.service.d/daemon.conf
[Service]
ExecStart=
ExecStart=/usr/bin/docker daemon -H fd:// --storage-driver=devicemapper --storage-opt=dm.thinpooldev=/dev/mapper/vgdocker-thinpool --storage-opt dm.use_deferred_removal=true
[root@srv00 ~]# systemctl daemon-reload
[root@srv00 ~]# systemctl start docker
```

> `ExecStart=` 第一行是空.否则启动会报错,并且修改daemon参数需要reload

5.检查确认

```
[root@srv00 docker]# docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 1.11.1
Storage Driver: devicemapper
 Pool Name: vgdocker-thinpool
 Pool Blocksize: 524.3 kB
 Base Device Size: 10.74 GB
 Backing Filesystem: xfs
 ...
 [root@srv00 docker]# lvs -a
  LV               VG       Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root             centos   -wi-ao----  35.47g                                                    
  swap             centos   -wi-ao----   4.00g                                                    
  [lvol0_pmspare]  vgdocker ewi------- 304.00m                                                    
  thinpool         vgdocker twi-a-t---  28.50g             0.07   0.02                            
  [thinpool_tdata] vgdocker Twi-ao----  28.50g                                                    
  [thinpool_tmeta] vgdocker ewi-ao---- 304.00m                                                    
[root@srv00 docker]# lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sr0                        11:0    1 114.4M  0 rom  
xvda                      202:0    0    40G  0 disk 
├─xvda1                   202:1    0   500M  0 part /boot
└─xvda2                   202:2    0  39.5G  0 part 
  ├─centos-root           253:0    0  35.5G  0 lvm  /
  └─centos-swap           253:1    0     4G  0 lvm  [SWAP]
xvdb                      202:16   0    30G  0 disk 
├─vgdocker-thinpool_tmeta 253:2    0   304M  0 lvm  
│ └─vgdocker-thinpool     253:5    0  28.5G  0 lvm  
└─vgdocker-thinpool_tdata 253:3    0  28.5G  0 lvm  
  └─vgdocker-thinpool     253:5    0  28.5G  0 lvm  
 ```

然后就可以正常的docker run 了

## `devicemapper`的性能影响

-  `Allocate-on-demand`性能影响

之前讲说,如果容器中要写数据,则会在容器层分配新的块,每个快64k,如果要写的数据大于64k,则会分配多个块.
如果容器中有许多小文件的写操作..则会影响性能.

-  `Copy-on-write` 性能影响

容器中第一次更新一个已有数据时,就会进行cow操作,如果更新大文件的某一部分,则只会copy相应的数据快,这个性能提升很大.
如果更新大量小文件(<=64k),`devicemapper`的性能就比AUFS差.

- 其他

	- `loop-lvm`模式的性能很差,不推荐使用在生产环境.生产环境使用`direct-lvm`模式,它是直接操作raw设备.
	-  使用ssd的话提升更明显
	- `devicemapper`内存使用不是最有效的.运行n个容器会加载n次相同文件到内存.所以不是运行pass平台和高密度容器最好的选择.

- 当然容器中写操作多的话要尽量使用data volume,它是绕过storage driver,性能有保证

//END

