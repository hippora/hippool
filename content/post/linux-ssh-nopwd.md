+++
categories = ["linux"]
date = "2015-11-19T11:00:05+08:00"
description = ""
tags = ["ssh"]
thumbnail = ""
title = "linux ssh 免密码登录"

+++

linux ssh 免密码登录

<!--more-->

## 方法1:ssh-copy-id (推荐)

hosta ssh 到hostb 免密码

### 生成密钥

```
$ ssh-keygen -t rsa
$ ssh-copy-id user01@hostb
```

> 第一次需要输入密码

### 验证

```
$ ssh user01@hostb date
```

## 方法2:手动操作

将hosta的pub key 添加到hostb的authorized_keys中

### a生成密钥

```
$ ssh-keygen -t rsa
$ cat ~/.ssh/id_rsa.pub
```

### 将公钥复制到hostb用户下的.ssh/authorized_keys中

> 略,可以通过scp

### 权限问题

> 检查.ssh目录是700,authorized_keys是600
> 否则还是会让你输入密码

### 双向免密码

将hostb的pub key添加到a的authorized_keys中

### 验证

```
$ ssh user01@hosta date
```

//END

