---
layout: post
title: 一次cpu iowait偏高故障处理
category: 技术
---



# 故障现象

uptime 的平均负载比其他正常节点偏高，正常节点不到1

```
root@pveNode38:~# uptime 
 15:26:56 up 17 days,  5:43,  4 users,  load average: 8.83, 8.96, 7.91
root@pveNode38:~# 

```



top 显示upload 高的原因是 iowait 偏高，达到 20左右

```
root@pveNode38:~# top
top - 15:25:16 up 17 days,  5:42,  4 users,  load average: 8.86, 8.98, 7.80
Tasks: 373 total,   1 running, 193 sleeping,   0 stopped,   0 zombie
%Cpu(s):  3.5 us,  1.6 sy,  0.0 ni, 75.2 id, 19.6 wa,  0.0 hi,  0.1 si,  0.0 st
KiB Mem : 65999920 total,   299716 free, 40101476 used, 25598728 buff/cache
KiB Swap:  8388604 total,  7280836 free,  1107768 used. 25040156 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                     
```





# 处理过程

所以现在目标是找到 iowait 高的原因。iowait高的原因一般是等待io设备，可能是磁盘，也可能是网络。

所以，希望找到cpu的负载和io设备的统计，可以同时查看的工具。



## 尝试pidstat



pidstat -d 虽然可以观察到进程的 读写io情况，也可以观察到进程的 cpu 使用情况（cpu的统计中 缺少iowait 指标），无法关联

pidstat的cpu统计指标如下，（没有iowait指标）

```
03:33:44 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
03:33:47 PM     0      1909    0.33    1.00    0.00    1.33    15  qemu-img
03:33:47 PM     0      2087    0.33    0.67    0.00    1.00    21  corosync
03:33:47 PM     0      2141    0.67    0.67    0.00    1.33     9  pvestatd
03:33:47 PM     0      2739    0.67    0.00    0.00    0.67     0  pvedaemon worke
03:33:47 PM    33      3813    0.33    0.00    0.00    0.33    13  pveproxy worker
03:33:47 PM     0      4799    0.33    0.00    0.00    0.33    12  pvedaemon worke
03:33:47 PM     0      5881    0.00   11.67   39.00   50.67    18  kvm
03:33:47 PM     0      5923    0.00    0.33    0.00    0.33     5  vhost-5881
03:33:47 PM     0      5964    0.00    1.33    0.00    1.33    14  vhost-5881
03:33:47 PM    33      7062    0.67    0.00    0.00    0.67     3  pveproxy worker
03:33:47 PM     0      7794    0.00    2.67    0.00    2.67     3  kworker/3:0H
03:33:47 PM    33      8881    0.33    0.00    0.00    0.33     5  pveproxy worker
03:33:47 PM     0      9338    0.33    0.33    0.00    0.67     7  pidstat
03:33:47 PM     0     16120    0.00    0.33    0.00    0.33     3  kworker/3:0
03:33:47 PM     0     19925    0.67   17.67   32.67   51.00     2  kvm
03:33:47 PM     0     19967    0.00    0.67    0.00    0.67     3  vhost-19925

```



## 尝试vmstat

vmstat 可以观察到 b（block的进程，也叫不可中断进程）一直维持在8， 正好符合 cpu的upload的数值。也可以观察到 cpu的iowait数值和 top中的一致。 所以，至此，能够判断 有几个进程在等待资源，但是无法进一步确定是等待什么资源，排查进入 阻塞阶段。

```
root@pveNode38:~# vmstat 5 -w 800
procs -----------------------memory---------------------- ---swap-- -----io---- -system-- --------cpu--------
 r  b         swpd         free         buff        cache   si   so    bi    bo   in   cs  us  sy  id  wa  st
 0 10      3066564      6697668       105684     20778468    1    1     3    77    1    0   4   2  94   0   0
 0  8      3066308      6590440       105688     20885932    8    0     8 11270 13875 47109   5   2  72  20   0

 0  8      3062468      6465872       105696     21008816   30    0   134 18427 13007 41522   4   2  75  19   0
 3  8      3060932      6297536       105704     21177828  154    0   270 13623 13828 41968   4   2  75  19   0
 2  8      3060420      6149920       105708     21315960 1270    0  1374 11906 15004 48566   6   2  72  21   0
 0  8      3059908      6013652       105712     21454104   25    0    26 13512 13357 39642   4   2  68  26   0
 0  8      3059396      5859260       105720     21607708   62    0   166 13208 13164 39697   4   2  71  24   0
 3  8      3057348      5769636       105728     21700124   21    0    28 19522 14054 51206  10   2  66  22   0
 0  8      3056580      5583244       105732     21884320   14    0   118 11832 15110 46003   5   2  72  21   0
 2  8      3055812      5430152       105740     22037812   18    0    21 17146 14748 44715   4   2  73  21   0

```



## dstat

临时抱拂脚，找到 倪朋飞的极客时间专栏，根据前辈倪朋飞的推荐，知道了 dstat工具，直接apt-get安装。

dstat 3

```
----total-cpu-usage---- -dsk/total- -net/total- ---paging-- ---system--
usr sys idl wai hiq siq| read  writ| recv  send|  in   out | int   csw 
  3   1  76  20   0   0|   0   584k|  60M 1623k|   0    28k|8980    22k
  2   1  72  24   0   0|  12k  973k|  72M 1426k|   0    19k|9284    23k
  4   1  68  26   0   0| 173k  557k|  83M 1254k|   0    28k|9809    22k
  3   1  74  22   0   0|   0   671k|  62M 1372k|   0    19k|8749    22k
  2   1  71  25   0   0|2731B  681k|  40M 1112k|1365B   28k|7993    21k
  5   2  73  21   0   0| 891k  443k|  64M 1192k|   0    21k|9582    26k
  4   2  75  19   0   0|  14M 2249k|  78M 1610k|   0    23k|  12k   29k
  3   1  76  19   0   0|5461B   20M|  94M 1133k|   0    44k|  11k   24k
  3   1  69  26   0   0| 291k  559k|  65M 1219k|   0    24k|9614    23k
  3   1  75  20   0   0| 265k  619k|  59M 1202k|   0    21k|9391    23k
  3   1  77  18   0   0| 139k  439k|  39M 1130k|   0     0 |7868    20k
  8   2  73  17   0   0|1365B  565k|  59M 1354k|1365B   24k|9411    28k
  7   1  75  17   0   0|   0   723k|  45M 1058k|   0    23k|9071    24k
  3   1  70  25   0   0| 181k  571k| 111M 3407k|   0    23k|  10k   23k
  2   1  69  28   0   0|1365B  675k|  49M 1114k|   0    25k|8231    20k
  3   1  71  25   0   0| 173k  521k|  44M 1177k|   0     0 |8202    21k
  3   1  74  22   0   0|1365B  717k|  56M 1624k|1365B   20k|9125    26k
  2   1  79  18   0   0|   0   615k|  63M 1169k|   0    20k|8434    17k
  4   1  79  16   0   0| 175k  617k|  24M 1030k|1365B   23k|7639    22k
  2   1  77  19   0   0|2731B   11M|  61M 1090k|   0     0 |8726    21k
  3   1  72  25   0   0|   0   449k|  68M 1168k|   0    20k|8540    20k
  3   1  71  25   0   0|   0   741k|  79M 1345k|   0    49k|  10k   27k
  3   1  69  26   0   0| 173k  372k|  41M 1132k|   0     0 |8674    21k
  4   1  72  23   0   0|1365B  629k|  65M 1230k|1365B   20k|8585    21k

```

至此，可以看到，iowait值 在 20左右的时候，net的 收流量一直有 几十兆，所以怀疑的重心就转移到网络上来。

通过 iftop命令观察，发现最大的流量，是和外部的一个 nas服务器。





## strace



```
ppoll([{fd=3, events=POLLIN}, {fd=5, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 4, NULL, NULL, 8) = 1 ([{fd=7, revents=POLLIN}])
read(7, "\1\0\0\0\0\0\0\0", 512)        = 8
futex(0x7f5f6309e178, FUTEX_WAKE_PRIVATE, 1) = 1
ppoll([{fd=3, events=POLLIN}, {fd=5, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 4, NULL, NULL, 8) = 1 ([{fd=7, revents=POLLIN}])
read(7, "\1\0\0\0\0\0\0\0", 512)        = 8
futex(0x7f5f6309e178, FUTEX_WAKE_PRIVATE, 1) = 1
ppoll([{fd=3, events=POLLIN}, {fd=5, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 4, NULL, NULL, 8^Cstrace: Process 19286 detached
 <detached ...>
root@pveNode38:~# ^C
root@pveNode38:~# strace -p 19286


```

 











# 知识点记录







