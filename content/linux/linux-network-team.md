+++
categories = ["linux"]
date = "2017-02-28T13:27:06+08:00"
description = ""
tags = ["nmcli","teamdctl","team","bond"]
thumbnail = ""
title = "NetworkManager配置网络team"

+++

NetworkManager用起来还挺顺手的，比直接改ifcfg方便.
team呢也在慢慢替换老的bond了.

<!--more-->

## team

这次用nmcli 来配置个team

```
[root@store01 ~]# vi team0.sh
#!/usr/bin/bash
nmcli c add type team con-name team0 ifname team0
nmcli c mod team0 ipv4.addresses 192.168.210.11/24 ipv4.gateway 192.168.210.1 ipv4.method manual ipv6.method ignore
nmcli c mod team0 team.config '{"runner":{"name":"loadbalance"}}'
nmcli c add type team-slave con-name team0-em1 ifname em1 master team0
nmcli c add type team-slave con-name team0-em2 ifname em2 master team0
nmcli c up team0-em1
nmcli c up team0-em2
nmcli c 
[root@store01 ~]# sh team0.sh
...
```

> runner就是各种负载方式,有几种可以搜一下redhat的官方文档
> 可以装一个iperf3来测试一下网络

teamdctl

```
teamdctl team0 config dump
```

## bond

有些情况下bond反而还稳定些，创建方式跟team类似

```
[root@store01 ~]# vi bond0.sh
#!/usr/bin/bash
nmcli c add type bond con-name bond0 ifname bond0
nmcli c mod bond0 ipv4.addresses 192.168.210.${1}/24 ipv4.gateway 192.168.210.1 ipv4.method manual ipv6.method ignore
nmcli c mod bond0 bond.options "miimon=100,updelay=0,downdelay=0,mode=balance-alb"
nmcli c add type bond-slave con-name bond0-em1 ifname em1 master bond0
nmcli c add type bond-slave con-name bond0-em2 ifname em2 master bond0
nmcli c up bond0-em1
nmcli c up bond0-em2
nmcli c
[root@store01 ~]# sh bond0.sh 10
...
```

> bond几种模式redhat官网也有

//END
