---
layout: post
title: linux multipah切换相关测试
category: 技术
---

linux multipah切换时间测试

[TOC]



# 测试结果

> 业务中断时间通过 fio持续对mpath设备写入来观察，down path 通过 端 存储节点的public网卡 实现
>
> 

| 客户端    | multipath 策略 | 故障模拟说明        | 业务中断时间 | 优化后中断时间 |
| --------- | -------------- | ------------------- | ------------ | -------------- |
| centos7.0 | active-backup  | down active path    | 20s          |                |
|           |                | down stand-by  path | 0s           |                |
|           | roud-robin     | down any one path   | 20s          |                |
| centos6.6 | active-backup  | down active path    | 120s         | 12s            |
|           |                | down stand-by  path | 120s         | 12s            |
|           | roud-robin     | down any one path   | 120s         | 12s            |
|           |                |                     |              |                |



# 知识点记录

1. 确认当前多路径策略需要看 活动路径的状态，不能光看multipath输出有 roudrobin，就认为是 roud-robin。

2. 发现 centos 6.6的 默认 path select 用的时 roud-robin，7.0 用的是 service time（类似 active-backup）。所谓的默认是 /etc/multipath.conf 配置文件没有指定 配置项 ` path_grouping_policy `,该配置项 一般取值如下： multibus（即roud-robin模式），failover（即 active-backup模式）

3. 相关参数的默认值 可以通过 man multipath.conf去确认



# 测试过程记录



## centos7.0 版本测试

> 后来补充 stremwrite 测试，mount 创建文件系统后，默认multipath配置下，中断一般 10-14s左右
>
> 

### multipath 策略采用 actvie-backup

```
[root@test01 multpath_test]# iscsiadm -m session
tcp: [6] 10.16.172.124:3260,1 iqn.2017-10.com.test:storage (non-flash)
tcp: [7] 10.16.172.125:3260,1 iqn.2017-10.com.test:storage (non-flash)

[root@test01 yum.repos.d]# multipath -ll
mpatha (23446d5dc97d4401f) dm-3 Bigtera ,VirtualStor_Conv
size=100G features='0' hwhandler='0' wp=rw
|-+- policy='service-time 0' prio=1 status=active
| `- 4:0:0:1 sdc 8:32 active ready running
`-+- policy='service-time 0' prio=1 status=enabled
  `- 5:0:0:1 sde 8:64 active ready running
[root@test01 yum.repos.d]# 

断开 active path 后
size=100G features='0' hwhandler='0' wp=rw
|-+- policy='service-time 0' prio=0 status=enabled
| `- 10:0:0:1 sdc 8:32 failed faulty running
`-+- policy='service-time 0' prio=1 status=active
  `- 9:0:0:1  sde 8:64 active ready running


通过fio对 mpathX 设备打流量，为了防止流量过大，造成磁盘过度繁忙，影响测试，所以控制 rate_iops，iodepth也设置为1，通过 fio写入过程中的中断时间来计算影响时间
fio --name=seqwrite --rw=write --bs=64k --size=20G  --rate_iops=80 --ioengine=libaio  --numjobs=1 --filename=/dev/mapper/mpathd --direct=1 --group_reporting
```





### multipath 采用 roud-robin

```
[root@test01 ~]# multipath -ll
mpathb (232e33f007a22e45b) dm-4 Bigtera ,VirtualStor_Conv
size=5.0T features='0' hwhandler='0' wp=rw
`-+- policy='round-robin 0' prio=1 status=active
  |- 10:0:0:0 sdb 8:16 active ready running
  `- 9:0:0:0  sdd 8:48 active ready running
mpatha (23446d5dc97d4401f) dm-3 Bigtera ,VirtualStor_Conv
size=100G features='0' hwhandler='0' wp=rw
`-+- policy='round-robin 0' prio=1 status=active
  |- 10:0:0:1 sdc 8:32 active ready running
  `- 9:0:0:1  sde 8:64 active ready running
[root@test01 ~]# 

测试方法同上
```







## centos6.6 版本测试



### multipath active-standby

```
mpatha (23446d5dc97d4401f) dm-2 Bigtera,VirtualStor_Conv
size=100G features='0' hwhandler='0' wp=rw
|-+- policy='round-robin 0' prio=0 status=enabled
| `- 3:0:0:1 sdc 8:32 failed faulty running
`-+- policy='round-robin 0' prio=1 status=active
  `- 5:0:0:1 sde 8:64 active ready running
```





### multipath roudrobin

```
mpatha (23446d5dc97d4401f) dm-2 Bigtera,VirtualStor_Conv
size=100G features='0' hwhandler='0' wp=rw
`-+- policy='round-robin 0' prio=1 status=active
  |- 3:0:0:1 sdc 8:32 active ready running
  `- 5:0:0:1 sde 8:64 active ready running

```



### 调整 iscsi参数后，测试 结果

```
修改 /etc/iscsi/iscsid.conf
node.session.timeo.replacement_timeout = 3



mpatha (23446d5dc97d4401f) dm-2 Bigtera,VirtualStor_Conv
size=100G features='0' hwhandler='0' wp=rw
`-+- policy='round-robin 0' prio=1 status=active
  |- 6:0:0:1 sdc 8:32 active ready running
  `- 7:0:0:1 sde 8:64 active ready running
  
  
  中断一条path，中断 在 12s 左右
```







# 手动切换路径

> 手动切换路径是 针对于 事先知道哪条路径将会有问题，提前切换，从而避免自动切换（对应影响时间见文档开始）对业务造成的影响。但是不适用 centos6.6的版本，因为该版本 断掉 非active path的情况下，仍然会有 120s的中断。 所以如果是 centos6.6的版本，需要通过 fail path的方式来实现，后面会说明



```

[root@localhost multipath]# multipathd show maps
name   sysfs uuid             
mpathc dm-2  271092d0f9c9ba322

[root@localhost multipath]# multipathd show paths
hcil     dev dev_t pri dm_st  chk_st dev_st  next_check     
2:0:0:0  sda 8:0   1   undef  undef  unknown orphan         
31:0:0:0 sdb 8:16  1   active ready  running XXX....... 7/20
32:0:0:0 sdc 8:32  1   active ready  running XXX....... 7/20
[root@localhost multipath]# 


手动切换路径
multipath -k 可以进入交互模式

每条路径有编号，依次为 group 1

查看设备下的所有路径
multipathd> show topology
mpathd (2ff2b09e40c04d8f0) dm-2 Bigtera,VirtualStor_Conv
size=10G features='0' hwhandler='0' wp=rw
|-+- policy='round-robin 0' prio=1 status=enabled
| `- 33:0:0:0 sdb 8:16 active ready running
|-+- policy='round-robin 0' prio=1 status=active
| `- 34:0:0:0 sdc 8:32 active ready running
|-+- policy='round-robin 0' prio=1 status=enabled
| `- 35:0:0:0 sdd 8:48 active ready running
`-+- policy='round-robin 0' prio=1 status=enabled
  `- 36:0:0:0 sde 8:64 active ready running

可以看到上面一共有4条路径，分别为 group 1 ，group 2...., 可以看到当前 active path在 group 2

如果想 将 active path 切换到 第四条路径，命令如下
multipathd> switch map mpathd group 3
ok


备注：切换路径时，有时候需要确认某个盘符 是通过哪个 sesssion 创建的，可以通过 iscsiadm找到，具体命令见后面章节。手动切换路径后，fio测试 发现 没有中断。



```



# 手动失效路径（fail path）

> 针对 centos6.6 down 非active path 也会有影响的情况，需要通过 fail path的方式，将对应不需要的path 提前failed，这样该path的变化就不会有影响



```
1,确认盘符对应哪个 session（ip），需要注意 如果有多个 target，一个ip可能会有多个session，后面会单独补充说明
[root@localhost ~]# iscsiadm -m session -P 3 |grep -Ei "Current Portal|disk"
	Current Portal: 10.16.172.124:3260,1
			Attached scsi disk sdb		State: running
	Current Portal: 10.16.172.125:3260,1
			Attached scsi disk sdc		State: running
可以看到 sdb 是通过10.16.172.124 挂载，现在想 fail 该 路径

2,进入交互模式
multipathd -k

3，查看当前多路径的信息
multipathd> show topology 
mpathd (2ff2b09e40c04d8f0) dm-2 Bigtera,VirtualStor_Conv
size=10G features='0' hwhandler='0' wp=rw
|-+- policy='round-robin 0' prio=1 status=enabled
| `- 3:0:0:0 sdb 8:16 active ready running
`-+- policy='round-robin 0' prio=1 status=active
  `- 4:0:0:0 sdc 8:32 active ready running

4，fail sdb path，并且确认结果
multipathd> fail path sdb
ok
multipathd> show topology 
mpathd (2ff2b09e40c04d8f0) dm-2 Bigtera,VirtualStor_Conv
size=10G features='0' hwhandler='0' wp=rw
|-+- policy='round-robin 0' prio=0 status=enabled
| `- 3:0:0:0 sdb 8:16 failed faulty running
`-+- policy='round-robin 0' prio=0 status=enabled
  `- 4:0:0:0 sdc 8:32 failed faulty running



5，恢复 sdb path
multipathd> reinstate path sdb
ok
multipathd> show topology 
mpathd (2ff2b09e40c04d8f0) dm-2 Bigtera,VirtualStor_Conv
size=10G features='0' hwhandler='0' wp=rw
|-+- policy='round-robin 0' prio=1 status=active
| `- 3:0:0:0 sdb 8:16 active ready running
`-+- policy='round-robin 0' prio=1 status=enabled
  `- 4:0:0:0 sdc 8:32 active ready running




```

## 补充说明

多个 target 的情况，应该首先 确认 到 某个ip 的 总 session数目

每一个 到target的连接 称为一个 session，如果多个 target，会产生到 同一个ip的多个session

```
[root@localhost ~]# iscsiadm -m session
tcp: [1] 10.16.172.124:3260,1 iqn.2018-08.com:wsg (non-flash)
tcp: [2] 10.16.172.125:3260,1 iqn.2018-08.com:wsg (non-flash)
tcp: [5] 10.16.172.126:3260,1 iqn.2018-08.com:wsg (non-flash)
tcp: [6] 10.16.172.124:3260,1 iqn.2018-08.com:wsg2 (non-flash)

所以 当 客户端 挂载 磁盘 比较多时，需要 手动 fail path 时，容易忽略掉其他的 盘。
[root@localhost ~]# iscsiadm -m session -P 3 |grep -Ei "Current Portal|disk"
	Current Portal: 10.16.172.124:3260,1
			Attached scsi disk sdb		State: running
	Current Portal: 10.16.172.125:3260,1
			Attached scsi disk sdc		State: running
	Current Portal: 10.16.172.126:3260,1
			Attached scsi disk sdf		State: running
	Current Portal: 10.16.172.124:3260,1
			Attached scsi disk sdg		State: running

```







# 手动删除路径



```
 add path $path
 remove|del path $path
 
 add map|multipath $map
 remove|del map|multipath $map

del path sdd

删除后，发现无法 add，重启 multipath也不能恢复，需要 进一步 找原因。。？？


 
 
 

```



# 多路径policy 对性能影响（补充）

37 物理节点 多路径挂载 247物理集群, 共2条 万兆路径



测试结果汇总如下：

|               | iodepth=4  |         | iodepth=32 |         | iodepth=64 |         | iodepth=128 |         |
| ------------- | ---------- | ------- | ---------- | ------- | ---------- | ------- | ----------- | ------- |
|               | 带宽(MB/s) | latency | 带宽(MB/s) | latency | 带宽(MB/s) | latency | 带宽(MB/s)  | latency |
| 单路径        | 270        | 15ms    | 314        | 102ms   | 550        | 116ms   | 345         | 370ms   |
| roud2         |            |         | 569        | 56ms    |            |         | 559         | 228ms   |
|               |            |         | 508        | 63ms    |            |         |             |         |
|               |            |         |            |         |            |         |             |         |
| active+backup | 290        | 14ms    | 484        | 66ms    | 573        | 112ms   | 515         | 248ms   |
| roud2         |            |         | 512        | 62ms    |            |         |             |         |
|               |            |         |            |         |            |         |             |         |
|               |            |         |            |         |            |         |             |         |
| roud-robin    | 277        | 14ms    | 518        | 62ms    | 480        | 133ms   | 510         | 250ms   |
|               |            |         | 389        | 82ms    |            |         |             |         |

> 单路径  第一轮的结果 可能 不准。
>
> 

测试过程记录如下

## 单路径的性能





```
root@node37:~# multipath -ll
21fe9f8718ffb8161 dm-8 Bigtera ,VirtualStor_Scal
size=2.0T features='0' hwhandler='0' wp=rw
|-+- policy='round-robin 0' prio=1 status=active
| `- 9:0:0:1  sdf 8:80 active ready running
`-+- policy='round-robin 0' prio=1 status=enabled
  `- 10:0:0:1 sdg 8:96 active ready running
root@node37:~# fio --name=seqwrite --rw=write --bs=1M --size=20G  --ioengine=libaio --iodepth=128 --numjobs=1 --filename=/dev/sdf --direct=1 --group_reporting
seqwrite: (g=0): rw=write, bs=1M-1M/1M-1M/1M-1M, ioengine=libaio, iodepth=128
fio-2.1.3
Starting 1 process
Jobs: 1 (f=1): [W] [100.0% done] [0KB/346.0MB/0KB /s] [0/346/0 iops] [eta 00m:00s]
seqwrite: (groupid=0, jobs=1): err= 0: pid=822839: Mon Jul  8 15:03:31 2019
  write: io=20480MB, bw=353806KB/s, iops=345, runt= 59274msec
    slat (usec): min=60, max=2430, avg=131.99, stdev=78.12
    clat (msec): min=17, max=1380, avg=370.25, stdev=183.71
     lat (msec): min=17, max=1381, avg=370.39, stdev=183.71
    clat percentiles (msec):
     |  1.00th=[  116],  5.00th=[  143], 10.00th=[  174], 20.00th=[  223],
     | 30.00th=[  258], 40.00th=[  293], 50.00th=[  330], 60.00th=[  371],
     | 70.00th=[  429], 80.00th=[  506], 90.00th=[  611], 95.00th=[  734],
     | 99.00th=[  996], 99.50th=[ 1045], 99.90th=[ 1188], 99.95th=[ 1237],
     | 99.99th=[ 1369]
    bw (KB  /s): min=99219, max=655683, per=100.00%, avg=358190.29, stdev=132592.23
    lat (msec) : 20=0.01%, 50=0.01%, 100=0.19%, 250=27.35%, 500=51.51%
    lat (msec) : 750=16.75%, 1000=3.33%, 2000=0.86%
  cpu          : usr=2.85%, sys=2.31%, ctx=7355, majf=0, minf=360809
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.2%, >=64=99.7%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued    : total=r=0/w=20480/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
  WRITE: io=20480MB, aggrb=353806KB/s, minb=353806KB/s, maxb=353806KB/s, mint=59274msec, maxt=59274msec

Disk stats (read/write):
  sdf: ios=3/10217, merge=0/10223, ticks=1328/3770472, in_queue=3780480, util=99.75%
root@node37:~# 


```





fio的iodepth 下降可以 降低 latency

```
root@node37:~# fio --name=seqwrite --rw=write --bs=1M --size=20G  --ioengine=libaio --iodepth=4 --numjobs=1 --filename=/dev/sdf --direct=1 --group_reporting
seqwrite: (g=0): rw=write, bs=1M-1M/1M-1M/1M-1M, ioengine=libaio, iodepth=4
fio-2.1.3
Starting 1 process
Jobs: 1 (f=1): [W] [100.0% done] [0KB/232.0MB/0KB /s] [0/232/0 iops] [eta 00m:00s]
seqwrite: (groupid=0, jobs=1): err= 0: pid=861883: Mon Jul  8 15:09:13 2019
  write: io=20480MB, bw=276874KB/s, iops=270, runt= 75744msec
    slat (usec): min=52, max=699, avg=112.73, stdev=32.06
    clat (msec): min=6, max=279, avg=14.68, stdev=19.34
     lat (msec): min=6, max=279, avg=14.79, stdev=19.34
    clat percentiles (msec):
     |  1.00th=[    8],  5.00th=[    9], 10.00th=[   10], 20.00th=[   11],
     | 30.00th=[   11], 40.00th=[   12], 50.00th=[   13], 60.00th=[   14],
     | 70.00th=[   15], 80.00th=[   16], 90.00th=[   18], 95.00th=[   19],
     | 99.00th=[   32], 99.50th=[  217], 99.90th=[  223], 99.95th=[  225],
     | 99.99th=[  273]
    bw (KB  /s): min=80984, max=355640, per=100.00%, avg=278316.16, stdev=61307.79
    lat (msec) : 10=18.75%, 20=77.87%, 50=2.48%, 250=0.89%, 500=0.02%
  cpu          : usr=1.37%, sys=2.14%, ctx=16571, majf=0, minf=10338
  IO depths    : 1=0.1%, 2=0.1%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=20480/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
  WRITE: io=20480MB, aggrb=276873KB/s, minb=276873KB/s, maxb=276873KB/s, mint=75744msec, maxt=75744msec

Disk stats (read/write):
  sdf: ios=4/20465, merge=0/0, ticks=24/300292, in_queue=300220, util=99.94%


```





## 多路径（active backup）

客户端

```
root@node37:~# multipath -ll
21fe9f8718ffb8161 dm-8 Bigtera ,VirtualStor_Scal
size=2.0T features='0' hwhandler='0' wp=rw
|-+- policy='round-robin 0' prio=1 status=active
| `- 9:0:0:1  sdf 8:80 active ready running
`-+- policy='round-robin 0' prio=1 status=enabled
  `- 10:0:0:1 sdg 8:96 active ready running
root@node37:~# 

```



```
root@node37:~# fio --name=seqwrite --rw=write --bs=1M --size=20G  --ioengine=libaio --iodepth=4 --numjobs=1 --filename=/dev/dm-8 --direct=1 --group_reporting
seqwrite: (g=0): rw=write, bs=1M-1M/1M-1M/1M-1M, ioengine=libaio, iodepth=4
fio-2.1.3
Starting 1 process
Jobs: 1 (f=1): [W] [100.0% done] [0KB/319.7MB/0KB /s] [0/319/0 iops] [eta 00m:00s]
seqwrite: (groupid=0, jobs=1): err= 0: pid=886978: Mon Jul  8 15:12:32 2019
  write: io=20480MB, bw=297976KB/s, iops=290, runt= 70380msec
    slat (usec): min=49, max=555, avg=107.63, stdev=30.82
    clat (msec): min=6, max=223, avg=13.63, stdev=16.77
     lat (msec): min=6, max=223, avg=13.74, stdev=16.77
    clat percentiles (msec):
     |  1.00th=[    8],  5.00th=[    9], 10.00th=[   10], 20.00th=[   10],
     | 30.00th=[   11], 40.00th=[   12], 50.00th=[   12], 60.00th=[   13],
     | 70.00th=[   14], 80.00th=[   15], 90.00th=[   16], 95.00th=[   18],
     | 99.00th=[   26], 99.50th=[  215], 99.90th=[  219], 99.95th=[  221],
     | 99.99th=[  223]
    bw (KB  /s): min=74704, max=364544, per=100.00%, avg=300092.54, stdev=58268.84
    lat (msec) : 10=22.04%, 20=75.59%, 50=1.70%, 100=0.01%, 250=0.66%
  cpu          : usr=1.37%, sys=1.99%, ctx=19590, majf=0, minf=1040
  IO depths    : 1=0.1%, 2=0.1%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=20480/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
  WRITE: io=20480MB, aggrb=297975KB/s, minb=297975KB/s, maxb=297975KB/s, mint=70380msec, maxt=70380msec

Disk stats (read/write):
    dm-8: ios=275/20430, merge=0/0, ticks=220/278492, in_queue=278656, util=100.00%, aggrios=194/10240, aggrmerge=0/0, aggrticks=146/139042, aggrin_queue=139186, aggrutil=100.00%
  sdf: ios=385/20480, merge=0/0, ticks=288/278084, in_queue=278368, util=100.00%
  sdg: ios=3/0, merge=0/0, ticks=4/0, in_queue=4, util=0.01%


```







## 多路径 roud-robin



```
root@node37:~# multipath -ll
21fe9f8718ffb8161 dm-8 Bigtera ,VirtualStor_Scal
size=2.0T features='0' hwhandler='0' wp=rw
`-+- policy='round-robin 0' prio=1 status=active
  |- 10:0:0:1 sdg 8:96 active ready running
  `- 9:0:0:1  sdf 8:80 active ready running
root@node37:~# 




root@node37:~# fio --name=seqwrite --rw=write --bs=1M --size=20G  --ioengine=libaio --iodepth=4 --numjobs=1 --filename=/dev/dm-8 --direct=1 --group_reporting
seqwrite: (g=0): rw=write, bs=1M-1M/1M-1M/1M-1M, ioengine=libaio, iodepth=4
fio-2.1.3
Starting 1 process
Jobs: 1 (f=1): [W] [100.0% done] [0KB/193.9MB/0KB /s] [0/193/0 iops] [eta 00m:00s]
seqwrite: (groupid=0, jobs=1): err= 0: pid=391841: Tue Jul  9 09:47:08 2019
  write: io=20480MB, bw=284028KB/s, iops=277, runt= 73836msec
    slat (usec): min=48, max=1947, avg=114.27, stdev=54.29
    clat (msec): min=6, max=222, avg=14.30, stdev=19.82
     lat (msec): min=6, max=223, avg=14.42, stdev=19.82
    clat percentiles (msec):
     |  1.00th=[    8],  5.00th=[    9], 10.00th=[   10], 20.00th=[   11],
     | 30.00th=[   11], 40.00th=[   12], 50.00th=[   12], 60.00th=[   13],
     | 70.00th=[   14], 80.00th=[   15], 90.00th=[   17], 95.00th=[   18],
     | 99.00th=[   79], 99.50th=[  215], 99.90th=[  221], 99.95th=[  221],
     | 99.99th=[  223]
    bw (KB  /s): min=142788, max=370512, per=100.00%, avg=285726.13, stdev=61431.16
    lat (msec) : 10=20.21%, 20=77.29%, 50=1.47%, 100=0.07%, 250=0.96%
  cpu          : usr=1.42%, sys=2.01%, ctx=19395, majf=0, minf=23744
  IO depths    : 1=0.1%, 2=0.1%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=20480/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
  WRITE: io=20480MB, aggrb=284028KB/s, minb=284028KB/s, maxb=284028KB/s, mint=73836msec, maxt=73836msec

Disk stats (read/write):
    dm-8: ios=276/20437, merge=0/0, ticks=224/292556, in_queue=292704, util=100.00%, aggrios=181/10240, aggrmerge=0/0, aggrticks=230/145940, aggrin_queue=146170, aggrutil=51.27%
  sdf: ios=280/10442, merge=0/0, ticks=400/149064, in_queue=149460, util=51.27%
  sdg: ios=82/10038, merge=0/0, ticks=60/142816, in_queue=142880, util=49.09%

```



增加fio的 iodepth到128

```
root@node37:~# fio --name=seqwrite --rw=write --bs=1M --size=20G  --ioengine=libaio --iodepth=128 --numjobs=1 --filename=/dev/dm-8 --direct=1 --group_reporting
seqwrite: (g=0): rw=write, bs=1M-1M/1M-1M/1M-1M, ioengine=libaio, iodepth=128
fio-2.1.3
Starting 1 process
Jobs: 1 (f=1): [W] [100.0% done] [0KB/483.6MB/0KB /s] [0/483/0 iops] [eta 00m:00s]
seqwrite: (groupid=0, jobs=1): err= 0: pid=450548: Tue Jul  9 09:54:44 2019
  write: io=20480MB, bw=522551KB/s, iops=510, runt= 40133msec
    slat (usec): min=61, max=3885, avg=196.10, stdev=202.43
    clat (msec): min=6, max=1978, avg=250.54, stdev=175.43
     lat (msec): min=6, max=1978, avg=250.74, stdev=175.44
    clat percentiles (msec):
     |  1.00th=[   20],  5.00th=[   61], 10.00th=[  104], 20.00th=[  143],
     | 30.00th=[  159], 40.00th=[  174], 50.00th=[  188], 60.00th=[  215],
     | 70.00th=[  273], 80.00th=[  383], 90.00th=[  449], 95.00th=[  553],
     | 99.00th=[  906], 99.50th=[ 1074], 99.90th=[ 1631], 99.95th=[ 1647],
     | 99.99th=[ 1762]
    bw (KB  /s): min=60711, max=968704, per=100.00%, avg=529317.22, stdev=211070.89
    lat (msec) : 10=0.16%, 20=0.85%, 50=2.59%, 100=5.97%, 250=56.88%
    lat (msec) : 500=27.53%, 750=4.17%, 1000=1.21%, 2000=0.65%
  cpu          : usr=4.04%, sys=6.46%, ctx=15280, majf=0, minf=801334
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.2%, >=64=99.7%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.1%
     issued    : total=r=0/w=20480/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
  WRITE: io=20480MB, aggrb=522550KB/s, minb=522550KB/s, maxb=522550KB/s, mint=40133msec, maxt=40133msec

Disk stats (read/write):
    dm-8: ios=279/19746, merge=0/706, ticks=1856/4953844, in_queue=4976604, util=100.00%, aggrios=214/9887, aggrmerge=0/0, aggrticks=1194/2479762, aggrin_queue=2480954, aggrutil=69.16%
  sdf: ios=6/9996, merge=0/0, ticks=1072/2952836, in_queue=2953908, util=69.16%
  sdg: ios=422/9778, merge=0/0, ticks=1316/2006688, in_queue=2008000, util=52.91%
root@node37:~# 

```





iodetph to 64

```
root@node37:~# fio --name=seqwrite --rw=write --bs=1M --size=20G  --ioengine=libaio --iodepth=64 --numjobs=1 --filename=/dev/dm-8 --direct=1 --group_reporting
seqwrite: (g=0): rw=write, bs=1M-1M/1M-1M/1M-1M, ioengine=libaio, iodepth=64
fio-2.1.3
Starting 1 process
Jobs: 1 (f=1): [W] [100.0% done] [0KB/580.5MB/0KB /s] [0/580/0 iops] [eta 00m:00s]
seqwrite: (groupid=0, jobs=1): err= 0: pid=719302: Tue Jul  9 10:32:03 2019
  write: io=20480MB, bw=491804KB/s, iops=480, runt= 42642msec
    slat (usec): min=53, max=7122, avg=144.34, stdev=73.69
    clat (msec): min=8, max=789, avg=133.05, stdev=98.09
     lat (msec): min=8, max=790, avg=133.20, stdev=98.09
    clat percentiles (msec):
     |  1.00th=[   24],  5.00th=[   57], 10.00th=[   64], 20.00th=[   73],
     | 30.00th=[   80], 40.00th=[   86], 50.00th=[   94], 60.00th=[  103],
     | 70.00th=[  119], 80.00th=[  163], 90.00th=[  306], 95.00th=[  322],
     | 99.00th=[  510], 99.50th=[  529], 99.90th=[  758], 99.95th=[  775],
     | 99.99th=[  783]
    bw (KB  /s): min=141575, max=958464, per=100.00%, avg=497120.71, stdev=175001.43
    lat (msec) : 10=0.02%, 20=0.65%, 50=2.87%, 100=54.19%, 250=24.27%
    lat (msec) : 500=16.93%, 750=0.95%, 1000=0.11%
  cpu          : usr=4.02%, sys=3.37%, ctx=15874, majf=0, minf=49218
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.2%, >=64=99.7%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.1%, >=64=0.0%
     issued    : total=r=0/w=20480/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
  WRITE: io=20480MB, aggrb=491804KB/s, minb=491804KB/s, maxb=491804KB/s, mint=42642msec, maxt=42642msec

Disk stats (read/write):
    dm-8: ios=279/20478, merge=0/0, ticks=4672/2709984, in_queue=2715024, util=100.00%, aggrios=185/10240, aggrmerge=0/0, aggrticks=2466/1354548, aggrin_queue=1357012, aggrutil=57.39%
  sdf: ios=220/9989, merge=0/0, ticks=1816/1301084, in_queue=1302896, util=51.80%
  sdg: ios=151/10491, merge=0/0, ticks=3116/1408012, in_queue=1411128, util=57.39%
root@node37:~# 

```







# multipath 默认参数



一般 /etc/multipath.conf 中，配置的不多，所以很多参数 用的是默认的。修给 multipath.conf后，需要重启 multipath ，需要确认这些默认值,下面是 centos6.6 的默认

```
可以通过 multpathd show config 或者 man multipath.conf 来看。

no_path_retry 如果设置很大，会导致 切换变长。
failback 控制故障路径恢复后的行为，如果设置 immediate，表示路径恢复后（比如故障路径原来是active path，恢复后，立即又变为 active path）。 如果设置 manual，故障路径恢复后，不进行任何操作。

        path_selector "round-robin 0"
        path_grouping_policy failover

        path_checker directio
        failback manual
        user_friendly_names yes
        find_multipaths no



```



man 手册

```
       no_path_retry    Specify the number of retries until disable queueing, or fail for immediate failure (no queueing), queue for never stop queueing.  Default
                        is 0.
                        
                        

```







## centos7.0 



默认的multipath.conf 

```
[root@localhost ~]# cat /etc/multipath.conf |grep -v '^#'
defaults {
	user_friendly_names yes
	find_multipaths yes
}

```





```
        path_selector "service-time 0"
        path_grouping_policy "failover"

        path_checker "directio"
        alias_prefix "mpath"

```



