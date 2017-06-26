+++
categories = ["linux"]
date = "2017-06-26T11:00:05+08:00"
description = ""
tags = ["ssh","autossh"]
thumbnail = ""
title = "linux autossh"

+++

linux autossh

<!--more-->

## 安装autossh

```
yum install -y autossh
```

## 设置面密码登录

略

## 配置服务自启

```
[root@srv00 ~]# vi /etc/systemd/system/autossh.service
[Unit]
Description=Keeps a tunnel open
After=network.target sshd.service

[Service]
ExecStart=/usr/bin/autossh -M 0 -q -o ServerAliveInterval=30 -o ServerAliveCountMax=30 -N -R 50099:localhost:22 user1@example.com -i /root/.ssh/id_rsa
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
[root@srv00 ~]# systemctl enable autossh
[root@srv00 ~]# systemctl start autossh
```

//END
