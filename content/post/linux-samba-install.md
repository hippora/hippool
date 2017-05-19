+++

categories = ["linux"]

date = "2017-05-19T15:14:11+08:00"

description = ""

tags = ["samba"]

thumbnail = ""

title = "samba 安装配置"

+++

记录一下samba的安装配置

<!--more-->

## centos 下安装

```
[root@samba-srv ~]# yum install samba samba-client -y
```

## 配置

### 准备

为每个部门添加一个组。
```
[root@samba-srv ~]# groupadd legal
[root@samba-srv ~]# groupadd tech
[root@samba-srv ~]# groupadd ecomm
[root@samba-srv ~]# groupadd sales
```

创建几个文件夹

```
[root@samba-srv ~]# mkdir -p /data/samba/{public,tmp,legal,tech,ecomm,sales}
[root@samba-srv ~]# chmod 777 /data/samba/*
```

### 配置samba服务

总配置

```
[root@samba-srv ~]# cd /etc/samba/
[root@samba-srv samba]# cat smb.conf
[global]
	# server
	workgroup = WORDGROUP
	server string = Samba(%v) on %L
;	netbios name = SAMBA-SRV
;	hosts allow = 127. 192.168.1. 192.168.10. 192.168.30.

	# log
	log level = 2
	log file = /var/log/samba/log.%m
	max log size = 10240

	security = user
	passdb backend = tdbsam
	map to guest = Bad User
;	guest account = pcguest

;	printing = cups
;	printcap name = cups
	load printers = no
;	cups options = raw

	include = /etc/samba/smb.conf.%G
[public]
	comment = Public Space
	path = /data/samba/public
	browseable = yes
	guest ok = yes
	write list = @tech
;	create mode = 0660
;	directory mode = 0660

[tmp]
	comment = Tmp Space
	path = /data/samba/tmp
	volume = volumn
	writeable = yes
	browseable = yes
	guest ok = yes
```

> `tmp` 匿名可以读写，同事交换文件用，
> `public`放点同事可以用的软件，只由技术部同事可写

几个部门配置

```
[root@samba-srv samba]# cat smb.conf.tech
[技术部]
	comment = Tech Department
	path = /data/samba/tech
	browseable = yes
	valid users = @tech
	write list = @tech
;	create mode = 0660
;	directory mode = 0660
```

> 其他几个部门配置复制改一下,配置名字要跟总配置的include名字匹配。
> 只有同部门的同事可以读写

### 启动samba

```
[root@samba-srv ~]# systemctl enable smb
[root@samba-srv ~]# systemctl enable nmb
[root@samba-srv ~]# systemctl start smb
[root@samba-srv ~]# systemctl start nmb
```

### 防火墙，selinux 

略

### 添加用户

```
[root@samba-srv ~]# useradd -M -g tech -s /usr/sbin/nologin jack
[root@samba-srv ~]# pdbedit -a jack
```

写了个建用户的脚本

```
[root@samba-srv ~]# cat add_samba_account.sh
#!/usr/bin/bash

USER=${1}
GROUP=${2}
GROUPNAME=
EMAIL=$USER@example.com
PASSWD=`date | md5sum |cut -c1-8`

usage () {
	echo
	echo "usage: $0 user group"
	echo
	echo "group should in : legal,tech,sales,ecomm"
	return 0
}

if [ $# -ne 2 ]; then usage; exit 1; fi;


for i in legal=法务部 tech=技术部 sales=销售部 ecomm=电商部; do
	if [ "$GROUP" = `echo $i | cut -d'=' -f1` ]; then GROUPNAME=`echo $i | cut -d'=' -f2` ; fi
done;

if [ -z $GROUPNAME ]; then
	echo "group error!"
	usage
	exit 1;
fi

# =====

useradd -M -g $GROUP -s /usr/sbin/nologin $USER
echo -e "$PASSWD\n$PASSWD" | pdbedit -a $USER -t

echo -e "Hi,$USER\n" > tmp.txt
echo -e "你的网络共享账号已经创建成功！\n" >> tmp.txt
echo '访问地址: \\192.168.1.200' >> tmp.txt
echo '部门目录: \\192.168.1.200\'$GROUP >> tmp.txt
echo -e "账号：$USER\t密码：$PASSWD\n"  >> tmp.txt

cat tmp.txt | mutt -s "Samba网络共享账号" $EMAIL

rm -f tmp.txt
```

### 手动修改密码

```
[root@samba-srv ~]# smbpasswd jack
```

### 删除用户

```
[root@samba-srv ~]# pdbedit -x jack
```

## 其他

有些windows系统，需要修改安全策略才能使用samba。账号密码正确却登录不了的可以设置试试：

开始运行 secpol.msc -> 本地策略 -> 安全选项 -> 网络安全: LAN 管理器身份验证级别 -> 改成"发送 LM & NTLM - 如果协商一致，则使用 NTLMv2 会话安全"

//END
