+++
categories = ["postgresql"]
date = "2015-02-25T23:34:24+08:00"
description = ""
tags = ["string"]
thumbnail = ""
title = "postgresql字符串函数与操作符号(二)"

+++

接上一篇字符串函数

<!--more-->

## trim 函数相关 

```
db01=> select trim(leading 'x' from 'xxTxTxx'),trim(trailing 'x' from 'xxTxTxx'),trim(both 'x' from 'xxTxTxx'),trim('x' from 'xxTxTxx');
 TxTxx | xxTxT | TxT   | TxT
```

单独的函数

```
db01=> select ltrim('xxTxTxx','x'),rtrim('xxTxTxx','x'),btrim('xxTxTxx','x');
 TxTxx | xxTxT | TxT
```

这些trim函数默认是去空格

```
db01=> select trim('  T T  '),char_length(trim('  T T  '));
 T T   |           3
```

## ascii 码 

```
db01=> select ascii('aa'),ascii('a'),chr(97);
    97 |    97 | a
```

> ascii只会转第一个字符
> 还有个函数to_ascii从一种编码转到另一个编码的ascii,感觉我们很少用到

## 字符串填充 

```
db01=> select lpad('a',5),length(lpad('a',5)),lpad('a',5,'1'),rpad('a',5,'1');
     a |      5 | 1111a | a1111
```

> 默认用空格填充

## 字符串重复,字符串反转 

```
db01=> select repeat('ab',5),reverse('abcdef');
 ababababab | fedcba
```

## MD5加密 

```
db01=> select md5('abc');
 900150983cd24fb0d6963f7d28e17f72
```

## 查看客户端编码 

```
db01=> select pg_client_encoding();
 UTF8
```

## 10进制转16进制 

```
db01=> select to_hex(4294967295);
 ffffffff
```

## 字符串引用转义相关 

函数quote_ident

```
db01=> select quote_ident('abc'),quote_ident('Abc'),quote_ident('rate ''(%)');
 abc         | "Abc"       | "rate '(%)"
```

> 帮助我们生成能被sql使用的标识,比如字段别名等

函数quote_literal

```
db01=> select quote_literal('Abc'),quote_literal(E'rate \'(%)'),quote_literal(null);
 'Abc'         | 'rate ''(%)'  | 
```

> 帮助我们生成可以被sql接受的字符串

函数quote_nullable

```
db01=> select quote_nullable('Abc'),quote_nullable(E'rate \'(%)'),quote_nullable(null);
 'Abc'          | 'rate ''(%)'   | NULL
```

> 跟quote_literal区别是quote_nullable接受null值会返回字符串NULL

## 正则表达式 

regexp_matches函数

```
db01=> select regexp_matches('abc1111bcd2222eee3333','\d+');
 {1111}

db01=> select regexp_matches('abc1111bcd2222eee3333','\d+','g');
 {1111}
 {2222}
 {3333}

db01=> select regexp_matches('abc1111bcd2222eee3333','B','i');    --忽略大小写
 {b}
```

> 从字符串中查找符合pattern的字符串,返回数组表记录

regexp_replace函数

```
db01=> select regexp_replace('abc1111bcd2222eee3333','\d+','X');
 abcXbcd2222eee3333

db01=> select regexp_replace('abc1111bcd2222eee3333','\d+','X','g');
 abcXbcdXeeeX
```

regexp_split_to_array函数

```
db01=> select E'aaa bbb\n ccc' str, regexp_split_to_array(E'aaa bbb\n ccc','\s+');
   str   | regexp_split_to_array 
---------+-----------------------
 aaa bbb+| {aaa,bbb,ccc}
  ccc    | 
(1 row)
```

regexp_split_to_table 函数

```
db01=> select E'aaa bbb\n ccc' str, regexp_split_to_table(E'aaa bbb\n ccc','\s+');
   str   | regexp_split_to_table 
---------+-----------------------
 aaa bbb+| aaa
  ccc    | 
 aaa bbb+| bbb
  ccc    | 
 aaa bbb+| ccc
  ccc    | 
(3 rows)
```

> 正则细节比较复杂,可以参考[官方文档][1] 


  [1]: http://www.postgresql.org/docs/9.4/static/functions-matching.html

//END

