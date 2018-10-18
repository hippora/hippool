+++
categories = ["linux"]
date = "2018-07-30T10:19:21+08:00"
description = ""
tags = ["rsync","sersync"]
thumbnail = ""
title = "rsync+sersync配置"

+++

正好需要用到rsync+sersync.记录一下基本同步配置

<!--more-->

# rsync daemon 方式配置

## 实验机器

两台均为安装centos 7.5 的机器

- srv1:192.168.1.101  作为sersync client
- srv2:192.168.1.102  作为rsync server

两边配好/etc/hosts先

## server端

### 安装

```
[root@srv2 ~]# yum install -y rsync
```

### 配置文件设置

```

[root@srv2 ~]# vi /etc/rsyncd.conf
uid = root
gid = root
use chroot = yes
max connections = 4
pid file = /var/run/rsyncd.pid
exclude = lost+found/
transfer logging = yes
timeout = 900
ignore nonreadable = yes
dont compress   = *.gz *.tgz *.zip *.z *.Z *.rpm *.deb *.bz2

[images]
path = /data/images/
comment = images directory
read only = false
#auth users = rsync
#secrets file = /etc/rsyncd.secrets
hosts allow = srv1
hosts deny = 0.0.0.0/32

```

### 设置开机自启动

```
[root@srv1 ~]# systemctl enable rsyncd
[root@srv1 ~]# systemctl start rsyncd
```

## client 端

### 下载sersync,解压到/opt/sersync

```
https://github.com/wsgzao/sersync

```

### 配置sersync

```
[root@srv1 ~]# vi /opt/sersync/confxml.xml 
...
    <fileSystem xfs="true"/>
    <filter start="false">
        <exclude expression="(.*)\.svn"></exclude>
        <exclude expression="(.*)\.gz"></exclude>
        <exclude expression="^info/*"></exclude>
        <exclude expression="^static/*"></exclude>
    </filter>
    <inotify>
        <delete start="true"/>
        <createFolder start="true"/>
        <createFile start="false"/>
        <closeWrite start="true"/>
        <moveFrom start="true"/>
        <moveTo start="true"/>
        <attrib start="true"/>
        <modify start="true"/>
    </inotify>

    <sersync>
        <localpath watch="/data/images">
            <remote ip="srv1" name="images"/>
...

[root@srv1 ~]# vi /etc/systemd/system/sersync.service 
[Unit]
Description=sersync daemon
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/sersync/
ExecStart=/opt/sersync/sersync2
ExecStop=/usr/bin/killall sersync2
Restart=always
RestartSec=30

[Install]
WantedBy=multi-user.target

[root@srv1 ~]# systemctl daemon-reload
[root@srv1 ~]# systemctl enable sersync
[root@srv1 ~]# systemctl start sersync

```

### 双向同步配置

按照上面步骤srv1当rsync服务器，srv2当客户端再配置一下就好了。

## 测试

略

//END

