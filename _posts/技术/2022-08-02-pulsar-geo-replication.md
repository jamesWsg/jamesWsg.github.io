---
layout: post
title: pulsar geo-replication原理和实践
category: 技术
---
# 01 - Geo Replication

# 原理

![Untitled](../../images/01%20-%20Geo%20Replication%209fb03f3d8a724a99bdfb2eeaa03ce9ef/Untitled.png)

- topic中消息的同步
- topic的schema同步（）
- topic的订阅同步（）

- 关于schema同步的说明
    
    这个[issue](https://github.com/streamnative/platform-support-tickets/issues/682)处理schema没有同步到对端集群问题，但是鹏辉在处理该问题发现了另一个潜在[issue](https://github.com/apache/pulsar/pull/17154)，所以推荐在2.10.1.6之后的版本使用（讨论的[slack](https://streamnative.slack.com/archives/C01J6DMU24W/p1661260785491439?thread_ts=1660100680.456329&cid=C01J6DMU24W)）
    

# 操作

## 基本配置

> 目前大部分环境都没有Global zk，这里的配置也都是在没有global zk的情况下，如果有配置会更简单
> 

具体可以参考 [slab文档](https://streamnative.slab.com/posts/apache-pulsar-%E8%B7%A8%E5%9C%B0%E5%9F%9F%E5%A4%8D%E5%88%B6%E9%85%8D%E7%BD%AE-nadfqzwq#hod3v-3-%E5%BC%80%E5%90%AF-namespace%E7%B2%92%E5%BA%A6%E7%9A%84%E5%A4%8D%E5%88%B6) 

简单总结，配置过程主要分为3步：

1. 2端集群分别创建对端的cluster信息
2. 2端集群分别创建相同的tenant()
3. 根据同步的业务需求（基于namespace级别同步或者topic级别同步）来配置相应的策略

需要注意的是namespace同步 和topic同步的优先级关系：

即namespace同步和topic同步不能共存，如果需要topic级别的同步，需要关闭namespace级别的同步。

## 单向同步配置

在基本配置的步骤里，只需要在集群1中配置2个cluster信息，在集群2中只配置本地cluster信息

以cluster1

## 双向同步配置

按照基本配置的方法，同步的模式就是双向同步

## 3集群 22 同步

在配置tenant和namespace增加一个cluster配置

```jsx
[pulsar@shengguo2 current]$ bin/pulsar-admin tenants get geo-tenant
{
  "adminRoles" : [ ],
  "allowedClusters" : [ "cluster1", "cluster3", "cluster2" ]
}

[pulsar@shengguo2 current]$  bin/pulsar-admin namespaces get-clusters geo-tenant/geo-ns
cluster1
cluster3
cluster2

//topics stats上会可以看到有2个replication的信息
[pulsar@shengguo2 current]$ bin/pulsar-admin topics stats geo-tenant/geo-ns/cluster3-topic
...
"replication" : {
    "cluster2" : {
      "msgRateIn" : 0.0,
      "msgThroughputIn" : 0.0,
      "msgRateOut" : 0.0,
      "msgThroughputOut" : 0.0,
      "msgRateExpired" : 0.0,
      "replicationBacklog" : 4163,
      "connected" : false,
      "replicationDelayInSeconds" : 0
    },
    "cluster3" : {
      "msgRateIn" : 0.0,
      "msgThroughputIn" : 0.0,
      "msgRateOut" : 0.0,
      "msgThroughputOut" : 0.0,
      "msgRateExpired" : 0.0,
      "replicationBacklog" : 4163,
      "connected" : false,
      "replicationDelayInSeconds" : 0
    }
  },
```

topics stats-internal上也会看到对应的cursor信息

```jsx

[pulsar@shengguo2 current]$ bin/pulsar-admin topics stats-internal geo-tenant/geo-ns/cluster3-topic
{
  "entriesAddedCounter" : 4163,
  "numberOfEntries" : 4163,
  "totalSize" : 4525688,
  "currentLedgerEntries" : 4163,
  "currentLedgerSize" : 4525688,
  "lastLedgerCreatedTimestamp" : "2022-09-07T01:08:23.234Z",
  "waitingCursorsCount" : 0,
  "pendingAddEntriesCount" : 0,
  "lastConfirmedEntry" : "348:4162",
  "state" : "LedgerOpened",
  "ledgers" : [ {
    "ledgerId" : 348,
    "entries" : 0,
    "size" : 0,
    "offloaded" : false,
    "underReplicated" : false
  } ],
  "cursors" : {
    "pulsar.repl.cluster2" : {
      "markDeletePosition" : "348:-1",
      "readPosition" : "348:0",
      "waitingReadOp" : false,
      "pendingReadOps" : 0,
      "messagesConsumedCounter" : 0,
      "cursorLedger" : 350,
      "cursorLedgerLastEntry" : 0,
      "individuallyDeletedMessages" : "[]",
      "lastLedgerSwitchTimestamp" : "2022-09-07T01:08:23.26Z",
      "state" : "Open",
      "numberOfEntriesSinceFirstNotAckedMessage" : 1,
      "totalNonContiguousDeletedMessagesRange" : 0,
      "subscriptionHavePendingRead" : false,
      "subscriptionHavePendingReplayRead" : false,
      "properties" : { }
    },
    "pulsar.repl.cluster3" : {
      "markDeletePosition" : "348:-1",
      "readPosition" : "348:0",
      "waitingReadOp" : false,
      "pendingReadOps" : 0,
      "messagesConsumedCounter" : 0,
      "cursorLedger" : 349,
      "cursorLedgerLastEntry" : 0, 
      "individuallyDeletedMessages" : "[]",
      "lastLedgerSwitchTimestamp" : "2022-09-07T01:08:23.259Z",
      "state" : "Open",
      "numberOfEntriesSinceFirstNotAckedMessage" : 1,
      "totalNonContiguousDeletedMessagesRange" : 0,
      "subscriptionHavePendingRead" : false,
      "subscriptionHavePendingReplayRead" : false,
      "properties" : { }
    }
  },
```

## 检查同步状态

Topics stats-internal 输出

```
 "pulsar.repl.htsc-pulsar-02" : {
      "markDeletePosition" : "50416:-1",
      "readPosition" : "50416:0",
      "waitingReadOp" : true,
      "pendingReadOps" : 0,
      "messagesConsumedCounter" : 0,
      "cursorLedger" : -1,
      "cursorLedgerLastEntry" : -1,
      "individuallyDeletedMessages" : "[]",
      "lastLedgerSwitchTimestamp" : "2022-05-11T14:08:10.844+08:00",
      "state" : "NoLedger",
      "numberOfEntriesSinceFirstNotAckedMessage" : 1,
      "totalNonContiguousDeletedMessagesRange" : 0,
      "subscriptionHavePendingRead" : false,
      "subscriptionHavePendingReplayRead" : false,
      "properties" : { }

  --- peer cluster
  "pulsar.repl.htsc-pulsar-01" : {
      "markDeletePosition" : "131118:-1",
      "readPosition" : "131118:0",
      "waitingReadOp" : true,
      "pendingReadOps" : 0,
      "messagesConsumedCounter" : 0,
      "cursorLedger" : 131119,
      "cursorLedgerLastEntry" : 0,
      "individuallyDeletedMessages" : "[]",
      "lastLedgerSwitchTimestamp" : "2022-05-06T22:45:29.443+08:00",
      "state" : "Open",
      "numberOfEntriesSinceFirstNotAckedMessage" : 1,
      "totalNonContiguousDeletedMessagesRange" : 0,
      "subscriptionHavePendingRead" : false,
      "subscriptionHavePendingReplayRead" : false,
      "properties" : { }
    }
```

### 分区topic的同步状态

需要加–per-partition的选项，才能看到同步的对端集群信息(需要注意分区topic和非分区topic的差异)

```
[pulsar@shengguo3 current]$ bin/pulsar-admin topics partitioned-stats geo-tenant/geo-ns/pt5 --per-partition

"persistent://geo-tenant/geo-ns/pt5-partition-3" : {
      "msgRateIn" : 0.0,
      "msgThroughputIn" : 0.0,
      "msgRateOut" : 0.0,
      "msgThroughputOut" : 0.0,
      "bytesInCounter" : 0,
      "msgInCounter" : 0,
      "bytesOutCounter" : 0,
      "msgOutCounter" : 0,
      "averageMsgSize" : 0.0,
      "msgChunkPublished" : false,
      "storageSize" : 0,
      "backlogSize" : 0,
      "publishRateLimitedTimes" : 0,
      "earliestMsgPublishTimeInBacklogs" : 0,
      "offloadedStorageSize" : 0,
      "lastOffloadLedgerId" : 0,
      "lastOffloadSuccessTimeStamp" : 0,
      "lastOffloadFailureTimeStamp" : 0,
      "publishers" : [ ],
      "waitingPublishers" : 0,
      "subscriptions" : { },
      "replication" : {
        "cluster1" : {
          "msgRateIn" : 0.0,
          "msgThroughputIn" : 0.0,
          "msgRateOut" : 0.0,
          "msgThroughputOut" : 0.0,
          "msgRateExpired" : 0.0,
          "replicationBacklog" : 0,
          "connected" : true,
          "replicationDelayInSeconds" : 0,
          "inboundConnection" : "/172.24.0.5:46512",
          "inboundConnectedSince" : "2022-09-02T07:23:07.215828Z",
          "outboundConnection" : "[id: 0x2f0738e3, L:/172.24.0.6:57162 - R:172.24.0.5/172.24.0.5:6650]",
          "outboundConnectedSince" : "2022-09-02T07:23:07.148868Z"
        }
```

> 如果不加该选项，只能看到 replication字段里面如下信息：
> 
> 
> ```
> [pulsar@shengguo3 current]$ bin/pulsar-admin topics partitioned-stats geo-tenant/geo-ns/pt5
> {
>   "msgRateIn" : 0.0,
>   "msgThroughputIn" : 0.0,
>   "msgRateOut" : 0.0,
>   "msgThroughputOut" : 0.0,
>   "bytesInCounter" : 0,
>   "msgInCounter" : 0,
>   "bytesOutCounter" : 0,
>   "msgOutCounter" : 0,
>   "averageMsgSize" : 0.0,
>   "msgChunkPublished" : false,
>   "storageSize" : 0,
>   "backlogSize" : 0,
>   "publishRateLimitedTimes" : 0,
>   "earliestMsgPublishTimeInBacklogs" : 0,
>   "offloadedStorageSize" : 0,
>   "lastOffloadLedgerId" : 0,
>   "lastOffloadSuccessTimeStamp" : 0,
>   "lastOffloadFailureTimeStamp" : 0,
>   "publishers" : [ ],
>   "waitingPublishers" : 0,
>   "subscriptions" : { },
>   "replication" : {
>     "cluster1" : {
>       "msgRateIn" : 0.0,
>       "msgThroughputIn" : 0.0,
>       "msgRateOut" : 0.0,
>       "msgThroughputOut" : 0.0,
>       "msgRateExpired" : 0.0,
>       "replicationBacklog" : 0,
>       "connected" : false,
>       "replicationDelayInSeconds" : 0
>     }
>   },
>   "nonContiguousDeletedMessagesRanges" : 0,
>   "nonContiguousDeletedMessagesRangesSerializedSize" : 0,
>   "compaction" : {
>     "lastCompactionRemovedEventCount" : 0,
>     "lastCompactionSucceedTimestamp" : 0,
>     "lastCompactionFailedTimestamp" : 0,
>     "lastCompactionDurationTimeInMills" : 0
>   },
>   "metadata" : {
>     "partitions" : 5
>   },
>   "partitions" : { }
> }
> ```
> 

### 查看topic上消息同步的速率统计

```
 [pulsar@shengguo3 current]$ bin/pulsar-admin topics partitioned-stats geo-tenant/geo-ns/pt5

....
"replication" : {
    "cluster1" : {
      "msgRateIn" : 100.0002801442639,
      "msgThroughputIn" : 108700.30451681487,
      "msgRateOut" : 0.0,
      "msgThroughputOut" : 0.0,
      "msgRateExpired" : 0.0,
      "replicationBacklog" : 0,
      "connected" : false,
      "replicationDelayInSeconds" : 0
    }
  },
```

## 检查同步配置

> 因为不同级别查看同步配置的关键字不一样，记录下以免混淆
> 

检查tenant的配置

```
[pulsar@shengguo2 current]$ bin/pulsar-admin tenants get geo-tenant
{
  "adminRoles" : [ ],
  "allowedClusters" : [ "cluster1", "cluster2" ]
}
```

检查namespace的配置

```
[pulsar@shengguo2 current]$ bin/pulsar-admin namespaces get-clusters geo-tenant/geo-ns
cluster1
cluster2
[pulsar@shengguo2 current]$

如果namespace没有配置远端集群，输出如下，只能看到一个cluster信息：
[pulsar@shengguo2 current]$ bin/pulsar-admin namespaces get-clusters public/default
cluster1
```

> 还可以通过namespaces policies的输出判断同步的配置，如下：
> 
> 
> ```
> [pulsar@shengguo2 current]$ bin/pulsar-admin namespaces policies geo-tenant/geo-ns
> {
>   "auth_policies" : {
>     "namespace_auth" : { },
>     "destination_auth" : { },
>     "subscription_auth_roles" : { }
>   },
>   "replication_clusters" : [ "cluster1", "cluster2" ],
> ```
> 

检查topic的配置

```
[pulsar@shengguo2 current]$ bin/pulsar-admin topics get-replication-clusters geo-tenant/geo-ns/pt4
null

```

## 修改tenant中的allowed-clusters

```
[pulsar@shengguo1 current]$ bin/pulsar-admin tenants get geo-tenant-direction
{
  "adminRoles" : [ ],
  "allowedClusters" : [ "test-cluster", "Cluster7" ]
}

//geo-tenant-direction 的allowedClusters 中配置了2个，现在需要删除Cluster7， 命令如下：
[pulsar@shengguo1 current]$ bin/pulsar-admin tenants update geo-tenant-direction --allowed-clusters test-cluster
[pulsar@shengguo1 current]$

//检查是否生效
[pulsar@shengguo1 current]$ bin/pulsar-admin tenants get geo-tenant-direction
{
  "adminRoles" : [ ],
  "allowedClusters" : [ "test-cluster" ]
}
```

## cluster信息删除

配置过程中，如果不小心cluster信息写错，可以直接删掉cluster

```
[pulsar@shengguo1 current]$ bin/pulsar-admin clusters get Cluster7
{
  "serviceUrl" : "http://10.2.0.7:8080",
  "brokerServiceUrl" : "pulsar://10.2.0.7:6650",
  "brokerClientTlsEnabled" : false,
  "tlsAllowInsecureConnection" : false,
  "brokerClientTlsEnabledWithKeyStore" : false,
  "brokerClientTlsTrustStoreType" : "JKS"
}

[pulsar@shengguo1 current]$ bin/pulsar-admin clusters delete Cluster7

```

## 关闭同步配置

这里以关闭 namespace的 同步为例(在源端进行操作)

```
[pulsar@shengguo2 current]$ bin/pulsar-admin namespaces get-clusters geo-tenant/geo-ns
cluster1
cluster2

//需要停止同步到cluster2，命令如下：
[pulsar@shengguo2 current]$ bin/pulsar-admin namespaces set-clusters -c cluster1 geo-tenant/geo-ns

//检查是否生效
[pulsar@shengguo2 current]$ bin/pulsar-admin namespaces get-clusters geo-tenant/geo-ns
cluster1
[pulsar@shengguo2 current]$

```

注意： 关闭同步后，从 topics的stats里面看似乎 还存在和远端集群的连接信息(如下)，但是此时 消息已经不会同步了

```
[pulsar@shengguo3 current]$ bin/pulsar-admin topics partitioned-stats geo-tenant/geo-ns/pt5 --per-partition
...
"persistent://geo-tenant/geo-ns/pt5-partition-4" : {
      "msgRateIn" : 0.0,
      "msgThroughputIn" : 0.0,
      "msgRateOut" : 0.0,
      "msgThroughputOut" : 0.0,
      "bytesInCounter" : 7324909,
      "msgInCounter" : 6739,
      "bytesOutCounter" : 0,
      "msgOutCounter" : 0,
      "averageMsgSize" : 0.0,
      "msgChunkPublished" : false,
      "storageSize" : 7384340,
      "backlogSize" : 0,
      "publishRateLimitedTimes" : 0,
      "earliestMsgPublishTimeInBacklogs" : 0,
      "offloadedStorageSize" : 0,
      "lastOffloadLedgerId" : 0,
      "lastOffloadSuccessTimeStamp" : 0,
      "lastOffloadFailureTimeStamp" : 0,
      "publishers" : [ ],
      "waitingPublishers" : 0,
      "subscriptions" : { },
      "replication" : {
        "cluster1" : {
          "msgRateIn" : 0.0,
          "msgThroughputIn" : 0.0,
          "msgRateOut" : 0.0,
          "msgThroughputOut" : 0.0,
          "msgRateExpired" : 0.0,
          "replicationBacklog" : 0,
          "connected" : true,
          "replicationDelayInSeconds" : 0,
          "outboundConnection" : "[id: 0x2f0738e3, L:/172.24.0.6:57162 - R:172.24.0.5/172.24.0.5:6650]",
          "outboundConnectedSince" : "2022-09-02T07:23:06.738923Z"
        }
```

## geo-replication 限速

### 总结

1. 可以基于namespace和topic级别来限速，topic级别的优先级高于namespces级别 （即如果同时设置了namespace和topic的限速，最终会按topic的设置来执行）
2. topic级别的限速会应用到该topic下的所有分区（比如给topicA设置了5MB限速，假如topicA下面有3个分区，则每个分区的限速为5MB，所以最终该topic的速度可能会达到15MB）
3. namespaces级别的限速会应用到该namespace下面的所有分区topic（比如 namespace 设置限流1M，如果ns下有个 2分区的topic，最终限流结果为2M； 如果有3分区topic，最终结果为3M）
4. 当前不管配置namespace，还是topic的限速额度，实际生效的都是分区topic的限速额度。

### namespace 级别限速配置

```jsx
// 设置30Mb的限速，可以以tps和带宽的维度设置
[root@168-64-5-166 apache-pulsar-2.8.2.4]# bin/pulsar-admin namespaces set-replicator-dispatch-rate --byte-dispatch-rate 3145728  rtdc-tenant/rdtc-xtrader-namespace

//check
[root@168-64-5-166 apache-pulsar-2.8.2.4]# bin/pulsar-admin namespaces get-replicator-dispatch-rate  rtdc-tenant/rdtc-xtrader-namespace
{
  "dispatchThrottlingRateInMsg" : -1,
  "dispatchThrottlingRateInByte" : 3145728,
  "relativeToPublishRate" : false,
  "ratePeriodInSecond" : 1
}

//删除限速
bin/pulsar-admin namespaces remove-replicator-dispatch-rate  rtdc-tenant/rdtc-xtrader-namespace
```

### topic级别限速配置

配置方法同上，只是将namespaces 更改为topics，如下

```jsx
bin/pulsar-admin topics get-replicator-dispatch-rate geo-wsg/geo-ns/pt
```

### 限流检查的时间精度

默认是1s，设置0.3 就会不合法

```jsx
--- dispatch-rate-period default 1s 

[pulsar@168-64-26-100 current]$ bin/pulsar-admin namespaces set-replicator-dispatch-rate --byte-dispatch-rate 1048576 --dispatch-rate-period 0.3  geo-wsg/geo-ns
"--dispatch-rate-period": couldn't convert "0.3" to an integer
```

### 限流dashboard

增加了带宽的panel

![Untitled](../../images/01%20-%20Geo%20Replication%209fb03f3d8a724a99bdfb2eeaa03ce9ef/Untitled%201.png)

如果需要使用，可以直接import下面的dashboard 

[Messaging Metrics-with-geo-throught-1661607815948.json](01%20-%20Geo%20Replication%209fb03f3d8a724a99bdfb2eeaa03ce9ef/Messaging_Metrics-with-geo-throught-1661607815948.json)

# 常见问题处理

## 配置geo后，对端broker的ip有过调整，如何处理？

可以删除旧的cluster，然后创建一个新的cluster（参考上面的cluster信息删除）

## 配置geo后，想删掉某个cluster里的某个topic？

直接删除会报错，因为topic上还存在同步（replication）的cursor引用，可以先将同步停止（参考上面关闭同步），再做删除topic的操作

# 下一步待解决问题

订阅的同步（因为每个集群的message id是不一样的，）分布式snapshot的原理

双活需要涉及到订阅的同步，（订阅的同步是无法避免重复消息的情况，，）

