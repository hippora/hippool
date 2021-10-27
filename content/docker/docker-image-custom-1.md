+++
categories = ["docker"]
date = "2016-05-20T11:20:25+08:00"
description = ""
tags = ["image"]
thumbnail = ""
title = "制作自己的docker镜像(一)"

+++

手动制作docker镜像

<!--more-->

## 制作自己的镜像(一)

### 对容器进行修改进行制作

我们运行个容器并进入,并进行定制化(安装修改)

```
[root@srv00 ~]# docker run -it centos
[root@8a15586148ee /]# cat << EOF > /etc/yum.repos.d/nginx.repo
> [nginx]
> name=nginx repo
> baseurl=http://nginx.org/packages/centos/7/\$basearch/
> gpgcheck=0
> enabled=1
> EOF
[root@8a15586148ee /]# yum install -y nginx
[root@8a15586148ee /]# exit
```

对容器的修改进行保存.有点像git commit

```
[root@srv00 ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                          PORTS                     NAMES
8a15586148ee        centos              "/bin/bash"         21 minutes ago      Exited (0) About a minute ago                             naughty_dubinsky
[root@srv00 ~]# docker commit -m "install nginx" -a "hippo" 8a15586148ee hippo/nginx:v1.10
sha256:1e20546f8434d72cb350ae2081fc77690bdc599dbd984b5cdec09a7868013042
```

> `-m` 注释,`-a` 作者,`8a15586148ee`容器id,`hippo/nginx`镜像名,`v1.10`是tag名

查看下创建的镜像

```
[root@srv00 ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
hippo/nginx         v1.10               1e20546f8434        About a minute ago   281.1 MB
centos              latest              8596123a638e        2 days ago           196.7 MB
```

运行一下试试

```
[root@srv00 ~]# docker run -d -p 80 hippo/nginx:v1.10 nginx -g "daemon off;"
616d942216ee7050ecff07ca66d2d7583d4093820e9d1448f96b417d7d779408
[root@srv00 ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS                          PORTS                     NAMES
616d942216ee        hippo/nginx:v1.10   "nginx -g 'daemon off"   4 seconds ago        Up 2 seconds                    0.0.0.0:32770->80/tcp     high_shockley
[root@srv00 ~]# curl -L localhost:32770
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
......
```

> `-d` 后台运行,`-p`让容器端口映射到主机上.通过`docker ps`看到是映射到`32770`上.
> `nginx -g "daemon off;" `容器中运行的命令需要在前台运行.否则启动容器后,会马上退出.

未完待续...

//END

