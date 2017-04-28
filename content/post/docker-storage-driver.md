+++
categories = ["docker"]
date = "2016-05-25T11:37:09+08:00"
description = ""
tags = ["storage-driver"]
thumbnail = ""
title = "docker选择存储驱动器"

+++

docker Storage driver

<!--more-->

## 可插拔(pluggable)的存储驱动架构

docker 支持多种存储驱动器.每种驱动器基于文件系统或linux的卷管理系统.
在不同的环境下.各驱动器的性能特点各有不同.可根据需要自己选择.

> 一个docker实例只可使用一个存储驱动器.

docker 支持下列存储驱动:

| Technology        | Storage driver name           |
| ------------- |-------------|
| OverlayFS | `overlay` |
| AUFS | `aufs` |
| Btrfs | `btrfs` |
| Device Mapper | `devicemapper` |
| VFS*	 | `vfs` |
| ZFS	 | `zfs` |


通过运行`docker info`查看正使用的驱动器.

```
[root@srv00 ~]# docker info
Containers: 5
 Running: 5
 Paused: 0
 Stopped: 0
Images: 12
Server Version: 1.11.1
Storage Driver: devicemapper
 Pool Name: docker-253:0-67305550-pool
 Pool Blocksize: 65.54 kB
 Base Device Size: 10.74 GB
 Backing Filesystem: xfs
 ...
```

 > 存储驱动是`devicemapper`,底层文件系统是xfs.也就是本地的存储区域`/var/lib/docker`所在的文件系统.

有些存储驱动要求底层的文件系统必须符合条件.比如`btrfs`和`zfs`,有些则没这个规定.

通过运行参数`--storage-driver`让docker使用指定的存储驱动,或者设置`DOCKER_OPTS`变量.

```
$ docker daemon --storage-driver=overlay &
```

## 如何选择合适的存储驱动

有两点需要注意:

1.	没有一个驱动适合所有场景
2. 存储驱动总是在不断改善和革新的.

牢记这两点再来看看其他方面

#### 稳定性(Stability)

- 使用linux分发版的默认驱动器.

一般来说,默认的驱动器都是比较稳定的,修改成非默认的可能会遇到bug等.

#### 使用经验(Experience and expertise)

使用自己熟悉的.比如一直使用centos,比较熟悉`LVM`和`Device Mapper`,则使用`devicemapper`比较好.熟悉ubuntu的使用`aufs`较好.

#### 远瞻性(Future-proofing)

很多人认为`OverlayFS`是未来的docker存储驱动器.但是相比`aufs`和`devicemapper`,overlay尚未稳定,可能还存在更多的bug,所以在使用时要千万小心.

各驱动器的特点:

![driver-pros-cons](https://docs.docker.com/engine/userguide/storagedriver/images/driver-pros-cons.png)

//END

