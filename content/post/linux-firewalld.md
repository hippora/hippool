+++
categories = ["linux"]
date = "2015-08-02T10:24:57+08:00"
description = ""
tags = ["firewall"]
thumbnail = ""
title = "firewalld屏蔽ip"

+++

firewalld屏蔽ip

<!--more-->

## firewalld-cmd

```bash
# firewall-cmd --permanent --add-rich-rule="rule family=ipv4 source address=43.229.53.61 reject"
```

> ip支持CIDR,reject=drop

```bash
# firewall-cmd --list-rich-rules
rule family="ipv4" source address="43.229.53.61" reject
```

//END

