---
layout: post
title: linux kernel ko build for ceph krbd
category: 技术
---


# 前言

近期和中兴通讯合作超融合，对方使用CentOS 7.2，用OpenStack+KVM和我们的RBD对接，但是ZTE不想使用用户态对接的方法，因此，直接装ceph client RPM，让用户使用librbd对接的方式就不行了，客户需将rbd map成块设备，在内核态直接操作rbd。这就牵扯到对应内核模块的重编译。

ceph相关的内核模块有:

* libceph.ko   对应内核代码 net/ceph ，主要是通信层代码
* ceph.ko       对应内核代码 fs/ceph ，主要是cephfs文件系统的代码
* rbd.ko          对应内核代码为 drivers/block下的rbd.c和rbd_types.c

这三个ko文件需要重新编译。使之能够于我们的Ceph集群对接。

# 编译方法

## 下载centos 的SRC RPM



```
##install dep
wget http://vault.centos.org/7.6.1810/os/Source/SPackages/kernel-3.10.0-957.el7.src.rpm

```

安装rpm后

```
[root@localhost rpmbuild]# ll
total 4
drwxr-xr-x. 3 root root   35 Jul 18 01:18 BUILD
drwxr-xr-x. 2 root root    6 Jul 17 23:33 BUILDROOT
drwxr-xr-x. 2 root root    6 Jul 17 23:33 RPMS
drwxr-xr-x. 2 root root 4096 Jul 17 23:30 SOURCES
drwxr-xr-x. 2 root root   25 Jul 17 23:30 SPECS
drwxr-xr-x. 2 root root    6 Jul 17 23:33 SRPMS
[root@localhost rpmbuild]# pwd
/root/rpmbuild
[root@localhost rpmbuild]# 

```



执行 SPECS中的kernel.spec，会检查相关依赖包，需要根据提示把 缺少的包装上

> 如果rpmbuild命令没有，需要 yum install rpm-build

```
cd ~/rpmbuild/SPECS
rpmbuild -bp kernel.spec

该命令中会把压缩的kernel源代码解压到相应目录，效果如下：
[root@localhost linux-3.10.0-862.el7.x86_64]# pwd
/root/rpmbuild/BUILD/kernel-3.10.0-862.el7/linux-3.10.0-862.el7.x86_64

[root@localhost linux-3.10.0-862.el7.x86_64]# ll
total 568
drwxr-xr-x.  32 root root   4096 Jul 18 01:18 arch
drwxr-xr-x.   3 root root   4096 Mar 21  2018 block
drwxr-xr-x.   2 root root     82 Jul 18 01:18 configs
-rw-r--r--.   1 root root  18693 Mar 21  2018 COPYING
-rw-r--r--.   1 root root  95409 Mar 21  2018 CREDITS
drwxr-xr-x.   4 root root   4096 Jul 18 01:18 crypto
drwxr-xr-x. 107 root root   8192 Jul 18 01:18 Documentation
drwxr-xr-x. 120 root root   4096 Mar 21  2018 drivers
drwxr-xr-x.  36 root root   4096 Jul 18 01:18 firmware
drwxr-xr-x.  75 root root   4096 Mar 21  2018 fs
drwxr-xr-x.  27 root root   4096 Jul 18 01:18 include
drwxr-xr-x.   2 root root    254 Mar 21  2018 init
drwxr-xr-x.   2 root root    256 Mar 21  2018 ipc
-rw-r--r--.   1 root root   2536 Mar 21  2018 Kbuild
-rw-r--r--.   1 root root    505 Mar 21  2018 Kconfig
drwxr-xr-x.  12 root root   8192 Jul 18 01:18 kernel
drwxr-xr-x.  10 root root   8192 Jul 18 01:18 lib
-rw-r--r--.   1 root root 274130 Mar 21  2018 MAINTAINERS
-rw-r--r--.   1 root root  51181 Mar 21  2018 Makefile
-rw-r--r--.   1 root root   2305 Mar 21  2018 Makefile.qlock
drwxr-xr-x.   2 root root   4096 Mar 21  2018 mm
drwxr-xr-x.  60 root root   4096 Mar 21  2018 net
-rw-r--r--.   1 root root  18736 Mar 21  2018 README
-rw-r--r--.   1 root root   7485 Mar 21  2018 REPORTING-BUGS
drwxr-xr-x.  14 root root    220 Mar 21  2018 samples
drwxr-xr-x.  13 root root   4096 Jul 18 01:18 scripts
drwxr-xr-x.   9 root root   4096 Mar 21  2018 security
drwxr-xr-x.  24 root root   4096 Mar 21  2018 sound
drwxr-xr-x.  21 root root    270 Mar 21  2018 tools
drwxr-xr-x.   2 root root     84 Jul 18 01:18 usr
drwxr-xr-x.   4 root root     44 Mar 21  2018 virt
[root@localhost linux-3.10.0-862.el7.x86_64]# 


```



## 下载对应内核的devel包

> 该包建议不要用yum 方式安装，yum 源中只保留了最新版本的devel包，要单独下载rpm安装，下载url如下
>
> https://buildlogs.centos.org/c7.1708.00/kernel/20170822030048/3.10.0-693.el7.x86_64/

```
[root@localhost ~]# ll
total 110640
-rw-r--r--. 1 root root 15065444 Jul 17 23:12 kernel-devel-3.10.0-862.el7.x86_64.rpm
drwxr-xr-x. 8 root root       89 Jul 17 23:33 rpmbuild

[root@localhost ~]# rpm -qa |grep kernel
kernel-3.10.0-862.el7.x86_64
kernel-tools-libs-3.10.0-957.21.3.el7.x86_64
kernel-3.10.0-957.21.3.el7.x86_64
kernel-tools-3.10.0-957.21.3.el7.x86_64
kernel-headers-3.10.0-957.21.3.el7.x86_64
kernel-devel-3.10.0-862.el7.x86_64
[root@localhost ~]# 


安装完devel包，会在/usr/src/kernels生产对应内核版本的目录，其中有关键的Module.symvers，后面会用

[root@localhost ~]# ll /usr/src/kernels/3.10.0-862.el7.x86_64/
total 4496
drwxr-xr-x.  32 root root    4096 Jul 24 01:50 arch
drwxr-xr-x.   3 root root      78 Jul 24 01:50 block
drwxr-xr-x.   4 root root      76 Jul 24 01:50 crypto
drwxr-xr-x. 119 root root    4096 Jul 24 01:50 drivers
drwxr-xr-x.   2 root root      22 Jul 24 01:50 firmware
drwxr-xr-x.  75 root root    4096 Jul 24 01:50 fs
drwxr-xr-x.  28 root root    4096 Jul 24 01:51 include
drwxr-xr-x.   2 root root      37 Jul 24 01:51 init
drwxr-xr-x.   2 root root      22 Jul 24 01:51 ipc
-rw-r--r--.   1 root root     505 Apr 12  2018 Kconfig
drwxr-xr-x.  12 root root     236 Jul 24 01:51 kernel
drwxr-xr-x.  10 root root     219 Jul 24 01:51 lib
-rw-r--r--.   1 root root   51197 Apr 12  2018 Makefile
-rw-r--r--.   1 root root    2305 Apr 12  2018 Makefile.qlock
drwxr-xr-x.   2 root root      58 Jul 24 01:51 mm
-rw-r--r--.   1 root root 1093137 Apr 12  2018 Module.symvers
drwxr-xr-x.  60 root root    4096 Jul 24 01:51 net
drwxr-xr-x.  14 root root     220 Jul 24 01:51 samples
drwxr-xr-x.  13 root root    4096 Jul 24 01:51 scripts
drwxr-xr-x.   9 root root     136 Jul 24 01:51 security
drwxr-xr-x.  24 root root    4096 Jul 24 01:51 sound
-rw-r--r--.   1 root root 3409143 Apr 12  2018 System.map
drwxr-xr-x.  17 root root     221 Jul 24 01:51 tools
drwxr-xr-x.   2 root root      37 Jul 24 01:51 usr
drwxr-xr-x.   4 root root      44 Jul 24 01:51 virt
-rw-r--r--.   1 root root      41 Apr 12  2018 vmlinux.id


```



## patch源码

主要改动在网络层，如下所示：

```c
--- kernel-3.10.0-327.el7/linux-3.10.0-327.el7.centos.x86_64/include/linux/ceph/osdmap.h	2015-10-30 04:56:51.000000000 +0800
+++ kernel-3.10.0-327.el7-patch/linux-3.10.0-327.el7.centos.x86_64/include/linux/ceph/osdmap.h	2017-10-11 15:31:40.928820886 +0800
@@ -95,6 +95,7 @@ struct ceph_osdmap {
 	u32 max_osd;       /* size of osd_state, _offload, _addr arrays */
 	u8 *osd_state;     /* CEPH_OSD_* */
 	u32 *osd_weight;   /* 0 = failed, 0x10000 = 100% normal */
+	u32 *recovery_weight;   /* 0 = failed, 0x10000 = 100% normal */
 	struct ceph_entity_addr *osd_addr;
 
 	struct rb_root pg_temp;
--- kernel-3.10.0-327.el7/linux-3.10.0-327.el7.centos.x86_64/net/ceph/osdmap.c	2015-10-30 04:56:51.000000000 +0800
+++ kernel-3.10.0-327.el7-patch/linux-3.10.0-327.el7.centos.x86_64/net/ceph/osdmap.c	2017-10-11 15:31:12.296822923 +0800
@@ -678,6 +678,7 @@ void ceph_osdmap_destroy(struct ceph_osd
 	}
 	kfree(map->osd_state);
 	kfree(map->osd_weight);
+	kfree(map->recovery_weight);
 	kfree(map->osd_addr);
 	kfree(map->osd_primary_affinity);
 	kfree(map);
@@ -691,7 +692,7 @@ void ceph_osdmap_destroy(struct ceph_osd
 static int osdmap_set_max_osd(struct ceph_osdmap *map, int max)
 {
 	u8 *state;
-	u32 *weight;
+	u32 *weight, *rweight;
 	struct ceph_entity_addr *addr;
 	int i;
 
@@ -705,6 +706,11 @@ static int osdmap_set_max_osd(struct cep
 		return -ENOMEM;
 	map->osd_weight = weight;
 
+	rweight = krealloc(map->recovery_weight, max*sizeof(*rweight), GFP_NOFS);
+	if (!rweight)
+		return -ENOMEM;
+	map->recovery_weight = rweight;
+
 	addr = krealloc(map->osd_addr, max*sizeof(*addr), GFP_NOFS);
 	if (!addr)
 		return -ENOMEM;
@@ -713,6 +719,7 @@ static int osdmap_set_max_osd(struct cep
 	for (i = map->max_osd; i < max; i++) {
 		map->osd_state[i] = 0;
 		map->osd_weight[i] = CEPH_OSD_OUT;
+		map->recovery_weight[i] = CEPH_OSD_IN;
 		memset(map->osd_addr + i, 0, sizeof(*map->osd_addr));
 	}
 
@@ -735,7 +742,7 @@ static int osdmap_set_max_osd(struct cep
 	return 0;
 }
 
-#define OSDMAP_WRAPPER_COMPAT_VER	7
+#define OSDMAP_WRAPPER_COMPAT_VER	8
 #define OSDMAP_CLIENT_DATA_COMPAT_VER	1
 
 /*
      
      
      
@@ -1096,6 +1103,7 @@ static int osdmap_decode(void **p, void
 	/* osd_state, osd_weight, osd_addrs->client_addr */
                                                                                         
 	ceph_decode_need(p, end, 3*sizeof(u32) +
 			 map->max_osd*(1 + sizeof(*map->osd_weight) +
+				       sizeof(*map->recovery_weight) +
 				       sizeof(*map->osd_addr)), e_inval);
 
 	if (ceph_decode_32(p) != map->max_osd)
@@ -1112,6 +1120,12 @@ static int osdmap_decode(void **p, void
 	if (ceph_decode_32(p) != map->max_osd)
 		goto e_inval;
 
+	for (i = 0; i < map->max_osd; i++)
+		map->recovery_weight[i] = ceph_decode_32(p);
+
+	if (ceph_decode_32(p) != map->max_osd)
+		goto e_inval;
+
 	ceph_decode_copy(p, map->osd_addr, map->max_osd*sizeof(*map->osd_addr));
 	for (i = 0; i < map->max_osd; i++)
 		ceph_decode_addr(&map->osd_addr[i]);
@@ -1334,6 +1348,20 @@ struct ceph_osdmap *osdmap_apply_increme    ？？ 这个 7.6 和7.5 也没找到。。就没有修改
 			map->osd_weight[osd] = off;
 	}
 
                                                ？？？？？？缺少？？
                                                
+	/* new_recovery_weight */
+	ceph_decode_32_safe(p, end, len, e_inval);
+	while (len--) {
+		u32 osd, off;
+		ceph_decode_need(p, end, sizeof(u32)*2, e_inval);
+		osd = ceph_decode_32(p);
+		off = ceph_decode_32(p);
+		pr_info("osd%d recovery weight 0x%x %s\n", osd, off,
+		     off == CEPH_OSD_IN ? "(in)" :
+		     (off == CEPH_OSD_OUT ? "(out)" : ""));
+		if (osd < map->max_osd)
+			map->recovery_weight[osd] = off;
+	}
+
 	/* new_pg_temp */
 	err = decode_new_pg_temp(p, end, map);
 	if (err)
@@ -1530,11 +1558,16 @@ static int pg_to_raw_osds(struct ceph_os
  */
 static int raw_to_up_osds(struct ceph_osdmap *osdmap,
 			  struct ceph_pg_pool_info *pool,
+			  struct ceph_pg pgid,
 			  int *osds, int len, int *primary)
 {
 	int up_primary = -1;
 	int i;
 
+	/* raw_pg -> pg */
+	pgid.seed = ceph_stable_mod(pgid.seed, pool->pg_num,
+				    pool->pg_num_mask);
+
 	if (ceph_can_shift_osds(pool)) {
 		int removed = 0;
 
@@ -1543,6 +1576,12 @@ static int raw_to_up_osds(struct ceph_os
 				removed++;
 				continue;
 			}
+			if (pgid.seed > (pool->pg_num *
+					 osdmap->recovery_weight[osds[i]] /
+					 CEPH_OSD_IN)) {
+				removed++;
+				continue;
+			}
 			if (removed)
 				osds[i - removed] = osds[i];
 		}
                                                 
                                                 
                                                 
  下面这个没有了。另外，raw_to_up_osds的一个 调用 ，参数个数不对。。。
          
                                                 
                                                         raw_to_up_osds(osdmap, pi, &pgid, up);

                                                         raw_to_up_osds(osdmap, pi, &pgid, up);

                                                 
                                                 liang
@@ -1730,7 +1769,7 @@ int ceph_calc_pg_acting(struct ceph_osdm
 		return len;
 	}
 
-	len = raw_to_up_osds(osdmap, pool, osds, len, primary);
+	len = raw_to_up_osds(osdmap, pool, pgid, osds, len, primary);
 
 	apply_primary_affinity(osdmap, pps, pool, osds, len, primary);
 

```

> osdmap.h中只有一行改动，其余 改动 都是在osdmap.c(上面得patch在bean的文档上有过修改，在patch centos7.5和7.6时，发现有些函数找不到，所以直接删除了。)
>
> 

## 编译libceph.ko

注意，我们的patch主要在net/ceph网络层部分，因此需要首先生成libceph.ko，同时我们需要对方操作系统的symbols，即如下文件：

```
/usr/src/kernels/3.10.0-327.el7.x86_64/Module.symvers 
```

如果没有对方操作系统的这个文件，我们直接用源码编译出来的ko，可能无法正确地加载：

```bash
[root@localhost RPMS]# modprobe libceph ceph rbd
modprobe: ERROR: could not insert 'libceph': Exec format error
```

dmesg会有如下的信息：

```bash
[  111.185502] libceph: disagrees about version of symbol module_layout
[  114.217421] libceph: disagrees about version of symbol module_layout
[  313.849037] libceph: disagrees about version of symbol module_layout
[  326.745409] libceph: disagrees about version of symbol module_layout
[  333.415711] libceph: disagrees about version of symbol module_layout
[  337.793039] libceph: disagrees about version of symbol module_layout
```

因此第一步是将对方操作系统的Module.sysvers文件（/usr/src/kernels/3.10.0-327.el7.x86\_64/Module.symvers）拷贝到 ~/rpmbuild/BUILD/kernel-3.10.0-327.el7/linux-3.10.0-327.el7.centos.x86_64/目录下，然后在代码的顶层目录即：

```
~/rpmbuild/BUILD/kernel-3.10.0-327.el7/linux-3.10.0-327.el7.centos.x86_64
```

执行如下命令：

```bash
make oldconfig && make prepare && make prepare scripts
make modules SUBDIRS=net/ceph
```

执行完毕后，我们得到了libceph.ko，第一个ko文件就完成了。除此意外，我们在net/ceph目录下得到了新的Module.sysvers文件，在编译后面的ceph.ko和rbd.ko的时候，会用到这个发生改变的Module.sysvers文件。

## 编译ceph.ko和rbd.ko

将编译libceph.ko过程中产生的Module.sysvers文件放到内核代码的 fs/ceph/和drivers/block/目录下，完整路径如下：

```bash
~/rpmbuild/BUILD/kernel-3.10.0-327.el7/linux-3.10.0-327.el7.centos.x86_64/fs/ceph
~/rpmbuild/BUILD/kernel-3.10.0-327.el7/linux-3.10.0-327.el7.centos.x86_64/drivers/block
```

分别执行如下命令：

```bash
make modules SUBDIRS=fs/ceph
make modules SUBDIRS=drivers/block
```

执行成功后，就可以在fs/ceph下找到ceph.ko，在drivers/block下找到rbd.ko。

至此，编译部分就完成了，接下来就要替换对应的内核模块了。

# 替换内核模块

内核模块存放位置如下：

```bash
[root@localhost linux-3.10.0-327.el7.centos.x86_64]# modinfo libceph
filename:       /lib/modules/3.10.0-327.el7.x86_64/kernel/net/ceph/libceph.ko
license:        GPL
description:    Ceph filesystem for Linux
author:         Patience Warnick <patience@newdream.net>
author:         Yehuda Sadeh <yehuda@hq.newdream.net>
author:         Sage Weil <sage@newdream.net>
rhelversion:    7.2
srcversion:     3680132DA0EA395EBDD2736
depends:        libcrc32c,dns_resolver
vermagic:       3.10.0-327.el7.centos.x86_64 SMP mod_unload modversions 

[root@localhost linux-3.10.0-327.el7.centos.x86_64]# modinfo ceph
filename:       /lib/modules/3.10.0-327.el7.x86_64/kernel/fs/ceph/ceph.ko
license:        GPL
description:    Ceph filesystem for Linux
author:         Patience Warnick <patience@newdream.net>
author:         Yehuda Sadeh <yehuda@hq.newdream.net>
author:         Sage Weil <sage@newdream.net>
alias:          fs-ceph
rhelversion:    7.2
srcversion:     268CE83A90FA60A7654BE27
depends:        libceph
vermagic:       3.10.0-327.el7.centos.x86_64 SMP mod_unload modversions 

[root@localhost linux-3.10.0-327.el7.centos.x86_64]# modinfo rbd
filename:       /lib/modules/3.10.0-327.el7.x86_64/kernel/drivers/block/rbd.ko
license:        GPL
description:    RADOS Block Device (RBD) driver
author:         Jeff Garzik <jeff@garzik.org>
author:         Yehuda Sadeh <yehuda@hq.newdream.net>
author:         Sage Weil <sage@newdream.net>
author:         Alex Elder <elder@inktank.com>
rhelversion:    7.2
srcversion:     2D3BC30A44BCDD9CF47B4B0
depends:        libceph
vermagic:       3.10.0-327.el7.centos.x86_64 SMP mod_unload modversions 
parm:           single_major:Use a single major number for all rbd devices (default: false) (bool)

```

注意，我们只需要将新生成的ko文件放置到对应位置。如果客户环境ceph内核模块已经加载，那么需要先执行如下指令，卸载内核模块

```bash
modprobe -r rbd
modprobe -r ceph
modprobe -r libceph
```

然后我们就可以通过modprobe指令加载我们新生成的内核模块了。

```bash
modprobe libceph
modprobe ceph
modprobe rbd
```

注意两点：

* 老的内核模块做好备份，防范万一
* 重启测试下，modprobe能否加载我们的ko

# 效果测试

加载成功之后，需要和我们的存储对接，需要装对应的ceph client RPM：

```
rpm -ivh python-ceph-0.87.2-0.el7.x86_64.rpm rbd-fuse-0.87.2-0.el7.x86_64.rpm librbd1-0.87.2-0.el7.x86_64.rpm ceph-0.87.2-0.el7.x86_64.rpm ceph-common-0.87.2-0.el7.x86_64.rpm librados2-0.87.2-0.el7.x86_64.rpm libcephfs1-0.87.2-0.el7.x86_64.rpm 
```

注意，版本要和Ceph 存储集群的版本一致，即最好是一份代码生成的RPM。

将存储集群的ceph.conf拷贝到CentOS 7.2的如下位置：

```
/etc/ceph/ceph.conf
```

如果ceph -s正常输出，表示用户态已经可以和我们存储对接了：

```bash
[root@localhost linux-3.10.0-327.el7.centos.x86_64]# ceph -s
    cluster cc591c4c-07f2-4ddb-8134-96c3764dda42
     health HEALTH_OK
     monmap e3: 3 mons at {gtiqf=10.11.12.2:6789/0,pfgdl=10.11.12.1:6789/0,uxypf=10.11.12.3:6789/0}, election epoch 290, quorum 0,1,2 pfgdl,gtiqf,uxypf
     mdsmap e67: 1/1/1 up {0=koabr=up:active}, 1 up:standby
     osdmap e251: 3 osds: 3 up, 3 in
      pgmap v812966: 9216 pgs, 18 pools, 544 MB data, 449 objects
            1416 MB used, 104 GB / 105 GB avail
                9216 active+clean
  client io 30103 B/s rd, 12955 B/s wr, 45 op/s
```

接下来我们要测试rbd能否在CentOS上map成块设备。在存储节点上有如下的rbd：

```bash
id pool image                                    snap device    
0  rbd  f6fdb115-11dc-415c-8cfe-ba0125ccd831.img -    /dev/rbd0 
```

在CentOS执行如下操作映射RBD设备：

```bash
rbd -p {pool-name} map {img-name}
rbd -p rbd map f6fdb115-11dc-415c-8cfe-ba0125ccd831.img
```

执行完毕后，lsblk可以看到设备：

```bash
[root@localhost linux-3.10.0-327.el7.centos.x86_64]# lsblk
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
fd0               2:0    1    4K  0 disk 
sda               8:0    0   80G  0 disk 
├─sda1            8:1    0  500M  0 part /boot
└─sda2            8:2    0 79.5G  0 part 
  ├─centos-root 253:0    0 48.1G  0 lvm  /
  ├─centos-swap 253:1    0  7.9G  0 lvm  [SWAP]
  └─centos-home 253:2    0 23.5G  0 lvm  /home
sr0              11:0    1    4G  0 rom  
rbd0            252:0    0   20G  0 disk 
```

我们就可以分区，文件系统格式化，挂载，使用了。



