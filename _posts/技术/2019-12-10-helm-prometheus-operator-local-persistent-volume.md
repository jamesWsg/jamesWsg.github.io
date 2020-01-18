---
layout: post
title: helm Prometheus-operator use host local persistent volume
category: 技术
---



# 背景

kubernets集群流行的监控平台是prometheus，为了更方便的部署，出现了Prometheus-Operator，随着helm的发展，helm在 ali的容器平台，使用也越来越多。

默认helm的配置中，prometheus的 TSDB数据库放在emptyDir中，这样pod 重启时，历史的监控数据就找不到了。因为lab中本来就已经有ceph集群，一开始打算通过kubernets的 StoarageClass来实现pv的自动创建，因为涉及到和ceph的storage network通信，所以放到后面再研究。

所以本次研究的重点是，通过在host节点挂载单独的磁盘做成pv，供Prometheus-Operator来使用。考虑到本地的pv只会挂载在特定的host上，所以 需要考虑到 node affinity



helm 是由CNCF孵化和管理的项目，用于需要在k8s上部署复杂应用进行定义，安装和更新。 helm以chart方式对应用软件进行描述，可以方便的创建，版本化，共享和发布复杂应用软件。相关术语chart： 一个helm包，包含了一个应用所需的工具和资源定义，可能还包括kubernets集群中的服务定义，类比apt中的deb，或者yum中的rpm文件release：k8s上运行的chart 实例。同一个集群上，一个chart可以安装多次，每次安装都会生成新的release，拥有独立的release名称repository： 存放和共享chart的仓库


helm由2部分组成，helmclient(拥有对repository，chart，release对象的管理能力)； tillerServer（负责客户端指令和k8s集群之间的交互，根据chart定义生成和管理各种k8s的资源对象），helm3 已经没有 tiller server了



#helm安装Prometheus-Operator

helm是go开发的，可以直接在官网下载，解压出来就是binary可执行文件

需要提前安装好 helm，参考命令如下：

```

https://get.helm.sh/helm-v3.0.2-linux-amd64.tar.gz
```



## helm 安装prometheus-operator

helm 安装可以通过源的方式在线安装，也可以通过 下载好chart包，通过本地安装。 也可以将 chart 包解压开，修改value.yml后再安装。

```
helm install prometheus-operator-release ./prometheus-operator

```





# 修改Prometheus-Operator的配置声明local-persistent-volume



## 默认的配置文件

values.yaml

```
    storage: {}
    # volumeClaimTemplate:
    #   spec:
    #     storageClassName: gluster
    #     accessModes: ["ReadWriteOnce"]
    #     resources:
    #       requests:
    #         storage: 50Gi
    #   selector: {}

```

可以看到storage默认的值是空的，所以默认存储用的是host的emptyDir





##修改后的文件

```
1605     ## Prometheus StorageSpec for persistent data
1606     ## ref: https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/storage.md
1607     ##
1608     storageSpec:
1609       volumeClaimTemplate:
1610         spec:
1611           storageClassName: local-storage
1612           accessModes: ["ReadWriteOnce"]
1613           resources:
1614             requests:
1615               storage: 200Gi
1616         selector: {}
1617 

```

可以看到这里我们申明了，一个 local-storage 类型的 storage class（本质上定义了一个pvc），所以下一步，我们需要进一步定义storage class。

这里补充下pvc，pv，storageclass之间的关系，学习自 极客时间 阿里 张磊的《深入剖析kubernets》：

```
而 StorageClass 对象的作用，其实就是创建 PV 的模板。具体地说，StorageClass 对象会定义如下两个部分内容：
第一，PV 的属性。比如，存储类型、Volume 的大小等等。
第二，创建这种 PV 需要用到的存储插件。比如，Ceph 等等。
有了这样两个信息之后，Kubernetes 就能够根据用户提交的 PVC，找到一个对应的 StorageClass 了。然后，Kubernetes 就会调用该 StorageClass 声明的存储插件，创建出需要的 PV。
```





## 定义上面 storage class

具体定义如下：

```
wsg@ubuntu16:~/kube_ymal/pv_pvc/local_pv$ cat local-storage-class.yaml 
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer

```

需要注意 其中的 provisioner 值，



## 定义 对应pv

```
wsg@ubuntu16:~/kube_ymal/pv_pvc/local_pv$ cat local-pv.yaml 

apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 200Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/disks/vol1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - ubuntu16-template
```



 注意，该pv 有 host主机 ubuntu16-template才会存在，所以 pv的定义中 增加了 nodeAffinity，所以该pv 只会调度在 hostname 为 ubuntu16-template的host上。

为了保证 pod重启后，pv中的数据还在，需要 设置 persistentVolumeReclaimPolicy: Retain

accessModes设置为ReadWriteOnce，保证只有一个 pod有读写权限

该host上的磁盘已经挂载到 /mnt/disks/vol1



## 启用 StorageClass 和pv



```
kubectl apply -f local-storage-class.yaml 
kubectl apply -f local-pv.yaml

```



检查pvc的情况，可以看到pvc 已经处于bond状态，也就是找到了我们定义的pv，可以看到是通过 local-storage（STORAGECLASS）来找到的。

```
wsg@ubuntu16:~/kube_ymal/pv_pvc/local_pv$ kubectl get pvc
NAME                                                                                               STATUS    VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
data-wsg-logstash-0                                                                                Pending                                                        2d23h
prometheus-wsg-prometheus-operator-prometheus-db-prometheus-wsg-prometheus-operator-prometheus-0   Bound     local-pv   200Gi      RWO            local-storage   5d23h


```



进一步检查pv的状态，可以看到pv处于bond状态，并且可以看到 谁使用了 该pv

```
wsg@ubuntu16:~/kube_ymal/pv_pvc/local_pv$ kubectl get pv
NAME                    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                                                                                                      STORAGECLASS    REASON   AGE
local-pv                200Gi      RWO            Retain           Bound      default/prometheus-wsg-prometheus-operator-prometheus-db-prometheus-wsg-prometheus-operator-prometheus-0   local-storage            8d
wsg@ubuntu16:~/kube_ymal/pv_pvc/local_pv$ 

```



这样就完成了 将Prometheus使用本地存储实现监控数据的持久化。

