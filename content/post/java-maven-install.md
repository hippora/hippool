+++
categories = ["java"]
date = "2015-09-25T10:29:52+08:00"
description = ""
tags = ["maven"]
thumbnail = ""
title = "maven安装"

+++

linux下maven安装

<!--more-->

## 下载解压

```
curl -#O http://mirrors.tuna.tsinghua.edu.cn/apache/maven/maven-3/3.3.9/binaries/apache-maven-3.3.9-bin.tar.gz
tar -xzf apache-maven-3.3.9-bin.tar.gz -C /opt
```

## 设置环境变量

```
vi /etc/profile.d/custom.sh

PATH=/opt/apache-maven-3.3.9/bin:$PATH; export PATH
```

立即生效

```
. /etc/profile
mvn -v
```

> 这是配置系统级别所有用户都可以使用mvn,单独用户请配置~/.bash_profile

//END

