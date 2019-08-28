---
layout: post
title: linux捕获内核crash信息(kdump)
category: 技术
---

# 前言

> 本文是在bean的blog基础上修改和测试

kdump 是一种先进的基于 kexec 的内核崩溃转储机制。当系统崩溃时，kdump 使用 kexec 启动到第二个内核。第二个内核通常叫做捕获内核，以很小内存启动以捕获转储镜像。第一个内核保留了内存的一部分给第二内核启动用。由于 kdump 利用 kexec 启动捕获内核，绕过了 BIOS，所以第一个内核的内存得以保留。这是内核崩溃转储的本质。

现在整理下在Ubuntu 14.04上的部署方法。

# 部署

第一步是通过如下指令安装：

``` 
root@46:~# sudo apt-get install linux-crashdump
```

安装过程中，会弹出选择框

注意选择YES。选择完成后，安装过程完毕，该安装会自动修改grub.cfg:

```
linux	/boot/vmlinuz-4.1.49-server root=UUID=cfb17230-a084-4117-a9b8-38bcd48f7c07 ro video=VGA-1:800x600 quiet 915.modeset=0 nomodeset net.ifnames=1 biosdevname=0 nomdmonddf nomdmonisw crashkernel=384M-:128M
```
后面新增 crashkernel=384M -:128M 了部分。

安装好了之后，执行kdump-config show会发现，该功能尚未启用。

```
root@46:~# kdump-config show
Usage: /usr/sbin/kdump-config {help|test|show|status|load|unload|savecore}
root@46:~# kdump-config show
 * /etc/default/kdump-tools: USE_KDUMP is not set or zero
 * no crashkernel= parameter in the kernel cmdline
DUMP_MODE:        kdump
USE_KDUMP:        0
KDUMP_SYSCTL:     kernel.panic_on_oops=1
KDUMP_COREDIR:    /var/crash
crashkernel addr: 
current state:    Not ready to kdump

kexec command:
  no kexec command recorded
```

第二步是要启用kdump，方法是修改/etc/default/kdump-tools 中的USE_KDUMP
```
# kdump-tools configuration
# ---------------------------------------------------------------------------
# USE_KDUMP - controls kdump will be configured
#     0 - kdump kernel will not be loaded
#     1 - kdump kernel will be loaded and kdump is configured
# KDUMP_SYSCTL - controls when a panic occurs, using the sysctl 
#     interface.  The contents of this variable should be the
#     "variable=value ..." portion of the 'sysctl -w ' command.
#     If not set, the default value "kernel.panic_on_oops=1" will
#     be used.  Disable this feature by setting KDUMP_SYSCTL=" "
#     Example - also panic on oom:
#         KDUMP_SYSCTL="kernel.panic_on_oops=1 vm.panic_on_oom=1"
#
USE_KDUMP=0
#KDUMP_SYSCTL="kernel.panic_on_oops=1"
```
将USE_KDUMP=0改成 USE_KDUMP=1。
修改之后，需要重启机器，方能生效。

第三步重启机器，然后检查 kdump-config show：
(换了一台机器展示)
```
root@44:~# kdump-config show 
DUMP_MODE:        kdump
USE_KDUMP:        1
KDUMP_SYSCTL:     kernel.panic_on_oops=1
KDUMP_COREDIR:    /var/crash
crashkernel addr: 0x2d000000
current state:    ready to kdump

kexec command:
  /sbin/kexec -p --command-line="BOOT_IMAGE=/boot/vmlinuz-4.1.49-server root=UUID=3cc9983b-3d4d-413a-a9a5-6edc9e785c3e ro video=VGA-1:800x600 quiet 915.modeset=0 nomodeset net.ifnames=1 biosdevname=0 nomdmonddf nomdmonisw irqpoll maxcpus=1 nousb" --initrd=/boot/initrd.img-4.1.49-server /boot/vmlinuz-4.1.49-server
```

# 测试配置是否有效

如何触发crash，验证能否产生crash文件：
手动触发的话，比较简单：
```
echo c> /proc/sysrq-trigger
```
就会触发crash，搜集资料，机器启动之后会看到如下内容
```
root@44:/var/crash/201808021451# crash  /usr/lib/debug/lib/modules/4.1.49-server/vmlinux  dump.201808021451 

crash 7.0.3
Copyright (C) 2002-2013  Red Hat, Inc.
Copyright (C) 2004, 2005, 2006, 2010  IBM Corporation
Copyright (C) 1999-2006  Hewlett-Packard Co
Copyright (C) 2005, 2006, 2011, 2012  Fujitsu Limited
Copyright (C) 2006, 2007  VA Linux Systems Japan K.K.
Copyright (C) 2005, 2011  NEC Corporation
Copyright (C) 1999, 2002, 2007  Silicon Graphics, Inc.
Copyright (C) 1999, 2000, 2001, 2002  Mission Critical Linux, Inc.
This program is free software, covered by the GNU General Public License,
and you are welcome to change it and/or distribute copies of it under
certain conditions.  Enter "help copying" to see the conditions.
This program has absolutely no warranty.  Enter "help warranty" for details.
 
GNU gdb (GDB) 7.6
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-unknown-linux-gnu"...

      KERNEL: /usr/lib/debug/lib/modules/4.1.49-server/vmlinux
    DUMPFILE: dump.201808021451  [PARTIAL DUMP]
        CPUS: 32
        DATE: Thu Jan  1 08:00:00 1970
      UPTIME: 00:01:35
LOAD AVERAGE: 5.72, 2.41, 0.89
       TASKS: 2109
    NODENAME: 44
     RELEASE: 4.1.49-server
     VERSION: #2 SMP Wed Aug 1 14:31:39 CST 2018
     MACHINE: x86_64  (2099 Mhz)
      MEMORY: 127.9 GB
       PANIC: "Oops: 0002 [#1] SMP " (check log for details)
         PID: 5191
     COMMAND: "bash"
        TASK: ffff8810249be500  [THREAD_INFO: ffff880fd8144000]
         CPU: 1
       STATE: TASK_RUNNING (PANIC)

crash> bt
PID: 5191   TASK: ffff8810249be500  CPU: 1   COMMAND: "bash"
 #0 [ffff880fd8147ab0] machine_kexec at ffffffff81042cef
 #1 [ffff880fd8147b00] crash_kexec at ffffffff810e7a33
 #2 [ffff880fd8147bd0] oops_end at ffffffff810076c8
 #3 [ffff880fd8147c00] no_context at ffffffff8173cf5f
 #4 [ffff880fd8147c60] __bad_area_nosemaphore at ffffffff8173d01e
 #5 [ffff880fd8147cb0] bad_area_nosemaphore at ffffffff8173d18a
 #6 [ffff880fd8147cc0] __do_page_fault at ffffffff8104ce25
 #7 [ffff880fd8147d30] do_page_fault at ffffffff8104d331
 #8 [ffff880fd8147d60] page_fault at ffffffff8174aa62
    [exception RIP: sysrq_handle_crash+22]
    RIP: ffffffff8145e156  RSP: ffff880fd8147e18  RFLAGS: 00010292
    RAX: 000000000000000f  RBX: ffffffff81caa660  RCX: 0000000000000000
    RDX: 0000000000000001  RSI: ffff88103fa4d678  RDI: 0000000000000063
    RBP: ffff880fd8147e18   R8: 00000000000000c2   R9: ffffffff81eafe9c
    R10: 000000000000064a  R11: 0000000000000649  R12: 0000000000000063
    R13: 0000000000000000  R14: 0000000000000003  R15: 0000000000000000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #9 [ffff880fd8147e20] __handle_sysrq at ffffffff8145e927
#10 [ffff880fd8147e50] write_sysrq_trigger at ffffffff8145edd3
#11 [ffff880fd8147e70] proc_reg_write at ffffffff8123107d
#12 [ffff880fd8147ea0] __vfs_write at ffffffff811ca688
#13 [ffff880fd8147eb0] vfs_write at ffffffff811cacd9
#14 [ffff880fd8147f00] sys_write at ffffffff811cbaa6
#15 [ffff880fd8147f50] system_call_fastpath at ffffffff81748f5b
    RIP: 00007f9b22df2390  RSP: 00007fff9441ffd8  RFLAGS: 00000246
    RAX: ffffffffffffffda  RBX: 0000000000000003  RCX: 00007f9b22df2390
    RDX: 0000000000000002  RSI: 0000000001d27408  RDI: 0000000000000001
    RBP: 000000000000000a   R8: 000000000000000a   R9: 00007f9b2370c740
    R10: 00007f9b230c46a0  R11: 0000000000000246  R12: 00000000004b8a97
    R13: 0000000000000073  R14: 0000000000000030  R15: 0000000001efdb40
    ORIG_RAX: 0000000000000001  CS: 0033  SS: 002b
crash> 

```

凑巧的时，我们产品恰好发生了crash：

当搜集信息完成之后，会自动重启，重启之后，可以在/var/crash下看到对应的信息：

```bash
root@44:~# ll /var/crash
total 56
drwxrwxrwt  4 root root  4096 Aug  2 15:53 ./
drwxr-xr-x 16 root root  4096 Aug  2 14:43 ../
drwxr-xr-x  2 root root  4096 Aug  2 14:54 201808021451/
drwxr-xr-x  2 root root  4096 Aug  2 15:56 201808021553/
-rw-r--r--  1 root root   313 Aug  2 15:58 kexec_cmd
-rw-r-----  1 root root 36332 Aug  2 14:56 linux-image-4.1.49-server-201808021451.crash

root@44:/var/crash/201808021553# ll
total 1434560
drwxr-xr-x 2 root root       4096 Aug  2 15:56 ./
drwxrwxrwt 4 root root       4096 Aug  2 15:53 ../
-rw------- 1 root root     240194 Aug  2 15:56 dmesg.201808021553
-rw------- 1 root root 1468733533 Aug  2 15:56 dump.201808021553
```

也可以 通过dmesg 判断 是否加载了 crash kernel

```

```





# 分析crash

要分析crash文件，需要安装内核符号表：

```
root@44:~# dpkg -l |grep linux-image
ii  linux-image-4.1.49-server            201808011426~cf7c219                 amd64        Linux kernel binary image for version 4.1.49-server
ii  linux-image-4.1.49-server-dbg        201808011426~cf7c219                 amd64        Linux kernel debug image for version 4.1.49-server
```

安装后，使用如下指令开始分析crash

```
crash  /usr/lib/debug/lib/modules/4.1.49-server/vmlinux dump.201808021553
```

对于本次的crash，可以查出是由于scst引起的，如下所示：

```
root@44:/var/crash/201808021553# crash  /usr/lib/debug/lib/modules/4.1.49-server/vmlinux dump.201808021553 

crash 7.0.3
Copyright (C) 2002-2013  Red Hat, Inc.
Copyright (C) 2004, 2005, 2006, 2010  IBM Corporation
Copyright (C) 1999-2006  Hewlett-Packard Co
Copyright (C) 2005, 2006, 2011, 2012  Fujitsu Limited
Copyright (C) 2006, 2007  VA Linux Systems Japan K.K.
Copyright (C) 2005, 2011  NEC Corporation
Copyright (C) 1999, 2002, 2007  Silicon Graphics, Inc.
Copyright (C) 1999, 2000, 2001, 2002  Mission Critical Linux, Inc.
This program is free software, covered by the GNU General Public License,
and you are welcome to change it and/or distribute copies of it under
certain conditions.  Enter "help copying" to see the conditions.
This program has absolutely no warranty.  Enter "help warranty" for details.
 
GNU gdb (GDB) 7.6
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-unknown-linux-gnu"...

      KERNEL: /usr/lib/debug/lib/modules/4.1.49-server/vmlinux
    DUMPFILE: dump.201808021553  [PARTIAL DUMP]
        CPUS: 32
        DATE: Thu Jan  1 08:00:00 1970
      UPTIME: 00:55:56
LOAD AVERAGE: 23.89, 6.95, 4.06
       TASKS: 2485
    NODENAME: 44
     RELEASE: 4.1.49-server
     VERSION: #2 SMP Wed Aug 1 14:31:39 CST 2018
     MACHINE: x86_64  (2099 Mhz)
      MEMORY: 127.9 GB
       PANIC: "Oops: 0000 [#1] SMP " (check log for details)
         PID: 133396
     COMMAND: "scstd30"
        TASK: ffff882036574bc0  [THREAD_INFO: ffff881c1253c000]
         CPU: 14
       STATE: TASK_RUNNING (PANIC)

crash> 
crash> bt
PID: 133396  TASK: ffff882036574bc0  CPU: 14  COMMAND: "scstd30"
 #0 [ffff881c1253f970] machine_kexec at ffffffff81042cef
 #1 [ffff881c1253f9c0] crash_kexec at ffffffff810e7a33
 #2 [ffff881c1253fa90] oops_end at ffffffff810076c8
 #3 [ffff881c1253fac0] no_context at ffffffff8173cf5f
 #4 [ffff881c1253fb20] __bad_area_nosemaphore at ffffffff8173d01e
 #5 [ffff881c1253fb70] bad_area_nosemaphore at ffffffff8173d18a
 #6 [ffff881c1253fb80] __do_page_fault at ffffffff8104ce25
 #7 [ffff881c1253fbf0] do_page_fault at ffffffff8104d331
 #8 [ffff881c1253fc20] page_fault at ffffffff8174aa62
    [exception RIP: scst_update_lat_stats+224]
    RIP: ffffffffa0c2d4f0  RSP: ffff881c1253fcd8  RFLAGS: 00010206
    RAX: 0000000000000048  RBX: ffff881cf3ed0000  RCX: 0000000000000010
    RDX: 0000000000000000  RSI: 0000000000000000  RDI: 0000000000000000
    RBP: ffff881c1253fd58   R8: 0000000000000000   R9: 0000000000000092
    R10: 0000000000000000  R11: 0000000000000010  R12: 0000000000000000
    R13: ffff881db84e7640  R14: ffff881cf3ed0490  R15: 0000000000000092
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #9 [ffff881c1253fd60] scst_finish_cmd at ffffffffa0c0bc68 [scst]
#10 [ffff881c1253fdb0] scst_process_active_cmd at ffffffffa0c0e415 [scst]
#11 [ffff881c1253fdf0] scst_do_job_active at ffffffffa0c0fe10 [scst]
#12 [ffff881c1253fe40] scst_cmd_thread at ffffffffa0c100cf [scst]
#13 [ffff881c1253fec0] kthread at ffffffff8107c749
#14 [ffff881c1253ff50] ret_from_fork at ffffffff81749362
crash> 

```

通过反编译 scst.ko文件：

```
objdump -S　scst.ko |tee /tmp/scst.ko.obj
```

我们打开该反编译的文件：

```
0000000000034410 <scst_update_lat_stats>:

void scst_update_lat_stats(struct scst_cmd *cmd)
{
   34410:       e8 00 00 00 00          callq  34415 <scst_update_lat_stats+0x5>
   34415:       55                      push   %rbp
   34416:       48 89 e5                mov    %rsp,%rbp
   34419:       41 57                   push   %r15
   3441b:       41 56                   push   %r14
   3441d:       41 55                   push   %r13
   3441f:       49 89 fd                mov    %rdi,%r13
   34422:       41 54                   push   %r12
   34424:       53                      push   %rbx
   34425:       48 83 ec 58             sub    $0x58,%rsp
        uint64_t finish, scst_time, tgt_time, dev_time, total_time;
        struct scst_session *sess = cmd->sess;

```

注意这一行：

````
    [exception RIP: scst_update_lat_stats+224]
````

我们可以计算crash的代码位置：

```bash
0x0000000000034410 + 224(十进制) = 344F0
```

去反编译的文件中寻找对应的未知：

```
#ifdef CONFIG_SCST_MEASURE_LATENCY
        if (cmd->dev->vdev_update_lat_stats) {
   344f0:       4c 8b 97 40 02 00 00    mov    0x240(%rdi),%r10
        else
                dev_latency_stat = NULL;
                
   
```

原因是：cmd->dev在更新统计新的时候已经变成了NULL，这和之前的分析对应。

```
10925 #ifdef CONFIG_SCST_MEASURE_LATENCY
10926         if (cmd->dev->vdev_update_lat_stats) {
10927                 cmd->dev->vdev_update_lat_stats(cmd->dev, cmd->data_direction,
10928                                                 total_time);  
10929         }
10930 #endif
```







# ubuntu 12 上测试



```
root@node1:~# kdump-config show
DUMP_MODE:        kdump
USE_KDUMP:        1
KDUMP_SYSCTL:     kernel.panic_on_oops=1
KDUMP_COREDIR:    /var/crash
crashkernel addr: 0x2d000000
current state:    ready to kdump

kexec command:
  /sbin/kexec -p --command-line="BOOT_IMAGE=/boot/vmlinuz-4.1.49-server root=UUID=8bd2bcd2-fca8-4c56-8fd7-07b9b65a2ee4 ro video=VGA-1:800x600 quiet 915.modeset=0 nomodeset net.ifnames=1 biosdevname=0 nomdmonddf nomdmonisw irqpoll maxcpus=1 nousb" --initrd=/boot/initrd.img-4.1.49-server /boot/vmlinuz-4.1.49-server
root@node1:~# 



##ubuntu 12 需要 单独 安装 kdump-tool

root@wsg-6:~# kdump-config show
USE_KDUMP:        1
KDUMP_SYSCTL:     kernel.panic_on_oops=1
KDUMP_COREDIR:    /var/crash
crashkernel addr: 0x2e000000
current state:    ready to kdump

kernel link: 
  

kexec command:
  /sbin/kexec -p --command-line="BOOT_IMAGE=/boot/vmlinuz-4.1.35-server root=UUID=715c78f8-3fbc-40df-bc53-632c752cdb9d ro video=VGA-1:800x600 quiet 915.modeset=0 nomodeset irqpoll maxcpus=1 nousb" --initrd=/boot/initrd.img-4.1.35-server /boot/vmlinuz-4.1.35-server

```





##kdump 与kexec



```
Kexec 是实现 kdump 机制的关键，它包括 2 个组成部分：一是内核空间的系统调用 kexec_load，负责在生产内核（production kernel 或 first kernel）启动时将捕获内核（capture kernel 或 sencond kernel）加载到指定地址。二是用户空间的工具 kexec-tools，他将捕获内核的地址传递给生产内核，从而在系统崩溃的时候能够找到捕获内核的地址并运行。没有 kexec 就没有 kdump。先有 kexec 实现了在一个内核中可以启动另一个内核，才让 kdump 有了用武之地。kexec 原来的目的是为了节省 kernel 开发人员重启系统的时间，谁能想到这个“偷懒”的技术却孕育了最成功的内存转存机制呢？

Kdump 的概念出现在 2005 左右，是迄今为止最可靠的内核转存机制，已经被主要的 linux™ 厂商选用。kdump 是一种先进的基于 kexec 的内核崩溃转储机制。当系统崩溃时，kdump 使用 kexec 启动到第二个内核。第二个内核通常叫做捕获内核，以很小内存启动以捕获转储镜像。第一个内核保留了内存的一部分给第二内核启动用。由于 kdump 利用 kexec 启动捕获内核，绕过了 BIOS，所以第一个内核的内存得以保留。这是内核崩溃转储的本质。

kdump 需要两个不同目的的内核，生产内核和捕获内核。生产内核是捕获内核服务的对像。捕获内核会在生产内核崩溃时启动起来，与相应的 ramdisk 一起组建一个微环境，用以对生产内核下的内存进行收集和转存。
```



关于 grub.cfg 中的 craskerner 配置

> ubuntu12 默认的 grub.cfg 中crashkerner 大小，会 导致 转存时 系统 OOM 卡死，放大 到 256M后就不会卡死了。
>
> dmesg |grep -i crash 可以看到 系统 启动时 ，预留的 转存 kerner 的大小。
>
> 

| **ram size** | **crashkernel parameter** | **ram / crashkernel factor** |
| ------------ | ------------------------- | ---------------------------- |
| >0GB         | 128MB                     | 15                           |
| >2GB         | 256MB                     | 23                           |
| >6GB         | 512MB                     | 15                           |
| >8GB         | 768MB                     | 31                           |

```
crashkernel=0M-2G:128M,2G-6G:256M,6G-8G:512M,8G-:768M
```





# 遗留问题

> centos的情况需要测试。。
>
> 

```
默认的 crashkerner 
? kexec 的权限 ，
6.1版本 看起来 每次 模拟 crash 后，每个进程 会 记录 自己的crash，和 ubuntu14 不一样。

6.1 kdump-config show 中 显示 的不太一样，DUMP_MODE 选项没有， 尝试 build 安装 kdump？？




root@wsg-6:/var/crash# ll
total 464
drwxrwsrwt  2 root    whoopsie   4096 Nov 26 17:55 ./
drwxr-xr-x 14 root    root       4096 Nov  7 13:44 ../
-rw-r--r--  1 root    whoopsie    263 Nov 26 17:54 kexec_cmd
-rw-rw----  1 root    whoopsie      0 Nov 26 12:09 .lock
-rw-r-----  1 ganglia whoopsie      0 Nov 26 17:54 _usr_sbin_gmond.114.crash
-rw-r-----  1 root    whoopsie 219754 Nov 26 17:56 _usr_sbin_nmbd.0.crash
-rw-r-----  1 root    whoopsie 239526 Nov 26 13:47 _usr_sbin_winbindd.0.crash

ubuntu14 版本，crash
root@node1:/var/crash# ll
total 48
drwxrwxrwt  3 root root  4096 Nov 27 10:54 ./
drwxr-xr-x 16 root root  4096 Nov 26 10:29 ../
drwxr-xr-x  2 root root  4096 Nov 27 10:55 201811261148/
-rw-r--r--  1 root root   313 Nov 26 11:49 kexec_cmd
-rw-r-----  1 root root 30826 Nov 26 11:49 linux-image-4.1.49-server-201811261148.crash
root@node1:/var/crash# ezs3-version 
v7.0-473 (88d823fa3)


```



