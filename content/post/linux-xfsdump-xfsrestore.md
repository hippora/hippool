+++
categories = ["linux"]
date = "2017-05-24T13:53:12+08:00"
description = ""
tags = ["xfsdump","xfsrestore"]
thumbnail = ""
title = "xfs备份恢复"

+++

xfsdump,xfsrestore 使用

<!--more-->

# xfs备份恢复

## 使用xfsdump 备份

没有就先安装

```sh
[root@samba-srv ~]# yum install xfsdump -y
```



执行`level 0` 备份，全量备份

```sh
[root@samba-srv tmp]# xfsdump -l 0 -L "Backup level 0 of /data/samba/legal" -M "local" -f /tmp/backup_2.xfsdump /data/samba/legal
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.4 (dump format 3.0) - type ^C for status and control
xfsdump: level 0 dump of samba-srv:/data/samba/legal
xfsdump: dump date: Wed May 24 16:00:08 2017
xfsdump: session id: 8e066a94-f439-4fbe-835c-f894ecb2e92d
xfsdump: session label: "Backup level 0 of /data/samba/legal"
xfsdump: ino map phase 1: constructing initial dump list
xfsdump: ino map phase 2: skipping (no pruning necessary)
xfsdump: ino map phase 3: skipping (only one dump stream)
xfsdump: ino map construction complete
xfsdump: estimated dump size: 19015872 bytes
xfsdump: creating dump session media file 0 (media 0, file 0)
xfsdump: dumping ino map
xfsdump: dumping directories
xfsdump: dumping non-directory files
xfsdump: ending media file
xfsdump: media file size 19013408 bytes
xfsdump: dump size (non-dir files) : 18990312 bytes
xfsdump: dump complete: 5 seconds elapsed
xfsdump: Dump Summary:
xfsdump:   stream 0 /tmp/backup_2.xfsdump OK (success)
xfsdump: Dump Status: SUCCESS
```

首先得有`level 0` 的全量备份，才能继续增量备份,

基于`level 0`之上的增量备份，就是`level 1`，基于`level 1`之上的增量备份就是`level 2`，以此类推，一共到`level 9`

我们执行一个`level 1`的：

```sh
[root@samba-srv legal]# pwd
/data/samba/legal
[root@samba-srv legal]# echo "111111" > xfs.txt
[root@samba-srv legal]# xfsdump -l 1 -L "Backup level 1 of /data/samba/legal" -M "local" -f /tmp/backup_level1_1.xfsdump /data/samba/legal
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.4 (dump format 3.0) - type ^C for status and control
xfsdump: level 1 incremental dump of samba-srv:/data/samba/legal based on level 0 dump begun Wed May 24 15:56:38 2017
xfsdump: dump date: Wed May 24 16:03:35 2017
...
```

检查下增量备份大小比较小。

```sh
[root@samba-srv legal]# ls -lh /tmp/backup_*
-rw-r--r-- 1 root root 19M May 24 16:00 /tmp/backup_2.xfsdump
-rw-r--r-- 1 root root 22K May 24 16:03 /tmp/backup_level1_1.xfsdump
...
```

再执行个`level 2`的：

```sh
[root@samba-srv legal]# echo "2222222" >> xfs.txt 
[root@samba-srv legal]# xfsdump -l 2 -L "Backup level 2 of /data/samba/legal" -M "local" -f /tmp/backup_level2_1.xfsdump /data/samba/legal
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.4 (dump format 3.0) - type ^C for status and control
xfsdump: level 2 incremental dump of samba-srv:/data/samba/legal based on level 1 dump begun Wed May 24 16:03:35 2017
xfsdump: dump date: Wed May 24 16:09:37 2017
xfsdump: session id: ad1f1bb7-6c50-4ccb-a7ce-3aa56fd2f9b8
...
```

## 查看备份信息

备份信息会记录在`/var/lib/xfsdump/inventory/`中。如果不想记录，加`-J`参数。

查看备份信息：

```sh
[root@samba-srv legal]# xfsdump -I
file system 0:
        fs id:          be538072-5cb7-4fa5-9073-086d7dfc1572
        session 0:
                mount point:    samba-srv:/data/samba/legal
                device:         samba-srv:/dev/xvdb1
                time:           Wed May 24 15:50:41 2017
                session label:  "level 0 of /data/samba/legal Wed May 24 15:50:41 CST 2017"
                session id:     8a83b78b-0eb1-4ac6-a8a6-2922820e7dd9
                level:          0
                resumed:        NO
                subtree:        NO
                streams:        1
                stream 0:
                        pathname:       /tmp/backup_l0.xfsdump
                        start:          ino 67 offset 0
                        end:            ino 73 offset 0
                        interrupted:    NO
                        media files:    1
                        media file 0:
                                mfile index:    0
                                mfile type:     data
                                mfile size:     19013408
                                mfile start:    ino 67 offset 0
                                mfile end:      ino 73 offset 0
                                media label:    "local"
                                media id:       eebe1aff-9214-4d48-8373-b2b030bef987
        session 1:
        。。。
```

## 使用xfsrestore恢复

破坏现场先

```
[root@samba-srv legal]# rm -rf /data/samba/legal/*
[root@samba-srv legal]# ll /data/samba/legal/
total 0
```

先恢复`level 0`

```sh
[root@samba-srv legal]# xfsrestore -f /tmp/backup_2.xfsdump /data/samba/legal/
xfsrestore: using file dump (drive_simple) strategy
xfsrestore: version 3.1.4 (dump format 3.0) - type ^C for status and control
xfsrestore: searching media for dump
xfsrestore: examining media file 0
...
xfsrestore:   stream 0 /tmp/backup_2.xfsdump OK (success)
xfsrestore: Restore Status: SUCCESS
[root@samba-srv legal]# ll /data/samba/legal/
total 18548
-rw-rw---- 1 heqing legal 6879099 Feb 21 18:04 Charles-Proxy-4.0.2-Crack.zip
-rw-rw---- 1 heqing legal 4718689 May  9 17:26 NexentaV1.2.pdf
-rw-rw---- 1 heqing legal   27048 May 22 11:50 p1atkkbhll1ao2113kj608rpaem4 (1).docx
...
```

> 数据回来了。而且确实是在创建xfs.txt文件之前（做`level 1`备份前）
>
> 通过`xfsrestore -I`或`xfsdump -I`检查可用备份

继续恢复增量备份`level 1`

```sh
[root@samba-srv legal]# xfsrestore -f /tmp/backup_level1_1.xfsdump /data/samba/legal
xfsrestore: using file dump (drive_simple) strategy
xfsrestore: version 3.1.4 (dump format 3.0) - type ^C for status and control
xfsrestore: searching media for dump
xfsrestore: examining media file 0
xfsrestore: dump description: 
xfsrestore: hostname: samba-srv
xfsrestore: mount point: /data/samba/legal
xfsrestore: volume: /dev/xvdb1
xfsrestore: session time: Wed May 24 16:03:35 2017
xfsrestore: level: 1
xfsrestore: session label: "Backup level 1 of /data/samba/legal"
...
[root@samba-srv legal]# cat xfs.txt 
111111
```

恢复`level 2`

```sh
[root@samba-srv legal]# xfsrestore -f /tmp/backup_level2_1.xfsdump /data/samba/legal
[root@samba-srv legal]# cat xfs.txt 
111111
2222222
```

## 探索一下

只恢复`level 2`到一个空目录下

```sh
[root@samba-srv legal]# mkdir xfs_test
[root@samba-srv legal]# xfsrestore -f /tmp/backup_level2_1.xfsdump /data/samba/legal/xfs_test
[root@samba-srv legal]# ls xfs_test/
xfs.txt
[root@samba-srv legal]# cat xfs_test/xfs.txt 
111111
2222222
```

> 只有变动过的文件，文件内容是完整的。



## 备份策略

根据增量数据的大小，来决定什么时候做全量备份，什么时候做增量备份。因为会影响到恢复时间。

写个定时脚本：

```sh
[root@samba-srv ~]# cat /opt/backup_samba_xfsdump.sh 
#!/usr/bin/bash

LEVEL=`date +%w`

MOUNT="/data/samba/legal"
BAKDIR="/data/backup"
BAKFILE="backup${LEVEL}_`date +%Y%m%d`.xfsdump"


# backup
xfsdump -l ${LEVEL} -L "Backup level ${LEVEL} of ${MOUNT}" -M local -f ${BAKDIR}/${BAKFILE} $MOUNT

# cleanup
find ${BAKDIR} -name "*.xfsdump" -mtime +15 -exec rm -f {} \;
xfsinvutil -F -M ${MOUNT} `date -d "-15 day" +%m/%d/%Y`

[root@samba-srv ~]# crontab -e
0 3 * * * /opt/backup_samba_xfsdump.sh
```

> crontab装好后，执行的时候`LEVEL`可能是4，`level 0,1,2,3`先手动执行一下，不然会报错


//END
