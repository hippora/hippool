+++
categories = ["postgresql"]
date = "2015-02-13T17:17:50+08:00"
description = ""
tags = ["backup","restore"]
thumbnail = ""
title = "postgresql在线备份与恢复(四)"

+++

其他需要注意的地方

<!--more-->

## 完全的物理备份

这种备份相当于冷备份,需要将数据库干净的关闭,然后将相关的物理文件备份起来.  
因为需要关闭数据库,使用场景比较少,比如需要将数据库备份出来搭一个演示环境使用.  

## 基于Timeline的备份简要说明

                            /3---------------  
    ______$_________________|______1  
                 |2________  

最初的数据库是Timeline1 ..  
假如我在$处做了基础备份及相关归档文件.  
由于某些原因,比如误操作弄乱了某些业务数据,需要还原到时间点2处,则生成了Timeline:2  
正常运作一段时间后发现更重要的某些资料没有正确还原,确认恢复到Timeline:1的位置3处一则可以最大避免错误的数据,二则可以恢复这些资料.  
因为在$处备份的的归档只备份到位置2跟3之间,2-3处的归档部分未备份,还在归档目录中.  
此时如果使用$基础备份恢复的话,默认只会使用Timeline:2的归档来恢复,不符合条件.  
如需要使用Timeline:1的归档来恢复的话,recovery.conf文件中需要额外设置一个参数recovery_target_timeline='1'  
恢复完正常使用后生成Timeline:3
注意:归档目录下的.history文件不要删除,恢复时需要读取.

## 在线备份一个独立的备份  

pg_basebackup默认不备份wal日志,所以单独使用不能打开数据库.  
如果带上-X参数,则会一起备份在备份期间生成的WAL日志,这样的话可以将此备份恢复到最近的一致点.
比起冷备份来无需关闭数据库.

//END

