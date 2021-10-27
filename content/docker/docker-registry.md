+++
categories = ["docker"]
date = "2016-06-01T11:47:07+08:00"
description = ""
tags = ["registry"]
thumbnail = ""
title = "Docker registry"

+++

Docker registry

<!--more-->

## 本地运行一个registry

```
hippo@ubuntu:~$ docker run -d -p 5000:5000 --restart=always --name registry registry:2
hippo@ubuntu:~$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
f53a7e7353ec        registry:2          "/bin/registry serve "   33 seconds ago      Up 30 seconds       0.0.0.0:5000->5000/tcp   registry
```

给镜像打标签,并push到registry

```
hippo@ubuntu:~$ docker pull ubuntu
hippo@ubuntu:~$ docker tag ubuntu localhost:5000/ubuntu
hippo@ubuntu:~$ docker images
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
localhost:5000/ubuntu   latest              2fa927b5cdd3        2 days ago          122 MB
ubuntu                  latest              2fa927b5cdd3        2 days ago          122 MB
registry                2                   34bccec54793        11 days ago         171.2 MB
hippo@ubuntu:~$ docker push localhost:5000/ubuntu
```

## 存储

默认registry的镜像数据存放在匿名数据卷里,可以通过docker inspect查看

我们可以通过-v来指定存放路径

```
docker run -d -p 5000:5000 --restart=always --name registry -v `pwd`/data:/var/lib/registry registry:2
```

## 通过域名访问registry

如果希望通过将registry放在互联网上访问,则需要配置ssl,并且为了安全,最好配置访问认证.

> 可以通过letsencrypt来获取个证书.

运行:

```
# docker run -d -p 5000:5000 --restart=always --name registry \
-v /etc/letsencrypt/archive/registry.xxxx.com/:/certs \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/fullchain1.pem \
-e REGISTRY_HTTP_TLS_KEY=/certs/privkey1.pem \
registry:2
```

其他主机上执行push操作:

```
[root@srv00 ~]# docker tag ubuntu registry.xxxx.com:5000/ubuntu
[root@srv00 ~]# docker push registry.xxxx.com:5000/ubuntu
...
```

> push成功则验证完成

## 添加访问认证

创建密码文件

```
[root@iZ23mynm1ezZ ~]# mkdir auth
[root@iZ23mynm1ezZ ~]# docker run --rm --entrypoint htpasswd registry:2 -Bbn hippo password > auth/htpasswd
```

运行registry要多加些参数,(删除原先的容器registry,注意备份镜像)

```
[root@iZ23mynm1ezZ ~]# docker stop registry
[root@iZ23mynm1ezZ ~]# docker rm -v registry	<===!!备份!!
[root@iZ23mynm1ezZ ~]# docker run -d -p 5000:5000 --restart=always --name registry \
-v /etc/letsencrypt/archive/registry.xxxx.com/:/certs \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/fullchain1.pem \
-e REGISTRY_HTTP_TLS_KEY=/certs/privkey1.pem \
-v `pwd`/auth:/auth \
-e "REGISTRY_AUTH=htpasswd" \
-e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
-e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
registry:2
```

到另一台机器上验证下

```
[root@srv00 ~]# docker push registry.xxxx.com:5000/ubuntu
The push refers to a repository [registry.xxxx.com:5000/ubuntu]
5f70bf18a086: Image push failed
a3b5c80a4eba: Image push failed
7f18b442972b: Image push failed
3ce512daaf78: Image push failed
7aae4540b42d: Image push failed
no basic auth credentials
```
> 直接push失败,需要先登录

```
[root@srv00 ~]# docker login registry.xxxx.com:5000
Username: hippo
Password:
Login Succeeded[root@srv00 ~]# docker push registry.xxxx.com:5000/ubuntu
The push refers to a repository [registry.xxxx.com:5000/ubuntu]
5f70bf18a086: Pushed
a3b5c80a4eba: Pushed
7f18b442972b: Pushed
3ce512daaf78: Pushed
7aae4540b42d: Pushed
latest: digest: sha256:92c80b28023de63d528c722c295bbe82a20722e3fd7a9b4f14a688bea2cacdac size: 1356
```

> 也可以通过前端通过nginx,apache等当反向代理来实现ssl及访问认证.

通过Compose管理
docker启动参数多了之后,每次运行都比较复杂麻烦..可以通过docker compose来管理.
具体不细说了.

//END

