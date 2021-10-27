+++
categories = ["linux"]
date = "2015-09-27T10:31:41+08:00"
description = ""
tags = ["logrotate"]
thumbnail = ""
title = "日志自动切换logrotate"

+++

linux下日志自动切换logrotate

<!--more-->

## 安装

centos下默认已安装了logrotate

```
rpm -ql logrotate-3.7.8-12.el6_0.1.x86_64

/etc/cron.daily/logrotate
/etc/logrotate.conf
/etc/logrotate.d
/usr/sbin/logrotate
/usr/share/doc/logrotate-3.7.8
/usr/share/doc/logrotate-3.7.8/CHANGES
/usr/share/doc/logrotate-3.7.8/COPYING
/usr/share/man/man5/logrotate.conf.5.gz
/usr/share/man/man8/logrotate.8.gz
/var/lib/logrotate.status
```

* /etc/cron.daily/logrotate 此文件是定时脚本
* /etc/logrotate.conf  此文件是全局配置文件
* /etc/logrotate.d 此目录是存放单独定制的配置,我们创建的配置放这里
* /usr/sbin/logrotate 二进制应用程序

## 配置

```
vi /etc/logrotate.d/mynginx

/var/log/xxx.log {
    daily
    rotate 30
    dateext
    dateformat -%Y%m%d%s
    #compress
    #delaycompress
    missingok
    notifempty
    #create 600 www www
    olddir arch
    #minsize 200M
    sharedscripts
    postrotate
        ps -A | grep 'gunicorn: maste'|cut -f1 -d' ' | xargs kill -HUP
    endscript
}
```

说明:

- `daily` 每天切换一次,其他可用还有`weekly`,`monthly`
- `rotate` 在删除最旧的日志前保留几份,这里每天切一次的话保留30天
- `dateext` 日志的后缀,默认是`-%Y%m%d`,如果没有此选项,日志后缀是`.1`,`.2`诸如此类
- `dateformat` 配合`dateext`一起使用,可用的参数`%Y%m%d%s`这4个
- `compress` 使用gzip压缩
- `delaycompress` 配合`compress`一起使用,最新一个切换的日志不压缩
- `missingok` 切换中遇到日志不存在忽略错误
- `notifempty` 空日志文件不切换
- `create` 以哪个用户身份权限创建新日志
- `olddir` 将日志备份到此目录,相对于日志的目录
- `minsize` 超过此大小就触发日志切换,但不会在满足的时间前切换
- `size` 与`minsize`类似,区别是不管时间的限制,一满足大小就切换
- `sharedscripts` 对于`prerotate`,`postrotate` 脚本块而言,比如日志是使用通配符`/var/log/xxx*.log`,命令只会执行一次.默认是切一个日志触发一次.
- `postrotate/endscript` 切换日志后执行的脚本,基本是让进程重新读取配置,因为日志文件句柄切换并新建后,不再关联进程.使用`kill -HUP`较多,使用程序自带的也可,比如`nginx -s reload`.

> 其他参数`man logrotate`

## 测试配置

查看配置有无问题

```
logrotate -d /etc/logrotate.d/mynginx
```

手动切换一次日志

```
logrotate -vf /etc/logrotate.d/mynginx
```

//END

