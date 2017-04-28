+++
categories = ["linux"]
date = "2016-05-19T11:14:11+08:00"
description = ""
tags = ["lvm"]
thumbnail = ""
title = "lvm 扩容"

+++

给虚拟机扩容常用.

<!--more-->

> 如果是centos 7,文件系统可能是xfs.那分区及更新文件系统的命令会有所不同.

## 虚拟机新加块磁盘
```
  [root@app-dev ~]# fdisk -l
...
Disk /dev/mapper/VolGroup-lv_root: 28.5 GB, 28462546944 bytes
255 heads, 63 sectors/track, 3460 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000
...
Disk /dev/xvdb: 32.2 GB, 32212254720 bytes
255 heads, 63 sectors/track, 3916 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000
```

## 新磁盘划个分区

```
[root@app-dev ~]# fdisk /dev/xvdb
Device contains neither a valid DOS partition table, nor Sun, SGI or OSF disklabel
Building a new DOS disklabel with disk identifier 0x87f4dc6c.
Changes will remain in memory only, until you decide to write them.
After that, of course, the previous content won't be recoverable.

Warning: invalid flag 0x0000 of partition table 4 will be corrected by w(rite)

WARNING: DOS-compatible mode is deprecated. It's strongly recommended to
         switch off the mode (command 'c') and change display units to
         sectors (command 'u').

Command (m for help): n
Command action
   e   extended
   p   primary partition (1-4)
p
Partition number (1-4): 1
First cylinder (1-3916, default 1):
Using default value 1
Last cylinder, +cylinders or +size{K,M,G} (1-3916, default 3916):
Using default value 3916

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
```

## 分区变成pv

```
[root@app-dev ~]# pvcreate /dev/xvdb1
  Physical volume "/dev/xvdb1" successfully created
[root@app-dev ~]# pvscan
  PV /dev/xvda2   VG VolGroup   lvm2 [29.51 GiB / 0    free]
  PV /dev/xvdb1                 lvm2 [30.00 GiB]
  Total: 2 [59.51 GiB] / in use: 1 [29.51 GiB] / in no VG: 1 [30.00 GiB]
[root@app-dev ~]# pvdisplay
  --- Physical volume ---
  PV Name               /dev/xvda2
  VG Name               VolGroup
  PV Size               29.51 GiB / not usable 3.00 MiB
  Allocatable           yes (but full)
  PE Size               4.00 MiB
  Total PE              7554
  Free PE               0
  Allocated PE          7554
  PV UUID               AsBSbC-3piI-LIjs-S2TF-aAVt-JWzn-84hVUH

  "/dev/xvdb1" is a new physical volume of "30.00 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/xvdb1
  VG Name
  PV Size               30.00 GiB
  Allocatable           NO
  PE Size               0
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               ctes1o-Nu0w-Zxwl-HxdT-Ac13-NDre-AqAJXl
```

## pv加入到vg

```
[root@app-dev ~]# vgextend VolGroup /dev/xvdb1
Volume group "VolGroup" successfully extended
[root@app-dev ~]# vgdisplay
  --- Volume group ---
  VG Name               VolGroup
  System ID
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  4
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               2
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               59.50 GiB
  PE Size               4.00 MiB
  Total PE              15233
  Alloc PE / Size       7554 / 29.51 GiB
  Free  PE / Size       7679 / 30.00 GiB
  VG UUID               Ldd34L-2fWI-6IXW-HEWV-EeHL-bXFG-0LOVVP
```

## 扩大需要的lv

```
[root@app-dev ~]# lvextend -l +7679 /dev/mapper/VolGroup-lv_root
Size of logical volume VolGroup/lv_root changed from 26.51 GiB (6786 extents) to 56.50 GiB (14465 extents).
Logical volume lv_root successfully resized
```

## 更新下文件系统

```
[root@app-dev ~]# resize2fs /dev/mapper/VolGroup-lv_root
resize2fs 1.41.12 (17-May-2010)
Filesystem at /dev/mapper/VolGroup-lv_root is mounted on /; on-line resizing required
old desc_blocks = 2, new_desc_blocks = 4
Performing an on-line resize of /dev/mapper/VolGroup-lv_root to 14812160 (4k) blocks.

The filesystem on /dev/mapper/VolGroup-lv_root is now 14812160 blocks long.
```

## 看看磁盘变大了没

```
[root@app-dev ~]# df -h
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup-lv_root
                       56G   24G   30G  44% /
tmpfs                 3.9G  4.0K  3.9G   1% /dev/shm
/dev/xvda1            477M  112M  340M  25% /boot
```

//END


