+++
categories = ["linux"]
date = "2018-04-08T14:21:11+08:00"
description = ""
tags = ["lio"]
thumbnail = ""
title = "LIO的使用"

+++

LIO的使用

<!--more-->


### 创建block

```
[root@store05 ~]# targetcli 
/> cd /backstores/block 
/backstores/block> create xen-ha /dev/rbd0 
Created block storage object xen-ha using /dev/rbd0.
/backstores/block> create xen-repo /dev/rbd1   
Created block storage object xen-repo using /dev/rbd1.
/backstores/block> ls
```

> tab键有代码提示

### 创建iscsi目标

```
/backstores/block> cd /iscsi 
/iscsi> create 
Created target iqn.2003-01.org.linux-iscsi.store05.x8664:sn.20853701f8ce.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
/iscsi> ls 
...
```

### 创建lun

```
/iscsi> cd iqn.2003-01.org.linux-iscsi.store05.x8664:sn.20853701f8ce/tpg1/luns     
/iscsi/iqn.20...8ce/tpg1/luns> create /backstores/block/xen-ha 
Created LUN 0.
/iscsi/iqn.20...8ce/tpg1/luns> create /backstores/block/xen-repo 
Created LUN 1.
```

### 创建portal

相当于监听在哪个网卡上。默认已经监听在所有网卡上了。修改的话需要先删除掉0.0.0.0

```
/iscsi/iqn.20...8ce/tpg1/luns> cd ../portals/
/iscsi/iqn.20.../tpg1/portals> delete ip_address=0.0.0.0 ip_port=3260 
Deleted network portal 0.0.0.0:3260
/iscsi/iqn.20.../tpg1/portals> create 192.168.220.105 
Using default IP port 3260
Created network portal 192.168.220.105:3260.
/iscsi/iqn.20.../tpg1/portals> ls
o- portals ............................................................................................................ [Portals: 1]
  o- 192.168.220.105:3260 ..................................................................................................... [OK]
```

### 设置访问权限

```
/iscsi/iqn.20...8ce/tpg1/luns> cd ..
/iscsi/iqn.20...3701f8ce/tpg1> set attribute demo_mode_write_protect=0 generate_node_acls=1 cache_dynamic_acls=1
Parameter demo_mode_write_protect is now '0'.
Parameter generate_node_acls is now '1'.
Parameter cache_dynamic_acls is now '1'.
```

//END
