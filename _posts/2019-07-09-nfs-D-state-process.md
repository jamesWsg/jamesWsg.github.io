---
layout: post
title: nfs进程变D故障处理
category: 技术
---



# 问题现象

客户端是opentack，通过nfs挂载我们存储，发现有个服务进程变D了，长时间无法恢复，如下

    [root@ECM-043 ~]# ps aux|grep nova-compute
    nova     21409  1.4  0.0 2117692 110916 ?      Dl   15:06   1:24 /opt/server/python27/bin/python /usr/bin/nova-compute --logfile /var/log/nova/compute.log

#排查步骤

##客户端排查，查找该进程hang 在哪里，lsof 查找进程打开的文件


    lsof -p 21409 
    nova-comp 21409 nova   22w   REG               0,29        0 2199054015793 /var/lib/nova/instances/locks/nova-storage-registry-lock (block.beijing.wocloud.cn:/var/share/ezfs/shareroot/block-bj)
    可以看到客户端的进程打开了nfs上的nova-storage-registry-lock文件，下一步在服务端看下该文件的状态

##服务端查看nova-storage-registry-lock 文件的inode号

知道了inode号，在所有的nfs服务端，看下是否有其他进程 拿着 该lock 文件

> /proc/locks 各字段的含义见后面的延伸章节

    root@Storage5:~# cat /proc/locks |grep inode
    1: POSIX  ADVISORY  WRITE 28331 00:14:2199053494291 0 EOF
    2: POSIX  ADVISORY  WRITE 28165 00:14:1099511627783 0 0
    3: POSIX  ADVISORY  READ  360769 00:0f:21590 4 4
    4: POSIX  ADVISORY  WRITE 28331 00:14:2199054015793 0 EOF
    5: POSIX  ADVISORY  WRITE 1022761 00:0f:673303453 0 EOF
    6: POSIX  ADVISORY  WRITE 98842 00:0f:47181 0 EOF
    7: POSIX  ADVISORY  WRITE 688677 00:0f:1186403778 0 EOF
    8: POSIX  ADVISORY  READ  38754 00:0f:21590 4 4

## 服务端 确认哪些 nfs客户端连接
nfs 是无状态协议，和cifs不一样，不是TCP 吗？

    root@Storage5:/var/lib/nfs/sm# ll
    total 48
    drwxr-xr-x 2 statd root 4096 Apr 13 09:03 ./
    drwxr-xr-x 5 statd root 4096 Apr 10 18:21 ../
    -rw------- 1 statd root   88 Apr 11 16:08 10.55.4.1
    -rw------- 1 statd root   89 Apr 10 14:47 10.55.4.15
    -rw------- 1 statd root   89 Apr 10 14:41 10.55.4.16
    -rw------- 1 statd root   90 Apr  7 02:10 10.55.4.199
    -rw-r----- 1 statd root 1350 Apr 11 13:42 10.55.4.205
    -rw------- 1 statd root   89 Apr 11 16:00 10.55.4.31
    -rw------- 1 statd root   89 Apr 10 14:44 10.55.4.32
    -rw------- 1 statd root   89 Apr 10 14:34 10.55.4.37
    -rw------- 1 statd root   89 Apr 11 16:04 10.55.4.54
    -rw-r----- 1 statd root 2288 Apr 13 09:03 10.55.4.9
    root@Storage5:/var/lib/nfs/sm# pwd
    /var/lib/nfs/sm

# 总结

会有多个客户端可能会同时访问这个所文件，存储端的ceph版本为0.67的版本，mds在并发访问控制上并不能处理所有异常。所以只能用一个临时的快速恢复的方法来处理，即删掉该 lock 文件，让客户端重新生成该lock文件。



# 延伸


linux下lock类型

## 文件锁，主要分为 flock 和fcntl
2者的粒度不一样
###flock
锁住整个文件，如下，

    root@scal61:/usr/share/pyshared/ezs3# cat /proc/locks 
    1: FLOCK  ADVISORY  WRITE 97862 00:12:229635 0 EOF
    2: FLOCK(lock类型）  ADVISORY（建议锁，非强制执行）  WRITE（holder能写lock文件） 4246（holder的pid） 00:12:20704（锁文件的MAJOR-DEVICE:MINOR-DEVICE:INODE-NUMBER） 0 EOF（锁文件的范围，0到EOF代表整个文件）

###fcntl
lock文件的某一部分，如下 posix

    root@scal61:/usr/share/pyshared/ezs3# cat /proc/locks 
    3: POSIX  ADVISORY  WRITE 97830 08:03:524404 0 EOF
    4: POSIX  ADVISORY  WRITE 4056 00:12:41028 0 EOF
    5: POSIX  ADVISORY  READ  2351 00:12:10095 4 4
    6: POSIX  ADVISORY  READ  2783 00:12:10052 4 4
    7: POSIX  ADVISORY  READ  2783 00:12:14432 4 4
    8: POSIX  ADVISORY  WRITE 2783 00:12:10050 0 0

----------
*** centos

 - FLOCK signifying the older-style UNIX file locks from a flock system
   call       
 - POSIX representing the newer POSIX locks from the lockf system call.
- ADVISORY means that the lock does not prevent other people from accessing the data; it only prevents other attempts to lock it
- MANDATORY means that no other access to the data is permitted while the lock is held

- The fourth column reveals whether the lock is allowing the holder READ or WRITE access to the file






## lsof 命令



```
lsof abc.txt 显示开启文件abc.txt的进程
lsof -i :22 知道22端口现在运行什么程序
lsof -c abc 显示abc进程现在打开的文件
lsof -g gid 显示归属gid的进程情况
lsof +d /usr/local/ 显示目录下被进程开启的文件
lsof +D /usr/local/ 同上，但是会搜索目录下的目录，时间较长
lsof -d 4 显示使用fd为4的进程
lsof -i 用以显示符合条件的进程情况

```



lsof 各字段

```
COMMAND     PID  TID       USER   FD      TYPE             DEVICE SIZE/OFF       NODE NAME
init          1            root  mem       REG                8,1   134296     529350 /lib/x86_64-linux-gnu/libselinux.so.1
init          1            root  mem       REG                8,1   281552     529234 /lib/x86_64-linux-gnu/libdbus-1.so.3.7.6

node : 文件的inode
```

命令示例

````
lsof -p 5527
ceph-osd 5527 root *627w   REG              252,0    282448   4196063 /data/cache/g-osd-0/current/18.137_head/omap/000471.log
root@converger-124:~# 


root@converger-124:/proc/5527/fd# ll |grep /data/cache/g-osd-0/current/18.137_head/omap/000471.log
l-wx------ 1 root root 64 Jul  5 16:47 17627 -> /data/cache/g-osd-0/current/18.137_head/omap/000471.log
root@converger-124:/proc/5527/fd# 

列出某个文件 被哪些进程在用
root@converger-124:/proc/5527/fd# lsof /data/cache/g-osd-0/current/18.137_head/omap/000471.log
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF    NODE NAME
ceph-osd 5527 root *627w   REG  252,0   292839 4196063 /data/cache/g-osd-0/current/18.137_head/omap/000471.log
root@converger-124:/proc/5527/fd# 


root@converger-124:/proc/5527/fd# stat /data/cache/g-osd-0/current/18.137_head/omap/000471.log
  File: ‘/data/cache/g-osd-0/current/18.137_head/omap/000471.log’
  Size: 294280    	Blocks: 584        IO Block: 4096   regular file
Device: fc00h/64512d	Inode: 4196063     Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2019-07-05 10:18:30.376381600 +0800
Modify: 2019-07-05 17:02:33.030906129 +0800
Change: 2019-07-05 17:02:33.030906129 +0800
 Birth: -
root@converger-124:/proc/5527/fd# 


````

><http://www.361way.com/lsof/176.html>

##/proc/目录和ps 命令

lsof 命令有时候输出很慢，更快的方式就是 通过/proc 目录去快速 查找 进程的相关信息。ps 命令的输出也是通过读取/proc目录

```
/proc/5527/fd
fd目录列出了进程打开的文件
task目录列出了进程的子进程

```



### ps 知识点记录

```
ps参数比较多，根据需要 显示进程的信息。比如进程的启动时间，运行时间，状态，

# 参数
参数 -e 显示所有进程信息，-o 参数控制输出。Pid,User 和 Args参数显示PID，运行应用的用户和该应用。
-u 显示用户进程

ps -efww   //david 常用
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0  2016 ?        00:00:02 /sbin/init
root         2     0  0  2016 ?        00:00:00 [kthreadd]

ps -aux //bean    
        USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
        root         1  0.0  0.0  24588  1224 ?        Ss    2016   0:02 /sbin/init
        root         2  0.0  0.0      0     0 ?        S     2016   0:00 [kthreadd]
            
ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0  2016 ?        00:00:02 /sbin/init
root         2     0  0  2016 ?        00:00:00 [kthreadd]
root         3     2  0  2016 ?        00:34:18 [ksoftirqd/0]

ps -eF
UID        PID  PPID  C    SZ   RSS PSR STIME TTY          TIME CMD
root         1     0  0  6147  1224  11  2016 ?        00:00:02 /sbin/init
root         2     0  0     0     0  10  2016 ?        00:00:00 [kthreadd]
root         3     2  0     0     0   0  2016 ?        00:34:18 [ksoftirqd/0]
root         5     2  0     0     0   0  2016 ?        00:00:00 [kworker/0:0H]



PROCESS STATE CODES
       Here are the different values that the s, stat and state output specifiers (header "STAT" or "S") will display to describe the state of a process:
       D    uninterruptible sleep (usually IO)
       R    running or runnable (on run queue)
       S    interruptible sleep (waiting for an event to complete)
       T    stopped, either by a job control signal or because it is being traced.
       W    paging (not valid since the 2.6.xx kernel)
       X    dead (should never be seen)
       Z    defunct ("zombie") process, terminated but not reaped by its parent.

       For BSD formats and when the stat keyword is used, additional characters may be displayed:
       <    high-priority (not nice to other users)
       N    low-priority (nice to other users)
       L    has pages locked into memory (for real-time and custom IO)
       s    is a session leader
       l    is multi-threaded (using CLONE_THREAD, like NPTL pthreads do)
       +    is in the foreground process group.
  

# 查看进程的线程
    ps -eLf  #L要大写，其中LWP ，light weighted process，代表thread
    
    UID          PID    PPID     LWP  C NLWP STIME TTY          TIME CMD
    root           1       0       1  0    1 Oct30 ?        00:08:28 /sbin/init
    root           2       0       2  0    1 Oct30 ?        00:03:56 [kthreadd]
    root           3       2       3  0    1 Oct30 ?        00:20:07 [ksoftirqd/0]
    root           5       2       5  0    1 Oct30 ?        00:00:00 [kworker/0:0H]
    root           7       2       7  0    1 Oct30 ?        00:55:00 [rcu_sched]
    root           8       2       8  0    1 Oct30 ?        00:00:00 [rcu_bh]
    root           9       2       9  0    1 Oct30 ?        00:07:19 [migration/0]
    root          10       2      10  0    1 Oct30 ?        00:00:04 [watchdog/0]
   
#查看 进程的 运行时间
    root@BTDISK-12:~# ps -C atop -o etime,pid,cmd
        ELAPSED     PID CMD
          00:29  654009 /usr/bin/atop -a -w /var/log/atop.log 600
```



















​    
