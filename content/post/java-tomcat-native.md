+++
categories = ["java"]
date = "2016-01-21T11:09:14+08:00"
description = ""
tags = ["tomcat","native"]
thumbnail = ""
title = "安装tomcat-native"

+++

 安装tomcat-native过程曲折,故记一下.

<!--more-->

# centos 下tomcat-native 安装

> root用户下

## 下载相关软件
```bash
wget http://apache.opencas.org//apr/apr-1.5.2.tar.gz
wget http://apache.opencas.org//apr/apr-util-1.5.4.tar.gz
wget http://apache.opencas.org//apr/apr-iconv-1.2.1.tar.gz
wget https://www.openssl.org/source/openssl-1.0.2e.tar.gz
wget http://mirrors.cnnic.cn/apache/tomcat/tomcat-connectors/native/1.2.4/source/tomcat-native-1.2.4-src.tar.gz
```

## 安装apr
```bash
tar -xzf apr-1.5.2.tar.gz && cd apr-1.5.2
./configure --prefix=/usr/local/apr
make && make install
...
rm: cannot remove `libtoolT': No such file or directory
...
```

> 30206行注释#    $RM "$cfgfile"

## 安装apr-iconv

```bash
tar -xzf apr-iconv-1.2.1.tar.gz && cd apr-iconv-1.2.1
./configure --prefix=/usr/local/apr-iconv --with-apr=/usr/local/apr
make && make install
```

## 安装apr-util
```bash
tar -xzf apr-util-1.5.4.tar.gz && cd apr-util-1.5.4
./configure --prefix=/usr/local/apr-util  --with-apr=/usr/local/apr  --with-apr-iconv=/usr/local/apr-iconv/bin/apriconv
make && make install
```

## 安装openssl1.0.2

> 如系统已经是1.0.2就没必要装

```bash
tar -xzf openssl-1.0.2e.tar.gz && cd openssl-1.0.2e
./config --prefix=/usr/local/openssl -fPIC
make && make install
```

> -fPIC参数不加,后面编译tomcat-native会报错,见下

## 安装tomcat-native

```bash
tar -xzf tomcat-native-1.2.4-src.tar.gz && cd tomcat-native-1.2.4-src/native
./configure --with-apr=/usr/local/apr --with-java-home=/etc/alternatives/java_sdk --with-ssl=/usr/local/openssl
make && make install
...

checking OpenSSL library version >= 1.0.2...
Found   OPENSSL_VERSION_NUMBER 0x1000105f (OpenSSL 1.0.1e 11 Feb 2013)
Require OPENSSL_VERSION_NUMBER 0x1000200f or greater (1.0.2)
not compatible
...
```

> centos 上用yum暂时只能更新到1.0.1e.需要手动安装

``` bash
...
/usr/local/openssl/lib/libssl.a: could not read symbols: Bad value
...
```

> 编译openssl时加上参数 `-fPIC`

## 配置

- *全局*

``` bash
vi /etc/profile.d/my.sh
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/apr/lib
```
> my.sh 自己取的名字,建议不直接修改`/etc/profile`

- *针对tomcat*

```bash
vi $CATALINA_HOME/bin/setenv.sh
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/apr/lib

```

> `setenv.sh`没有的话自己创建,不要写在`catalina.sh`中.


## 验证

```bash
tail -f catalina.out
......
21-Jan-2016 11:42:24.442 INFO [main] org.apache.catalina.core.AprLifecycleListener.lifecycleEvent Loaded APR based Apache Tomcat Native library 1.2.4 using APR version 1.5.2.
21-Jan-2016 11:42:24.443 INFO [main] org.apache.catalina.core.AprLifecycleListener.lifecycleEvent APR capabilities: IPv6 [true], sendfile [true], accept filters [false], random [true].
21-Jan-2016 11:42:24.446 INFO [main] org.apache.catalina.core.AprLifecycleListener.initializeSSL OpenSSL successfully initialized (OpenSSL 1.0.2e 3 Dec 2015)
21-Jan-2016 11:42:24.520 INFO [main] org.apache.coyote.AbstractProtocol.init Initializing ProtocolHandler ["http-apr-8080"]
......
```

//END

