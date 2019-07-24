---
layout: post
title: raid 运维记录
category: 技术
---

webBIOS 图形界面配置raid需要重启系统，并且UI操作很繁琐，通过megacli可以实现所有的配置。这里会逐步收集 各种场景下的raid操作命令


相关选项说明（megacli选项 是不区分大小写的）：
* ld（logic driver），即raid组，由1块或多块物理硬盘组成的逻辑磁盘，一般也叫做VD（virtual driver）
* pd（physical driver），即物理磁盘
* A （adapter），即raid卡，一般每个节点只有一个raid卡，所以常用 -A0来表示该节点的raid卡，当然如果有第二块raid卡，就需要用 -A1 来表示
* 另外，关于一些背景说明的 可以参考 [[运维手册]] 的相关章节
* 每个硬盘都是由 Eclosure Id:Slot number 简称[E:S]唯一确定，创建raid 和定位硬盘，查询rebuid进度等，都是要通过 [E:S]

本文目录如下：

[TOC]



# 特殊故障记录



## 模拟拔盘

```
/opt/MegaRAID/MegaCli/MegaCli64 -PDOffline -PhysDrv [23:7] -aN
/opt/MegaRAID/MegaCli/MegaCli64 -PDMarkMissing -PhysDrv [23:7] -a0

此时如果autorebuild enable时，就会自动开始rebuild
```





## 运行中的raid5 误拔了盘

一般有2中方式恢复



```
第一种（推荐方法）是首先对磁盘进行make good，然后再import foreigh (命令如下)，之后进入rebuild状态。
第二种（此方法可行，但不推荐）是对磁盘进行make good，然后clear foreign，最后将其设为
热备盘，然后进入rebuild状态。

/opt/MegaRAID/MegaCli/MegaCli64 -CfgForeign -import -a0
```









## raid5 rebuild 期间，其他磁盘的media erro 导致的-a-punctured-raid-array

在亚信和苏州电视台

需要关注 raid卡日志 中的几个关于 recovery的

```

Event Description: Puncturing bad block on PD 15(e0x00/s7) at 20d803e
Event Data:


Event Description: Corrected medium error during recovery on PD 15(e0x08/s4) at 14553b546
Event Data:
===========
Device ID: 21
Enclosure Index: 8
Slot Number: 4


```



## 正常运行情况下，模拟故障拔盘

有热备盘情况下，拔盘后，再插上：热备盘会立即顶上，rebuild结束后，需要将 拔掉的盘清除 foreigh key后，热备盘才会触发copyback（拔掉的盘上保存有raid的信息，只有将盘 makegood之后，raid卡才能 scan到 foreign key，否则raid卡不会认为该盘是 foreign ）。该情况是在 如下raid卡上验证过：

```
Product Name    : AVAGO MegaRAID SAS 9361-8i
Serial No       : SK84168002
FW Package Build: 24.21.0-0025
该raid卡上，还验证过，hotspare 盘 在完成使命后，会自动 恢复成hotspare。需要在 9271-8i的卡上 ，现象 是否 一致？？？

```



无热备盘情况，拔盘后，再插上（lab验证过）：raid 会一直处于degrade状态，并且拔掉的盘也会处于unconfigure bad状态，即使将 盘更改为 unconfigure good，raid也不会开始rebuild。 （当盘变为 unconfigure good 时，raid会扫描到 foreign key，所以需要 clear foreign key，然后再将 该盘设置为 hotspare，才能让 raid开始rebuild。）。测试的raid卡型号如下:

```
Product Name    : LSI MegaRAID SAS 9271-8i
Serial No       : SV45213767
FW Package Build: 23.12.0-0011
```



# 常用命令汇总



##raid有一个 auto rebuild功能

lsi 的9271raid卡上有，不确定其他型号 raid卡是否也有

```
/opt/MegaRAID/MegaCli/MegaCli64 -AdpAutoRbld -Enbl -A0
```



没有autorebuild的情况下，可以尝试 手动发起rebuild

```
/opt/MegaRAID/MegaCli/MegaCli64 -PDRbld -Start -PhysDrv [E:S] -aN
/opt/MegaRAID/MegaCli/MegaCli64 -PDRbld -Stop -PhysDrv [E:S] -aN

```



## 有foreign key的状态的磁盘 无法 offline

```
root@node37:~# /opt/MegaRAID/MegaCli/MegaCli64 -CfgForeign -Scan -aALL
                                     
There are 1 foreign configuration(s) on controller 0.

Exit Code: 0x00
root@node37:~#


有foreign key 情况下，无法 offline
root@node37:~# /opt/MegaRAID/MegaCli/MegaCli64 -PDOffline -PhysDrv[23:5] -a0
                                     
Adapter: 0: Failed to change PD state at EnclId-23 SlotId-5.

Exit Code: 0x01
root@node37:~#
```



## 查看节点上raid组（ld）的个数，类型，策略，状态，每个raid组有哪些物理磁盘组成

    /opt/MegaRAID/MegaCli/MegaCli64 ldinfo -lall -a0

* 关于raid组创建的一些要求，和策略说明，可以参考 [[运维手册]] 的相关章节


## 有时候测试，需要保证raid 一直处于 WB（避免因为bbu学习，或者没有bbu时，变成WT）
    /opt/MegaRAID/MegaCli/MegaCli64 LDSetProp cachedbadbbu -lall a0
    
    恢复的时候，
    /opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp -NoCachedBadBBU -Immediate -Lall -aAll

## pd offline 和online

    /opt/MegaRAID/MegaCli/MegaCli64 -PDOffline -PhysDrv[8:1] -a0
    /opt/MegaRAID/MegaCli/MegaCli64 -PDonline -PhysDrv [24:3] -a0

## 配置raid 的jbod模式

    查看 jbob配置模式
    root@node-44:~# /opt/MegaRAID/MegaCli/MegaCli64  -AdpGetProp -enablejbod -aALL                                   
    Adapter 0: JBOD: Disabled
    Exit Code: 0x00
    
    启用 jbod模式
    /opt/MegaRAID/MegaCli/MegaCli64 -AdpSetProp -EnableJBOD 1 -a0
    
    有时需要执行
    megacli -PDMakeJBOD -PhysDrv[32:${i}] -a0
    
    jbod 模式配置后，有时希望将某些 jbob盘继续做成raid，此时需要 将jbod盘 的 firmware state 更改
    MegaCli -PDMakeGood -PhysDrv[9:9] -a0
    更改完，需要检查下 是否有 foreign key，有的话 要 先 clear

## pdlist中也是可以查询到vd的信息，DiskGroup和 VD的关系
> 正常情况下，一个DiskGroup会把全部的空间做为一个VD，但是有时候空间紧张的情况下，也会创建多个VD。所以需要注意VD 的底层 是否用的是同一组 DiskGroup。
> 更准确的验证一台设备，到底有几组raid，应该 看pdlist中的DiskGroup 信息。

    root@node02:~# /opt/MegaRAID/MegaCli/MegaCli64 pdlist -a0 |grep DiskGr
    Drive's position: DiskGroup: 3, Span: 0, Arm: 0
    Drive's position: DiskGroup: 3, Span: 0, Arm: 1

## rebuild rate
    /opt/MegaRAID/MegaCli/MegaCli64 -AdpSetProp RebuildRate -30 -a0    ---修改raid的rebuid速度
    默认 30
## BBU learning
默认情况 ，BBU 每隔28天 会自动 进行learning，但也可以手动发起learning，命令如下
    
    手动发起 learn
    /opt/MegaRAID/MegaCli/MegaCli64 -AdpBbuCmd -BbuLearn -aALL

在某些情况下，需要关闭BBU的auto learn，方法如下：

    echo 'autoLearnMode=1' >/tmp/megaraid.conf
    MegaCli -AdpBbuCmd -SetBbuProperties -f /tmp/megaraid.conf -aAll
    1为Disable, 0为Enable, 从Disable切换到Enable时，Relearn操作会立刻执行
    
    确认是否生效
    MegaCli -AdpBbuCmd -GetBbuProperties -aALL

## raid卡的告警设置
    有时候会发现 设置 告警会失败，因为告警功能也是 raid卡的一个模块（和BBU类似），可以检查 alarm模块是否存在 （通过adpallinfo grep Alarm 关键字）
    MegaCli  -AdpSetProp  -AlarmSilence –a0  临时关闭，重启又变
    MegaCli  -AdpSetProp  -AlarmDsbl  –a0    永久关闭，重启后还
    MegaCli  -AdpSetProp  -Alarmenbl  –a0    开启       
    MegaCli  -AdpgetProp  -Alarmdsply  –a0   查看告警的状态  

## raid卡的cache flush

    /opt/MegaRAID/MegaCli/MegaCli64 -AdpCacheFlush -Aall

## 检查ssd 的寿命

这个命令其实 不属于 raid的命令，但是如果ssd 在raid 卡上，还是要 结合raid的命令来 组合查询

    如果ssd直接连在主板上面，找到ssd 的盘符，可以直接用下面的命令查询
    smartctl -A /dev/sdX   输出中WearOut 指示着，ssd 的使用寿命，全新的ssd 该值为 100 ，在亚信 有碰到过 该值是 5 的，官方说 30 就要注意该 ssd 就可能会损坏了。
    
    如果ssd 位于raid 卡上，需要先确认ssd 的device id（注意并不是 E：S 号），用如下命令
    smartctl -a -d megaraid,2  /dev/sda   其中 2 是ssd 的device id

## 新加入硬盘，重新插拔硬盘

    清除foreign key 
    /opt/MegaRAID/MegaCli/MegaCli64 -CfgForeign -Scan -aALL 
    /opt/MegaRAID/MegaCli/MegaCli64 -CfgForeign -Clear -aALL
    
    如何查看和清除 raid的 cache ##删除，重建raid时可能需要
    /opt/MegaRAID/MegaCli/MegaCli64 -GetPreservedCacheList -aALL 
    /opt/MegaRAID/MegaCli/MegaCli64 -DiscardPreservedCache -L7 -aall
    
    /opt/MegaRAID/MegaCli/MegaCli64 -PDMakeGood -PhysDrv[8:1] -a0
    /opt/MegaRAID/MegaCli/MegaCli64 -CfgForeign -import -a0

## 查看所有物理磁盘（pd）的信息

    /opt/MegaRAID/MegaCli/MegaCli64 pdlist -a0

## ld，pd一起查看，这样可以了解到每个ld下面由哪些pd组成
当熟悉了输出的内容之后，可以过滤掉一些不需要的信息。
    MegaCli -LdPdInfo -aALL


## raid的日常巡检
检查ld的状态，pd的状态，参考 [[Raid日常巡检]]

## 添加全局热备盘
    MegaCli -PDHSP Set -PhysDrv[E:S] -a0

## raid 卡日志，保存到文件
    /opt/MegaRAID/MegaCli/MegaCli64 -AdpEventLog -GetEvents -f raid.envent.log -a0

## 创建和删除raid


    ##删除raid,-L 指定logic driver
    root@scaler:~# /opt/MegaRAID/MegaCli/MegaCli64 -CfgLdDel -L2 -A0
    
    ##创建raid
    root@scaler:~# /opt/MegaRAID/MegaCli/MegaCli64 CfgLDAdd -r5 [41:5,41:6,41:7,41:8,41:9,41:10,41:11,41:12,41:13,41:14] -strpsz128 -A0 
    
    ## 删除raid，重新创建时，因为删除raid下面的磁盘存在foreign key，导致创建新raid时失败，尝试如下命令
    查看和清除foreign key 
    /opt/MegaRAID/MegaCli/MegaCli64 -CfgForeign -Scan -aALL 
    /opt/MegaRAID/MegaCli/MegaCli64 -CfgForeign -Clear -aALL
    
    如何查看和清除 raid的 cache ##删除，重建raid时可能需要
    /opt/MegaRAID/MegaCli/MegaCli64 -GetPreservedCacheList -aALL 
    /opt/MegaRAID/MegaCli/MegaCli64 -DiscardPreservedCache -L7 -aall

* 对于最新的6.1，在初次创建集群，配置raid时，可以用bean写的创建raid的工具



## 常用检查 磁盘erro 
    /opt/MegaRAID/MegaCli/MegaCli64 -pdlist -a0|grep -Ei "enc|Slot Number|Firmware stat|media error |other error"

## 在pdlist 中可以看到 硬盘类型
     Media Type: Hard Disk Device，，如果是固态盘，会显示 solid state device

## 闪烁硬盘，和关闭闪烁
让指定的 Enclosure ID:Slot number 的硬盘  闪烁

    MegaCli -PdLocate -start -physdrv [E:S]  -aALL
    MegaCli -PdLocate -stop -physdrv [E:S]  -aALL

## raid一致性检查
一致性检查时，会看到所有磁盘的红灯一直闪烁，默认每周都会做一次

    禁用一直性 检查
    /opt/MegaRAID/MegaCli/MegaCli64 -AdpCcSched -Dsbl -Aall
    启用一致性检查，
    /opt/MegaRAID/MegaCli/MegaCli64 -AdpCcSched -ModeConc -Aall
    查看一直性检查 信息
    /opt/MegaRAID/MegaCli/MegaCli64 -AdpCcSched -info -Aall
    
    ldinfo 中是可以看到 一致性检查 是否在进行和进度
    
    设置 raid卡cc的时间
    MegaCli -AdpCcSched -SetStartTime yyyymmdd hh -aN|-a0,1,2|-aALL
    /opt/MegaRAID/MegaCli/MegaCli64 -AdpCcSched -SetStartTime 20180515 11 -a0
    
    设置一致性检查的 rate，默认 30%
    /opt/MegaRAID/MegaCli/MegaCli64 -AdpSetProp CCRate 30 -aALL
    
    可以停止已经发起的cc，也可以 手动 发起 cc
    /opt/MegaRAID/MegaCli/MegaCli64 -LDCC -Stop -lall -aall
    手动发起
    /opt/MegaRAID/MegaCli/MegaCli64 -LDCC -Start -lall -aall
    
    查询cc进度
    /opt/MegaRAID/MegaCli/MegaCli64 -LDCC -ShowProg -lall -aall



## write through  改成 write back

    /opt/MegaRAID/MegaCli/MegaCli64 LDSetProp WB -Lall -Aall
    ##更改1个logic disk
    
     /opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp WB -L3 -A0
     /opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp WB -L4 -A0

## 查看电池状态 

    /opt/MegaRAID/MegaCli/MegaCli64 -AdpBbuCmd -GetBbuStatus -aALL
    -AdpBbuCmd -GetBbuCapacityInfo －A0

## 换硬盘 ，查看rebuild进度,有2中显示方式

    /opt/MegaRAID/MegaCli/MegaCli64 -PDRbld -showprog -physDrv [40:7] -a0
    另外一种显示方式，可以看到 持续的时间，如下，用 progdsply
    /opt/MegaRAID/MegaCli/MegaCli64 -pdrbld -progdsply -physdrv[E:S] -aALL

## 关闭 读的cache
/opt/MegaRAID/MegaCli/MegaCli64  LDSetProp Direct  -Lall -Aall


## 关闭 disk cache
/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp -DisDskCache -Lall -Aall   

## raid卡时间查看和设置
默认raid 卡的时间和实际时间相差8小时，

 1406  /opt/MegaRAID/MegaCli/MegaCli64 adpgettime -a0
 1409  /opt/MegaRAID/MegaCli/MegaCli64 adpsettime 20170710 14:18:00 -a0

## raid卡 恢复出厂设置
    MegaCli  -AdpFacDefSet –a0              恢复出厂的默认配置


