---
layout: post
title: ceph rbd header对象恢复和研究
category: 技术
---





# 背景

亚信线上ceph集群（v0.67），是我们老的集群，ceph的稳定性也不是很好，一直没有机会升级。这次故障的现象是云平台的vm在重启后，无法正常启动。找到vm对应的rbd img后，rbd info也无法正常输出,如下：

```
rbd信息无法获取
root@Storage-c1:~# rbd -p cn_pre-nova info d1783f0c-6aac-4568-bc73-603f670748aa_disk
rbd: error opening image d1783f0c-6aac-4568-bc73-603f670748aa_disk: (2) No such file or directory
2019-01-23 14:05:10.189804 7f306bea9780 -1 librbd::ImageCtx: error reading immutable metadata: (2) No such file or directory
root@Storage-c1:~# 
```

最后检查发现rbd img的rbd header 对象丢失，（也有可能header对象在，但是元数据有丢失）。



## 基本知识点

> rbd 元数据有3种存储方式：
>
> data: 元数据编码后以二进制存储在rados对象中
>
> xattr： 以key/value 方式存储在扩展属性
>
> omap： 存储在 omap中



rbd header对象本质上是个空文件，只是它的元数据信息 包含了rbd卷的很多信息。（详见后面的验证过程）

新创建 一个 rbd后,pool 中会产生如下对象

```
root@node1:~# rbd ls -p pool-wsg
b45cc982-506d-45f6-8aea-60764e394f4c.img
root@node1:~# rados ls -p pool-wsg
rbd_directory
rbd_id.b45cc982-506d-45f6-8aea-60764e394f4c.img
rbd_data.7uan0jo222ikj.0000000000000000
rbd_header.7uan0jo222ikj
rbd_header.7uan0jo222ikj.pr
root@node1:~# 

```



该对象 所在 pg

```
root@node1:~# ceph osd map pool-wsg rbd_header.7uan0jo222ikj
osdmap e157 pool 'pool-wsg' (17) object 'rbd_header.7uan0jo222ikj' -> pg 17.19d91b1 (17.1b1) -> up ([2,0], p2) acting ([2,0], p2)


需要注意该命令只是纯粹计算一个输入的object该如何映射，输入的object可以不存在。
root@node1:~# ceph osd map pool-wsg wsg
osdmap e157 pool 'pool-wsg' (17) object 'wsg' -> pg 17.4d58bb10 (17.310) -> up ([1,0], p1) acting ([1,0], p1)
root@node1:~# 

```



pg  2副本分别位于 primary osd.2，可以通过getfattr 提取出文件系统层面的扩展属性

```
root@node3:/data/osd.2/current# find . -name "*7uan0jo222ikj*"
./17.1b1_head/rbd\uheader.7uan0jo222ikj__head_019D91B1__11


root@node3:/data/osd.2/current/17.1b1_head# getfattr -d -m - rbd\\uheader.7uan0jo222ikj__head_019D91B1__11 
# file: rbd\134uheader.7uan0jo222ikj__head_019D91B1__11
user.ceph.snapset=0sAgIZAAAAAAAAAAAAAAABAAAAAAAAAAAAAAAAAAAAAA==
user.cephos.spill_out=0sMQA=




root@node3:/data/osd.2/current/17.1b1_head# rados -p pool-wsg listomapvals rbd_header.7uan0jo222ikj
features
value: (8 bytes) :
0000 : 01 00 00 00 00 00 00 00                         : ........

object_prefix
value: (26 bytes) :
0000 : 16 00 00 00 72 62 64 5f 64 61 74 61 2e 37 75 61 : ....rbd_data.7ua
0010 : 6e 30 6a 6f 32 32 32 69 6b 6a                   : n0jo222ikj

order
value: (1 bytes) :
0000 : 16                                              : .

size
value: (8 bytes) :
0000 : 00 00 00 00 05 00 00 00                         : ........

snap_seq
value: (8 bytes) :
0000 : 00 00 00 00 00 00 00 00                         : ........

```

可以获取object的所有metadata









## lab 模拟故障情况

> 在ceph 0.87版本验证

1，创建测试的rbd 卷，提前 保存后 rbd header 相关元数据

```
root@galen61-node1:~# rbd create pool-wsg/rbd-wsg --image-format 2 --size 40960

root@galen61-node1:~# rbd info pool-wsg/rbd-wsg
rbd image 'rbd-wsg':
	size 40960 MB in 10240 objects
	order 22 (4096 kB objects)
	used objects: 0
	block_name_prefix: rbd_data.6n2q5cs1h8ncj
	format: 2
	features: layering
	


root@galen61-node1:~/rbd_head_recover/rbd_header_recover# rados -p pool-wsg listomapvals rbd_header.6n2q5cs1h8ncj
features
value: (8 bytes) :
0000 : 01 00 00 00 00 00 00 00                         : ........

object_prefix
value: (26 bytes) :
0000 : 16 00 00 00 72 62 64 5f 64 61 74 61 2e 36 6e 32 : ....rbd_data.6n2
0010 : 71 35 63 73 31 68 38 6e 63 6a                   : q5cs1h8ncj

order
value: (1 bytes) :
0000 : 16                                              : .

size
value: (8 bytes) :
0000 : 00 00 00 00 0a 00 00 00                         : ........

snap_seq
value: (8 bytes) :
0000 : 00 00 00 00 00 00 00 00                         : ........

root@galen61-node1:~/rbd_head_recover/rbd_header_recover# 



root@galen61-node1:~/rbd_head_recover/rbd_header_recover# ceph osd map  pool-wsg rbd_header.6n2q5cs1h8ncj
osdmap e162 pool 'pool-wsg' (16) object 'rbd_header.6n2q5cs1h8ncj' -> pg 16.996eefe (16.2fe) -> up ([1,0], p1) acting ([1,0], p1)
root@galen61-node1:~/rbd_head_recover/rbd_header_recover# 



root@galen61-node3:~# ceph daemon osd.1 getomap pool-wsg rbd_header.6n2q5cs1h8ncj | head -c-2 | hexdump -v -e '"\\\\\\x" 1/1 "%02X"'
\\\\68\\\\65\\\\61\\\\64\\\\65\\\\72\\\\3D\\\\20\\\\6B\\\\65\\\\79\\\\3D\\\\66\\\\65\\\\61\\\\74\\\\75\\\\72\\\\65\\\\73\\\\20\\\\76\\\\61\\\\6C\\\\3D\\\\01\\\\00\\\\00\\\\00\\\\00\\\\00\\\\00\\\\00\\\\20\\\\6B\\\\65\\\\79\\\\3D\\\\6F\\\\62\\\\6A\\\\65\\\\63\\\\74\\\\5F\\\\70\\\\72\\\\65\\\\66\\\\69\\\\78\\\\20\\\\76\\\\61\\\\6C\\\\3D\\\\16\\\\00\\\\00\\\\00\\\\72\\\\62\\\\64\\\\5F\\\\64\\\\61\\\\74\\\\61\\\\2E\\\\36\\\\6E\\\\32\\\\71\\\\35\\\\63\\\\73\\\\31\\\\68\\\\38\\\\6E\\\\63\\\\6A\\\\20\\\\6B\\\\65\\\\79\\\\3D\\\\6F\\\\72\\\\64\\\\65\\\\72\\\\20\\\\76\\\\61\\\\6C\\\\3D\\\\16\\\\20\\\\6B\\\\65\\\\79\\\\3D\\\\73\\\\69\\\\7A\\\\65\\\\20\\\\76\\\\61\\\\6C\\\\3D\\\\00\\\\00\\\\00\\\\00\\\\0A\\\\00\\\\00\\\\00\\\\20\\\\6B\\\\65\\\\79\\\\3D\\\\73\\\\6E\\\\61\\\\70\\\\5F\\\\73\\\\65\\\\71\\\\20\\\\76\\\\61\\\\6C\\\\3D\\\\00\\\\00\\\\00\\\\00\\\\00\\\\00\\\\00\\\\00root@galen61-node3:~# 


root@galen61-node1:~/rbd_head_recover/rbd_header_recover# ceph daemon osd.0 getomap pool-wsg rbd_header.6n2q5cs1h8ncj | head -c-2 | hexdump -v -e '"\\\\\\x" 1/1 "%02X"'
\\\\68\\\\65\\\\61\\\\64\\\\65\\\\72\\\\3D\\\\20\\\\6B\\\\65\\\\79\\\\3D\\\\66\\\\65\\\\61\\\\74\\\\75\\\\72\\\\65\\\\73\\\\20\\\\76\\\\61\\\\6C\\\\3D\\\\01\\\\00\\\\00\\\\00\\\\00\\\\00\\\\00\\\\00\\\\20\\\\6B\\\\65\\\\79\\\\3D\\\\6F\\\\62\\\\6A\\\\65\\\\63\\\\74\\\\5F\\\\70\\\\72\\\\65\\\\66\\\\69\\\\78\\\\20\\\\76\\\\61\\\\6C\\\\3D\\\\16\\\\00\\\\00\\\\00\\\\72\\\\62\\\\64\\\\5F\\\\64\\\\61\\\\74\\\\61\\\\2E\\\\36\\\\6E\\\\32\\\\71\\\\35\\\\63\\\\73\\\\31\\\\68\\\\38\\\\6E\\\\63\\\\6A\\\\20\\\\6B\\\\65\\\\79\\\\3D\\\\6F\\\\72\\\\64\\\\65\\\\72\\\\20\\\\76\\\\61\\\\6C\\\\3D\\\\16\\\\20\\\\6B\\\\65\\\\79\\\\3D\\\\73\\\\69\\\\7A\\\\65\\\\20\\\\76\\\\61\\\\6C\\\\3D\\\\00\\\\00\\\\00\\\\00\\\\0A\\\\00\\\\00\\\\00\\\\20\\\\6B\\\\65\\\\79\\\\3D\\\\73\\\\6E\\\\61\\\\70\\\\5F\\\\73\\\\65\\\\71\\\\20\\\\76\\\\61\\\\6C\\\\3D\\\\00\\\\00\\\\00\\\\00\\\\00\\\\00\\\\00\\\\00root@galen61-node1:~/rbd_head_recover/rbd_header_recover# 

```





2，删除rbd header 对象

```
root@galen61-node1:~# rados -p pool-wsg rm rbd_header.6n2q5cs1h8ncj

删除后rbd info 已经无法读取
root@galen61-node1:~# rbd info pool-wsg/rbd-wsg
2019-08-08 15:04:15.719631 7f60c468e780 -1 librbd::ImageCtx: error reading immutable metadata: (2) No such file or directory
rbd: error opening image rbd-wsg: (2) No such file or directory
root@galen61-node1:~# 



```



3，构造 rbd header对象和元数据

> 因为rbd header并没有实际数据，只要 塞一个 空文件就可以了。

```
塞rbd_header对象前后的变化
root@galen61-node1:~# rados -p pool-wsg ls |grep rbd_header.6n2q5cs1h8ncj
rbd_header.6n2q5cs1h8ncj.bitmask
root@galen61-node1:~# 
root@galen61-node1:~# 
root@galen61-node1:~# rados -p pool-wsg  put  rbd_header.6n2q5cs1h8ncj a
root@galen61-node1:~# rados -p pool-wsg ls |grep rbd_header.6n2q5cs1h8ncj
rbd_header.6n2q5cs1h8ncj.bitmask
rbd_header.6n2q5cs1h8ncj
root@galen61-node1:~# 


空 rbd_header对象的属性
root@galen61-node1:~# rados -p pool-wsg listomapvals rbd_header.6n2q5cs1h8ncj
root@galen61-node1:~# 


```



构造rbd header的元数据

```
从rados的help中可以看到设置 对象的omap 命令，并且网上也看到有人做了类似的事情（参考链接：http://www.zphj1987.com/2016/07/02/%E9%87%8D%E6%9E%84rbd%E9%95%9C%E5%83%8F%E7%9A%84%E5%85%83%E6%95%B0%E6%8D%AE/），可是在我的环境上，用这个方法行不同，无法将 16进制数据直接喂给 rados命令

 setomapval <obj-name> <key> <val>
   rmomapkey <obj-name> <key>
   getomapheader <obj-name> [file]
   setomapheader <obj-name> <val>



所以只能绕过rados，通过level db的接口直接给对象设置元数据，python脚本已经放到了我的gitlab仓库（参考链接： https://gitlab.com/dragon200518/my-own/tree/master/PYTHON/omaptool）


根据之前保存的key value 值，手动执行（执行前 停掉osd ）
root@galen61-node3:~/omaptools# python set_omap.py /data/osd.1/current/omap/ rbd_header.6n2q5cs1h8ncj order \\x16 snap_seq \\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00 size \\x00\\x00\\x00\\x00\\x19\\x00\\x00\\x00 object_prefix \\x16\\x00\\x00\\x00\\x72\\x62\\x64\\x5F\\x64\\x61\\x74\\x61\\x2E\\x36\\x6E\\x32\\x71\\x35\\x63\\x73\\x31\\x68\\x38\\x6E\\x63\\x6A features \\x01\\x00\\x00\\x00\\x00\\x00\\x00\\x00
root@galen61-node3:~/omaptools# 


启用osd后再次查询 rbd info，可以看到 rbd img的信息有了
root@galen61-node3:~/omaptools# rbd info pool-wsg/rbd-wsg
rbd image 'rbd-wsg':
	size 102400 MB in 25600 objects
	order 22 (4096 kB objects)
	used objects: 70
	block_name_prefix: rbd_data.6n2q5cs1h8ncj
	format: 2
	features: layering
root@galen61-node3:~/omaptools# 

检查之前的数据完整性
root@galen61-node3:~/omaptools# mount /dev/rbd2p2 /mnt/wsg/
root@galen61-node3:~/omaptools# cd /mnt/wsg/
root@galen61-node3:/mnt/wsg# ls
lost+found  wsg.tst
root@galen61-node3:/mnt/wsg# cat wsg.tst 
hello
wsg add
rbd2 
pool-wsg/rbd-wsg
two partion
5G
35G

root@galen61-node3:/mnt/wsg# df -h
Filesystem                          Size  Used Avail Use% Mounted on
/dev/sda3                            50G  6.9G   41G  15% /
udev                                2.0G   12K  2.0G   1% /dev
tmpfs                               396M  708K  395M   1% /run
none                                5.0M     0  5.0M   0% /run/lock
none                                2.0G   36K  2.0G   1% /run/shm
/dev/sdb1                            99G   34G   65G  35% /data/osd.1
10.0.0.122,10.0.0.123,10.0.0.124:/   99G   34G   65G  35% /var/share/ezfs
ceph-fuse                           295G  101G  195G  35% /mnt/ezfs_fuse
/dev/rbd2p2                          35G   48M   33G   1% /mnt/wsg
root@galen61-node3:/mnt/wsg# 

```







# 相关知识点记录

## 进制转换

asci码 和16进制转换

```
linux下可以
man ascii 
NAME
       ascii - ASCII character set encoded in octal, decimal, and hexadecimal

DESCRIPTION
       ASCII is the American Standard Code for Information Interchange.  It is a 7-bit code.  Many 8-bit codes (such as ISO 8859-1, the Linux default character set) contain ASCII as their lower half.  The international counterpart
       of ASCII is known as ISO 646.

       The following table contains the 128 ASCII characters.

       C program '\X' escapes are noted.

       Oct   Dec   Hex   Char                        Oct   Dec   Hex   Char
       ────────────────────────────────────────────────────────────────────────
       000   0     00    NUL '\0'                    100   64    40    @
       001   1     01    SOH (start of heading)      101   65    41    A
       002   2     02    STX (start of text)         102   66    42    B
       003   3     03    ETX (end of text)           103   67    43    C




order
1a	26
16	22


用UltraEdit转换大小写
alt+F5转大写；
ctrl+F5转小写； 
F5每个单词的首字母大写；
Shift+F5大小写互换。
```





## 手动设置 xattr



```
root@node1:/data/osd.0# setfattr -n "user.wsg" -v 888 wsg.txt 
root@node1:/data/osd.0# getfattr -d -m - wsg.txt 
# file: wsg.txt
user.wsg="888"

-n name 需要 以 user.打头。

cp 之后 拓展属性就没有了， mv 之后 拓展属性 还在。
rsync -aX 可以保留 xattr


```





attr -q 输出 会有不一样



```
Without the -q flag, a message showing  the  attribute  name  and  the
              entire value will be printed.
              
```





## 确认rbd image的rbd header

```
通过rbd img name可以查到rbd的prefix（每个rbd会产生rbd_id."rbdname"的对象，该对象里面保存的就是prefix，所以get 该对象内容即可）
root@galen61-node1:~# rados -p pool-wsg get rbd_id.rbd-wsg -
6n2q5cs1h8ncjroot@galen61-node1:~# 


根据rbd 的prefix 才能 关联到 rbd header
root@galen61-node1:~# rados ls -p pool-wsg |grep 6n2q5cs1h8ncj
rbd_header.6n2q5cs1h8ncj

```



## rbd header 对象的大小和内容

```
root@galen61-node3:/data/osd.1/current# find . -name "*6n2q5cs1apd0j*"
./16.3b8_head/rbd\uheader.6n2q5cs1apd0j__head_8760EBB8__10
root@galen61-node3:/data/osd.1/current# ll ./16.3b8_head/rbd\uheader.6n2q5cs1apd0j__head_8760EBB8__10
ls: cannot access ./16.3b8_head/rbduheader.6n2q5cs1apd0j__head_8760EBB8__10: No such file or directory

# rbdheader 名字中有特殊字符，需要 转义。
root@galen61-node3:/data/osd.1/current# ll ./16.3b8_head/rbd\\uheader.6n2q5cs1apd0j__head_8760EBB8__10
-rw-r--r-- 1 root root 0 Aug 16 15:27 ./16.3b8_head/rbd\uheader.6n2q5cs1apd0j__head_8760EBB8__10
root@galen61-node3:/data/osd.1/current# cat ./16.3b8_head/rbd\\uheader.6n2q5cs1apd0j__head_8760EBB8__10
root@galen61-node3:/data/osd.1/current# 

```



作为对比，我们把一个rbd data 对象的内容，（和nas 中的对象不太一样，rbd 里面的内容看起来被编码了，直接cat 无法打开）

```
root@galen61-node3:/data/osd.1/current# cat ./16.1f3_head/rbd\\udata.6n2q5cs1h8ncj.0000000000001530__head_A88F59F3__10

```

 



## 8.0 ceph 12.2 版本可以直接设置 对象元数据

元数据 以 16进制格式



```
root@node-195:~# echo -en \\x16|rados -p rbd setomapval obj-wsg key2
root@node-195:~# rados -p rbd listomapvals obj-wsg
key1
value (1 bytes) :
00000000  16                                                |.|
00000001

key2
value (1 bytes) :
00000000  16                                                |.|
00000001

order
value (9 bytes) :
00000000  74 65 73 74 6f 72 64 65  72                       |testorder|
00000009

root@node-195:~# echo -en \\x01\\x00\\x00\\x00\\x00\\x00\\x00\\x00|rados -p rbd setomapval obj-wsg features
root@node-195:~# rados -p rbd listomapvals obj-wsg
features
value (8 bytes) :
00000000  01 00 00 00 00 00 00 00                           |........|
00000008

key1
value (1 bytes) :
00000000  16                                                |.|
00000001

key2
value (1 bytes) :
00000000  16                                                |.|
00000001

order
value (9 bytes) :
00000000  74 65 73 74 6f 72 64 65  72                       |testorder|
00000009

root@node-195:~# 


```









# 测试总结

有2中恢复rbd header元数据的方法：

1，通过rados 命令直接修改，但是只有 ceph v12版本之上的版本才可以

2，通过python调用 leveldb 接口，直接操作数据库（脚本可参考：https://gitlab.com/dragon200518/my-own/tree/master/PYTHON/omaptool）





# 遗留问题



不同 ceph 版本，rados 命令 对于 接受 stdin 的 表现不一样？ ceph 12 版本的 rados 才可以 接受 管道的 stdin ？？？



## rbd img中object_prefix 如何计算？？

看起来 并不是 pool id，david 的shell 脚本好像有 计算的过程



**新建rbd-wsg-1**



```
root@galen61-node1:~# rados -p pool-wsg get rbd_id.rbd-wsg-1 -
6n2q5cs1apd0jroot@galen61-node1:~#

rados -p pool-wsg listomapvals rbd_header.6n2q5cs1apd0j
features
value: (8 bytes) :
0000 : 01 00 00 00 00 00 00 00                         : ........

object_prefix
value: (26 bytes) :
0000 : 16 00 00 00 72 62 64 5f 64 61 74 61 2e 36 6e 32 : ....rbd_data.6n2
0010 : 71 35 63 73 31 61 70 64 30 6a                   : q5cs1apd0j

order
value: (1 bytes) :
0000 : 16                                              : .

size
value: (8 bytes) :
0000 : 00 00 00 80 0c 00 00 00                         : ........

snap_seq
value: (8 bytes) :
0000 : 00 00 00 00 00 00 00 00                         : ........

root@galen61-node1:~# 

```





** 新建 pool和rbd**

```
root@galen61-node1:~# rados -p pool-wsg2 get rbd_id.rbd-test1 -
6n2q5cs1apmj3root@galen61-node1:~# rados -p pool-wsg2 get rbd_id.rbd-test1^C
root@galen61-node1:~# rados -p pool-wsg listomapvals rbd_header.6n2q5cs1apmj3
error getting omap keys pool-wsg/rbd_header.6n2q5cs1apmj3: (2) No such file or directory


root@galen61-node1:~# rados -p pool-wsg2 listomapvals rbd_header.6n2q5cs1apmj3
features
value: (8 bytes) :
0000 : 01 00 00 00 00 00 00 00                         : ........

object_prefix
value: (26 bytes) :
0000 : 16 00 00 00 72 62 64 5f 64 61 74 61 2e 36 6e 32 : ....rbd_data.6n2
0010 : 71 35 63 73 31 61 70 6d 6a 33                   : q5cs1apmj3

order
value: (1 bytes) :
0000 : 16                                              : .

size
value: (8 bytes) :
0000 : 00 00 00 40 00 00 00 00                         : ...@....

snap_seq
value: (8 bytes) :
0000 : 00 00 00 00 00 00 00 00                         : ........

root@galen61-node1:~# 


```



新建

```
root@galen61-node1:~# rbd create pool-wsg2/rbd-test2 --image-format 2 --size 1024
root@galen61-node1:~# rados -p pool-wsg2 get rbd_id.rbd-test2 -
6n2q5cs1k7pqqroot@galen61-node1:~# 
root@galen61-node1:~# rados -p pool-wsg2 listomapvals rbd_header.6n2q5cs1k7pqq
features
value: (8 bytes) :
0000 : 01 00 00 00 00 00 00 00                         : ........

object_prefix
value: (26 bytes) :
0000 : 16 00 00 00 72 62 64 5f 64 61 74 61 2e 36 6e 32 : ....rbd_data.6n2
0010 : 71 35 63 73 31 6b 37 70 71 71                   : q5cs1k7pqq

order
value: (1 bytes) :
0000 : 16                                              : .

size
value: (8 bytes) :
0000 : 00 00 00 40 00 00 00 00                         : ...@....

snap_seq
value: (8 bytes) :
0000 : 00 00 00 00 00 00 00 00                         : ........

root@galen61-node1:~# 


```







# 附录





## 亚信 rbd-header-recover 脚本执行情况记录



```
对指定pool 检查
 bash  recover_rbd_header_wsg.sh cn_pre-cinder
 
 

Checking pool: cn_pre-cinder
..............
Image volume-0ec4cc5c-e307-4202-80e2-d897b68d3076 have error: 
	rbd_header.14ae38af5e701d1f object does not exists in primary osd
	osd [13 25 19] of [13 25 19] missing omap
	volume-0ec4cc5c-e307-4202-80e2-d897b68d3076 need info to recover
...............................................
Image volume-30d55c34-7d98-4838-971e-978145a9c281 have error: 
	osd [10 24] of [13 10 24] missing omap
Save backup omaps for volume-30d55c34-7d98-4838-971e-978145a9c281 in 2019-08-07_14_18_19/rbd_header.1e6a25b819d7582f.omap

..................
Image volume-42b7a869-5413-4e65-ac0a-124dee366edd have error: 
	rbd_header.a97bd1e3002e8ae object does not exists in primary osd
	osd [13 10 24] of [13 10 24] missing omap
	volume-42b7a869-5413-4e65-ac0a-124dee366edd need info to recover
```

从recover.log 检查

```

pool=cn_pre-cinder
rbd_header.14ae38af5e701d1f object does not exists in primary osd
osd [13 25 19] of [13 25 19] missing omap
volume-0ec4cc5c-e307-4202-80e2-d897b68d3076 need info to recover


###### rbd-head存在情况。
osd [10 24] of [13 10 24] missing omap
volume-30d55c34-7d98-4838-971e-978145a9c281 will try to recover


rbd_header.a97bd1e3002e8ae object does not exists in primary osd
osd [13 10 24] of [13 10 24] missing omap
volume-42b7a869-5413-4e65-ac0a-124dee366edd need info to recover


rbd_header.a65f33a64a9f8e2 object does not exists in primary osd
osd [13 4] of [13 18 4] missing omap
volume-97e456b4-845b-475b-b3eb-415f057e3dfc will try to recover
```

对于日志格式： 

 rbd_head对象在 primary osd不存在的情况，会首先 打印 rbd head

rbd_head对象在 primary osd存在的情况，则不会打印head









## 亚信 故障处理(snap产生的rbd )

### 问题现象

```
rbd信息无法获取
root@Storage-c1:~# rbd -p cn_pre-nova info d1783f0c-6aac-4568-bc73-603f670748aa_disk
rbd: error opening image d1783f0c-6aac-4568-bc73-603f670748aa_disk: (2) No such file or directory
2019-01-23 14:05:10.189804 7f306bea9780 -1 librbd::ImageCtx: error reading immutable metadata: (2) No such file or directory
root@Storage-c1:~# 
```



因为有问题的rbd 是快照产生的，业务部门那边可以比对出 问题rbd 是来自哪个parent，然后将获取到的parent rbd 元数据按照固定格式写入文件，就可以利用david的修复脚本修复损坏的rbd_header信息。

这次找到的parent rbd 信息如下：

```
root@Storage-c1:~# rbd info cn_pre-glance/cca78713-1f68-47be-8031-741e5c9eb983
rbd image 'cca78713-1f68-47be-8031-741e5c9eb983':
	size 40960 MB in 5120 objects
	order 23 (8192 KB objects)
	block_name_prefix: rbd_data.582c9e859103084
	format: 2
	features: layering
```



### 脚本输入参数

第一个是 待修复rbd，第二个是 rbd元 数据的文件

```
rbd元 数据的文件的内容如下：格式要求 clone-img size order parent-img@snapip
snapid可以通过 rbd snap ls 获取

下面是我此次修复手动生成的文件
root@Storage-c1:~/recover# cat wsg_img 
d1783f0c-6aac-4568-bc73-603f670748aa_disk 40960 23 cn_pre-glance/cca78713-1f68-47be-8031-741e5c9eb983@7
root@Storage-c1:~/recover# 
```



###脚本执行结果

脚本 执行过程中为设置osd的omap，所以会停止osd

```

root@Storage-c1:~/recover# vim wsg_img
root@Storage-c1:~/recover# bash recover_rbd_header.sh cn_pre-nova wsg_img 
Waiting cluster healthy...done
OSD 0 at 172.16.250.1
OSD 1 at 172.16.250.1
OSD 2 at 172.16.250.1
OSD 3 at 172.16.250.1
OSD 4 at 172.16.250.2
OSD 5 at 172.16.250.2
OSD 6 at 172.16.250.2
OSD 7 at 172.16.250.2
OSD 8 at 172.16.250.3
OSD 9 at 172.16.250.3
OSD 10 at 172.16.250.3
OSD 11 at 172.16.250.3
OSD 12 at 172.16.250.4
OSD 13 at 172.16.250.4
OSD 14 at 172.16.250.4
OSD 15 at 172.16.250.4
OSD 16 at 172.16.250.5
OSD 17 at 172.16.250.5
OSD 18 at 172.16.250.5
OSD 19 at 172.16.250.5
OSD 20 at 172.16.250.6
OSD 21 at 172.16.250.6
OSD 22 at 172.16.250.6
OSD 23 at 172.16.250.6
OSD 24 at 172.16.250.7
OSD 25 at 172.16.250.7
OSD 26 at 172.16.250.7
OSD 27 at 172.16.250.7

Checking pool: cn_pre-nova

Image d1783f0c-6aac-4568-bc73-603f670748aa_disk have error: 
	osd [13 1 25] of [13 1 25] missing omap
	d1783f0c-6aac-4568-bc73-603f670748aa_disk(size 40960) need info to recover
Save backup omaps for d1783f0c-6aac-4568-bc73-603f670748aa_disk in 2019-01-23_17_41_16/rbd_header.973fed283df0c163.omap


************************************************************
OSD: 25 13 1 needs to restart for omap injection
************************************************************


************************************************************
Waiting cluster healthy...done
set_omap.py                                                                       100% 1213     1.2KB/s   00:00    
Packing rbd_header.973fed283df0c163 for osd.25
Going to stop osd.25 for omap injection, continue?(Y/n) y
=== osd.25 === 
Stopping Ceph osd.25 on 172.16.250.7...kill 23456...kill 23456...done
=== osd.25 === 
Starting Ceph osd.25 on 172.16.250.7...
starting osd.25 at :/0 osd_data /data/osd.25 /dev/disk/by-partlabel/v2-journal
************************************************************


************************************************************
Waiting cluster healthy................
```



修复后再次查看 问题rbd

```
root@Storage-c1:~/recover# rbd -p cn_pre-nova info d1783f0c-6aac-4568-bc73-603f670748aa_disk
rbd image 'd1783f0c-6aac-4568-bc73-603f670748aa_disk':
	size 40960 MB in 5120 objects
	order 23 (8192 KB objects)
	block_name_prefix: rbd_data.973fed283df0c16
	format: 2
	features: layering
	parent: cn_pre-glance/cca78713-1f68-47be-8031-741e5c9eb983@snap
	overlap: 40960 MB
	
```



## 代码



```
src/osd/osd_types.h:struct object_info_t {


2686 struct object_info_t {
2687   hobject_t soid;
2688   string category;
2689 
2690   eversion_t version, prior_version;
2691   version_t user_version;
2692   osd_reqid_t last_reqid;
2693 
2694   uint64_t size;
2695   utime_t mtime;
2696   utime_t local_mtime; // local mtime
2697 
2698   // note: these are currently encoded into a total 16 bits; see
2699   // encode()/decode() for the weirdness.
2700   typedef enum {
2701     FLAG_LOST     = 1<<0,
2702     FLAG_WHITEOUT = 1<<1,  // object logically does not exist
2703     FLAG_DIRTY    = 1<<2,  // object has been modified since last flushed or undirtied
2704     FLAG_OMAP     = 1 << 3,  // has (or may have) some/any omap data



```







encode

```
src/include/encoding.h: * - Normal classes will use WRITE_CLASS_ENCODER, with that features=0 default.
src/include/encoding.h: * - Classes that _require_ features will use WRITE_CLASS_ENCODER_FEATURES, which
src/include/encoding.h:#define WRITE_CLASS_ENCODER(cl)						\
src/include/encoding.h:#define WRITE_CLASS_ENCODER_FEATURES(cl)		


```





### encoding.h

```
146 #define WRITE_CLASS_ENCODER(cl)                     \
147   inline void encode(const cl &c, bufferlist &bl, uint64_t features=0) { \
148     ENCODE_DUMP_PRE(); c.encode(bl); ENCODE_DUMP_POST(cl); }        \
149   inline void decode(cl &c, bufferlist::iterator &p) { c.decode(p); }
150 


811 /*                                                                                                                                                                                                                                      
812  * guards
813  */
814 
815 /*
816  * struct_v + struct_compat + struct_len
817  */
818 #define ENCODE_START_BYTES (1 + 1 + 4)
819 
820 /**
821  * start encoding block
822  *
823  * @param v current (code) version of the encoding
824  * @param compat oldest code version that can decode it
825  * @param bl bufferlist to encode to
826  */
827 #define ENCODE_START(v, compat, bl)                          \
828   __u8 struct_v = v, struct_compat = compat;                 \
829   ::encode(struct_v, (bl));                                  \
830   ::encode(struct_compat, (bl));                             \
831   buffer::list::iterator struct_compat_it = (bl).end();      \
832   struct_compat_it.advance(-1);                              \
833   ceph_le32 struct_len;                                      \
834   struct_len = 0;                                            \
835   ::encode(struct_len, (bl));                                \
836   buffer::list::iterator struct_len_it = (bl).end();         \
837   struct_len_it.advance(-4);                                 \
838   do {
839 
840 

```





### osd/osd_types.cc,6.3版本code

```
4326 void object_info_t::encode(bufferlist& bl) const

4337   // kludge to reduce xattr size for most rbd objects
4338   int ev = 15;
4339   if (!(flags & FLAG_OMAP_DIGEST) &&
4340       !(flags & FLAG_DATA_DIGEST) &&
4341       local_mtime == utime_t()) {
4342     ev = 13;
4343   }
4344 
4345   ENCODE_START(ev, 8, bl);
4346   ::encode(soid, bl);
4347   ::encode(myoloc, bl); //Retained for compatibility
4348   ::encode((__u32)0, bl); // was category, no longer used
4349   ::encode(version, bl);
4350   ::encode(prior_version, bl);
4351   ::encode(last_reqid, bl);
4352   ::encode(size, bl);
4353   ::encode(mtime, bl);
4354   if (soid.snap() == CEPH_NOSNAP)
4355     ::encode(wrlock_by, bl);
4356   else
4357     ::encode(snaps, bl);                                                                                                                                          
4358   ::encode(truncate_seq, bl);
4359   ::encode(truncate_size, bl);
4360   ::encode(is_lost(), bl);
4361   ::encode(old_watchers, bl);
4362   /* shenanigans to avoid breaking backwards compatibility in the disk format.
4363    * When we can, switch this out for simply putting the version_t on disk. */
4364   eversion_t user_eversion(0, user_version);
4365   ::encode(user_eversion, bl);
4366   ::encode(test_flag(FLAG_USES_TMAP), bl);
4367   ::encode(watchers, bl);
4368   __u32 _flags = flags;
4369   ::encode(_flags, bl);
4370   if (ev >= 14) {
4371     ::encode(local_mtime, bl);
4372   }                                                                                                                                                               
4373   if (ev >= 15) {
4374     ::encode(data_digest, bl);
4375     ::encode(omap_digest, bl);
4376   }
4377   ENCODE_FINISH(bl);
4378 }

```





### osd/osd_type.cc,6.1版本



```
3818 void object_info_t::encode(bufferlist& bl) const
3819 {
3820   object_locator_t myoloc(soid);
3821   map<entity_name_t, watch_info_t> old_watchers;
3822   for (map<pair<uint64_t, entity_name_t>, watch_info_t>::const_iterator i =
3823      watchers.begin();
3824        i != watchers.end();
3825        ++i) {
3826     old_watchers.insert(make_pair(i->first.second, i->second));
3827   }
3828   ENCODE_START(14, 8, bl);
3829   ::encode(soid, bl);
3830   ::encode(myoloc, bl); //Retained for compatibility
3831   ::encode(category, bl);
3832   ::encode(version, bl);
3833   ::encode(prior_version, bl);
3834   ::encode(last_reqid, bl);
3835   ::encode(size, bl);
3836   ::encode(mtime, bl);
3837   if (soid.snap == CEPH_NOSNAP)
3838     ::encode(wrlock_by, bl);
3839   else
3840     ::encode(snaps, bl);
3841   ::encode(truncate_seq, bl);
3842   ::encode(truncate_size, bl);
3843   ::encode(is_lost(), bl);
3844   ::encode(old_watchers, bl);
3845   /* shenanigans to avoid breaking backwards compatibility in the disk format.
3846    * When we can, switch this out for simply putting the version_t on disk. */
3847   eversion_t user_eversion(0, user_version);
3848   ::encode(user_eversion, bl);
3849   ::encode(test_flag(FLAG_USES_TMAP), bl);
3850   ::encode(watchers, bl);
3851   __u32 _flags = flags;
3852   ::encode(_flags, bl);
3853   ::encode(local_mtime, bl);
3854   ENCODE_FINISH(bl);
3855 }
3856 

```



### hobject_t soid

？？ 需要 在 代码 里找出 hobject_t 每个成员变量 的 定义和 大小



object_t 就是对于的低层文件系统的一个文件，name就对于的文件名

sobject_t 就是 加了snapshot相关信息的object_t ，snap 就是该对象对于的snapshot的以snap 号， 



hobject_t 是名字应该是 hash object的缩写。 
其除了已经介绍过的数据成员： 
object_t oid; 保存对象的名字 
snapid_t snap; 保存对象的snap号 
新增加了成员： 
int64_t pool 所在的pool的id 
string nspace 
string key 
string hash



ghobject 在对象hobject_t的基础上，添加了 generation字段 和 shard_id 字段



### object_locator_t myoloc(soid)





## 通用编码规范

[](https://zh.wikipedia.org/wiki/%E5%AD%97%E8%8A%82%E5%BA%8F)

little endian value

linux  xxd 和 hexdump 输出 的 顺序不一样，xxd 输出 和 



```
root@node1:~# echo "2018-08-22 10:23:23.718986" |hexdump
0000000 3032 3831 302d 2d38 3232 3120 3a30 3332
0000010 323a 2e33 3137 3938 3638 000a          
000001b
root@node1:~# echo "2018-08-22 10:23:23.718986" |od -A d -x
0000000 3032 3831 302d 2d38 3232 3120 3a30 3332
0000016 323a 2e33 3137 3938 3638 000a
0000027
root@node1:~# echo "2018-08-22 10:23:23.718986" |od -x
0000000 3032 3831 302d 2d38 3232 3120 3a30 3332
0000020 323a 2e33 3137 3938 3638 000a
0000033
root@node1:~# vim hex.tst
root@node1:~# vim -b hex.tst 
0000000: 3230 3138 2d30 382d 3232 2031 303a 3233  2018-08-22 10:23
0000010: 3a32 332e 3731 3839 3836 0a              :23.718986.
~                                                                   


root@node1:~# echo "2018-08-22 10:23:23.718986" |od -tx
0000000 38313032 2d38302d 31203232 33323a30
0000020 2e33323a 39383137 000a3638
0000033
root@node1:~# 


root@node1:~# echo "2018-08-22 10:23:23.718986" |xxd 
0000000: 3230 3138 2d30 382d 3232 2031 303a 3233  2018-08-22 10:23
0000010: 3a32 332e 3731 3839 3836 0a              :23.718986.


```











