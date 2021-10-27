+++
categories = ["docker"]
date = "2016-05-27T11:41:37+08:00"
description = ""
tags = ["tls"]
thumbnail = ""
title = "docker daemon的HTTP socket TLS加密连接"

+++

默认docker daemon是通过非网络的unix socket监听客户端连接的.如果我们需要客户端通过网络来安全的连接到docker daemon,则因该配置TLS加密方式,通过http的方式来连接.

<!--more-->

## 使用openssl来创建ca证书,并签发密钥.

```
[root@srv00 ~]# openssl genrsa -aes256 -out ca-key.pem 4096
Generating RSA private key, 4096 bit long modulus
.........................................................................................................................................................................++
........................++
e is 65537 (0x10001)
Enter pass phrase for ca-key.pem:
Verifying - Enter pass phrase for ca-key.pem:
[root@srv00 ~]# openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
Enter pass phrase for ca-key.pem:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:Shanghai
Locality Name (eg, city) [Default City]:Shanghai
Organization Name (eg, company) [Default Company Ltd]:docker
Organizational Unit Name (eg, section) []:IT
Common Name (eg, your name or your server's hostname) []:srv00
Email Address []:h@xxx.com
```

> ca证书颁发好.可以申请证书签名请求(CSR)了,注意common name填主机名

服务端证书:

```
[root@srv00 ~]# openssl genrsa -out server-key.pem 4096
Generating RSA private key, 4096 bit long modulus
.........................................++
..................................................................++
e is 65537 (0x10001)
[root@srv00 ~]# openssl req -subj "/CN=srv00" -sha256 -new -key server-key.pem -out server.csr
[root@srv00 ~]# echo subjectAltName = IP:192.168.1.80,IP:127.0.0.1 > extfile.cnf
[root@srv00 ~]# openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem -extfile extfile.cnf
Signature ok
subject=/CN=srv00
Getting CA Private Key
Enter pass phrase for ca-key.pem:
```

客户端证书:

```
[root@srv00 ~]# openssl genrsa -out key.pem 4096
Generating RSA private key, 4096 bit long modulus
............................................................++
..............................................................................................................................................................++
e is 65537 (0x10001)
[root@srv00 ~]# openssl req -subj '/CN=client' -new -key key.pem -out client.csr
[root@srv00 ~]# echo extendedKeyUsage = clientAuth > extfile.cnf
[root@srv00 ~]# openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out cert.pem -extfile extfile.cnf
Signature ok
subject=/CN=client
Getting CA Private Key
Enter pass phrase for ca-key.pem:
```

CSR 没用可以删了

```
[root@srv00 ~]# rm -rfv client.csr server.csr
removed ‘client.csr’
removed ‘server.csr’
```

## 安装证书

```
[root@srv00 ~]# chmod 400 *.pem <==收紧权限
[root@srv00 ~]# mkdir /etc/docker/cert.d
[root@srv00 ~]# cp ca.pem server-key.pem server-cert.pem /etc/docker/cert.d/
[root@srv00 ~]# vi /etc/systemd/system/docker.service.d/daemon.conf
[Service]
ExecStart=
ExecStart=/usr/bin/docker daemon -H fd:// \
--storage-driver=devicemapper --storage-opt=dm.thinpooldev=/dev/mapper/vgdocker-thinpool --storage-opt dm.use_deferred_removal=true \
--tlsverify --tlscacert=/etc/docker/cert.d/ca.pem --tlscert=/etc/docker/cert.d/server-cert.pem --tlskey=/etc/docker/cert.d/server-key.pem \
-H=0.0.0.0:2376
[root@srv00 ~]# systemctl daemon-reload
[root@srv00 ~]# systemctl restart docker
```

## 客户端的连接

```
[root@srv00 ~]# docker --tlsverify --tlscacert=ca.pem --tlscert=cert.pem --tlskey=key.pem -H=192.168.1.80:2376 version
Client:
 Version:      1.11.1
 API version:  1.23
 Go version:   go1.5.4
 Git commit:   5604cbe
 Built:        Wed Apr 27 00:34:42 2016
 OS/Arch:      linux/amd64

Server:
 Version:      1.11.1
 API version:  1.23
 Go version:   go1.5.4
 Git commit:   5604cbe
 Built:        Wed Apr 27 00:34:42 2016
 OS/Arch:      linux/amd64
 ```

 客户端证书移到另一台机器上测试

 ```
 [root@srv00 ~]# scp ca.pem key.pem cert.pem hippo@192.168.1.81:/home/hippo
hippo@192.168.1.81's password:
ca.pem                                                                                                             100% 2069     2.0KB/s   00:00
key.pem                                                                                                            100% 3243     3.2KB/s   00:00
cert.pem                                                                                                           100% 1846     1.8KB/s   00:00
```

ubuntu 机器上配置
```
hippo@ubuntu:~$ mkdir .docker
hippo@ubuntu:~$ mv ca.pem cert.pem key.pem .docker/
hippo@ubuntu:~$ export DOCKER_HOST=tcp://192.168.1.80:2376
hippo@ubuntu:~$ export DOCKER_TLS_VERIFY=1
hippo@ubuntu:~$ docker version
Client:
 Version:      1.10.3
 API version:  1.22
 Go version:   go1.6.1
 Git commit:   20f81dd
 Built:        Wed, 20 Apr 2016 14:19:16 -0700
 OS/Arch:      linux/amd64
An error occurred trying to connect: Get https://192.168.1.80:2376/v1.22/version: dial tcp 192.168.1.80:2376: getsockopt: no route to host
```

  > 通过配置环境变量而不是通过传递参数也可

可能服务端防火墙的问题..我们开放2376端口就好

```
[root@srv00 ~]# firewall-cmd --state
running
[root@srv00 ~]# firewall-cmd --add-port=2376/tcp --permanent
[root@srv00 ~]# firewall-cmd --reload
[root@srv00 ~]# firewall-cmd --list-port
```

再在ubuntu上试一下

```
hippo@ubuntu:~$ docker version
Client:
 Version:      1.10.3
 API version:  1.22
 Go version:   go1.6.1
 Git commit:   20f81dd
 Built:        Wed, 20 Apr 2016 14:19:16 -0700
 OS/Arch:      linux/amd64

Server:
 Version:      1.11.1
 API version:  1.23
 Go version:   go1.5.4
 Git commit:   5604cbe
 Built:        Wed Apr 27 00:34:42 2016
 OS/Arch:      linux/amd64
hippo@ubuntu:~$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
centos              latest              8596123a638e        9 days ago          196.7 MB
ubuntu              latest              c5f1cf30c96b        3 weeks ago         120.7 MB
```

测试成功.

> 如果将客户端证书放在用户的.docker目录下,则`--tlscacert --tlscert --tlskey` 这些参数无需指定.如果是daemon的本机,`-H`参数也无需指定.

//END

