+++
categories = ["postgresql"]
date = "2015-02-25T23:18:30+08:00"
description = ""
tags = ["string"]
thumbnail = ""
title = "postgresql字符串函数与操作符号(一)"

+++

常用的字符串函数与操作符号

<!--more-->

## 字符串拼接

使用 || 拼接

```
db01=> select 'sss' || 'aaa','sss' || 123;
 sssaaa   | sss123

db01=> select 111 || 222;
ERROR:  operator does not exist: integer || integer
LINE 1: select 111 || 222;
                   ^
HINT:  No operator matches the given name and argument type(s). You might need to add explicit type casts.
db01=> select 111 || 222::text;
 111222
```

> 都是数字,至少要有一个显式转成字符串.

使用函数拼接

```
db01=> select concat('aa','bb'),concat('aa','bb',11),concat(11,22);
 aabb   | aabb11 | 1122
```

> 函数对于处理int类型当做字符串处理

使用分隔符拼接
```
db01=> select concat_ws('$','aa','bb','cc');
 aa$bb$cc
```

### 长度相关 ###

```
[postgres@fnddb ~]$ export LANG=zh_CN.GBK
[postgres@fnddb ~]$ psql db01 hippo
psql (9.4.1)
Type "help" for help.

db01=> show client_encoding;
 GBK

db01=> show server_encoding;
 UTF8
```

length函数

```
postgres=# select length('abc'),length('河马'),length('河马','UTF8'),length('河马','GBK');
      3 |      2 |      2 |      3
```

> length计算字符长度

length变种

```
db01=> select bit_length('abc'),char_length('abc'),octet_length('abc');
         24 |           3 |            3

db01=> select bit_length('河马'),char_length('河马'),octet_length('河马');
         48 |           2 |            6
```

> octet_length 是字节长度
> 这里的字符长度是2,使用的服务器编码UTF8

## 大小写转换

```
db01=> select lower('aBc'),upper('aBc'),initcap('i am HIPPO');
 abc   | ABC   | I Am Hippo
```

> initcap 首字母大写

## 字符串定位与替换

字符串定位

```
db01=> select position('x' in 'bbbxcccxddd'),strpos('bbbxcccxddd','x');
    4 |      4
```

> 两种函数,参数方向相反
> pg中很多函数使用关键字而不是逗号来区分参数

字符串替换overlay

```
db01=> select overlay('aaa1111bbb' placing 'XXXX' from 4 for 3);
 aaaXXXX1bbb
```

字符串替换replace

```
db01=> select replace('abacadaeaf','a','|');
 |b|c|d|e|f
```

字符串替换translate

```
db01=> select translate('abcdef','adc','123'),translate('abcdef','adc','1'),translate('abcdef','ad','1234');
 1b32ef    | 1bef      | 1bc2ef
```

> 被替换字符与替换字符一一对应,如果替换字符少了,用null代替.如果替换字符多了,则没用

### 字符串截取

还有个substr函数,使用逗号区分参数,不支持正则

```
db01=> select substr('aaa1111bbb',3,2);
 a1
```

变种substr

```
db01=> select substring('aaa1111bbb' from 4 for 4),substring('aaa1111bbb' from '\d+'),substring('aaa1111bbb' from '\db{2}');
 1111      | 1111      | 1bb
```

> 支持POSIX及SQL的正则表达式

left和right

```
db01=> select left('abcdef',2),left('abcdef',-2);
 ab   | abcd

db01=> select right('abcdef',2),right('abcdef',-2);
 ef    | cdef
```

split_part函数

```
db01=> select split_part('abc:bbc:ccc:ddd',':',3);
 ccc
```

> 指定分隔符,获取某一段,感觉截取linux下密码文件很好用

未完待续......

//END


