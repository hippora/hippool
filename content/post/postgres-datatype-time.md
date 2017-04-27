+++
categories = ["postgresql"]
date = "2015-02-28T00:48:16+08:00"
description = ""
tags = ["datatype","date","time","timestamp"]
thumbnail = ""
title = "postgresql数据类型之时间类型"

+++

包括date,time,timestamp,interval

<!--more-->

## date 

```
postgres=# select date'2015-2-28',date'2015-2-28 12:22:30';
    date    |    date
------------+------------
 2015-02-28 | 2015-02-28
(1 row)
```

> 日期只有年月日

## time 

```
postgres=# select time '000001.111',time'24:00:00',time '2013-11-11 11:23:44';
     time     |   time   |   time
--------------+----------+----------
 00:00:01.111 | 24:00:00 | 11:23:44
(1 row)
```

> 时间有时分秒毫秒,可以出现24:00:00.

## timestamp 

```
postgres=# select timestamp '2015-2-28 12:11:22.123411',timestamp with time zone '2015-2-28 24:00:00';
         timestamp          |      timestamptz
----------------------------+------------------------
 2015-02-28 12:11:22.123411 | 2015-03-01 00:00:00+08
(1 row)
```

> timestamp中的24点变成第二天的0点

## 特殊关键字 

epoch

```
postgres=# select date 'epoch',timestamp 'epoch';
 1970-01-01 | 1970-01-01 00:00:00
```

> 特定时间戳

infinity

```
postgres=# select date 'infinity',timestamp '-infinity';
 infinity | -infinity
```

> 正负无穷大

now

```
postgres=# select date 'now',timestamp 'now',time 'now';
 2015-02-28 | 2015-02-28 23:16:27.991058 | 23:16:27.991058
```

tomorrow,today,yesterday

```
postgres=# select date'tomorrow',date 'today',date'yesterday';
 2015-03-01 | 2015-02-28 | 2015-02-27

postgres=# select timestamp'tomorrow',timestamp'today',timestamp'yesterday';
 2015-03-01 00:00:00 | 2015-02-28 00:00:00 | 2015-02-27 00:00:00
```

> 昨天今天明天,时分秒都是0

但是你也可以这样写:

```
postgres=# select timestamp 'yesterday 11:00:00',timestamp 'today 12:';
 2015-02-14 11:00:00 | 2015-02-15 12:00:00
```

allballs

```
postgres=# select time'allballs';
 00:00:00
```

> 只适用time类型

## interval 

```
postgres=# select interval '1-2 3 4:5:6.777777',interval '1 year 3 hour 87 second';
 1 year 2 mons 3 days 04:05:06.777777 | 1 year 03:01:27

postgres=# select interval 'P1Y2M3DT4H5M6S',interval 'P1-2-3T4:5:6.7';
 1 year 2 mons 3 days 04:05:06 | 1 year 2 mons 3 days 04:05:06.7
```

## 影响时间输出的格式 

datestyle

时间相关的显示格式

```
postgres=# show datestyle;
 ISO, MDY
postgres=# select now();
 2015-02-28 23:42:39.748022+08

postgres=# set datestyle='postgres,dmy';
SET
postgres=# select now();
 Sat 28 Feb 23:42:12.606023 2015 CST

postgres=# reset datestyle;
RESET
```


intervalstyle

interval显示的格式

```
postgres=# show intervalstyle;
 postgres

postgres=# select interval '1-2 3 4:5:6.777777';
 1 year 2 mons 3 days 04:05:06.777777

postgres=# set intervalstyle=sql_standard;
SET
postgres=# select interval '1-2 3 4:5:6.777777';
 +1-2 +3 +4:05:06.777777

postgres=# set intervalstyle=postgres_verbose;
SET
postgres=# select interval '1-2 3 4:5:6.777777';
 @ 1 year 2 mons 3 days 4 hours 5 mins 6.777777 secs

postgres=# set intervalstyle=iso_8601;
SET
postgres=# select interval '1-2 3 4:5:6.777777';
 P1Y2M3DT4H5M6.777777S

postgres=# reset intervalstyle;
RESET
```

> 跟输入格式相对应

//END
