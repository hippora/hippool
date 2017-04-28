+++
categories = ["docker"]
date = "2016-06-01T11:48:39+08:00"
description = ""
tags = ["registry","aliyun oss"]
thumbnail = ""
title = "Docker registry 存储到aliyun oss"

+++

Docker registry 存储到aliyun oss

<!--more-->

# Docker registry 存储到aliyun oss

registry有许多配置,通常需要修改是通过-e传入环境变量.

默认registry的数据存储在本地磁盘`/var/lib/registry`

```
[root@iZ23mynm1ezZ ~]# docker exec registry cat /etc/docker/registry/config.yml
version: 0.1
log:
  fields:
    service: registry
storage:
    cache:
        blobdescriptor: inmemory
    filesystem:
        rootdirectory: /var/lib/registry
http:
    addr: :5000
    headers:
        X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
```

为了更好的扩展.比如registry要配置负载均衡.则包括存储数据的地方,ssl证书,redis都是相同的..

这里我们配置oss,
环境变量的名字是按照yml的层级组合成的.比如

```
storage:
  filesystem:
    rootdirectory: /var/lib/registry
```

对应的环境变量名为'REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY`(大写)

```
docker run -d -p 5001:5000 --restart=always --name registry1 \
-v /etc/letsencrypt/archive/registry.xxxx.com/:/certs \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/fullchain1.pem \
-e REGISTRY_HTTP_TLS_KEY=/certs/privkey1.pem \
-v `pwd`/auth:/auth \
-e "REGISTRY_AUTH=htpasswd" \
-e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
-e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
-e REGISTRY_STORAGE=oss \
-e REGISTRY_STORAGE_OSS_ACCESSKEYID=xxxx \
-e REGISTRY_STORAGE_OSS_ACCESSKEYSECRET=xxxxxxx \
-e REGISTRY_STORAGE_OSS_REGION=oss-cn-hangzhou \
-e REGISTRY_STORAGE_OSS_BUCKET=bkt_name \
registry:2
```

测试

```
[root@srv00 ~]# docker login registry.xxxx.com:5001
Username: hippo
Password:
Login Succeeded
[root@srv00 ~]# docker push registry.xxxx.com:5001/ubuntu
The push refers to a repository [registry.xxxx.com:5001/ubuntu]
5f70bf18a086: Pushed
a3b5c80a4eba: Pushed
7f18b442972b: Pushed
3ce512daaf78: Pushed
7aae4540b42d: Pushed
latest: digest: sha256:92c80b28023de63d528c722c295bbe82a20722e3fd7a9b4f14a688bea2cacdac size: 1356
```

> 登录oss可以看到多了个docker文件夹

参数参考:[oss](https://docs.docker.com/registry/storage-drivers/oss/)

完整的registry参数列表参考: [registry config](https://docs.docker.com/registry/configuration/)

> 如果通过环境变量不能满足你的条件,可以创建个`config.yml`文件,通过挂载数据卷文件方式覆盖容器内的配置文件(/etc/docker/registry/config.yml)

//END

