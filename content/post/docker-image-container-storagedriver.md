+++
categories = ["docker"]
date = "2016-05-25T11:31:46+08:00"
description = ""
tags = ["image","container","storage-drivers"]
thumbnail = ""
title = "docker镜像,容器和存储驱动"

+++

这篇都是概念性的.主要就是官方文档的笔记.更详细的请参考官方文档.

<!--more-->

## 镜像和层
镜像是由一些只读并且描绘文件系统区别的层组成的.它们一层一层堆叠起来,组成了镜像的基础文件系统.这些层都是只读的.
下图是由4层构成的ubuntu镜像:

![image](https://docs.docker.com/engine/userguide/storagedriver/images/image-layers.jpg)

而docker存储驱动的职责就是将这些层堆叠起来.并提供一个统一的视图.让我们觉得跟普通文件系统没什么区别.

当你创建了一个容器时.会在镜像的层上再添加一层叫做"容器层(container layer)",所有在运行中的容器中所做的修改操作,都是影响的这一层. 看下图

![container](https://docs.docker.com/engine/userguide/storagedriver/images/container-layers.jpg)

## 内容可寻址的存储

docker 1.10 开始,介绍了一种全新的寻址镜像和层上数据的存储模型.之前的镜像和层上数据的存放,都是通过随机UUID存放.新模型采用一种安全的内容hash来存放.
新模型特点包括提高了安全性.防止了id冲突,保证了pull,push,load,save操作后的数据一致性.并且更好的提供了层之间的共享问题.

参考图片:

![新模型](https://docs.docker.com/engine/userguide/storagedriver/images/container-layers-cas.jpg)

> 容器层还是random uuid
> 对于新的模型,原先旧模型的镜像数据需要重新计算id来迁移.这些操作会在你运行新版本docker进程后自动执行.
> 镜像较多的话会比较耗资源.如需手工迁移请参考官方文档.

## 容器和层

容器和镜像主要区别就在于容器多了最上层的容器层.所有在容器中的修改操作,都是影响的容器各自的容器层.底下镜像的层都是只读不变的.

下图展示了多个容器共享同一个镜像,但是拥有各自的容器层.互相不受影响.

![sharing-layers](https://docs.docker.com/engine/userguide/storagedriver/images/sharing-layers.jpg)

docker的存储驱动器的作用就是管理这些层.不同的驱动实现方式也不同.主要两个关键点是如何堆叠这些层并提供统一的视图,还有就是数据修改copy-on-write (CoW).

## copy-on-write (CoW)

### 镜像层之间的共享

所有镜像层和容器层,都存储在主机的存储区域,然后通过存储驱动来管理.通常是'/var/lib/docker/'目录

```
[root@srv00 ~]# docker pull ubuntu
Using default tag: latest
latest: Pulling from library/ubuntu

6d28225f8d96: Pull complete
166102ec41af: Pull complete
d09bfba2bd6a: Pull complete
c80dad39a6c0: Pull complete
a3ed95caeb02: Pull complete
Digest: sha256:5718d664299eb1db14d87db7bfa6945b28879a67b74f36da3e34f5914866b71c
Status: Downloaded newer image for ubuntu:latest
```

我们拉取一个ubuntu镜像,可以看到由5层组成,在docker1.10前的版本.镜像层存放目录就是id.1.10之后就不同了.

```
[root@srv00 layerdb]# docker inspect ubuntu
...
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:7aae4540b42d10456f8fdc316317b7e0cf3194ba743d69f82e1e8b10198be63c",
                "sha256:3ce512daaf78307e3a2c5adef7741d9ce9d61449a9a642cafd9f474a50e8c5d0",
                "sha256:7f18b442972bf737eadfff445088375d38f0f455f25ea94277b70050de3ae4b1",
                "sha256:a3b5c80a4ebaf7a0a1b92219b3dfb7109a14b38bebd6b55870a3aec90743a263",
                "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef"
            ]
        }
...
[root@srv00 ~]# ll /var/lib/docker/image/devicemapper/layerdb/sha256
...
```

> '1.10'版本前后尽管管理方式不同.docker 还是会处理好层的共享问题.

我们制作个镜像,看看镜像间的层共享.

```
[root@srv00 ~]# mkdir test-ubuntu
[root@srv00 ~]# cd test-ubuntu/
[root@srv00 test-ubuntu]# cat Dockerfile
FROM ubuntu:latest
RUN echo "Hello world" > /tmp/newfile
[root@srv00 test-ubuntu]# docker build -t test-ubuntu .
Sending build context to Docker daemon 2.048 kB
Step 1 : FROM ubuntu:latest
 ---> c5f1cf30c96b
Step 2 : RUN echo "Hello world" > /tmp/newfile
 ---> Running in 2ae7ab3a167b
 ---> 21c10e6721e0
Removing intermediate container 2ae7ab3a167b
Successfully built 21c10e6721e0
```

> 我们创建了一个新的镜像,镜像ID是`21c10e6721e0`.

```
[root@srv00 test-ubuntu]# docker history 21c10e6721e0
IMAGE               CREATED              CREATED BY                                      SIZE                COMMENT
21c10e6721e0        About a minute ago   /bin/sh -c echo "Hello world" > /tmp/newfile    12 B
c5f1cf30c96b        3 weeks ago          /bin/sh -c #(nop) CMD ["/bin/bash"]             0 B
<missing>           3 weeks ago          /bin/sh -c sed -i 's/^#\s*\(deb.*universe\)$/   1.895 kB
<missing>           3 weeks ago          /bin/sh -c rm -rf /var/lib/apt/lists/*          0 B
<missing>           3 weeks ago          /bin/sh -c set -xe   && echo '#!/bin/sh' > /u   701 B
<missing>           3 weeks ago          /bin/sh -c #(nop) ADD file:ffc85cfdb5e66a5b4f   120.7 MB
[root@srv00 test-ubuntu]# docker inspect test-ubuntu
......
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:7aae4540b42d10456f8fdc316317b7e0cf3194ba743d69f82e1e8b10198be63c",
                "sha256:3ce512daaf78307e3a2c5adef7741d9ce9d61449a9a642cafd9f474a50e8c5d0",
                "sha256:7f18b442972bf737eadfff445088375d38f0f455f25ea94277b70050de3ae4b1",
                "sha256:a3b5c80a4ebaf7a0a1b92219b3dfb7109a14b38bebd6b55870a3aec90743a263",
                "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef",
                "sha256:bf8dd5e3aefb6c212ecfbc1fc72f014d9f93c30ca78546b78079423c4b4e7665"
            ]
        }
...
```

>  `21c10e6721e0`镜像层堆叠在ubuntu镜像之上.`inspect`的结果也是5层的id相同.多了一层.
> 新的模型由于历史数据不再存在每层的配置文件中.而是以文本形式存放在整个镜像级别的配置文件中.所以以上的镜像层`<missing>`是正常的.

官方的图:

![saving-space](https://docs.docker.com/engine/userguide/storagedriver/images/saving-space.jpg)

> 新模型的存储方式极大的节省了空间.

### 容器中的cow

之前也提到了.多个容器如何共享底层的镜像层.但是拥有各自可写的容器层.

当容器中需要修改数据时,就会进行cow操作.但是不同的存储驱动.工作方式会有所不同.

`AUFS`和`OverlayFS`存储驱动的cow操作方式:

- 从最上层依次往下层查找需要修改的数据.
- 一旦找到,就将数据副本copy到容器层.
- 在容器层进行修改操作.

> Btrfs, ZFS, 等工作方式有些区别..centos 默认使用`devicemapper`,使用`docker info`查看.
> 所以说如果对容器有大量的写操作.会极大增加容器体积.官方推荐使用数据卷.

这种操作方式会导致性能方面的影响.各个存储驱动的影响各不相同.
大文件的修改,大量的镜像层.很深的目录树.影响更甚.(所以官方推荐写dockerfile时要缩减指令.多个指令合并).

我们看看运行同一个镜像的多个容器,产生的容器层

```
[root@srv00 ~]# docker run -itd test-ubuntu /bin/bash
76233b0ee89b2a55a449ac4778bf1f1146e77b928ab3474ee030895c247d257d
[root@srv00 ~]# docker run -itd test-ubuntu /bin/bash
2bacfc63c3941d0c817b5f02e55d526344b1a769a3443b35110021c3bcfc9978
[root@srv00 ~]# docker run -itd test-ubuntu /bin/bash
65447ded6aa8fe4b965f30d370d9707301f069be8416f704b6d3de57de5ece58
[root@srv00 ~]# ll /var/lib/docker/containers/
total 20
drwx------ 3 root root 4096 May 25 14:46 2bacfc63c3941d0c817b5f02e55d526344b1a769a3443b35110021c3bcfc9978
drwx------ 3 root root 4096 May 25 14:46 65447ded6aa8fe4b965f30d370d9707301f069be8416f704b6d3de57de5ece58
drwx------ 3 root root 4096 May 25 14:46 76233b0ee89b2a55a449ac4778bf1f1146e77b928ab3474ee030895c247d257d
```

> 容器一旦创建.docker会分配一个容器层.然后分配一个UUID.
> docker通过cow的方式同样也加快了容器的启动速度.只需简单创建一个很小的容器层.

### 数据卷和存储驱动.

当容器被删除,所有容器内的修改都会随着容器一起删除..所以如果需要持久化的数据,要使用数据卷.并且数据卷也可以在多个容器中共享...
数据卷是绕过docker的存储驱动的.

关于数据卷很多内容在上篇blog中已说过.

## 详细参考:
[Understand images, containers, and storage drivers](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/)

//END

