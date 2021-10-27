+++
categories = ["docker"]
date = "2015-07-31T10:22:54+08:00"
description = ""
tags = ["etcd","upgrade"]
thumbnail = ""
title = "etcd 2.0 rolling upgrade 升级到 etcd 2.1"

+++

etcd2.0开始支持rolling update 正好手上有个测试环境,升级一下做个记录

<!--more-->

# etcd2.0升级到etcd2.1

etcd2.0开始支持rolling upgrade

## 环境说明

- 3台都是centos 7
- etcd 版本2.0.11
- kubernetes 1.0
- flannel 0.2.0

## 升级前检查

```console
[root@srv01 ~]# etcdctl -v
etcdctl version 2.0.11
[root@srv01 ~]# curl -L http://127.0.0.1:2379/version
etcd 2.0.11
[root@srv01 ~]# etcdctl cluster-health
cluster is healthy
member 9ec800ecd4c0c36c is healthy
member a033297505a2b430 is healthy
member c7b0353c799612f2 is healthy
```

## 停止etcd进程

```console
[root@srv01 ~]# systemctl stop etcd
[root@srv01 ~]# systemctl status etcd -l
etcd.service - Etcd Server
   Loaded: loaded (/usr/lib/systemd/system/etcd.service; enabled)
   Active: inactive (dead) since Thu 2015-07-30 11:28:45 CST; 9s ago
 Main PID: 12027 (code=killed, signal=TERM)

Jul 30 11:28:45 srv01 systemd[1]: Stopping Etcd Server...
Jul 30 11:28:45 srv01 bash[12027]: 2015/07/30 11:28:45 received terminated signal, shutting down...
Jul 30 11:28:45 srv01 bash[12027]: 2015/07/30 11:28:45 rafthttp: encountered error reading the client log stream: read tcp 192.168.1.206:2380: use of closed network connection
Jul 30 11:28:45 srv01 bash[12027]: 2015/07/30 11:28:45 rafthttp: client streaming to c7b0353c799612f2 at term 8595 has been stopped
Jul 30 11:28:45 srv01 systemd[1]: Stopped Etcd Server.
Warning: Journal has been rotated since unit was started. Log output is incomplete or unavailable.
```

## 备份数据

查看数据存放目录

```console
[root@srv01 ~]# grep DATA /etc/etcd/etcd.conf
ETCD_DATA_DIR="/var/lib/etcd/srv01.etcd"
[root@srv01 ~]# etcdctl backup --data-dir=/var/lib/etcd/srv01.etcd/ --backup-dir etcd-backup
```

## 下载etcd2.1并进行升级

```console
[root@srv01 ~]# curl -L  https://github.com/coreos/etcd/releases/download/v2.1.1/etcd-v2.1.1-linux-amd64.tar.gz -o etcd-v2.1.1-linux-amd64.tar.gz
[root@srv01 ~]# tar -xzvf etcd-v2.1.1-linux-amd64.tar.gz
[root@srv01 ~]# cp etcd-v2.1.1-linux-amd64/etcd* /usr/bin/
```

## 启动

可以查看一下systemd启动脚本检查下

```console
[root@srv01 ~]# cat /usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target

[Service]
Type=simple
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
User=etcd

# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/bin/etcd"

[Install]
WantedBy=multi-user.target
[root@srv01 ~]# systemctl start etcd
```

## 检查一下状态

```console
[root@srv01 ~]# etcdctl -v
etcdctl version 2.1.1
[root@srv01 ~]#
[root@srv01 ~]# systemctl status etcd -l
etcd.service - Etcd Server
   Loaded: loaded (/usr/lib/systemd/system/etcd.service; enabled)
   Active: active (running) since Thu 2015-07-30 11:58:07 CST; 2min 7s ago
 Main PID: 4404 (etcd)
   CGroup: /system.slice/etcd.service
           4404 /usr/bin/etcd

Jul 30 11:58:08 srv01 bash[4404]: 2015/07/30 11:58:08 etcdserver: advertise client URLs = http://192.168.1.205:2379,http://localhost:2379
Jul 30 11:58:08 srv01 bash[4404]: 2015/07/30 11:58:08 etcdserver: loaded cluster information from store: <nil>
Jul 30 11:58:08 srv01 bash[4404]: 2015/07/30 11:58:08 etcdserver: restarting member 9ec800ecd4c0c36c in cluster 9a7804897ebf8b5d at commit index 2373689
Jul 30 11:58:08 srv01 bash[4404]: 2015/07/30 11:58:08 raft: 9ec800ecd4c0c36c became follower at term 8595
Jul 30 11:58:08 srv01 bash[4404]: 2015/07/30 11:58:08 raft: newRaft 9ec800ecd4c0c36c [peers: [9ec800ecd4c0c36c,a033297505a2b430,c7b0353c799612f2], term: 8595, commit: 2373689, applied: 2370325, lastindex: 2373691, lastterm: 8595]
Jul 30 11:58:08 srv01 bash[4404]: 2015/07/30 11:58:08 etcdserver: starting server... [version: 2.1.1, cluster version: to_be_decided]
Jul 30 11:58:08 srv01 bash[4404]: 2015/07/30 11:58:08 rafthttp: the connection with c7b0353c799612f2 became active
Jul 30 11:58:08 srv01 bash[4404]: 2015/07/30 11:58:08 rafthttp: the connection with a033297505a2b430 became active
Jul 30 11:58:08 srv01 bash[4404]: 2015/07/30 11:58:08 raft: raft.node: 9ec800ecd4c0c36c elected leader c7b0353c799612f2 at term 8595
Jul 30 11:58:08 srv01 bash[4404]: 2015/07/30 11:58:08 etcdserver: published {Name:srv01 ClientURLs:[http://192.168.1.205:2379 http://localhost:2379]} to cluster 9a7804897ebf8b5d
[root@srv01 ~]# etcdctl cluster-health
cluster is healthy
member 9ec800ecd4c0c36c is healthy
member a033297505a2b430 is healthy
member c7b0353c799612f2 is healthy
```

看看此时的etcd集群版本

``` console
[root@srv01 ~]# curl -L http://127.0.0.1:2379/version
{"etcdserver":"2.1.1","etcdcluster":"not_decided"}
```

> 现在是2.0和2.1的混合状态,2.1版本的新功能在此状态下尚未可用.待全部节点升级到2.1时才可用

## 其他节点依次升级

参考上面步骤,备份可以略过

## 检查集群状态

最后一个节点升级并启动后..可以看到log变化

```console
[root@srv03 ~]# systemctl status etcd
......
Jul 30 13:14:32 srv03 bash[25997]: 2015/07/30 13:14:32 raft: raft.node: a033297505a2b430 elected leader 9ec800ecd4c0c36c at term 8596
Jul 30 13:14:33 srv03 bash[25997]: 2015/07/30 13:14:33 etcdserver: published {Name:srv03 ClientURLs:[http://192.168.1.207:2379 http://loc...97ebf8b5d
**Jul 30 13:14:37 srv03 bash[25997]: 2015/07/30 13:14:37 etcdserver: updated the cluster version from 2.0.0 to 2.1.0**
```

此时再查看集群版本

```console
[root@srv03 ~]# curl -L http://127.0.0.1:2379/version
{"etcdserver":"2.1.1","etcdcluster":"2.1.0"}
```

检查集群状态

```console
[root@srv03 ~]# etcdctl cluster-health
cluster is healthy
member 9ec800ecd4c0c36c is healthy
member a033297505a2b430 is healthy
member c7b0353c799612f2 is healthy
```

kubernetes集群也正常

```console
[root@srv01 ~]# kubectl get nodes,pods
NAME      LABELS                         STATUS
srv01     kubernetes.io/hostname=srv01   Ready
srv02     kubernetes.io/hostname=srv02   Ready
srv03     kubernetes.io/hostname=srv03   Ready
NAME          READY     STATUS    RESTARTS   AGE
nginx-l51p3   1/1       Running   0          1d
nginx-mtnu5   1/1       Running   0          1d
```

## 参考资料

https://coreos.com/etcd/docs/latest/upgrade_2_1.html

//END

