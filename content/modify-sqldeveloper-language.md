+++
categories = ["oracle"]
date = "2014-04-14T14:20:17+08:00"
description = ""
tags = ["sqldeveloper"]
thumbnail = "img/oracle-logo.png"
title = "修改sqldeveloper的语言"

+++

修改sqldeveloper的界面的语言

<!--more-->

- 环境:MACOSX

/Applications/SQLDeveloper.app/Contents/Resources/sqldeveloper/ide/bin/ide.conf 找到一段话:

```bash
# Relative paths are resolved against the parent directory of this file.
#
# The format of this file is:
#
#    "Directive      Value" (with one or more spaces and/or tab characters
#    between the directive and the value)  This file can be in either UNIX
#    or DOS format for end of line terminators.  Any path seperators must be
#    UNIX style forward slashes '/', even on Windows.
#
# This configuration file is not intended to be modified by the user.  Doing so
# may cause the product to become unstable or unusable.  If options need to be
# modified or added, the user may do so by modifying the custom configuration files
# located in the user's home directory.  The location of these files is dependent
# on the product name and host platform, but may be found according to the
# following guidelines:
#
# Windows Platforms:
#   The location of user/product files are often configured during installation,
#   but may be found in:
#     %APPDATA%\<product-name>\<product-version>\product.conf
#     %APPDATA%\<product-name>\<product-version>\jdev.conf
#
# Unix/Linux/Mac/Solaris:
#   $HOME/.<product-name>/<product-version>/product.conf
#   $HOME/.<product-name>/<product-version>/jdev.conf
#
# In particular, the directives to set the initial and maximum Java memory
# and the SetJavaHome directive to specify the JDK location can be overridden
# in that file instead of modifying this file.
#
```

so....

找到路径追加进此文件后重启搞定~

```bash
vi ~/.sqldeveloper/4.0.0/product.conf
AddVMOption -Duser.language=en
AddVMOption -Duser.country=US
```

//END
