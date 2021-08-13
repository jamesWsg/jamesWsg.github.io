---
layout: post
title: zookeeper部署和使用实践
category: 技术
---



# zookeeper

java开发

应用场景

配置管理，分布式锁， master角色， master监控worker节点状态。

数据量 几百MB





## 与chubby比较

google 内部，分布式 lock服务

使用 advisory locking



## 与etcd 比较

数据量几个GB,bbot，B+tree



## 部署

需要手动创建 myid文件，用来标识 该节点，2节点，3个zk 实例

active vm部署一个 zookeeper实例， slave vm 部署2个 zookeeper 实例

官方网站下载zookeeper ，本次用的版本为 apache-zookeeper-3.6.3

### active vm配置

- 将下载好的zoozkeeper 解压到下面的目录

  ````
  root@active:/opt/zookeeper# ll
  total 12
  drwxr-xr-x 3 root root 4096 Jul 14 09:33 ./
  drwxr-xr-x 7 root root 4096 Jul 14 09:38 ../
  drwxr-xr-x 7 root root 4096 Jul 14 09:57 apache-zookeeper-3.6.3-bin/
  root@active:/opt/zookeeper# 
  
  ````

- 创建zookeeper 数据目录，并且在其中创建 myid(myid 是用来区分不同 zookeeper实例)

  ````
  mkdir /data-zookeeper/
  
  root@active:/data-zookeeper# cat myid 
  1
  root@active:/data-zookeeper# pwd
  /data-zookeeper
  
  ````

- 配置 zookeeper

  配置文件位于解压目录的 conf/zoo.cfg，其中主要配置 dataDir，dataLogDir为上面步骤的目录

  ```
  root@active:/opt/zookeeper/apache-zookeeper-3.6.3-bin# cat conf/zoo.cfg  |grep -v "^#"
  tickTime=2000
  initLimit=10
  syncLimit=5
  dataDir=/data-zookeeper
  dataLogDir=/data-zookeeper
  clientPort=2181
  
  server.1=2.2.2.67:2222:2223
  server.2=2.2.2.68:2222:2223
  server.3=2.2.2.68:3333:3334
  
  4lw.commands.whitelist=*
  
  ```

- zookeeper 开机启动

  ```
  创建 systemctl 配置文件 /usr/lib/systemd/zookeeper.service，内容如下
  [Unit]
  Description=zookeeper
  After=network.target remote-fs.target nss-lookup.target
  [Service]
  Type=forking
  ExecStart=/opt/zookeeper/apache-zookeeper-3.6.3-bin/bin/zkServer.sh start
  ExecReload=/opt/zookeeper/apache-zookeeper-3.6.3-bin/bin/zkServer.sh restart
  ExecStop=/opt/zookeeper/apache-zookeeper-3.6.3-bin/bin/zkServer.sh stop
  [Install]
  
  然后服务可执行权限
  chmod 777 zookeeper.service
  
  设置开机启动
  systemctl enable /usr/lib/systemd/zookeeper.service
  
  然后启动服务
  systemctl start zookeeper.service
  
  
  ```

  



### slave vm 配置

- 将下载好的 zookeeper 解压到 下面2个目录（因为需要2个实例，所以有2个目录）

  ````
  root@slave:/opt# ll
  drwxr-xr-x  3 root root 4096 Jul 14 09:56 zookeeper1/
  drwxr-xr-x  3 root root 4096 Jul 14 10:27 zookeeper2/
  root@slave:/opt# 
  
  ````

- 创建zookeeper 数据目录，并且在其中创建 myid,

  ```
  因为该节点需要部署2个 实例，所以创建了2个目录,每个目录都需要生产一个myid
  
  root@slave:/data-zookeeper# ll
  drwxr-xr-x  4 root root 4096 Aug  5 10:35 zk1/
  drwxr-xr-x  3 root root 4096 Aug  5 10:35 zk2/
  root@slave:/data-zookeeper# pwd
  /data-zookeeper
  
  root@slave:/data-zookeeper# cat zk1/myid 
  2
  root@slave:/data-zookeeper# cat zk2/myid 
  3
  root@slave:/data-zookeeper# 
  
  
  
  ```

  

- 配置zookeeper

  实例1配置

  ````
  root@slave:/opt/zookeeper1# cat apache-zookeeper-3.6.3-bin/conf/zoo.cfg |grep -v "^#"
  tickTime=2000
  initLimit=10
  syncLimit=5
  dataDir=/data-zookeeper/zk1
  clientPort=2181
  
  
  server.1=2.2.2.67:2222:2223
  server.2=2.2.2.68:2222:2223
  server.3=2.2.2.68:3333:3334
  
  4lw.commands.whitelist=*
  
  root@slave:/opt/zookeeper1# 
  
  ````

  实例2配置

  ````
  root@slave:/opt/zookeeper2# cat apache-zookeeper-3.6.3-bin/conf/zoo.cfg |grep -v "^#"
  tickTime=2000
  initLimit=10
  syncLimit=5
  dataDir=/data-zookeeper/zk2
  clientPort=2182
  
  server.1=2.2.2.67:2222:2223
  server.2=2.2.2.68:2222:2223
  server.3=2.2.2.68:3333:3334
  
  
  4lw.commands.whitelist=*
  
  
  ````

  

- zookeeper 开机启动

  和active vm 配置类似，只是需要配置2个实例

  实例1配置

  ```
  root@slave:/opt# cat /usr/lib/systemd/zookeeper1.service
  [Unit]
  Description=zookeeper1
  After=network.target remote-fs.target nss-lookup.target
  [Service]
  Type=forking
  ExecStart=/opt/zookeeper1/apache-zookeeper-3.6.3-bin/bin/zkServer.sh start
  ExecReload=/opt/zookeeper1/apache-zookeeper-3.6.3-bin/bin/zkServer.sh restart
  ExecStop=/opt/zookeeper1/apache-zookeeper-3.6.3-bin/bin/zkServer.sh stop
  [Install]
  WantedBy=multi-user.target
  root@slave:/opt# 
  
  ```

  实例2 配置

  ```
  root@slave:/opt# cat /usr/lib/systemd/zookeeper2.service
  
  [Unit]
  Description=zookeeper2
  After=network.target remote-fs.target nss-lookup.target
  [Service]
  Type=forking
  ExecStart=/opt/zookeeper2/apache-zookeeper-3.6.3-bin/bin/zkServer.sh start
  ExecReload=/opt/zookeeper2/apache-zookeeper-3.6.3-bin/bin/zkServer.sh restart
  ExecStop=/opt/zookeeper2/apache-zookeeper-3.6.3-bin/bin/zkServer.sh stop
  [Install]
  WantedBy=multi-user.target
  root@slave:/opt# 
  
  ```

  加入开机启动，并且启动

  ```
  systemctl enable /usr/lib/systemd/zookeeper1.service
  systemctl enable /usr/lib/systemd/zookeeper2.service
  
  systemctl start zookeeper1.service 
  systemctl start zookeeper2.service 
  ```

  



### 









````
 vim /usr/lib/systemd/zookeeper.service
 增加如下配置
 [Unit]
Description=zookeeper
After=network.target remote-fs.target nss-lookup.target
[Service]
Type=forking
ExecStart=/opt/zookeeper/apache-zookeeper-3.6.3-bin/bin/zkServer.sh start
ExecReload=/opt/zookeeper/apache-zookeeper-3.6.3-bin/bin/zkServer.sh restart
ExecStop=/opt/zookeeper/apache-zookeeper-3.6.3-bin/bin/zkServer.sh stop
[Install]
WantedBy=multi-user.target

然后配置文件权限
  415  chmod 777 zookeeper.service 
  417  systemctl daemon-reload 

 
 421  systemctl enable /usr/lib/systemd/zookeeper.service
  422  systemctl start zookeeper.service 


````







### zkCli

```
[zk: localhost:2181(CONNECTED) 0] create /app1
Created /app1
[zk: localhost:2181(CONNECTED) 1] create /app2
Created /app2
[zk: localhost:2181(CONNECTED) 2] create /app2/ps1
Created /app2/ps1
[zk: localhost:2181(CONNECTED) 3] create /app2/ps2
Created /app2/ps2

org.apache.commons.cli.UnrecognizedOptionException: Unrecognized option: -lR
[zk: localhost:2181(CONNECTED) 6] ls -R /
/
/app1
/app2
/zookeeper
/app2/ps1
/app2/ps2
/zookeeper/config
/zookeeper/quota
[zk: localhost:2181(CONNECTED) 7] 


```





## zk python client

```
root@idcSingle:/usr/lib/systemd# pip3 install kazoo


```





## 常用命令



### 查看集群成员



查询leader 节点

```
root@idcSingle:~# echo mntr |nc 2.2.2.68 2181
zk_version	3.6.3--6401e4ad2087061bc6b9f80dec2d69f2e3c8660a, built on 04/08/2021 16:35 GMT
zk_server_state	leader
zk_peer_state	leading - broadcast
zk_ephemerals_count	1
zk_avg_latency	0.1736
zk_num_alive_connections	1
zk_min_proposal_size	36
zk_outstanding_requests	0
zk_znode_count	11
zk_global_sessions	1
zk_non_mtls_remote_conn_count	0
zk_last_client_response_size	16
zk_packets_sent	10432
zk_packets_received	10431
zk_max_proposal_size	110
zk_max_client_response_size	131
zk_synced_followers	2


```



查询 非 leader 节点，没有 followers 的值

```
root@idcSingle:~# echo mntr |nc 2.2.2.67 2181
zk_version	3.6.3--6401e4ad2087061bc6b9f80dec2d69f2e3c8660a, built on 04/08/2021 16:35 GMT
zk_server_state	follower
zk_peer_state	following - broadcast
zk_ephemerals_count	1
zk_avg_latency	0.0
zk_num_alive_connections	1
zk_outstanding_requests	0
zk_znode_count	11
zk_global_sessions	1
zk_non_mtls_remote_conn_count	0
zk_last_client_response_size	-1
zk_packets_sent	2
zk_packets_received	3
zk_max_client_response_size	-1
zk_connection_drop_probability	0.0
zk_watch_count	0
zk_auth_failed_count	0
zk_min_latency	0
zk_max_file_descriptor_count	4096
zk_approximate_data_size	238
zk_open_file_descriptor_count	74
zk_local_sessions	0
zk_uptime	155134117
zk_max_latency	0
zk_outstanding_tls_handshake	0
zk_min_client_response_size	-1
zk_non_mtls_local_conn_count	0
zk_quorum_size	3


```



### 查看节点状态



stat

````
root@idcSingle:/opt/zookeeper/apache-zookeeper-3.6.3-bin# echo stat |nc 2.2.2.68 2182
Zookeeper version: 3.6.3--6401e4ad2087061bc6b9f80dec2d69f2e3c8660a, built on 04/08/2021 16:35 GMT
Clients:
 /2.2.2.67:41734[0](queued=0,recved=1,sent=0)

Latency min/avg/max: 0/0.0/0
Received: 1
Sent: 0
Connections: 1
Outstanding: 0
Zxid: 0x200000000
Mode: leader
Node count: 9
Proposal sizes last/min/max: -1/-1/-1



````





## 错误处理记录







### ruok

```
root@idcSingle:~# echo ruok |netcat 2.2.2.67 2181
ruok is not executed because it is not in the whitelist.

```

zoo.cfg 添加配置

```
4lw.commands.whitelist=*

```



# python 使用zookeeper

将zookeeper 作为分布式 lock服务，详见gitlab 代码

https://gitlab.com/bigtera/SEG-Tools/-/tree/master/python-zookeepr

# 参考

https://www.codenong.com/cs110872163/

# 2数据中心灾备



## 2个数据中心 vm 准备

2个vm 部署drbd， 并且部署3节点 zookeeper



选择一个 vm 作为 创建 ceph-mon（通过 命令行，ui 创建的ceph-mon无法使用vip）

需要保证该vm

- drbd 挂载到 /data目录 （因为创建ceph-mon 时 会在该目录创建 ceph-mon 的level db数据库）
- 集群 ceph.conf 拷贝过来
- vip 设置好
- 手动创建好 ceph-mon ,手动操作ceph的文档里有记录
- 



## 高可用脚本部署

需要 将 ceph-mon 的systemctl 启动 disable掉， 否则 systectl 启动会报错。 2个vm 上都需要 disable

但是 ceph-mgr 的启动 需要 保留。



master node 脚本启动时，需要 保证 master drbd为primary



python 脚本启动 放在 /etc/rc.local 中







