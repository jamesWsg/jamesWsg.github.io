---
layout: post
title: ceph线上故障处理记录
category: 技术
---



ceph故障记录

[TOC]





# mon故障

## mon无法启动
* 现象描述
  因为某些原因，导致ceph-mon无法启动。ceph-mon 无法启动，原因可能是mointor level db 损坏，数据的丢失等。在OS 正常的情况下（osd运行正常），只是ceph-mon 无法启动。
  这种情况下，剔除节点然后再 加入，会导致数据的2次均衡，如果数据量大，耗时比较久。所以，可以通过重建ceph-mon的方式来实现


* 处理
  ​

  ````
  * 处理
  1，在当前节点，停掉ceph-mon，导出monmap，然后通过monmaptool工具手动修改 monmap，修改完 再导入到 剩余的mon map里，最后通过ceph mon dump验证是否删除成功

      ceph-mon -i {mon-id} --extract-monmap {map-path}
      # for example,
      ceph-mon -i a --extract-monmap /tmp/monmap
      
      monmaptool {map-path} --rm {mon-id}
      #for example
      monmaptool /tmp/monmap --rm b
      
      ceph-mon -i {mon-id} --inject-monmap {map-path}
      # for example,
      ceph-mon -i a --inject-monmap /tmp/monmap
      
  2，删除成功后，开始重新添加 ，通过ceph-mon leader 导出 现有的monmap

      ceph -m 10.10.10.62 mon getmap -o monmap

  3，在/data/目录生成ceph-mon 目录，mon-id 是5个字符串，

      ceph-mon -i wsgws --mkfs --monmap monmap

  4，将新的ceph-mon 加入monmap

      ceph mon add wsgws 10.10.10.61:6789

  4,重启 ezs3-agent 会将 新的mon加入到 ceph.conf
  ````

  ​


# OSD故障

## osd level db膨胀compact无效

> ceph 0.67版本



现象

```
osd 所在的路径 omap大小超过50G
```



处理

```
通过 level db tool 工具来修复，步骤如下：，

停掉要修复的osd.11

执行  ./leveldb_tool -i 11

执行完，新 生产 omap_new 目录

将老的 omap 目录备份，mv omap omap_bak

然后 mv omap_new omap

启动 osd.11

```

工具链接地址：<https://gitlab.com/dragon200518/my-own/tree/master/PYTHON/omaptool/leveldb_tool>





## osd的omap leveldb sst文件损坏



亚信环境，0.67ceph 版本

打开sst的文件，发现有java的log 记录

打开osd level db的LOG

```
2019/06/21-23:09:58.442027 7fe1fa1b9bc0 Too many L0 files; waiting...
2019/06/21-23:10:06.442146 7fe1f2156700 Compacting 1@2 + 1@3 files
2019/06/21-23:10:06.442200 7fe1fa1b9bc0 Too many L0 files; waiting...
2019/06/21-23:10:06.493726 7fe1f2156700 compacted to: files[ 13 8 53 156 0 0 0 ]
2019/06/21-23:10:06.494047 7fe1f2156700 Compaction error: Invalid argument: not an sstable (bad magic number)
2019/06/21-23:10:06.494068 7fe1f2156700 Waiting after background compaction error: Invalid argument: not an sstable (bad magic number)
2019/06/21-23:10:06.494102 7fe1fa1b9bc0 Too many L0 files; waiting...
2019/06/21-23:10:14.494229 7fe1f2156700 Compacting 1@2 + 1@3 files
2019/06/21-23:10:14.494286 7fe1fa1b9bc0 Too many L0 files; waiting...
2019/06/21-23:10:14.541016 7fe1f2156700 compacted to: files[ 13 8 53 156 0 0 0 ]
2019/06/21-23:10:14.541365 7fe1f2156700 Compaction error: Invalid argument: not an sstable (bad magic number)
2019/06/21-23:10:14.541386 7fe1f2156700 Waiting after background compaction error: Invalid argument: not an sstable (bad magic number)
2019/06/21-23:10:14.541418 7fe1fa1b9bc0 Too many L0 files; waiting...

```

**问题原因**

sst 文件被篡改了，被误操作写入了java的日志，另外也了解到这台机器之前被挖矿了。 处理方法就是 重做这个osd







## osd 启动，osd日志 显示 journal 返回-5



日志如下：

```
   -3> 2018-07-08 10:15:55.659738 7f2ecae94bc0  0 filestore(/data/osd.0) mount: enabling WRITEAHEAD journal mode: checkpoint is not enabled
    -2> 2018-07-08 10:15:55.916367 7f2ecae94bc0  0 journal  kernel version is 3.10.41
    -1> 2018-07-08 10:16:01.469545 7f2ecae94bc0 -1 journal FileJournal::wrap_read_bl: safe_read_exact 4885336064~40 returned -5
    
    
     0> 2018-07-08 10:16:01.471787 7f2ecae94bc0 -1 os/FileJournal.cc: In function 'void FileJournal::wrap_read_bl(off64_t, int64_t, ceph::bufferlist*, off64_t*)' thread 7f2ecae94bc0 time 2018-07-08 10:16:01.469593
os/FileJournal.cc: 1650: FAILED assert(0)
 
```

处理

```
  
  将 默认的 "journal_ignore_corruption": "false",改为 ture，临时规避。
  
  

```





## eio，kern.log报 某些扇区 无法读取

osd日志 如下， 亚信环境发现，设置 osd eio的参数，临时规避

```
=0x3976680).accept connect_seq 22 vs existing 21 state standby
    -2> 2019-01-27 03:45:16.312168 7f39802c2700  0 <cls> cls/rgw/cls_rgw.cc:1457: gc_iterate_entries end_key=1_01548531916.312165000

    -1> 2019-01-27 03:45:17.752461 7f397f2c0700  0 <cls> cls/rgw/cls_rgw.cc:1457: gc_iterate_entries end_key=1_01548531917.752457000

     0> 2019-01-27 04:00:11.552521 7f397eabf700 -1 os/FileStore.cc: In function 'virtual int FileStore::read(const coll_t&, const hobject_t&, uint64_t, size_t, ceph::bufferlist&, bool)' thread 7f397eabf700 time 2019-01-27 04:00:11.390778
os/FileStore.cc: 2557: FAILED assert(allow_eio || !m_filestore_fail_eio || got != -5)

 ceph version 0.67.9-222-g014b35f (014b35fc1ee0a1ad1f699a3705f3481a88614d36)
 1: (FileStore::read(coll_t const&, hobject_t const&, unsigned long, unsigned long, ceph::buffer::list&, bool)+0x4f8) [0x7acf98]
 2: (ReplicatedPG::build_push_op(ObjectRecoveryInfo const&, ObjectRecoveryProgress const&, ObjectRecoveryProgress*, PushOp*)+0x3ca) [0x5fa23a]
 3: (ReplicatedPG::prep_push(int, ObjectContext*, hobject_t const&, int, eversion_t, interval_set<unsigned long>&, std::map<hobject_t, interval_set<unsigned long>, std::less<hobject_t>, std::allocator<std::pair<hobject_t const, interval_set<unsigned long> > > >&, PushOp*)+0x2a9) [0x606229]
 4: (ReplicatedPG::prep_push_to_replica(ObjectContext*, hobject_t const&, int, int, PushOp*)+0x465) [0x606875]
 5: (ReplicatedPG::prep_backfill_object_push(hobject_t, eversion_t, eversion_t, int, std::map<int, std::vector<PushOp, std::allocator<PushOp> >, std::less<int>, std::allocator<std::pair<int const, std::vector<PushOp, std::allocator<PushOp> > > > >*)+0x586) [0x6073f6]
 6: (ReplicatedPG::recover_backfill(int, ThreadPool::TPHandle&)+0x22d3) [0x624923]
 7: (ReplicatedPG::start_recovery_ops(int, PG::RecoveryCtx*, ThreadPool::TPHandle&)+0x865) [0x628005]
                                                                                                           
```









## op 一直 block 在某个 osd上

> 亚信 5.2集群，恢复过程中 发生了 osd down，然后就 发生了 op block



* 现象

  ceph health detail 显示 一个 op 卡了几千秒，并且 是在 osd.13 上面，进一步检查改op如下，发现在 waiting for missing object

  ```
  root@Storage-c4:~# ceph daemon osd.13 dump_ops_in_flight
  { "num_ops": 1,
    "ops": [
          { "description": "osd_op(client.754329591.0:45771588 rbd_header.6051ba239be4ee8 [watch add cookie 1 ver 0] 18.839a3da6 e43577)",
            "rmw_flags": 4,
            "received_at": "2018-05-24 15:07:21.442995",
            "age": "1018.638811",
            "duration": "83.825999",
            "flag_point": "delayed",
            "client_info": { "client": "client.754329591",
                "tid": 45771588},
            "events": [
                  { "time": "2018-05-24 15:07:33.051865",
                    "event": "waiting_for_osdmap"},
                  { "time": "2018-05-24 15:07:33.224930",
                    "event": "reached_pg"},
                  { "time": "2018-05-24 15:08:45.268952",
                    "event": "reached_pg"},
                  { "time": "2018-05-24 15:08:45.268994",
                    "event": "waiting for missing object"}]}]}
                    
  ```

* 处理 

  因为pool 设置了3副本，尽管 osd.13 上 该object的信息 已经丢失，但是 其他 osd 上还有副本，所以 处理思路就是 将 osd.13 上 该 object 所在的pg 重命名，让 osd.13 可以 从 它的 副本 osd 上 复制过来。

  查询，该object 位于 

  ```
  ^Croot@Storage-c4:/data/osd.13/current# ceph osd map pool2-vol rbd_header.6051ba239be4ee8
  osdmap e43630 pool 'pool2-vol' (18) object 'rbd_header.6051ba239be4ee8' -> pg 18.839a3da6 (18.1a6) -> up [13,16,7] acting [13,16,7]
  root@Storage-c4:/data/osd.13/current# 
  root@Storage-c4:/data/osd.13/current# cd 18.1a6_head/
  ```

  停掉 osd.13后，进入 /data/osd.13/current目录，`mv 18.1a6_head bak_18.1a6_head_bak`，然后重启 osd.13

  ​

  ​




## FAILED assert(oi.version == i->first)

* 现象描述

  5.5的集群在恢复的过程种，osd down了，检查osd 日志发现如下报错(关键是其中的assert输出），

  ```
      -5> 2018-05-19 19:05:25.095366 7fb7c377dbc0 20 read_log 41589'230169678 (41579'230169463) modify   645c9725/rbd_data.10cfd565c115fe13a.00000000000006bb/head//18 by client.217933292.0:3978897 2018-05-19 19:08:31.855998
      -4> 2018-05-19 19:05:25.095383 7fb7c377dbc0 20 read_log 41590'230169679 (41589'230169676) modify   ab2c325/rbd_data.101cdc19c5114325c.00000000000127b6/head//18 by client.30285026.0:217491869 2018-05-19 19:05:05.048896
      -3> 2018-05-19 19:05:25.095403 7fb7c377dbc0 20 read_log 1 divergent_priors
      -2> 2018-05-19 19:05:25.101481 7fb7c377dbc0 10 read_log checking for missing items over interval (0'0,41590'230169679]
      -1> 2018-05-19 19:05:25.115614 7fb7c377dbc0 15 read_log  missing 41568'230157800 (41568'230157799) modify   94f51325/rbd_data.1215645d44a26d389.0000000000000fd1/head//18 by client.558170568.0:79615064 2018-05-19 17:17:57.372314 (have 41529'230151680)
       0> 2018-05-19 19:05:25.117403 7fb7c377dbc0 -1 osd/PGLog.cc: In function 'static bool PGLog::read_log(ObjectStore*, coll_t, hobject_t, const pg_info_t&, std::map<eversion_t, hobject_t>&, PGLog::IndexedLog&, pg_missing_t&, std::ostringstream&, std::set<std::basic_string<char> >*)' thread 7fb7c377dbc0 time 2018-05-19 19:05:25.116180
  osd/PGLog.cc: 738: FAILED assert(oi.version == i->first)

  ceph version 0.67.9-239-g6915a1e (6915a1e36e9a7ae9365b8b4abd10f52d7fe249a9)
  1: (PGLog::read_log(ObjectStore*, coll_t, hobject_t, pg_info_t const&, std::map<eversion_t, hobject_t, std::less<eversion_t>, std::allocator<std::pair<eversion_t const, hobject_t> > >&, PGLog::IndexedLog&, pg_missing_t&, std::basic_ostringstream<char, std::char_traits<char>, std::allocator<char> >&, std::set<std::string, std::less<std::string>, std::allocator<std::string> >*)+0x7b0) [0x76bc60]
  2: (PG::read_state(ObjectStore*, ceph::buffer::list&)+0x26f) [0x735c3f]
  3: (OSD::load_pgs()+0x15bd) [0x6a0d8d]
  4: (OSD::init()+0x103b) [0x6a8d2b]
  5: (main()+0x1d00) [0x5ccdc0]
  6: (__libc_start_main()+0xed) [0x7fb7c0f9276d]
  7: /usr/bin/ceph-osd() [0x5d06f5]
  ```

* 处理

  默认osd的debug level 比较低，看不出  ，， 将osd debug （debug_filestore)以发现其中多了些 filestor的 debug 信息），知道 pg 后，将对应 pg mv走：

  ```
      -3> 2018-05-19 19:52:31.377997 7f967388cbc0 15 read_log  missing 41568'230157800 (41568'230157799) modify   94f51325/rbd_data.1215645d44a26d389.0000000000000fd1/head//18 by client.558170568.0:79615064 2018-05-19 17:17:57.372314 (have 41529'230151680)
      -2> 2018-05-19 19:52:31.378019 7f967388cbc0 15 filestore(/data/osd.21) getattr 18.325_head/4a89f325/volume-ead13e2c-3522-4150-aad3-de6386ef4e13.rbd/head//18 '_'
      -1> 2018-05-19 19:52:31.378393 7f967388cbc0 10 filestore(/data/osd.21) getattr 18.325_head/4a89f325/volume-ead13e2c-3522-4150-aad3-de6386ef4e13.rbd/head//18 '_' = 0
       0> 2018-05-19 19:52:31.379622 7f967388cbc0 -1 osd/PGLog.cc: In function 'static bool PGLog::read_log(ObjectStore*, coll_t, hobject_t, const pg_info_t&, std::map<eversion_t, hobject_t>&, PGLog::IndexedLog&, pg_missing_t&, std::ostringstream&, std::set<std::basic_string<char> >*)' thread 7f967388cbc0 time 2018-05-19 19:52:31.378418
  osd/PGLog.cc: 738: FAILED assert(oi.version == i->first)

  ceph version 0.67.9-239-g6915a1e (6915a1e36e9a7ae9365b8b4abd10f52d7fe249a9)
  1: (PGLog::read_log(ObjectStore*, coll_t, hobject_t, pg_info_t const&, std::map<eversion_t, hobject_t, std::less<eversion_t>, std::allocator<std::pair<eversion_t const, hobject_t> > >&, PGLog::IndexedLog&, pg_missing_t&, std::basic_ostringstream<char, std::char_traits<char>, std::allocator<char> >&, std::set<std::string, std::less<std::string>, std::allocator<std::string> >*)+0x7b0) [0x76bc60]
  2: (PG::read_state(ObjectStore*, ceph::buffer::list&)+0x26f) [0x735c3f]
  3: (OSD::load_pgs()+0x15bd) [0x6a0d8d]
  4: (OSD::init()+0x103b) [0x6a8d2b]
  5: (main()+0x1d00) [0x5ccdc0]
  ```

  ​

## FAILED assert(0 == "unexpected error")

* 现象描述
  ceph 0.67版本，osd的日志中有 如下报错

        -3> 2017-10-18 16:44:55.365325 7fdb5a6c0bc0  0 filestore(/data/osd.13)  error (39) Directory not empty not handled on operation 21 (28581659019.0.0, or op 0, counting from 0)
        -2> 2017-10-18 16:44:55.365340 7fdb5a6c0bc0  0 filestore(/data/osd.13) ENOTEMPTY suggests garbage data in osd data dir
        -1> 2017-10-18 16:44:55.365343 7fdb5a6c0bc0  0 filestore(/data/osd.13)  transaction dump:
    { "ops": [
            { "op_num": 0,
              "op_name": "rmcoll",
              "collection": "24.36_TEMP"}]}
         0> 2017-10-18 16:44:55.367380 7fdb5a6c0bc0 -1 os/FileStore.cc: In function 'unsigned int FileStore::_do_transaction(ObjectStore::Transaction&, uint64_t, int)' thread 7fdb5a6c0bc0 time 2017-10-18 16:44:55.365391
    os/FileStore.cc: 2471: FAILED assert(0 == "unexpected error")

* 处理
  可以看到日志 提示 24.36_TEMP 中存在垃圾数据，手动进入该目录，发现有个文件，删除即可。

## unfound object处理
* 现象描述
  ceph 0.67的版本，osd故障恢复后，发现集群有2个unfound object

* 处理

    1，确认unfoud 所在pg
     root@Storage-b6:/etc/cron.d# ceph health detail |grep unfound
    HEALTH_WARN 1219 pgs backfill; 1 pgs backfilling; 1079 pgs degraded; 2 pgs recovering; 1222 pgs stuck unclean; recovery 3661003/50331903 degraded (7.274%); 3/25061739 unfound (0.000%);  recovering 5 o/s, 25439KB/s; noout flag(s) set
    pg 17.2c3 is active+recovering+degraded, acting [20,2], 1 unfound
    pg 17.2b1 is active+recovering+degraded, acting [20,2], 2 unfound
    unable to open OSD superblock on /data/osd.2
    2，如果想要确认具体是哪些 object，可以用
    ceph pg 17.2b1 list_missing
    3，标记object
    ceph pg 2.5 mark_unfound_lost revert
    这个命令不一定是 roll back，如果object 是新的，ceph object 有版本信息吗？

* 参考
  http://docs.ceph.com/docs/argonaut/ops/manage/failures/osd/



## unable to open OSD superblock on /data/osd
* 现象描述

    2017-10-10 12:00:15.534831 7fda0e7b97c0 -1 ^[[0;31m ** ERROR: unable to open OSD superblock on /data/osd.2: (2) No such file or directory^[[0m
    2017-10-10 14:14:58.196144 7f71363c27c0  0 ceph version 0.94.9-1061-g9bcd143 (9bcd143a70b819086b9e58e0799ba93364d7ee31), process ceph-osd, pid 7250
    2017-10-10 14:14:58.198476 7f71363c27c0 -1 ^[[0;31m ** ERROR: unable to open OSD superblock on /data/osd.2: (2) No such file or directory^[[0m
    该报错说明osd盘并没有mount上，所以superblock文件无法打开（superblock ），df -h检查 osd盘确实没有mount上。

    检查/var/log/kern.log中有如下报错
    Oct 10 14:14:54 142 kernel: [  120.133734] EXT4-fs (dm-0): ext4_check_descriptors: Checksum for group 32640 failed (58283!=0)
    Oct 10 14:14:54 142 kernel: [  120.133739] EXT4-fs (dm-0): group descriptors corrupted!
    Oct 10 14:14:54 142 kernel: [  120.214674] EXT4-fs (dm-1): ext4_check_descriptors: Inode bitmap for group 34688 not in group (block 2037277037)!
    Oct 10 14:14:54 142 kernel: [  120.214679] EXT4-fs (dm-1): group descriptors corrupted!

> tune2fs 检查dm设备，发现并没有EXT-4 错误记录，也说明 tune2fs 也有不及时的时候。

* 处理方法

    e2fsck -p /dev/dm-0 提示不行
    e2fsck -y /dev/dm-0 进行勉强修复，后面需要reformat osd进行彻底修复


* 原因说明
  检查disk cache policy是disabled的，之前设备关机有强制关机（raid卡passthrough的converger环境）






# mds故障

## 南京lab converger 集群，mds damage

> ceph 10.X，jewer 版本

现象

```
[7/19 下午5:14] Bean Li
    root@converger-126:/var/share# ceph -s
    cluster 7225322e-ae69-431a-9e17-b2701ab76968
     health HEALTH_WARN
            mds0: Client converger-124:guest failing to respond to capability release
 
root@coverger-125:/var/share/ezfs/shareroot# cat /sys/kernel/debug/ceph/7225322e-ae69-431a-9e17-b2701ab76968.client263005808/mdsc
76608    mds0    getattr     #100000002eb
76636    mds0    getattr     #100000002eb
76642    mds0    getattr     #100000002eb
77444    mds0    getattr     #100000002eb
79328    mds0    getattr     #100000002eb
80376    mds0    getattr     #100000002eb
81392    mds0    getattr     #100000002eb

 
 root@converger-126:~# ceph daemon mds.gsnjf inode get_path 0x100000002eb
{
    "base_ino": 1,
    "relative_path": "shareroot\/share\/.iormstats.sf"
}


```



active mds log

```
2019-07-19 10:18:25.199411 7fa7907fe700  0 mds.0.cache creating system inode with ino:100
2019-07-19 10:18:25.200984 7fa7907fe700  0 mds.0.cache creating system inode with ino:1
2019-07-19 11:04:44.777559 7fa7937ff700 -1 mds.0.cache.strays Dentry [dentry #100/stray9/100013597bd [2,head] auth (dversion lock) v=168708994 inode=0x7fa7939af600 state=new | request=0 lock=0 inodepin=1 dirty=1 authpin=0 0x7fa79395e300] was purgeable but no longer is!
2019-07-19 11:04:49.778561 7fa7937ff700 -1 mds.0.cache.strays Dentry [dentry #100/stray9/100013597bd [2,head] auth (dversion lock) v=168708994 inode=0x7fa7939af600 state=new | request=0 lock=0 inodepin=1 dirty=1 authpin=0 0x7fa79395e300] was purgeable but no longer is!
2019-07-19 11:04:54.778648 7fa7937ff700 -1 mds.0.cache.strays Dentry [dentry #100/stray9/100013597bd [2,head] auth (dversion lock) v=168708994 inode=0x7fa7939af600 state=new | request=0 lock=0 inodepin=1 dirty=1 authpin=0 0x7fa79395e300] was purgeable but no longer is!
2019-07-19 11:04:59.778733 7fa7937ff700 -1 mds.0.cache.strays Dentry [dentry #100/stray9/100013597bd [2,head] auth (dversion lock) v=168708994 inode=0x7fa7939af600 state=new | request=0 lock=0 inodepin=1 dirty=1 authpin=0 0x7fa79395e300] was purgeable but no longer is!
2019-07-19 11:05:04.778809 7fa7937ff700 -1 mds.0.cache.strays Dentry [dentry #100/stray9/100013597bd [2,head] auth (dversion lock) v=168708994 inode=0x7fa7939af600 state=new | request=0 lock=0 inodepin=1 dirty=1 authpin=0 0x7fa79395e300] was purgeable but no longer is!
2019-07-19 11:05:09.778897 7fa7937ff700 -1 mds.0.cache.strays Dentry [dentry #100/stray9/100013597bd [2,head] auth (dversion lock) v=168708994 inode=0x7fa7939af600 state=new | request=0 lock=0 inodepin=1 dirty=1 authpin=0 0x7fa79395e300] was purgeable but no longer is!
2019-07-19 11:05:14.779030 7fa7937ff700 -1 mds.0.cache.strays Dentry [dentry #100/stray9/100013597bd [2,head] auth (dversion lock) v=168708994 inode=0x7fa7939af600 state=new | request=0 lock=0 inodepin=1 dirty=1 authpin=0 0x7fa79395e300] was purgeable but no longer is!
2019-07-19 11:05:19.779121 7fa7937ff700 -1 mds.0.cache.strays Dentry [dentry #100/stray9/100013597bd [2,head] auth (dversion lock) v=168708994 inode=0x7fa7939af600 state=new | request=0 lock=0 inodepin=1 dirty=1 authpin=0 0x7fa79395e300] was purgeable but no longer is!
2019

```

**原因**

> https://tracker.ceph.com/issues/36093



待整理命令

```
ceph daemon mds.dnyci config set debug_mds 20
 ceph daemon mds.dnyci config set debug_journaler 20
 ceph daemon mds.dnyci config set debug_mds 20
 
 onnode all 'grep "replay journaler" /var/log/ceph/ceph-mds.*.log'
 
ceph mds repaired 0
```



```

128
/var/log/history/history_19-07-22.log:19-07-22 10:07:34 ##### root pts/0 (172.17.74.102) #### /var/log/ezcloudstor #### tailf ezmds-agent.log
/var/log/history/history_19-07-22.log:19-07-22 10:07:38 ##### root pts/2 (172.17.74.102) #### /var/log/ceph #### tailf ceph-mds.uatqd.log
/var/log/history/history_19-07-22.log:19-07-22 10:08:50 ##### root pts/2 (172.17.74.102) #### /var/log/ceph #### ceph mds -h
/var/log/history/history_19-07-22.log:19-07-22 10:08:57 ##### root pts/2 (172.17.74.102) #### /var/log/ceph #### ceph mds stat
/var/log/history/history_19-07-22.log:19-07-22 10:09:49 ##### root pts/2 (172.17.74.102) #### /var/log/ceph #### ceph mds getmap
/var/log/history/history_19-07-22.log:19-07-22 10:10:15 ##### root pts/2 (172.17.74.102) #### /var/log/ceph #### ceph mds repaired 0
/var/log/history/history_19-07-22.log:19-07-22 10:11:14 ##### root pts/2 (172.17.74.102) #### /root #### ceph mds repaired -h
/var/log/history/history_19-07-22.log:19-07-22 10:11:29 ##### root pts/2 (172.17.74.102) #### /root #### ceph mds repaired
/var/log/history/history_19-07-22.log:19-07-22 10:11:41 ##### root pts/2 (172.17.74.102) #### /root #### ceph mds repaired rand0
/var/log/history/history_19-07-22.log:19-07-22 10:11:47 ##### root pts/2 (172.17.74.102) #### /root #### ceph mds repaired rank0
/var/log/history/history_19-07-22.log:19-07-22 10:11:54 ##### root pts/2 (172.17.74.102) #### /root #### ceph mds repaired rank 0
/var/log/history/history_19-07-22.log:19-07-22 10:11:57 ##### root pts/2 (172.17.74.102) #### /root #### ceph mds repaired 0
/var/log/history/history_19-07-22.log:19-07-22 10:15:15 ##### root pts/0 (172.17.74.102) #### /var/log/ezcloudstor #### tailf ezmds-agent.log
/var/log/history/history_19-07-22.log:19-07-22 10:15:15 ##### root pts/0 (172.17.74.102) #### /var/log/ezcloudstor #### tailf ezmds-agent.log
/var/log/history/history_19-07-22.log:19-07-22 10:15:15 ##### root pts/0 (172.17.74.102) #### /var/log/ezcloudstor #### tailf ezmds-agent.log
/var/log/history/history_19-07-22.log:19-07-22 13:47:40 ##### root pts/0 (10.10.10.124) #### /root #### ceph daemon mds.ptexh config set debug_mds 20
/var/log/history/history_19-07-22.log:19-07-22 13:47:44 ##### root pts/0 (10.10.10.124) #### /root #### ceph daemon mds.ptexh config set debug_journaler 20
/var/log/history/history_19-07-22.log:19-07-22 13:51:02 ##### root pts/0 (10.10.10.124) #### /root #### less /var/log/ceph/ceph-mds.ptexh.log
/var/log/history/history_19-07-22.log:19-07-22 13:52:37 ##### root pts/0 (10.10.10.124) #### /root #### ceph daemon mds.ptexh config set debug_journaler 20
/var/log/history/history_19-07-22.log:19-07-22 13:52:42 ##### root pts/0 (10.10.10.124) #### /root #### ceph daemon mds.ptexh config set debug_mds 20
/var/log/history/history_19-07-22.log:19-07-22 13:53:52 ##### root pts/0 (10.10.10.124) #### /root #### ceph daemon mds.ptexh config set debug_mds 0
/var/log/history/history_19-07-22.log:19-07-22 13:53:56 ##### root pts/0 (10.10.10.124) #### /root #### ceph daemon mds.ptexh config set debug_journaler 0
root@converger-128:~# 

```



##南医大 mds 的damage，，补充？？





现象

```
root@converger-124:~# ceph -s
    cluster 7225322e-ae69-431a-9e17-b2701ab76968
     health HEALTH_ERR
            mds rank 0 is damaged
            mds cluster is degraded
            noout flag(s) set
            mon.buvro low disk space
     monmap e5: 5 mons at {amehj=10.10.10.128:6789/0,buvro=10.10.10.124:6789/0,hcgqm=10.10.10.125:6789/0,kbzjw=10.10.10.126:6789/0,wivnm=10.10.10.127:6789/0}
            election epoch 25312, quorum 0,1,2,3,4 buvro,hcgqm,kbzjw,wivnm,amehj
      fsmap e26913: 0/1/1 up, 2 up:standby, 1 damaged
     osdmap e63146: 10 osds: 10 up, 10 in
            flags noout,sortbitwise,require_jewel_osds
      pgmap v19520923: 13056 pgs, 24 pools, 29060 GB data, 7547 kobjects
            58244 GB used, 16276 GB / 74521 GB avail
               13056 active+clean
  client io 143 kB/s rd, 182 op/s rd, 0 op/s wr

```











## mdsc 卡住

联通环境，ceph0.67版本



**现象**

```
cat /sys/kernel/debug/ceph/*/mdsc，发现有 部分op block 在相同的 文件上，并且每个节点上 都会看到这个inode


9070104625	mds0	getattr	 #1000000a3bf
9070104727	mds0	getattr	 #1000000a3bf
9070104783	mds0	getattr	 #1000000a3bf
9070104887	mds0	getattr	 #1000000a3bf
9070104938	mds0	getattr	 #1000000a3bf
9070105036	mds0	getattr	 #1000000a3bf
9070105088	mds0	getattr	 #1000000a3bf
9070105196	mds0	getattr	 #1000000a3bf
9070105243	mds0	getattr	 #1000000a3bf
9070105346	mds0	getattr	 #1000000a3bf
9070105400	mds0	getattr	 #1000000a3bf
9070105502	mds0	getattr	 #1000000a3bf
9070105556	mds0	getattr	 #1000000a3bf
9070105665	mds0	getattr	 #1000000a3bf
9070105712	mds0	getattr	 #1000000a3bf
9070105812	mds0	getattr	 #1000000a3bf

```



解决

```
1,尝试重启 mds 服务，没有效果
2, 为了防止 客户端的访问干扰，先停止了 所有客户端的nova-compute 进程，然后存储端 依次 重启各个节点，检查每个节点的mdsc是否还有卡住。

```





## mds/MDCache.cc: 216: FAILED assert(inode_map.count(in->vino()) == 0)

mds 状态为replay，一直无法恢复，ceph 的版本为 0.67的老版本

        -4> 2017-10-06 21:21:23.199440 7f5c63631700  0 mds.0.cache.dir(609) _fetched  badness: got (but i already had) [inode 1000004e2b0 [2,head] /shareroot/block-hz/routinebackup/2017-09-29/log/check_img.5b8ebf83-30c9-4c3c-b6be-bb37544d672f.log auth v296 s=0 n(v0 1=1+0) (iversion lock) 0x368f050] mode 33188 mtime 2017-09-29 05:09:03.364737
        -3> 2017-10-06 21:21:23.199470 7f5c63631700  0 log [ERR] : loaded dup inode 1000004e2b0 [2,head] v4917497405 at ~mds0/stray9/1000004e2b0, but inode 1000004e2b0.head v296 already exists at /shareroot/block-hz/routinebackup/2017-09-29/log/check_img.5b8ebf83-30c9-4c3c-b6be-bb37544d672f.log
        -2> 2017-10-06 21:21:23.199478 7f5c63631700  0 mds.0.cache.dir(609) _fetched  badness: got (but i already had) [inode 1000004e2b1 [2,head] /shareroot/block-hz/routinebackup/2017-09-29/log/check_img.bf45b4c0-717f-4ae4-b709-0f3cb8cd8650.log auth v298 s=0 n(v0 1=1+0) (iversion lock) 0x3695590] mode 33188 mtime 2017-09-29 05:09:03.413889
        -1> 2017-10-06 21:21:23.199500 7f5c63631700  0 log [ERR] : loaded dup inode 1000004e2b1 [2,head] v4917497407 at ~mds0/stray9/1000004e2b1, but inode 1000004e2b1.head v298 already exists at /shareroot/block-hz/routinebackup/2017-09-29/log/check_img.bf45b4c0-717f-4ae4-b709-0f3cb8cd8650.log
         0> 2017-10-06 21:21:23.201731 7f5c63631700 -1 mds/MDCache.cc: In function 'void MDCache::add_inode(CInode*)' thread 7f5c63631700 time 2017-10-06 21:21:23.199831
    mds/MDCache.cc: 216: FAILED assert(inode_map.count(in->vino()) == 0)

* 处理方法
  ceph.conf 中 加入可以把mds config: mds_wipe_ino_prealloc=true
  重启mds，然后 ceph.conf再 删除该配置项

## mds/journal.cc: 1397: FAILED assert(mds->sessionmap.version == cmapv)

active mds 的log 中有该assert

* 处理方法
  因为ceph 的版本0.67，重新 build 了mds 的binary，拿掉 该assert

## dir 1000002c8e2 object missing on disk
    一开始mds 如下报错,6.3 环境
    2018-03-14 21:00:13.997147 7fdf8dbf6700 -2 log_channel(cluster) log [ERR] : dir 1000002c8e2 object missing on disk; some files may be lost
    处理后
    2018-03-15 09:30:29.562972 7fea581937c0  0 ceph version 0.94.10-1158-gc4c02ae (c4c02aef50cc77c379bcc589adb331244183e0f8), process ceph-mds, pid 128708
    2018-03-15 09:30:29.565343 7fea581937c0 -1 mds.-1.0 log_to_monitors {default=true}
    2018-03-15 09:31:32.254899 7fea4fff6700  0 mds.0.cache creating system inode with ino:100
    2018-03-15 09:31:32.291740 7fea4fff6700  0 mds.0.cache creating system inode with ino:1
    2018-03-15 09:31:36.231516 7fea4fff6700 -2 log_channel(cluster) log [ERR] : bad backtrace on dir ino 1000002c8e2
    2018-03-15 09:31:36.254626 7fea4fff6700 -1 *** Caught signal (Aborted) **
     in thread 7fea4fff6700
     ceph version 0.94.10-1158-gc4c02ae (c4c02aef50cc77c379bcc589adb331244183e0f8)
     1: /usr/bin/ceph-mds() [0x8a0f28]
     2: (()+0xfcb0) [0x7fea57b41cb0]
     3: (gsignal()+0x35) [0x7fea56284035]
     4: (abort()+0x17b) [0x7fea5628779b]

* 解决方法，henry 提供
    rados -p metadata setomapheader 1000002c8e2.00000000 omapheader
