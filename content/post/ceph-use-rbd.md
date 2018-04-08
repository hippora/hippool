+++
date = "2017-04-01T09:12:25+08:00"
categories = ["ceph"]
tags = ["rbd"]
author = "Hippo"
title = "ceph中RBD使用"

+++

RBD使用

<!--more-->

## 在pool中创建个镜像

```sh
[root@store05 ~]# rbd create -s 20G xenpool/testdisk0
[root@store05 ~]# rbd ls -p xenpool
testdisk0
[root@store05 ~]# rbd info xenpool/testdisk0
rbd image 'testdisk0':
    size 20480 MB in 5120 objects
    order 22 (4096 kB objects)
    block_name_prefix: rbd_data.17129238e1f29
    format: 2
    features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
    flags:
```

## 映射成块设备

```sh
[root@store05 ~]# rbd map xenpool/testdisk0
rbd: sysfs write failed
RBD image feature set mismatch. You can disable features unsupported by the kernel with "rbd feature disable".
In some cases useful info is found in syslog - try "dmesg | tail" or so.
rbd: map failed: (6) No such device or address
[root@store05 ~]# dmesg | tail
...
[598181.779454] rbd: image testdisk0: image uses unsupported features: 0x38

##看看有哪些feature
[root@store05 ~]# rbd help feature disable
usage: rbd feature disable [--pool <pool>] [--image <image>]
                           <image-spec> <features> [<features> ...]

Disable the specified image feature.

Positional arguments
  <image-spec>         image specification
                       (example: [<pool-name>/]<image-name>)
  <features>           image features
                       [layering, striping, exclusive-lock, object-map,
                       fast-diff, deep-flatten, journaling, data-pool]
[root@store01 ~]# ceph --show-config | grep rbd.*feature
rbd_default_features = 61
```

默认支持的属性是61，换算成2进制：00111101，对应`rbd info`显示的`features: layering, exclusive-lock, object-map, fast-diff, deep-flatten`

那不支持其中的的0x38，换算成2进制：00**111**000，对应`object-map,fast-diff, deep-flatten`不支持

我们把它禁用

```sh
[root@store05 ~]# rbd feature disable xenpool/testdisk0 object-map fast-diff deep-flatten
[root@store05 ~]# rbd info xenpool/testdisk0
rbd image 'testdisk0':
    size 20480 MB in 5120 objects
    order 22 (4096 kB objects)
    block_name_prefix: rbd_data.17129238e1f29
    format: 2
    features: layering, exclusive-lock
    flags:
[root@store05 ~]# rbd map xenpool/testdisk0
/dev/rbd0
[root@store05 ~]# rbd showmapped
id pool    image     snap device
0  xenpool testdisk0 -    /dev/rbd0
```

> 也可以把rbd_default_features=5写入ceph.conf中，或者`rbd create` 带`--image-feature`参数

```sh
[root@store01 ~]# rbd create -s 1G xenpool/test --image-feature layering,exclusive-lock
[root@store01 ~]# rbd info xenpool/test
rbd image 'test':
    size 1024 MB in 256 objects
    order 22 (4096 kB objects)
    block_name_prefix: rbd_data.1bf49238e1f29
    format: 2
    features: layering, exclusive-lock
    flags:
```

## rbd设备开机自映射

```
[root@store05 ~]# echo "xenpool/testdisk0" >> /etc/ceph/rbdmap
[root@store05 ~]# systemctl enable rbdmap
```

> `/usr/bin/rbdmap`会读取配置文件`/etc/ceph/rbdmap` ,其他参数参考此配置文件

##  镜像删除

```sh
[root@store05 ~]# rbd showmapped
id pool    image     snap device    
0  xenpool testdisk0 -    /dev/rbd0 
[root@store05 ~]# rbd unmap /dev/rbd0 
[root@store05 ~]# rbd showmapped
[root@store05 ~]# rbd rm xenpool/testdisk0
Removing image: 100% complete...done.
```

> /etc/ceph/rbdmap里的条目删除

//END
