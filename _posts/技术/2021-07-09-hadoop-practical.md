---
layout: post
title: hadoopr实践一
category: 技术
---




# 背景

为基于k8s 部署做热身， 熟悉hadoop，hdfs，nameNode，dataNode，yarn，spark 相关概念

# 部署

部署步骤主要包括 各节点主机环境配置，hadoop的配置，集群初始化，集群服务启动。本次环境的规划信息如下：

| node            | ip           | 角色                                                        |
| --------------- | ------------ | ----------------------------------------------------------- |
| hd1.hdfscluster | 172.17.73.64 | nameNode, JournalNode                                       |
| hd2             | 172.17.73.65 | nameNode, JournalNode，dataNode,ResourceManger, NodeManager |
| hd3             | 172.17.73.66 | datanode，journalNode，ResourceManager，nodeManger          |

3节点，创建hadoop 用户

软件目录

```
sudo chown -R hadoop:hadoop /opt/hadoop/
```

数据目录

```
sudo mkdir /data/hadoop/data
sudo mkdir /data/hadoop/name
sudo mkdir /data/hadoop/tmp
sudo mkdir /data/hadoop/log
sudo mkdir /data/hadoop/checkpoint
sudo mkdir /data/hadoop/journalnode



```

## 主机配置

- 免密登陆

- /etc/hosts 文件修改(hostname 加了个域名)

  ```
  172.17.73.64 hd1 
  172.17.73.65 hd2 
  172.17.73.66 hd3 
  ```

  

- java 配置 和 hadoop

  ```
  export PATH=/opt/jdk-11.0.1/bin:$PATH
  
  
  export HADOOP_HOME=/opt/hadoop/hadoop-3.3.1
  export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
  export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
  
  ```

  



## hadoop 配置文件

主要涉及，nameNode 和yarn的高可用 通过外部zookeeper

```
hadoop@hd1:/opt/hadoop/hadoop-3.3.1/etc/hadoop$ vim core-site.xml 
hadoop@hd1:/opt/hadoop/hadoop-3.3.1/etc/hadoop$ vim hdfs-site.xml 
hadoop@hd1:/opt/hadoop/hadoop-3.3.1/etc/hadoop$ vim yarn-site.xml 


```

### core-site.xml

>  fs.defaultFS 用nameservice id 后一直有问题，先临时指定具体的namenode，但是 nameNode 切换会有影响。

```
hadoop@hd1:/opt/hadoop/hadoop-3.3.1$ cat etc/hadoop/core-site.xml 
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

<!-- Put site-specific property overrides in this file. -->
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://hd1:9000</value>     
  </property>
  <property>
    <name>dfs.journalnode.edits.dir</name>
    <value>/data/hadoop/journalnode</value>
  </property>
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/data/hadoop/tmp</value>
  </property>
  <property>
    <name>fs.trash.interval</name>
    <value>1440</value>
  </property>
  <property>
    <name>io.file.buffer.size</name>
    <value>65536</value>
  </property>
  <property>
    <name>ha.zookeeper.quorum</name>
    <value>172.17.73.67:2181,172.17.73.68:2181,172.17.73.68:2182</value>
  </property>
</configuration>
hadoop@hd1:/opt/hadoop/hadoop-3.3.1$ 


```



### hdfs-site.xml

```
hadoop@hd1:/opt/hadoop/hadoop-3.3.1$ cat etc/hadoop/hdfs-site.xml 
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

<!-- Put site-specific property overrides in this file. -->

<configuration>
  <property>
    <name>dfs.replication</name>
    <value>2</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>/data/hadoop/name</value>
  </property>
  <property>
    <name>dfs.blocksize</name>
    <value>67108864</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>/data/hadoop/data</value>
  </property>
  <property>
    <name>dfs.namenode.checkpoint.dir</name>
    <value>/data/hadoop/checkpoint</value>
  </property>
  <property>
    <name>dfs.namenode.handler.count</name>
    <value>10</value>
  </property>
  <property>
    <name>dfs.datanode.handler.count</name>
    <value>10</value>
  </property>
  <property>
    <name>dfs.nameservices</name>
    <value>ns</value>
  </property>
  <property>
    <name>dfs.ha.namenodes.ns</name>
    <value>nn1,nn2</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.ns.nn1</name>
    <value>hd1:9000</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.ns.nn2</name>
    <value>hd2:9000</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.ns.nn1</name>
    <value>hd1:50070</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.ns.nn2</name>
    <value>hd2:50070</value>
  </property>
  <property>
    <name>dfs.namenode.shared.edits.dir</name>
    <value>qjournal://hd1:8485;hd2:8485;hd3:8485/ns</value>
  </property>
  <property>
    <name>ds.client.failover.proxy.provider.ns:</name>
    <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
  </property>
  <property>
    <name>dfs.ha.automatic-failover.enabled.ns</name>
    <value>true</value>
  </property>
  <property>
    <name>dfs.ha.fencing.methods</name>
    <value>shell(/bin/true)</value>
  </property>
</configuration>
hadoop@hd1:/opt/hadoop/hadoop-3.3.1$ 

```



### yarn-site.xml

```
hadoop@hd1:/opt/hadoop/hadoop-3.3.1$ cat etc/hadoop/yarn-site.xml 
<configuration>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.resourcemanager.ha.enabled</name>
    <value>true</value>
  </property>
  <property>
    <name>yarn.resourcemanager.cluster-id</name>
    <value>ns</value>
  </property>
  <property>
    <name>yarn.resourcemanager.ha.rm-ids</name>
    <value>rm1,rm2</value>
  </property>
  <property>
    <name>yarn.resourcemanager.hostname.rm1</name>
    <value>hd2</value>
  </property>
  <property>
    <name>yarn.resourcemanager.hostname.rm2</name>
    <value>hd3</value>
  </property>
  <property>
    <name>yarn.resourcemanager.webapp.address.rm1</name>
    <value>hd2:8088</value>
  </property>
  <property>
    <name>yarn.resourcemanager.webapp.address.rm2</name>
    <value>hd3:8088</value>
  </property>
  <property>
    <name>yarn.resourcemanager.zk-address</name>
    <value>172.17.73.67:2181,172.17.73.68:2181,172.17.73.68:2182</value>
  </property>
</configuration>
hadoop@hd1:/opt/hadoop/hadoop-3.3.1$ 

```















## 初始化

active namenode 格式化后， standby namenode 配置从命令

````
#hd1 ,hd2, hd3 分别启动 journalNode
hdfs --daemon start journalnode

#hd1 执行格式化，并且启动 nameNode ，并且初始化 zookeeper 
hdfs namenode -format
hdfs --daemon start namenode
hdfs zkfc -formatZK


#hd2 同步nameNode，并且启动
hdfs namenode -bootstrapStandby
hdfs --daemon start namenode


#hd1 ,hd2 执行 zookeeper同步进程
hdfs --daemon start zkfc

````



## 启动服务

可以通过前台启动，和后台启动，根据开始规划的角色 在各个节点启动相应的服务

```
hdfs journalnode  #前台启动

后台启动
hadoop@hd1:~$ hdfs --daemon start journalnode

hdfs --daemon start namenode

hdfs --daemon start datanode


hadoop@hd3:~$ yarn --daemon start resourcemanager
yarn --daemon start nodemanager


```

也可以通过 start-all.sh 来统一启动。已经启动的程序，该脚本会检测到，不会重复启动。



## 各节点 服务检查



YARN总体上也Master/slave架构——ResourceManager/NodeManager。前者(RM)负责对各个NodeManager(NM)上的资源进行统一管理和调度



- resourceManger 状态

  ```
  hadoop@hd3:~$ yarn rmadmin -getServiceState rm1
  standby
  hadoop@hd3:~$ yarn rmadmin -getServiceState rm2
  active
  hadoop@hd3:~$ 
  
  
  ```


```
hadoop@hd1:/opt/hadoop/hadoop-3.3.1$ jps
5029 JournalNode
7546 DFSZKFailoverController
7307 NameNode
14461 Jps
hadoop@hd1:/opt/hadoop/hadoop-3.3.1$ 

hadoop@hd2:/data/hadoop$ jps
5202 NameNode
18884 Jps
3893 JournalNode
5515 NodeManager
5387 DFSZKFailoverController
hadoop@hd2:/data/hadoop$ 

hadoop@hd3:~$ jps
4786 ResourceManager
8883 Jps
4653 DataNode
3727 JournalNode
hadoop@hd3:~$ 


```



## web访问

### namenode 状态

````
http://172.17.73.64:50070/explorer.html#/

````

其中可以看到 data node的信息

| Node                                       | Http Address                        | Last contact | Last Block Report | Used  | Non DFS Used | Capacity  | Blocks | Block pool used | Version |
| :----------------------------------------- | :---------------------------------- | :----------- | :---------------- | :---- | :----------- | :-------: | :----- | :-------------- | :------ |
| /default-rack/hd3:9866 (172.17.73.66:9866) | [http://hd3:9864](http://hd3:9864/) | 0s           | 339m              | 28 KB | 12.89 GB     | 491.15 GB | 0      | 28 KB (0%)      | 3.3.1   |



name node web 整体输出

| Namespace:     | hdfscluster                                                  |
| :------------- | ------------------------------------------------------------ |
| Namenode ID:   | nn1                                                          |
| Started:       | Thu Aug 05 15:47:04 +0800 2021                               |
| Version:       | 3.3.1, ra3b9c37a397ad4188041dd80621bdeefc46885f2             |
| Compiled:      | Tue Jun 15 13:13:00 +0800 2021 by ubuntu from (HEAD detached at release-3.3.1-RC3) |
| Cluster ID:    | CID-8ce922b6-2d92-4e78-859f-118a98e3116d                     |
| Block Pool ID: | BP-273953418-172.17.73.64-1628148818373                      |





| Configured Capacity:                                         | 491.15 GB                                |
| :----------------------------------------------------------- | ---------------------------------------- |
| Configured Remote Capacity:                                  | 0 B                                      |
| DFS Used:                                                    | 28 KB (0%)                               |
| Non DFS Used:                                                | 12.89 GB                                 |
| DFS Remaining:                                               | 453.24 GB (92.28%)                       |
| Block Pool Used:                                             | 28 KB (0%)                               |
| DataNodes usages% (Min/Median/Max/stdDev):                   | 0.00% / 0.00% / 0.00% / 0.00%            |
| [Live Nodes](http://172.17.73.64:50070/dfshealth.html#tab-datanode) | 1 (Decommissioned: 0, In Maintenance: 0) |
| [Dead Nodes](http://172.17.73.64:50070/dfshealth.html#tab-datanode) | 0 (Decommissioned: 0, In Maintenance: 0) |
| [Decommissioning Nodes](http://172.17.73.64:50070/dfshealth.html#tab-datanode) | 0                                        |
| [Entering Maintenance Nodes](http://172.17.73.64:50070/dfshealth.html#tab-datanode) | 0                                        |
| [Total Datanode Volume Failures](http://172.17.73.64:50070/dfshealth.html#tab-datanode-volume-failures) | 0 (0 B)                                  |
| Number of Under-Replicated Blocks                            | 0                                        |
| Number of Blocks Pending Deletion (including replicas)       | 0                                        |
| Block Deletion Start Time                                    | Thu Aug 05 15:47:04 +0800 2021           |
| Last Checkpoint Time                                         | Fri Aug 06 11:52:31 +0800 2021           |
| Enabled Erasure Coding Policies                              | RS-6-3-1024k                             |



| Journal Manager                                              | State                                                        |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| QJM to [172.17.73.64:8485, 172.17.73.65:8485, 172.17.73.66:8485] | Writing segment beginning at txid 1211. 172.17.73.64:8485 (Written txid 1211), 172.17.73.65:8485 (Written txid 1211), 172.17.73.66:8485 (Written txid 1211) |
| FileJournalManager(root=/data/hadoop/name)                   | EditLogFileOutputStream(/data/hadoop/name/current/edits_inprogress_0000000000000001211) |



NameNode storage

| **Storage Directory** | **Type**        | **State** |
| --------------------- | --------------- | --------- |
| /data/hadoop/name     | IMAGE_AND_EDITS | Active    |





DFS Storage Type

| Storage Type | Configured Capacity | Capacity Used | Capacity Remaining | Block Pool Used | Nodes In Service |
| :----------- | :------------------ | :------------ | :----------------- | :-------------- | :--------------- |
| DISK         | 491.15 GB           | 28 KB (0%)    | 453.24 GB (92.28%) | 28 KB           | 1                |





### resource manger state

```
http://172.17.73.66:8088/cluster/cluster


Cluster ID:	1628150507329
ResourceManager state:	STARTED
ResourceManager HA state:	active
ResourceManager HA zookeeper connection state:	CONNECTED
ResourceManager RMStateStore:	org.apache.hadoop.yarn.server.resourcemanager.recovery.NullRMStateStore
ResourceManager started on:	Thu Aug 05 16:01:47 +0800 2021
ResourceManager version:	3.3.1 from a3b9c37a397ad4188041dd80621bdeefc46885f2 by ubuntu source checksum bac7931c6577eaa849b289247f5b4ab3 on 2021-06-15T05:22Z
Hadoop version:	3.3.1 from a3b9c37a397ad4188041dd80621bdeefc46885f2 by ubuntu source checksum 88a4ddb2299aca054416d6b7f81ca55 on 2021-06-15T05:13Z

```



## namenode HA

依赖之前搭建好的zookeeper 集群

发现

指定 standby nameNode 写入会报错

````
hadoop@hd1:~$ hdfs dfs -fs hdfs://hd2:9000 -mkdir /wsg2
mkdir: Operation category READ is not supported in state standby. Visit https://s.apache.org/sbnn-error
hadoop@hd1:~$ 


````



检查 ha 状态

````
hadoop@hd1:~$ hdfs haadmin -getAllServiceState
hd1:9000                                           active    
hd2:9000                                           standby   
hadoop@hd1:~$ 



hdfs namenode -bootstrapStandby 输出如下：
=====================================================
About to bootstrap Standby ID nn2 from:
           Nameservice ID: ns
        Other Namenode ID: nn1
  Other NN's HTTP address: http://hd1:50070
  Other NN's IPC  address: hd1/172.17.73.64:9000
             Namespace ID: 979716374
            Block pool ID: BP-662301418-172.17.73.64-1628737793187
               Cluster ID: CID-9126de85-f785-4816-b55a-92cbd1bf586e
           Layout version: -66
       isUpgradeFinalized: true

````



https://zhuanlan.zhihu.com/p/61888562



### nameservice

 namenode ha 配置后，default fs

### nameNode HA 配置注意点

需要增加 ha nameservice

```
  <property>
    <name>dfs.client.failover.proxy.provider.ns:</name>
    <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>

```



https://stackoverflow.com/questions/35568042/get-java-net-unknownhostexception-when-use-hadoop-ha

https://issues.apache.org/jira/browse/HDFS-12109

## zookeep上 nameNode 和ResourceManager 的lock

````
/hadoop-ha

/yarn-leader-election
/zookeeper

/hadoop-ha/hdfscluster
/hadoop-ha/hdfscluster/ActiveBreadCrumb
/hadoop-ha/hdfscluster/ActiveStandbyElectorLock
/idcmonitor/idclock
/yarn-leader-election/hdfscluster
/yarn-leader-election/hdfscluster/ActiveBreadCrumb
/yarn-leader-election/hdfscluster/ActiveStandbyElectorLock


````

# 知识点记录





配置文件中的 journal node 如果没有全部 启动，name node 启动会失败



## hdfs format 后的 数据

会在配置的namenode 存储上，看到 生成的 fsimage

```
hadoop@hd1:/opt/hadoop/hadoop-3.3.1/etc/hadoop$ tree /data/
/data/
└── hadoop
    ├── checkpoint
    ├── data
    ├── journalnode
    ├── log
    ├── name
    │   └── current
    │       ├── fsimage_0000000000000000000
    │       ├── fsimage_0000000000000000000.md5
    │       ├── seen_txid
    │       └── VERSION
    └── tmp


```



## 更改defaultFs 

````
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://hd1:9000</value>
  </property>


````

更改后，refresh-namenodes.sh 即可生效。





## refresh namenode

据说可以 重新 加载 hdfs-site.xml ??



```
$HADOOP_HOME/sbin/refresh-namenodes.sh 
```



## journal node 的作用

edit log 会保存到 journal node 目录， 启动name node 也会 校验 editlog 

edit log其实是一个Write-Ahead-Log：

https://www.qedev.com/bigdata/152182.html

https://cloud.tencent.com/developer/article/1665707

## 上传文件



```
用默认的 fs ，会报错，应该是 HA的某个配置 ，留到后面再看？？？
hadoop@hd1:/opt/hadoop/hadoop-3.3.1/etc/hadoop$ hdfs dfs -mkdir /wsg
2021-08-06 20:38:18,478 WARN fs.FileSystem: Failed to initialize fileystem hdfs://hdfscluster: java.lang.IllegalArgumentException: java.net.UnknownHostException: hdfscluster
-mkdir: java.net.UnknownHostException: hdfscluster

可以绕过的方法，就是 直接指定 nameNode，如下： 但是 这样对于 使用方，就失去了高可用。。

hdfs dfs -fs hdfs://hd1:9000 -mkdir /wsg

hadoop@hd1:/opt/hadoop/hadoop-3.3.1/etc/hadoop$ hdfs dfs -fs hdfs://hd1:9000 -put yarn-env.sh /wsg
hadoop@hd1:/opt/hadoop/hadoop-3.3.1/etc/hadoop$ hdfs dfs -fs hdfs://172.17.73.64:9000 -put yarn-site.xml /wsg


hadoop@hd1:/opt/hadoop/hadoop-3.3.1/etc/hadoop$ hdfs dfs -fs hdfs://172.17.73.64:9000 -ls /wsg
Found 2 items
-rw-r--r--   2 hadoop supergroup       6329 2021-08-08 10:41 /wsg/yarn-env.sh
-rw-r--r--   2 hadoop supergroup       1019 2021-08-08 10:42 /wsg/yarn-site.xml
hadoop@hd1:/opt/hadoop/hadoop-3.3.1/etc/hadoop$ 


```





# 问题记录



## jounal node

```
2021-08-07 21:47:03,628 WARN client.QuorumJournalManager: Quorum journal URI 'qjournal://hd1:8485;hd2:8485/ns' has an even number of Journal Nodes specified. This is not recommended!
Re-format filesystem in QJM to [172.17.73.64:8485, 172.17.73.65:8485] ? (Y or N) y



```



# 错误处理记录

## name node 报错

````
2021-08-11 19:41:36,201 INFO impl.MetricsSystemImpl: NameNode metrics system stopped.
2021-08-11 19:41:36,201 INFO impl.MetricsSystemImpl: NameNode metrics system shutdown complete.
2021-08-11 19:41:36,201 ERROR namenode.NameNode: Failed to start namenode.
java.io.IOException: There appears to be a gap in the edit log.  We expected txid 5559, but got txid 5560.
	at org.apache.hadoop.hdfs.server.namenode.MetaRecoveryContext.editLogLoaderPrompt(MetaRecoveryContext.java:95)
	at org.apache.hadoop.hdfs.server.namenode.FSEditLogLoader.loadEditRecords(FSEditLogLoader.java:268)
	at org.apache.hadoop.hdfs.server.namenode.FSEditLogLoader.loadFSEdits(FSEditLogLoader.java:182)
	at org.apache.hadoop.hdfs.server.namenode.FSImage.loadEdits(FSImage.java:915)
	at org.apache.hadoop.hdfs.server.namenode.FSImage.loadFSImage(FSImage.java:762)
	at org.apache.hadoop.hdfs.server.namenode.FSImage.recoverTransitionRead(FSImage.java:339)
	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.loadFSImage(FSNamesystem.java:1197)
	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.loadFromDisk(FSNamesystem.java:779)
	at org.apache.hadoop.hdfs.server.namenode.NameNode.loadNamesystem(NameNode.java:677)
	at org.apache.hadoop.hdfs.server.namenode.NameNode.initialize(NameNode.java:764)
	at org.apache.hadoop.hdfs.server.namenode.NameNode.<init>(NameNode.java:1018)


````





- 处理

  active namenode 和 stand by name node 都需要 下面命令恢复

  hdfs namenode recover

  ````
  hdfs namenode -recover
  
  
  STARTUP_MSG:   java = 11.0.1
  ************************************************************/
  2021-08-11 22:11:05,150 INFO namenode.NameNode: registered UNIX signal handlers for [TERM, HUP, INT]
  2021-08-11 22:11:05,275 INFO namenode.NameNode: createNameNode [-recover]
  You have selected Metadata Recovery mode.  This mode is intended to recover lost metadata on a corrupt filesystem.  Metadata recovery mode often permanently deletes data from your HDFS filesystem.  Please back up your edit log and fsimage before trying this!
  
  Are you ready to proceed? (Y/N)
   (Y or N) 
  
  
  2021-08-12 07:55:11,706 INFO namenode.RedundantEditLogInputStream: Fast-forwarding stream 'http://hd2:8480/getJournal?jid=ns&segmentTxId=5560&storageInfo=-66%3A1192395334%3A1628344027603%3ACID-e0007d89-f40a-416e-9fea-bca652209466&inProgressOk=true' to transaction ID 5548
  2021-08-12 07:55:11,708 ERROR namenode.MetaRecoveryContext: There appears to be a gap in the edit log.  We expected txid 5559, but got txid 5560.
  
  Enter 'c' to continue, ignoring missing  transaction IDs
  Enter 's' to stop reading the edit log here, abandoning any later edits
  Enter 'q' to quit without saving
  Enter 'a' to always select the first choice in the future without prompting. (c/s/q/a)
  c
  2021-08-12 07:55:17,724 INFO namenode.MetaRecoveryContext: Continuing
  2021-08-12 07:55:17,724 INFO namenode.FSEditLogLoader: replaying edit log: 2/3 transactions completed. (67%)
  2021-08-12 07:55:17,724 INFO namenode.FSImage: Reading org.apache.hadoop.hdfs.server.namenode.RedundantEditLogInputStream@2d72f75e expecting start txid #5562; suppressed logging for 6 edit reads
  
  ````

  



## 重启集群再次发生nodenode 启动失败

错误同下

````
格式化前，要先删除 journal 目录
hadoop@hd2:~$ rm -rf /tmp/hadoop/dfs/journalnode/ns
hadoop@hd2:~$ 

````







## 关闭集群后，重启 namenode 失败



```
org.apache.hadoop.hdfs.qjournal.client.QuorumException: Got too many exceptions to achieve quorum size 2/3. 3 exceptions thrown:
172.17.73.65:8485: Journal Storage Directory root= /tmp/hadoop/dfs/journalnode/hdfscluster; location= null not formatted ; journal id: hdfscluster


```

处理

重新格式化 journal node

```
hadoop@hd1:~$ hdfs namenode -initializeSharedEdits


```



https://www.cnblogs.com/liuchangchun/p/4629205.html









## /etc/hosts 文件在 更改hostname时 注意

```
127.0.0.1       localhost
127.0.1.1       wsg

虽然 hostname已经更改为了 hd1.hdfscluster ，但是hosts 文件中 127.0.1.1 解析的还是 老的hostname
```





## hdfs 写入报错

和自己配置 的hdfscluster 有关

```
hadoop@hd1:~$ hdfs dfs -mkdir /wsg
2021-08-06 17:49:11,424 WARN fs.FileSystem: Failed to initialize fileystem hdfs://hdfscluster: java.lang.IllegalArgumentException: java.net.UnknownHostException: hdfscluster
-mkdir: java.net.UnknownHostException: hdfscluster


```



处理：



添加hosts解些 hdfscluster 到 nameNode1 后

````
hadoop@hd1:~$ hdfs dfs -mkdir /wsg
mkdir: Call From hd1.hdfscluster/172.17.73.64 to hdfscluster:8020 failed on connection exception: java.net.ConnectException: Connection refused; For more details see:  http://wiki.apache.org/hadoop/ConnectionRefused

````



# 参考



https://ken.io/note/hadoop-cluster-deploy-guide#H3-8

上面文档 对于 hadoop的 nameNode ，dataNode ， resourceManager， journalNode的 角色划分规划没有说清楚，在配置文件里可以看到。 另外，启动hadoop的服务 可以通过 --daemon 方式。



https://www.wangfeng.live/2021/03/hadoop-3-3-0-fully-distributed-deployment-configuration/



