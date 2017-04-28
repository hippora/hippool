+++
categories = ["python"]
date = "2015-09-28T10:33:31+08:00"
description = ""
tags = ["virtualenvwrapper"]
thumbnail = ""
title = "安装虚拟环境virtualenvwrapper"

+++

安装python虚拟环境virtualenvwrapper

<!--more-->

## 单独下载python

### 安装编译需要的包

```
yum -y groupinstall "Development Tools"
yum -y install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel
useradd user01  #添加一个跑应用用户
```

### 编译及配置


```
su - user01
curl -O --progress https://www.python.org/ftp/python/2.7.10/Python-2.7.10.tar.xz
cd Python-2.7.10
mkdir ~/python
./configure --prefix=$HOME/python
make && make install
```

添加环境变量

```
vi ~/.bash_profile

PATH=$HOME/python/bin:$PATH; export PATH
```

测试

```
python -V
```

## 安装pip

安装 pip,会自动安装setuptools

```
curl https://bootstrap.pypa.io/get-pip.py -o - | python
```

> 官网说python 2.7.9 之后都包含了pip了,为啥偶没有.

## 安装虚拟环境virtualenvwrapper

```
pip install virtualenvwrapper
echo 'export WORKON_HOME=$HOME/.virtualenvs' >> ~/.bash_profile
echo '. ~/python/bin/virtualenvwrapper.sh' >> ~/.bash_profile
. ~/.bash_profile
```

## 创建虚拟环境

```
mkvirtualenv env1
mkvirtualenv env2
```

查看有哪些环境

```
workon
```

切换到某个环境

```
workon env1

```

//END


