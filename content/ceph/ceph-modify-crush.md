+++
date = "2017-03-29T19:12:25+08:00"
categories = ["ceph"]
tags = ["crush"]
author = "Hippo"
title = "ceph修改crush"

+++

修改crush图

<!--more-->

## 获取crush图并反编译

```sh
[root@store01 ~]# ceph osd getcrushmap -o crush.map
got crush map from osdmap epoch 4800
[root@store01 ~]# crushtool -d crush.map -o crush.map.txt
[root@store01 ~]# file crush.map;file crush.map.txt
crush.map: MS Windows icon resource - 8 icons, 1-colors
crush.map.txt: ASCII text
```

可以看一下现在的osdtree

```sh
[root@store01 ~]# ceph osd tree
ID WEIGHT   TYPE NAME        UP/DOWN REWEIGHT PRIMARY-AFFINITY
-1 82.45461 root default
-2 10.99065     host store01
 0  3.63689         osd.0         up  1.00000          1.00000
 2  3.63689         osd.2         up  1.00000          1.00000
 1  3.63689         osd.1         up  1.00000          1.00000
21  0.07999         osd.21        up  1.00000          1.00000
...
-8 11.91066     host store07
18  3.63689         osd.18        up  1.00000          1.00000
19  3.63689         osd.19        up  1.00000          1.00000
20  3.63689         osd.20        up  1.00000          1.00000
27  0.07999         osd.27        up  1.00000          1.00000
```

## 修改crush图

这里7台机器，每台都有一个ssd分区，创建一个单独的pool用ssd。

```sh
[root@store01 ~]# vi crush.map.txt
# begin crush map
tunable choose_local_tries 0
tunable choose_local_fallback_tries 0
tunable choose_total_tries 50
tunable chooseleaf_descend_once 1
tunable chooseleaf_vary_r 1
tunable straw_calc_version 1

# devices
device 0 osd.0
device 1 osd.1
...
device 26 osd.26
device 27 osd.27

# types
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 region
type 10 root

# buckets
host store01 {
        id -2           # do not change unnecessarily
        # weight 10.911
        alg straw
        hash 0  # rjenkins1
        item osd.0 weight 3.637
        item osd.2 weight 3.637
        item osd.1 weight 3.637
}
...
host store07 {
        id -8           # do not change unnecessarily
        # weight 10.911
        alg straw
        hash 0  # rjenkins1
        item osd.18 weight 3.637
        item osd.19 weight 3.637
        item osd.20 weight 3.637
}
host store01-ssd {
        id -9           # do not change unnecessarily
        # weight 0.08
        alg straw
        hash 0  # rjenkins1
        item osd.21 weight 0.080
}
...
host store07-ssd {
        id -9           # do not change unnecessarily
        # weight 0.08
        alg straw
        hash 0  # rjenkins1
        item osd.27 weight 0.080
}

root ssd {
        id -10          # do not change unnecessarily
        # weight 0.08 * 7
        alg straw
        hash 0  # rjenkins1
        item store01-ssd  weight 0.08
        item store02-ssd  weight 0.08
        item store03-ssd  weight 0.08
        item store04-ssd  weight 0.08
        item store05-ssd  weight 0.08
        item store06-ssd  weight 0.08
        item store07-ssd  weight 0.08
}
root default {
        id -1           # do not change unnecessarily
        # weight 3.637 * 3 * 7 = 76.377
        alg straw
        hash 0  # rjenkins1
        item store01 weight 10.911
        item store02 weight 10.911
        item store03 weight 10.911
        item store04 weight 10.911
        item store05 weight 10.911
        item store06 weight 10.911
        item store07 weight 10.911
}

# rules
rule replicated_ruleset {
        ruleset 0
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule ssd_ruleset {
        ruleset 1
        type replicated
        min_size 1
        max_size 10
        step take ssd
        step chooseleaf firstn 0 type host
        step emit
}
```

1. 为7台主机的ssd加7个bucket，编号是不重复的负整数。把7个ssd关联的osd移到对应的bucket中，(host store01移到 host store01-ssd),
2. 增加一个根节点bucket （root  ssd），将7个ssd相关的bucket添加进来成为叶子节点。注意修改（root default）中的weight，减掉移走的ssd的比重。
3. 增加一个ssd的规则，跟默认一样，就修改成只用ssd这个bucket。（rule ssd_ruleset)

## 编译crush map并注入

一旦修改好了crush map，就要编译成ceph用的格式

```sh
[root@store01 ~]# crushtool -c crush.map.txt -o crush.map.v1
[root@store01 ~]# ceph osd setcrushmap -i crush.map.v1
set crush map
```

再看一下osdtree的变化

```sh
[root@store01 ~]# ceph osd tree
ID  WEIGHT   TYPE NAME            UP/DOWN REWEIGHT PRIMARY-AFFINITY
-16  0.55991 root ssd
 -9  0.07999     host store01-ssd
 21  0.07999         osd.21            up  1.00000          1.00000
-10  0.07999     host store02-ssd
 22  0.07999         osd.22            up  1.00000          1.00000
-11  0.07999     host store03-ssd
 23  0.07999         osd.23            up  1.00000          1.00000
-12  0.07999     host store04-ssd
 24  0.07999         osd.24            up  1.00000          1.00000
-13  0.07999     host store05-ssd
 25  0.07999         osd.25            up  1.00000          1.00000
-14  0.07999     host store06-ssd
 26  0.07999         osd.26            up  1.00000          1.00000
-15  0.07999     host store07-ssd
 27  0.07999         osd.27            up  1.00000          1.00000
 -1 76.37697 root default
 -2 10.91100     host store01
  0  3.63699         osd.0             up  1.00000          1.00000
  2  3.63699         osd.2             up  1.00000          1.00000
  1  3.63699         osd.1             up  1.00000          1.00000
  ...
 -8 10.91100     host store07
 18  3.63699         osd.18            up  1.00000          1.00000
 19  3.63699         osd.19            up  1.00000          1.00000
 20  3.63699         osd.20            up  1.00000          1.00000
```

> 可以发现crush图生效了

## 创建一个pool使用ssd的crush规则

```sh
[root@store01 ~]# ceph osd pool create -h
[root@store01 ~]# ceph osd pool create xenpool 256 256 ssd_ruleset
pool 'xenpool' created

```

> pg_num，pgp_num不指定就使用ceph.conf中的配置的值

检查一下

```sh
[root@store01 ~]# ceph osd lspools
0 rbd,2 xenpool,
[root@store01 ~]# ceph osd pool get xenpool pg_num
pg_num: 256
[root@store01 ~]# ceph osd pool get xenpool crush_ruleset
crush_ruleset: 1
```



## pool 属性的修改

删除pool

```sh
[root@store01 ~]# ceph osd pool rm xenpool --yes-i-really-really-mean-it
Error EPERM: WARNING: this will *PERMANENTLY DESTROY* all data stored in pool xenpool.  If you are *ABSOLUTELY CERTAIN* that is what you want, pass the pool name *twice*, followed by --yes-i-really-really-mean-it.
[root@store01 ~]# ceph osd pool rm xenpool xenpool --yes-i-really-really-mean-it
Error EPERM: pool deletion is disabled; you must first set the mon_allow_pool_delete config option to true before you can destroy a pool
[root@store01 ~]# ceph tell mon.* injectargs --mon_allow_pool_delete=true
mon.store02: injectargs:mon_allow_pool_delete = 'true' (unchangeable)
mon.store03: injectargs:mon_allow_pool_delete = 'true' (unchangeable)
mon.store04: injectargs:mon_allow_pool_delete = 'true' (unchangeable)
[root@store01 ~]# ceph osd pool rm xenpool xenpool --yes-i-really-really-mean-it
pool 'xenpool' removed
[root@store01 ~]# ceph tell mon.* injectargs --mon_allow_pool_delete=false
```



创建一个默认规则的pool

```sh
[root@store01 ~]# ceph osd pool create xenpool 256
pool 'xenpool' created
[root@store01 ~]# ceph osd pool ls detail
pool 0 'rbd' replicated size 3 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 512 pgp_num 512 last_change 2966 flags hashpspool stripe_width 0
pool 3 'xenpool' replicated size 3 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 4914 flags hashpspool stripe_width 0
```

在线修改属性

```sh
[root@store01 ~]# ceph osd pool set xenpool size 2
set pool 3 size to 2
[root@store01 ~]# ceph osd pool set xenpool crush_ruleset 1
set pool 3 crush_ruleset to 1
```



**xenpool的编号变成3了，删除的pool编号（1，2）怎么重用呢？！**

## 奇怪的问题

修改了store01的网络后，store01的4个osd都down了，`systemctl start ceph-osd.target`重启后，发现crush图变了

```
[root@store01 test]# ceph osd tree
ID  WEIGHT   TYPE NAME            UP/DOWN REWEIGHT PRIMARY-AFFINITY
-16  0.47992 root ssd
 -9        0     host store01-ssd
-10  0.07999     host store02-ssd
 22  0.07999         osd.22            up  1.00000          1.00000
-11  0.07999     host store03-ssd
 23  0.07999         osd.23            up  1.00000          1.00000
 ...
 -1 76.45694 root default
 -2 10.99097     host store01
  0  3.63699         osd.0             up  1.00000          1.00000
  2  3.63699         osd.2             up  1.00000          1.00000
  1  3.63699         osd.1             up  1.00000          1.00000
 21  0.07999         osd.21            up  1.00000          1.00000
 ...
```

> osd.21又回到了root default？？？

通过ceph osd getcrushmap回来后查看，发现store01-ssd这个bucket还在，osd.21的item移动到store01这个bucket里去了，暂时还不知道为什么。

手动改回来先。

```
[root@store01 test]# ceph osd crush set osd.21 0.08 root=ssd host=store01-ssd
set item id 21 name 'osd.21' weight 0.08 at location {host=store01-ssd,root=ssd} to crush map
[root@store01 test]# ceph osd tree
ID  WEIGHT   TYPE NAME            UP/DOWN REWEIGHT PRIMARY-AFFINITY
-16  0.55991 root ssd
 -9  0.07999     host store01-ssd
 21  0.07999         osd.21            up  1.00000          1.00000
 ...
```

解决方法：

google一圈在这里找到了：https://elkano.org/blog/ceph-sata-ssd-pools-server-editing-crushmap/

修改ceph.conf

```
...

[osd.21]
crush_location = root=ssd host=store01-ssd

[osd.22]
crush_location = root=ssd host=store02-ssd
...
[hpec@store01 store-cluster]$ ceph-deploy --overwrite-conf config  push store0{1..7}
...
## 可以重启一下检验一下
[hpec@store01 store-cluster]$ ceph-crush-location --id 27 --type osd
root=ssd host=store07-ssd
```

//END
