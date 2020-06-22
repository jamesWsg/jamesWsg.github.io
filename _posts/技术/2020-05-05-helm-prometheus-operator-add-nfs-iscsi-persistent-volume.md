





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



# 警告：

nfs 作为 持久化存储时，nfs 目录有时权限会变， 有时可以写，有时 无法写入



在pod 内尝试创建目录 和文件，都没有权限。

```
wsg@k8smaster2:~/ceph-csi/examples/rbd$ kubectl exec -it prometheus-prometheus-operator-releas-prometheus-0 -- sh -c 'mkdir pod_in'
Defaulting container name to prometheus.
Use 'kubectl describe pod/prometheus-prometheus-operator-releas-prometheus-0 -n default' to see all of the containers in this pod.
mkdir: can't create directory 'pod_in': Permission denied
command terminated with exit code 1
wsg@k8smaster2:~/ceph-csi/examples/rbd$ kubectl exec -it prometheus-prometheus-operator-releas-prometheus-0 -- sh -c 'touch 1.st'
Defaulting container name to prometheus.
Use 'kubectl describe pod/prometheus-prometheus-operator-releas-prometheus-0 -n default' to see all of the containers in this pod.
touch: 1.st: Permission denied
command terminated with exit code 1

```



发现 nfs 文件的属主 uid 不同,有些是 472，有些是1000

```
root@scaler-single:/vol/k8s-prometheus/prometheus_db/prometheus-db# ll
total 20
drwxrwxrwx 1  472  472 263700463 Jun 16 22:12 ./
drwxrwxrwx 1  472  472 266441399 Jun 17 18:24 ../
drwxrwxrwx 1  472  472  32120955 Jun 17 06:56 01EAZNRY2JXM3J2BCY9B7Q580J/
drwxrwxrwx 1  472  472  50071518 Jun 17 10:56 01EB03GBMSZXTB2MS8J1BQA80H/
drwxr-xr-x 1 1000 root  13211040 Jun 17 18:23 01EB0X3QCE6AFM5WQTN114YF21/
drwxr-xr-x 1 1000 root  49041075 Jun 17 18:23 01EB0X3WWMGR3XGSGFN2YD1F8H/
-rw-r--r-- 1 1000 root         0 Jun 17 18:23 1.st
-rwxrwxrwx 1  472  472     20001 Jun 17 18:27 queries.active*
drwxrwxrwx 1  472  472 119235874 Jun 17 18:23 wal/
root@scaler-single:/vol/k8s-prometheus/prometheus_db/prometheus-db# cd ..



```





# 补充1：iscsi 作为持久化存储

> 注意： 需要将iscsi lun 不做分区，对整个盘创建文件系统。 测试过创建分区，在分区上创建ext4，但是k8s无法检测到。 看起来k8s 只能认整个盘，并且不会自动创建ext4。相关kern.log 报错信息见文后

可以看到 k8s mount 选项。

```
Mounting arguments: --description=Kubernetes transient mount for /var/lib/kubelet/plugins/kubernetes.io/iscsi/iface-default/172.17.73.37:3260-iqn.2020-02.com:wsg-lun-1 --scope -- mount -t ext4 -o defaults /dev/disk/by-path/ip-172.17.73.37:3260-iscsi-iqn.2020-02.com:wsg-lun-1 /var/lib/kubelet/plugins/kubernetes.io/iscsi/iface-default/172.17.73.37:3260-iqn.2020-02.com:wsg-lun-1

```



iscsi pv配置文件如下：

```
wsg@k8smaster2:~/helm/chart$ cat pv-iscsi-prometheus-grafana.yml 
--- 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus-pv
  labels:
    app: wsg-prometheus
spec:
  capacity:
    storage: 800Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  iscsi:
    targetPortal: 172.17.73.37:3260
    iqn: iqn.2020-02.com:wsg
    lun: 1
    fsType: ext4
    readOnly: false
    


---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus-grafana-pv
  labels:
    app: wsg-grafana
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  iscsi:
    targetPortal: 172.17.73.37:3260
    iqn: iqn.2020-02.com:wsg
    lun: 2
    fsType: ext4
    readOnly: false

wsg@k8smaster2:~/helm/chart$ 

```



## iscsi lun 分区后，k8s 挂载报错信息如下：

```
un 20 10:37:58 k8smaster2 kernel: [89069.266974]  connection1:0: detected conn error (1020)
Jun 20 10:38:09 k8smaster2 kernel: [89080.473511] sd 3:0:0:1: Power-on or device reset occurred
Jun 20 10:38:09 k8smaster2 kernel: [89080.637096]  sdc: sdc1
Jun 20 10:38:09 k8smaster2 kernel: [89080.648508] EXT4-fs (sdc): VFS: Can't find ext4 filesystem
Jun 20 10:38:09 k8smaster2 kernel: [89080.788179] EXT4-fs (sdc): VFS: Can't find ext4 filesystem
Jun 20 10:38:10 k8smaster2 kernel: [89081.284024]  connection1:0: detected conn error (1020)
Jun 20 10:38:10 k8smaster2 kernel: [89081.285509]  connection1:0: detected conn error (1020)

```



# 补充2： nfs 作为grafana的持久化存储



修改grafana中values配置，将其中的 persistent enalbe 就可以，然后创建 同样大小的pv，就可以正常bound。（看起来之前pvc 和pv bound 也不需要 显示指定 label来完成匹配）

```
197 persistence:
198   type: pvc
199   enabled: true
200   # storageClassName: default
201   accessModes:
202     - ReadWriteOnce
203   size: 10Gi
204   # annotations: {}
205   finalizers:
206     - kubernetes.io/pvc-protection
207   # subPath: ""
208   # existingClaim:
209 

```

pv的配置：

```
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus-grafana-pv
  labels:
    app: wsg-grafana
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 172.17.73.37
    path: /vol/k8s-prometheus/prometheus_db

```







# 补充3：nfs 作为prometheus DB

> 补充知识点： prometheus-operator重新部署后，如果原来的nfs -pv还存在，好像就无法bound成功，需要把pv 删除再 重新创建，pvc才能bound。





上面使用本地存储卷的方式在 kubernets v1.18环境中（上面的步骤我是在v1.14 版本上测试work），怀疑v.18在 local storage方面可能有过调整。 不过有替代方案了，可以通过nfs的方式，这样使用起来更方便些（不需要定义storageclass）。因为上面使用local volume时，需要定义一个storageclass。 后面如果nfs的性能不满足，可以进一步考虑用iscsi的方式替代。



**nfs的使用方式如下**

环境 基于kubernets v1.18

````
wsg@k8smater1:~/kube_deploy/jenkins$ kubectl get node -o wide
NAME         STATUS   ROLES    AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE           KERNEL-VERSION      CONTAINER-RUNTIME
k8smaster2   Ready    master   54d   v1.18.2   172.17.73.65   <none>        Ubuntu 16.04 LTS   4.4.0-184-generic   docker://18.9.7
k8smater1    Ready    master   54d   v1.18.2   172.17.73.66   <none>        Ubuntu 16.04 LTS   4.4.0-184-generic   docker://18.9.7
wsg@k8smater1:~/kube_deploy/jenkins$ 

````



- 修改prometheus-operator配置

  需要使用persistent volume的组件，除了 prometheus，granfan也需要使用。

  修改operator中value.yaml 中，prometheus部分 storage的配置，如下

  ```
      storageSpec:
        volumeClaimTemplate:
          spec:
            resources:
              requests:
                storage: 800Gi
            selector:
              matchLabels:
                app: wsg-prometheus
  wsg@k8smaster2:~/helm/chart/prometheus-operator$ vim values.yaml 
  
  
  ```

  

  然后定义对应的pv （因为用的是kubernet自带的storageclass，所以无需定义 storageclass）,根据coreos的官方说明，pvc中label和pv中的label要匹配（可以测试 如果pvc中不定义select，是否就无法bound到pv ？？测试见后面）。

  ```
  wsg@k8smaster2:~/helm/chart/prometheus-operator$ cat ../pv-pvc.yml 
  
  --- 
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: prometheus-pv
    labels:
      app: wsg-prometheus
  spec:
    capacity:
      storage: 800Gi
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Retain
    nfs:
      server: 172.17.73.37
      path: /vol/k8s-prometheus/prometheus_db
  
  ```

  

- apply上面的pv配置后，helm 部署prometheus

  检查 pvc 和pod状态，可以看到正常bound

  ```
  wsg@k8smater1:~/kube_deploy/jenkins$ kubectl get pvc
  NAME                                                                                                     STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   AGE
  prometheus-prometheus-operator-releas-prometheus-db-prometheus-prometheus-operator-releas-prometheus-0   Bound    prometheus-pv   800Gi      RWO                           4m18s
  wsg@k8smater1:~/kube_deploy/jenkins$ kubectl get pv
  NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                                                                            STORAGECLASS   REASON   AGE
  jenkins-pv      500Gi      RWX            Retain           Bound    ci/jenkins-pvc                                                                                                                           53d
  prometheus-pv   800Gi      RWO            Retain           Bound    default/prometheus-prometheus-operator-releas-prometheus-db-prometheus-prometheus-operator-releas-prometheus-0                           6m36s
  wsg@k8smater1:~/kube_deploy/jenkins$ 
  
  ```

  



## 测试 prometheus pvc申明中 不配置 selector

测试结论： 必须指定pvc中的selector， 否则无法正常bound。 测试过程如下：



values配置如下

````
    ## Prometheus StorageSpec for persistent data
    ## ref: https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/storage.md
    ##
    storageSpec:
      volumeClaimTemplate:
        spec:
          resources:
            requests:
              storage: 800Gi
    #      selector:
    #        matchLabels:
    #          app: wsg-prometheus


````

查看pvc的bound情况

```
wsg@k8smater1:~/kube_deploy/jenkins$ kubectl get pvc
NAME                                                                                                     STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
prometheus-prometheus-operator-releas-prometheus-db-prometheus-prometheus-operator-releas-prometheus-0   Pending                                                     75s
wsg@k8smater1:~/kube_deploy/jenkins$ 


```

查看pvc pending的原因

```
wsg@k8smater1:~/kube_deploy/jenkins$ kubectl describe pvc prometheus-prometheus-operator-releas-prometheus-db-prometheus-prometheus-operator-releas-prometheus-0 
Name:          prometheus-prometheus-operator-releas-prometheus-db-prometheus-prometheus-operator-releas-prometheus-0
Namespace:     default
StorageClass:  
Status:        Pending
Volume:        
Labels:        app=prometheus
               prometheus=prometheus-operator-releas-prometheus
Annotations:   <none>
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      
Access Modes:  
VolumeMode:    Filesystem
Mounted By:    prometheus-prometheus-operator-releas-prometheus-0
Events:
  Type    Reason         Age                From                         Message
  ----    ------         ----               ----                         -------
  Normal  FailedBinding  8s (x9 over 113s)  persistentvolume-controller  no persistent volumes available for this claim and no storage class is set
wsg@k8smater1:~/kube_deploy/jenkins$ 

```









