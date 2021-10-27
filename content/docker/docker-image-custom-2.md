+++
categories = ["docker"]
date = "2016-05-20T11:23:11+08:00"
description = ""
tags = ["image","dockerfile"]
thumbnail = ""
title = "制作自己的docker镜像(二)"

+++

使用dockerfile

<!--more-->

## 制作自己的镜像(二)

### 使用dockerfile

`docker commit`方式创建镜像比较直观.但是不容易分发共享.还有种方法比较常用,就是使用dockerfile

新建两个目录,创建两个文件

```
[root@srv00 ~]# mkdir df && cd df
[root@srv00 df]# cat nginx.repo
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=0
enabled=1
[root@srv00 df]# cat Dockerfile
FROM centos:latest
MAINTAINER hippo <x@xxx.com>
COPY nginx.repo /etc/yum.repos.d/
RUN yum install -y nginx && echo "daemon off;" >> /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx"]
```

> `Dockerfile`是默认文件名,`docker build -f`来指定自定义文件
> 一行一条指令.有点像shell.
> `FROM`基于哪个镜像.`MAINTAINER`维护者信息.`COPY`将本地文件copy到镜像目录中. `RUN`在镜像中运行的命令.
> `EXPOSE`暴露端口号给外部映射.`CMD`如果运行容器会执行的命令.(控制台执行.不然容器会马上退出).

运行`docker build`

```
[root@srv00 df]# docker build -t hippo/nginx:v1 .
Sending build context to Docker daemon 3.072 kB
Step 1 : FROM centos:latest
 ---> 8596123a638e
Step 2 : MAINTAINER hippo <x@xxx.com>
 ---> Using cache
 ---> c04988102337
Step 3 : COPY nginx.repo /etc/yum.repos.d/
 ---> 840a6358f3d1
Removing intermediate container 8cb81de3f7e9
Step 4 : RUN yum install -y nginx && echo "daemon off;" >> /etc/nginx/nginx.conf
 ---> Running in 5a27d8a4bc77
Loaded plugins: fastestmirror, ovl
......
Complete!
 ---> 45b53927ed9a
Removing intermediate container 5a27d8a4bc77
Step 5 : EXPOSE 80
 ---> Running in f022d6097efa
 ---> 29429605ebc7
Removing intermediate container f022d6097efa
Step 6 : CMD nginx
 ---> Running in c7faa5042715
 ---> 2e1513eeaa0a
Removing intermediate container c7faa5042715
Successfully built 2e1513eeaa0a
```

> 每条指令都相当于`git commit`一次.

运行测试下

```
[root@srv00 df]# docker run -d -p 80 hippo/nginx:v1
dead20777b6c1609ab968966b3589904d44f8a12c124c178fd5cb540052cce6f
[root@srv00 df]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                     NAMES
dead20777b6c        hippo/nginx:v1      "nginx"                  11 seconds ago      Up 8 seconds        0.0.0.0:32771->80/tcp     gloomy_cray
[root@srv00 df]# curl -L localhost:32771
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
......
```

//END

