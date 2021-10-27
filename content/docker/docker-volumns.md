+++
categories = ["docker"]
date = "2016-05-23T11:28:14+08:00"
description = ""
tags = ["volumns"]
thumbnail = ""
title = "docker管理容器的数据(VOLUMNS)"

+++

可以通过两种方式管理容器中的数据,数据卷(Data Volumns),数据卷容器(Data Volumns containers)

<!--more-->

## 数据卷(Data Volumns)

### 概念:

`data volume`存在一个或多个容器中的特殊的目录,它绕过容器的`Union File System`.用来持久或共享数据.它是独立于容器的生命周期的. 它有几个特点:

1. 容器创建好后会初始化数据卷.如果基础镜像在此挂载点上有数据,它会将数据复制到数据卷里.(挂在主机的目录则不会复制)
2. 数据卷可以被多个容器共享使用
3. 对数据卷的修改是直接的.
4. 对数据卷的更改,不会因为你更新镜像而被包括进去.
5. 就算容器被删除,数据卷还继续存在.

### 添加数据卷

```
[root@srv00 ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
hippo/nginx         v1                  2e1513eeaa0a        2 days ago          281.1 MB
hippo/nginx         v1.10               1e20546f8434        2 days ago          281.1 MB
[root@srv00 ~]# docker run -it -v /etc/nginx hippo/nginx:v1.10
[root@bab2e5725220 /]# ll /etc/nginx/
total 32
drwxr-xr-x 2 root root   25 May 20 07:40 conf.d
...
```
> 可以看到容器内的数据卷目录中,是原有的`/etc/nginx`目录的内容.

可以修改一个文件配置看看.并检查主机上对应的数据卷内容.

```
[root@srv00 ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS                      PORTS                   NAMES
bab2e5725220        hippo/nginx:v1.10   "/bin/bash"              About a minute ago   Exited (0) 15 seconds ago                           hungry_jennings
[root@srv00 ~]# docker inspect bab2e5725220
...
        "Mounts": [
            {
                "Name": "b6c506a65d5f495ca6ce06aeb30bb6472295d459bfbeb464e89681e17ffe3541",
                "Source": "/var/lib/docker/volumes/b6c506a65d5f495ca6ce06aeb30bb6472295d459bfbeb464e89681e17ffe3541/_data",
                "Destination": "/etc/nginx",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
...

[root@srv00 _data]# docker rm bab2e5725220
bab2e5725220
[root@srv00 ~]# cat /var/lib/docker/volumes/b6c506a65d5f495ca6ce06aeb30bb6472295d459bfbeb464e89681e17ffe3541/_data/nginx.conf
...
```

> 对数据卷的修改直接持久在主机对应的地方..就算容器已经关闭或删除.

### 挂载主机目录

```
[root@srv00 ~]# mkdir nginx
[root@srv00 ~]# docker run -it -v /root/nginx:/etc/nginx hippo/nginx:v1.10
[root@c0750f9779b3 /]# ll /etc/nginx/  <===无内容
[root@srv00 ~]# docker ps -l
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                          PORTS                   NAMES
c0750f9779b3        hippo/nginx:v1.10   "/bin/bash"              23 seconds ago      Exited (0) 11 seconds ago                               agitated_mclean
[root@srv00 ~]# docker inspect c0750f9779b3
...
        "Mounts": [
            {
                "Source": "/root/nginx",
                "Destination": "/etc/nginx",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            }
...
```

> 可以看到是主机的对应目录挂载到容器内.`RW`是可读写标志,效果和mount命令一致.

如果是相对目录:

```
[root@srv00 ~]# mkdir -p a/b
[root@srv00 ~]# touch a/b/c
[root@srv00 ~]# docker run -it -v a/b:/etc/nginx hippo/nginx:v1.10
docker: Error response from daemon: create a/b: "a/b" includes invalid characters for a local volume name, only "[a-zA-Z0-9][a-zA-Z0-9_.-]" are allowed.
See 'docker run --help'.
[root@srv00 ~]# docker run -it -v nginx:/etc/nginx hippo/nginx:v1.10
e4e2f137a7412f6474018e88985e2f605b5f77f2cd1ce27280142994e5feb030
[root@srv00 ~]# docker inspect e4e2f137a7412f6474018e88985e2f605b5f77f2cd1ce27280142994e5feb030
...
        "Mounts": [
            {
                "Name": "nginx",
                "Source": "/var/lib/docker/volumes/nginx/_data",
                "Destination": "/etc/nginx",
                "Driver": "local",
                "Mode": "z",
                "RW": true,
                "Propagation": "rprivate"
            }
...
```

> 主机不可以指定相对目录,只可以对数据卷命名.

当然你可以以只读方式挂载.

```
[root@srv00 ~]# docker run -itd -v nginx:/etc/nginx:ro hippo/nginx:v1.10
b0a67f835eaac812ab37086bef05be6b0726ba080d73113221a5cf8a25bfb513
[root@srv00 ~]# docker inspect b0a67f835eaac812ab37086bef05be6b0726ba080d73113221a5cf8a25bfb513
...
        "Mounts": [
            {
                "Name": "nginx",
                "Source": "/var/lib/docker/volumes/nginx/_data",
                "Destination": "/etc/nginx",
                "Driver": "local",
                "Mode": "ro",
                "RW": false,
                "Propagation": "rprivate"
            }
  ...
```

### 单独创建数据卷

之前都是通过运行容器的同时创建数据卷.当然也可以先创建好数据卷.然后挂载到容器中使用.
数据卷管理使用`docker volume`命令.我们看一下刚才创建了哪些数据卷.

```
[root@srv00 ~]# docker volume ls
DRIVER              VOLUME NAME
local               0159a70ae1a86b5c2947d323b5e6d539101770b66af5cb0ba6d87c12a1c2c742
local               b6c506a65d5f495ca6ce06aeb30bb6472295d459bfbeb464e89681e17ffe3541
local               nginx
[root@srv00 ~]# docker volume inspect 0159a70ae1a86b5c2947d323b5e6d539101770b66af5cb0ba6d87c12a1c2c742
[
    {
        "Name": "0159a70ae1a86b5c2947d323b5e6d539101770b66af5cb0ba6d87c12a1c2c742",
        "Driver": "local",
        "Mountpoint": "/var/lib/docker/volumes/0159a70ae1a86b5c2947d323b5e6d539101770b66af5cb0ba6d87c12a1c2c742/_data",
        "Labels": null
    }
]
[root@srv00 ~]# docker volume inspect nginx
[
    {
        "Name": "nginx",
        "Driver": "local",
        "Mountpoint": "/var/lib/docker/volumes/nginx/_data",
        "Labels": null
    }
]
```

> `Driver`是local,就是映射到本地主机.也可以使用其他驱动.映射到比如iscsi,NFS,FC 等共享磁盘上.

我们来单独创建一个数据卷并使用

```
[root@srv00 ~]# docker volume create --name dbdata
dbdata
[root@srv00 ~]# docker volume ls
DRIVER              VOLUME NAME
local               0159a70ae1a86b5c2947d323b5e6d539101770b66af5cb0ba6d87c12a1c2c742
local               b6c506a65d5f495ca6ce06aeb30bb6472295d459bfbeb464e89681e17ffe3541
local               dbdata
local               nginx
[root@srv00 ~]# docker volume inspect dbdata
[
    {
        "Name": "dbdata",
        "Driver": "local",
        "Mountpoint": "/var/lib/docker/volumes/dbdata/_data",
        "Labels": {}
    }
]
[root@srv00 ~]# docker run -itd -v dbdata:/etc/nginx hippo/nginx:v1.10
```

## 数据卷容器(Data Volumns containers)

我们先创建个数据卷容器.然后运行两个单独的容器,挂载数据卷容器.

```
[root@srv00 ~]# docker create -v /opt --name datastore centos
e5c37981c4f9e16f92b0d5215f5dfa5f79c86282e3018435a5e380350a5cce2c
[root@srv00 ~]# docker ps -l
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
e5c37981c4f9        centos              "/bin/bash"         11 seconds ago      Created                                 datastore
[root@srv00 ~]# docker run -itd --volumes-from datastore --name db1 centos
[root@srv00 ~]# docker run -itd --volumes-from datastore --name db2 centos
```

看看数据是否互相可访问.

```
[root@srv00 ~]# docker exec -it db1 /bin/bash
[root@cd8fa62d3b29 ~]# echo "db111111" > /opt/txt
[root@cd8fa62d3b29 ~]# exit
exit
[root@srv00 ~]# docker exec -it db2 /bin/bash
[root@72c21395c72b /]# cat /opt/txt
db111111
[root@72c21395c72b /]# exit
exit
[root@srv00 ~]# docker inspect datastore
[root@srv00 ~]# docker inspect db1
[root@srv00 ~]# docker inspect db2
...
        "Mounts": [
            {
                "Name": "6b650687d6d2a448006dcb2b966b5f69bb31d34cf3e93f5e3c3460b6fd6eea4a",
                "Source": "/var/lib/docker/volumes/6b650687d6d2a448006dcb2b966b5f69bb31d34cf3e93f5e3c3460b6fd6eea4a/_data",
                "Destination": "/opt",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
...
[root@srv00 ~]# docker volume inspect 6b650687d6d2a448006dcb2b966b5f69bb31d34cf3e93f5e3c3460b6fd6eea4a
[
    {
        "Name": "6b650687d6d2a448006dcb2b966b5f69bb31d34cf3e93f5e3c3460b6fd6eea4a",
        "Driver": "local",
        "Mountpoint": "/var/lib/docker/volumes/6b650687d6d2a448006dcb2b966b5f69bb31d34cf3e93f5e3c3460b6fd6eea4a/_data",
        "Labels": null
    }
]
```

> 几个容器挂在的数据卷都是相同的.内部其实是创建了一个数据卷.
> 就算删除所有相关的运行的容器..数据卷不会删除.

//END

