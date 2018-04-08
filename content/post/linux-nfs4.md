+++
date = "2018-04-08T10:19:21+08:00"
description = ""
categories = ["linux"]
tags = ["nfs4"]
thumbnail = ""
title = "centos7配置nfs4"

+++

cenots7下配置nfs4

<!--more-->

# 安装

```shell
# yum install -y nfs-utils
# systemctl enable nfs-server
# systemctl start nsf-server
```

# 配置

```shell
# vi /etc/exports
/samba/public   *(rw,sync,no_subtree_check,all_squash,anonuid=99,anongid=99)

# exportfs -rav #使配置立即生效
```

> 具体选项查看`man exports`
>
> /samba/public 目录注意权限



# 防火墙

```sh el
# firewall-cmd --permanent --add-service=nfs
# firewall-cmd --permanent --add-service=mountd
# firewall-cmd --permanent --add-service=rpc-bind
# firewall-cmd --reload
```



# 客户端挂载



```sh el
# showmount -e 192.168.210.10
# mount -t nfs4 192.168.210.10:/samba/public /mnt
# mount
```

开机启动就挂载就编辑/etc/fstab文件

```shell
# vi /etc/fstab
192.168.210.10:/samba/public   /mnt   nfs   defaults    0 0
# mount -a
```



//END

