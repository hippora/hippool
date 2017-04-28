+++
categories = ["linux"]
date = "2016-05-30T11:44:26+08:00"
description = ""
tags = ["letsencrypt"]
thumbnail = ""
title = "使用letsencrypt"

+++

使用letsencrypt

<!--more-->

安装

```
yum install certbot
```

生成证书

```
# certbot certonly -n --standalone  --pre-hook "systemctl stop nginx" --post-hook "systemctl start nginx" -d www.idyi.net --email xxx@xxx.com --agree-tos
```

nginx 配置

```
server {
    listen      80;
    server_name www.idyi.net idyi.net ;
    rewrite ^/(.*) https://www.idyi.net/$1 permanent;
}

server {
    listen       443 ;
    server_name  www.idyi.net;
    ssl on;
    ssl_certificate /etc/letsencrypt/live/www.idyi.net/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/www.idyi.net/privkey.pem;
...
```

> 80,443端口开放防火墙

加入crontab

```
crontab -e
0 5 1 * * certbot renew --quiet --pre-hook "systemctl stop nginx" --post-hook "systemctl start nginx"
```
//END

