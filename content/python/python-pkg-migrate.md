+++
categories = ["python"]
date = "2015-10-22T10:35:44+08:00"
description = ""
tags = ["pip"]
thumbnail = ""
title = "python 依赖包迁移"

+++

python 依赖包迁移(本地安装)

<!--more-->

有时候网络不给力,很多依赖包通过pip下载太慢了,可以通过已有的环境导出再导入新环境.


## 现有环境导出

```
pip freeze > requestments.txt    # 编辑此文件对需要的包进行删减
pip install -d pgk/ -r requestments.txt    # 导出需要的包
```

## 导入新环境

首先将刚才导出的包传到新环境,然后执行

```
pip install *
```

## 总结

关键是`pip install -d`
当然网络良好可以通过`pip install -r requestments.txt`

//END

