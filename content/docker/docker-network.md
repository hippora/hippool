+++
categories = ["docker"]
date = "2016-05-20T11:25:10+08:00"
description = ""
tags = ["network"]
thumbnail = ""
title = "docker的网络"

+++

docker的网络

<!--more-->

## docker network

docker有3个默认网络

```
[root@srv00 ~]# docker network ls
NETWORK ID          NAME                DRIVER
b891bd36a3f1        bridge              bridge
739cb6a018f7        host                host
d0e062995920        none                null
```

默认容器都是运行在 `bridge` 网络下

运行个容器测试下

```
[root@srv00 ~]# docker run -itd --name nw centos
5ffb7062805af03b00ecc6160a78c855a7d1b7c32bfcc14d78ce036f2df9a517
[root@srv00 ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS                   NAMES
5ffb7062805a        centos              "/bin/bash"              About a minute ago   Up About a minute                           nw
```

> `--name` 给容器命名,否则docker自动命名.

查看bridge网络下的容器有哪些:

```
[root@srv00 ~]# docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "b891bd36a3f12dc2bb1f4526c0fc2741c0f8ccf6011071116840b6b31f9b20ac",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.42.1"
                }
            ]
        },
        "Internal": false,
        "Containers": {
            "5ffb7062805af03b00ecc6160a78c855a7d1b7c32bfcc14d78ce036f2df9a517": {
                "Name": "nw",
                "EndpointID": "65f410576be2d9aadb697ebb820e372ffb2073db7868ec001e87da5c8cf40d71",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            },
            "616d942216ee7050ecff07ca66d2d7583d4093820e9d1448f96b417d7d779408": {
                "Name": "high_shockley",
                "EndpointID": "de09ec8b400b2724238d78ece5ff24201b7baffcdb7279809ea6b1beae5fec1d",
                "MacAddress": "02:42:ac:11:00:06",
                "IPv4Address": "172.17.0.6/16",
                "IPv6Address": ""
            },
            "dead20777b6c1609ab968966b3589904d44f8a12c124c178fd5cb540052cce6f": {
                "Name": "gloomy_cray",
                "EndpointID": "1ba62447dd2d89165fc7079442a2016fb1c44caa02cca7cda6bf363eec2841da",
                "MacAddress": "02:42:ac:11:00:07",
                "IPv4Address": "172.17.0.7/16",
                "IPv6Address": ""
            },
            "e709f66c9bdbcc20f92b804e483bb566bc4ee934fa1e2e5cdbf4759f7ac1923c": {
                "Name": "tiny_shockley",
                "EndpointID": "a070387813fc17fd54ab389b4afe40c80c5dbbe42c86983a89a84a65cf32e378",
                "MacAddress": "02:42:ac:11:00:01",
                "IPv4Address": "172.17.0.1/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```

> 可以看到刚运行的容器nw的ip是`172.17.0.2`

我们可以将某个容器从网络中移除

```
[root@srv00 ~]# docker exec -it high_shockley /bin/bash
[root@616d942216ee /]# ping 172.17.0.2
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.100 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.065 ms
^C
--- 172.17.0.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.065/0.082/0.100/0.019 ms
[root@616d942216ee /]# exit
exit
```

> 先进入同网络的另一个容器ping,可以ping通.

```
[root@srv00 ~]# docker network disconnect bridge nw
[root@srv00 ~]# docker exec -it high_shockley /bin/bash
[root@616d942216ee /]# ping 172.17.0.2
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
^C
--- 172.17.0.2 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 1999ms

[root@616d942216ee /]# exit
exit
[root@srv00 ~]# docker network inspect bridge
...
```

> 可以看到ping不通.也不在bridge网络里了.通过`docker inspect nw` 已经没有ip地址了.

## 创建自己的 bridge network

docker 支持 两个网络驱动类型'bridge'和`overlay`.
'bridge' 是运行在统一主机上的网络.`overlay`可以包括多个主机.

这里我们先创建自己的'bridge'网络

```
[root@srv00 ~]# docker network create -d bridge mynetwork
072dbe030461bc1f7e1bc4829017f13ade899b09b606d12dfb8b6dec496a08bf
[root@srv00 ~]# docker network ls
NETWORK ID          NAME                DRIVER
b891bd36a3f1        bridge              bridge
739cb6a018f7        host                host
072dbe030461        mynetwork           bridge
d0e062995920        none                null
[root@srv00 ~]# docker network inspect mynetwork
[
    {
        "Name": "mynetwork",
        "Id": "072dbe030461bc1f7e1bc4829017f13ade899b09b606d12dfb8b6dec496a08bf",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1/16"
                }
            ]
        },
        "Internal": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```

> `mynetwork`下一个容器都没有

运行个容器在`mynetwork`下

```
[root@srv00 ~]# docker run -itd --net mynetwork --name nw1 centos
c5b59a1e180753301460b597b87b1608b00ebe2b8620b41b8851938c6eab8044
[root@srv00 ~]# docker network inspect mynetwork
[
    {
        "Name": "mynetwork",
        "Id": "072dbe030461bc1f7e1bc4829017f13ade899b09b606d12dfb8b6dec496a08bf",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1/16"
                }
            ]
        },
        "Internal": false,
        "Containers": {
            "c5b59a1e180753301460b597b87b1608b00ebe2b8620b41b8851938c6eab8044": {
                "Name": "nw1",
                "EndpointID": "ed444519fd958ddd83d64f62719f485f62d284f2a647536e4f48a8128e62ad5e",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

测试下互通性
先把nw连到bridge网络

```
[root@srv00 ~]# docker network connect bridge nw
[root@srv00 ~]# docker exec -it nw /bin/bash
[root@5ffb7062805a /]# ping 172.18.0.2
PING 172.18.0.2 (172.18.0.2) 56(84) bytes of data.
^C
--- 172.18.0.2 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 1999ms
```

> 不同网络中的容器不通.

将nw也加入到mynetwork

```
[root@srv00 ~]# docker network connect mynetwork nw
[root@srv00 ~]# docker network inspect mynetwork
[root@srv00 ~]# docker network inspect bridge
```

> 可以看到2个网络中都有nw

```
[root@srv00 ~]# docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' nw
172.17.0.2172.18.0.3
[root@srv00 ~]# docker exec -it nw /bin/bash
[root@5ffb7062805a /]# ping 172.18.0.2 -c3
PING 172.18.0.2 (172.18.0.2) 56(84) bytes of data.
64 bytes from 172.18.0.2: icmp_seq=1 ttl=64 time=0.093 ms
64 bytes from 172.18.0.2: icmp_seq=2 ttl=64 time=0.057 ms
64 bytes from 172.18.0.2: icmp_seq=3 ttl=64 time=0.076 ms

--- 172.18.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2000ms
rtt min/avg/max/mdev = 0.057/0.075/0.093/0.016 ms
[root@5ffb7062805a /]# ping 172.17.0.6 -c3
PING 172.17.0.6 (172.17.0.6) 56(84) bytes of data.
64 bytes from 172.17.0.6: icmp_seq=1 ttl=64 time=0.102 ms
64 bytes from 172.17.0.6: icmp_seq=2 ttl=64 time=0.080 ms
^C
--- 172.17.0.6 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.080/0.091/0.102/0.011 ms
```

> nw 这台容器有两个ip,同时在两个网段中,两个网络中的容器都可以互通.

//END

