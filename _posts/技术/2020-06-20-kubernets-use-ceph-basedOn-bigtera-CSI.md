

# 背景

容器的持久化存储：容器的存储，也就是容器的volume，其实就是将host上的某个目录和容器内的某个目录绑定在一起。而所谓的持久化存储，也就是指 host上的目录具有持久性（所谓持久性就是 目录内的内容不会因为容器删除而删除，也不和该host 主机绑定）

```
hostPath 和 emptyDir 类型的 Volume 并不具备这个特征：它们既有可能被 kubelet 清理掉，也不能被“迁移”到其他节点上
```



kubernet 为容器提供的持久化存储，就是利用外部的存储（比如nfs，glusterfs，iscsi等），在host主机上创建出持久化的目录。 一般处理的流程是，pod 被调度到一个节点上后，kubelet会创建pod需要的volume， 创建的过程分为2个阶段，attach 和mount阶段。

使用NFS ，ISCSI的存储时，kubernet无需单独安装存储插件，但是这些存储需要提前手动部署好pv（static provisioning）；如果想使用ceph rbd这样的存储时，需要安装额外的存储插件，并且可以dynamic provisioning。本文整理整个存储插件的部署过程，以及相关原理探索。（官方也支持的ceph的支持。但是我们自己的ceph在开源的基础上有修改，需要用我们自己的ceph 包，所以需要用我们自己的存储插件。我们存储插件也经过了kubernet的认证: https://kubernetes-csi.github.io/docs/drivers.html）

# 部署步骤

> 注意kubernet OS版本要求：
>
> centos： CentOS 7.3-1611 (Kernel v3.10.0-514) 或以上
>
> ubuntu：ubuntu18.04（kernel 4.15以上）， 或者ubuntu16.04的版本直接在线升级到ubuntu18
>
> 



## 存储端配置

1. 启用 ceph的volume模块

   ```
   ceph mgr module enable volumes
   
   确认生效是否成功，如下，enabled的modules 中已经包括了 volumes
   root@node3:~# ceph mgr module ls
   {
       "enabled_modules": [
           "balancer",
           "status",
           "virtualstor",
           "volumes"
       ],
       "disabled_modules": [
           "dashboard",
           "influx",
           "localpool",
           "prometheus",
           "restful",
           "selftest",
           "zabbix"
       ]
   }
   
   ```

2. 修改ceph.conf

   增加下面的配置项

   ```
   client allow non bigtera = true
   ```

3. 重启ceph 集群

## 下载定制化的csi插件

https://github.com/ceph/ceph-csi

下载后的目录结构如下

```
wsg@ubuntu18:~/ceph-csi/examples/rbd$ ll
total 64
drwxrwxr-x 2 wsg wsg 4096 Jun 19 15:22 ./
drwxrwxr-x 4 wsg wsg 4096 Jun 19 15:30 ../
-rwxrwxr-x 1 wsg wsg  395 Jun 19 14:50 exec-bash.sh*
-rwxrwxr-x 1 wsg wsg  384 Jun 19 14:50 logs.sh*
-rwxrwxr-x 1 wsg wsg  332 Jun 19 14:50 plugin-deploy.sh*
-rwxrwxr-x 1 wsg wsg  332 Jun 19 14:50 plugin-teardown.sh*
-rw-rw-r-- 1 wsg wsg  332 Jun 19 14:50 pod-restore.yaml
-rw-rw-r-- 1 wsg wsg  316 Jun 19 14:50 pod.yaml
-rw-rw-r-- 1 wsg wsg  303 Jun 19 14:50 pvc-restore.yaml
-rw-rw-r-- 1 wsg wsg  191 Jun 19 14:50 pvc.yaml
-rw-rw-r-- 1 wsg wsg  372 Jun 19 14:50 raw-block-pod.yaml
-rw-rw-r-- 1 wsg wsg  217 Jun 19 14:50 raw-block-pvc.yaml
-rw-rw-r-- 1 wsg wsg  440 Jun 19 14:50 secret.yaml
-rw-rw-r-- 1 wsg wsg  768 Jun 19 14:50 snapshotclass.yaml
-rw-rw-r-- 1 wsg wsg  216 Jun 19 14:50 snapshot.yaml
-rw-rw-r-- 1 wsg wsg 2161 Jun 19 14:50 storageclass.yaml
wsg@ubuntu18:~/ceph-csi/examples/rbd$ pwd
/home/wsg/ceph-csi/examples/rbd

```

## 安装csi插件

```
wsg@k8smaster2:~/ceph-csi/examples/rbd$ ./plugin-deploy.sh 
serviceaccount/rbd-csi-provisioner created
clusterrole.rbac.authorization.k8s.io/rbd-external-provisioner-runner created
clusterrole.rbac.authorization.k8s.io/rbd-external-provisioner-runner-rules created
clusterrolebinding.rbac.authorization.k8s.io/rbd-csi-provisioner-role created
role.rbac.authorization.k8s.io/rbd-external-provisioner-cfg created
rolebinding.rbac.authorization.k8s.io/rbd-csi-provisioner-role-cfg created
serviceaccount/rbd-csi-nodeplugin created
clusterrole.rbac.authorization.k8s.io/rbd-csi-nodeplugin created
clusterrole.rbac.authorization.k8s.io/rbd-csi-nodeplugin-rules created
clusterrolebinding.rbac.authorization.k8s.io/rbd-csi-nodeplugin created
configmap/ceph-csi-config created
service/csi-rbdplugin-provisioner created
deployment.apps/csi-rbdplugin-provisioner created
daemonset.apps/csi-rbdplugin created
service/csi-metrics-rbdplugin created
wsg@k8smaster2:~/ceph-csi/examples/rbd$ 
可以看到部署过程除了创建k8s相关的资源，包括serviceaccount，rbac，service，deployment，daemonset等编排对象。 重点需要关注 其中的daemonset 和deployment。这些编排对象的yaml 配置文件如下（上面部署的脚本里面其实就是执行下面这些配置文件）

wsg@k8smaster2:~/ceph-csi/deploy/rbd/kubernetes$ ll
total 52
drwxrwxr-x 2 wsg wsg 4096 Jun 23 15:52 ./
drwxrwxr-x 3 wsg wsg 4096 Jun  8 16:18 ../
-rw-rw-r-- 1 wsg wsg  100 Jun  8 16:18 csi-config-map.yaml
-rw-rw-r-- 1 wsg wsg 1691 Jun  8 16:18 csi-nodeplugin-psp.yaml
-rw-rw-r-- 1 wsg wsg 1336 Jun  8 16:18 csi-nodeplugin-rbac.yaml
-rw-rw-r-- 1 wsg wsg 1312 Jun  8 16:18 csi-provisioner-psp.yaml
-rw-rw-r-- 1 wsg wsg 3402 Jun  8 16:18 csi-provisioner-rbac.yaml
-rw-rw-r-- 1 wsg wsg 5680 Jun  8 16:18 csi-rbdplugin-onenode.yaml
-rw-rw-r-- 1 wsg wsg 5382 Jun  8 16:18 csi-rbdplugin-provisioner.yaml
-rw-rw-r-- 1 wsg wsg 5652 Jun  8 16:18 csi-rbdplugin.yaml
wsg@k8smaster2:~/ceph-csi/deploy/rbd/kubernetes$ vim csi-rbdplugin.yaml 
wsg@k8smaster2:~/ceph-csi/deploy/rbd/kubernetes$ 

```

执行完上面的安装脚本，会在 default namespace创建出相应的管理pod(管理pod会运行在每个节点)

```
wsg@k8smaster2:~/ceph-csi/examples$ kubectl get pod
NAME                                                              READY   STATUS    RESTARTS   AGE
csi-rbdplugin-8jzhg                                               3/3     Running   0          39m
csi-rbdplugin-c59sm                                               3/3     Running   0          39m
csi-rbdplugin-provisioner-968f8d7c6-2kdj5                         6/6     Running   0          39m
csi-rbdplugin-provisioner-968f8d7c6-pwxct                         6/6     Running   0          39m
csi-rbdplugin-provisioner-968f8d7c6-zp95q                         6/6     Running   0          39m

```





## csi配置和生效

> 注意：有2个配置文件需要设置 cluster id，

- 1，配置集群的连接信息（csi-config-map-sample.yaml）

  如下，要设置cluster id 和cluster的monitor成员

  ```
  wsg@k8smaster2:~/ceph-csi/examples$ cat csi-config-map-sample.yaml 
  ---
  apiVersion: v1
  kind: ConfigMap
  data:
    config.json: |-
      [
        {
          "clusterID": "bf27339d-a4fc-42db-a9af-3f3fed8e1f2c",
          "monitors": [
            "172.17.73.60"
          ]
        }
      ]
  metadata:
    name: ceph-csi-config
  wsg@k8smaster2:~/ceph-csi/examples$ 
  
  ```

  

- 2，storageclass.yam 配置

  其中需要设置 ceph cluster id 和pool

  ```
  wsg@k8smaster2:~/ceph-csi/examples/rbd$ cat storageclass.yaml  过滤掉其中的注释
  ---
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
     name: csi-rbd-sc
  provisioner: csi.block.bigtera.com
  parameters:
     clusterID: bf27339d-a4fc-42db-a9af-3f3fed8e1f2c
     pool: rbd
     imageFeatures: layering
     csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
     csi.storage.k8s.io/provisioner-secret-namespace: default
     csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
     csi.storage.k8s.io/controller-expand-secret-namespace: default
     csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
     csi.storage.k8s.io/node-stage-secret-namespace: default
     csi.storage.k8s.io/fstype: ext4
  reclaimPolicy: Delete
  allowVolumeExpansion: true
  mountOptions:
     - discard
  ```

  

- 部署相关配置

  包括上面需要修改的2个配置文件

  ```
  
  kubectl apply -f csi-config-map-sample.yaml 
  kubectl apply -f secret.yaml
  kubectl apply -f storageclass.yaml 
  kubectl apply -f pvc.yaml
  
  ```

  

## 检查部署后的pvc

后面测试pod的定义中，会引用该pvc，所以需要先部署好pvc，再部署测试pod。

如下，可以看到pvc 已经正常bound（pvc能够正常bound，正是因为它的yaml文件中定义了 storageclass）

```
wsg@k8smaster2:~/ceph-csi/examples/rbd$ kubectl get pvc
NAME                                                                                                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
rbd-pvc                                                                                                  Bound    pvc-7a4097ef-5d59-4c78-bd5d-d3e8a50b0765   1Gi        RWO            csi-rbd-sc     113m
wsg@k8smaster2:~/ceph-csi/examples/rbd$

```



pvc的yaml 定义文件如下

```
wsg@k8smaster2:~/ceph-csi/examples/rbd$ cat pvc.yaml 
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-rbd-sc

```



查看bound的详细信息，可以看到具体的provisioner

```
wsg@k8smaster2:~/ceph-csi/examples/rbd$ kubectl describe pvc rbd-pvc
Name:          rbd-pvc
Namespace:     default
StorageClass:  csi-rbd-sc
Status:        Bound
Volume:        pvc-7a4097ef-5d59-4c78-bd5d-d3e8a50b0765
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
               volume.beta.kubernetes.io/storage-provisioner: csi.block.bigtera.com
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      1Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Mounted By:    <none>
Events:        <none>

```

进一步查看 pvc 所bound的pv信息

```
wsg@k8smaster2:~/ceph-csi/examples/rbd$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                                                                                                            STORAGECLASS   REASON   AGE
pvc-7a4097ef-5d59-4c78-bd5d-d3e8a50b0765   1Gi        RWO            Delete           Bound      default/rbd-pvc                                                                                                  csi-rbd-sc              105m
pvc-ad66c618-a2b0-48db-bea1-9f822f39d47b   1Gi        RWO            Delete           Released   default/rbd-pvc                                                                                                  csi-rbd-sc              3d22h
wsg@k8smaster2:~/ceph-csi/examples/rbd$ 

```



ceph存储端查看rbd 信息

```
root@node3:~# rbd ls
csi-vol-9333dd85-b506-11ea-953e-5235b1be5b28
root@node3:~# rbd info csi-vol-9333dd85-b506-11ea-953e-5235b1be5b28
rbd image 'csi-vol-9333dd85-b506-11ea-953e-5235b1be5b28':
	size 1GiB in 256 objects
	order 22 (4MiB objects)
	used objects: 0
	block_name_prefix: rbd_data.8c2a34dd154fe
	format: 2
	features: layering
	flags: 
	create_timestamp: Tue Jun 23 12:04:06 2020
root@node3:~# 

```



## 部署测试pod

pod 删除后重建，保留在rbd中的数据是可以再次使用的。

进入pod 后观察

```
root@csi-rbd-demo-pod:/# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0  500G  0 disk 
`-sda1                    8:1    0  500G  0 part 
  |-ubuntu18--vg-root   253:0    0  499G  0 lvm  /etc/hosts
  `-ubuntu18--vg-swap_1 253:1    0  980M  0 lvm  
sr0                      11:0    1  900M  0 rom  
rbd0                    252:0    0    1G  0 disk /var/lib/www/html
root@csi-rbd-demo-pod:/# 

/dev/rbd0 /var/lib/www/html ext4 rw,relatime,stripe=1024,data=ordered 0 0


```



host主上观察

```
wsg@ubuntu18:~$ lsblk 
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0  500G  0 disk 
└─sda1                    8:1    0  500G  0 part 
  ├─ubuntu18--vg-root   253:0    0  499G  0 lvm  /
  └─ubuntu18--vg-swap_1 253:1    0  980M  0 lvm  
sr0                      11:0    1  900M  0 rom  
rbd0                    252:0    0    1G  0 disk /var/lib/kubelet/pods/09eb0041-cc7b-4734-b11c-521f8e40637a/volumes/kubernetes.io~csi/pvc-3349781b-1367-43f8-8d1d-81d118644729/mount
wsg@ubuntu18:~$ 

```



## pod 性能

dd绕过内存，看起来

```
root@csi-rbd-demo-pod:/var/lib/www/html/in_pod# dd if=/dev/zero  of=dd.2 bs=1M count=200 oflag=direct
200+0 records in
200+0 records out
209715200 bytes (210 MB, 200 MiB) copied, 7.29218 s, 28.8 MB/s
root@csi-rbd-demo-pod:/var/lib/www/html/in_pod# 

```

# csi进阶





## 原理探索

正常使用ceph rbd一般通过 librbd（用户态）和krbd（内核态）的方式使用，k8s通过krbd的方式使用，通过pod 的特殊的mount option(bidirection)，在特定的pod(daemonset pod) 内将rbd map，并且mount信息同步到host主机上，然后再 分配给需要的pod。 

csi主要有2个模块，一个是controller component （deployment或者statefulset 类型），一个是node-component （daemonset 类型，与kubelet 交互，完成volume的）

* controller component 

  The controller component can be deployed as a Deployment or StatefulSet on any node in the cluster. It consists of the CSI driver that implements the CSI Controller service and one or more [sidecar containers](https://kubernetes-csi.github.io/docs/sidecar-containers.html). These controller sidecar containers typically interact with Kubernetes objects and make calls to the driver's CSI Controller service.
  
  It generally does not need direct access to the host and can perform all its operations through the Kubernetes API and external control plane services. Multiple copies of the controller component can be deployed for HA, however it is recommended to use leader election to ensure there is only one active controller at a time.

  Controller sidecars include the external-provisioner, external-attacher, external-snapshotter, and external-resizer. Including a sidecar in the deployment may be optional. See each sidecar's page for more details.

- node component 

  The node component should be deployed on every node in the cluster through a DaemonSet. It consists of the CSI driver that implements the CSI Node service and the [node-driver-registrar](https://kubernetes-csi.github.io/docs/node-driver-registrar) sidecar container.

  

https://kubernetes-csi.github.io/docs/deploying.html

csi相关的pod 如下：每个host节点会有daemonset pod （node component）和 provisioner pod （controller component ）

```
wsg@k8smater1:~$ kubectl get pod -o wide
NAME                                                              READY   STATUS    RESTARTS   AGE     IP                NODE         NOMINATED NODE   READINESS GATES
csi-rbdplugin-8jzhg                                               3/3     Running   0          3h10m   172.17.73.66      k8smater1    <none>           <none>
csi-rbdplugin-c59sm                                               3/3     Running   0          3h10m   172.17.73.65      k8smaster2   <none>           <none>
csi-rbdplugin-provisioner-968f8d7c6-2kdj5                         6/6     Running   0          3h10m   192.168.129.59    k8smater1    <none>           <none>
csi-rbdplugin-provisioner-968f8d7c6-pwxct                         6/6     Running   0          3h10m   192.168.137.95    k8smaster2   <none>           <none>
csi-rbdplugin-provisioner-968f8d7c6-zp95q                         6/6     Running   0          3h10m   192.168.137.100   k8smaster2   <none>           <none>



```



###daemonset pod的信息

 pod中会用到3个image

其中有 driver-registrar，image为 quay.io/k8scsi/csi-node-driver-registrar:v1.2.0

bigtera/bigtera-csi:v2.0-canary

liveness-prometheus，bigtera/bigtera-csi:v2.0-canary ？看起来只是为 提供监控用的？



完整的pod 信息如下

```
wsg@k8smater1:~$ kubectl describe pod csi-rbdplugin-8jzhg 
Name:         csi-rbdplugin-8jzhg
Namespace:    default
Priority:     0
Node:         k8smater1/172.17.73.66
Start Time:   Tue, 23 Jun 2020 11:04:43 +0800
Labels:       app=csi-rbdplugin
              controller-revision-hash=787744bcd8
              pod-template-generation=1
Annotations:  <none>
Status:       Running
IP:           172.17.73.66
IPs:
  IP:           172.17.73.66
Controlled By:  DaemonSet/csi-rbdplugin
Containers:
  driver-registrar:
    Container ID:  docker://847ff75f444d356cc703da8eb2d07cbe1156ffe60b1de683946cffea824c6a92
    Image:         quay.io/k8scsi/csi-node-driver-registrar:v1.2.0
    Image ID:      docker-pullable://quay.io/k8scsi/csi-node-driver-registrar@sha256:89cdb2a20bdec89b75e2fbd82a67567ea90b719524990e772f2704b19757188d
    Port:          <none>
    Host Port:     <none>
    Args:
      --v=5
      --csi-address=/csi/csi.sock
      --kubelet-registration-path=/var/lib/kubelet/plugins/csi.block.bigtera.com/csi.sock
    State:          Running
      Started:      Tue, 23 Jun 2020 11:04:44 +0800
    Ready:          True
    Restart Count:  0
    Environment:
      KUBE_NODE_NAME:   (v1:spec.nodeName)
    Mounts:
      /csi from socket-dir (rw)
      /registration from registration-dir (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from rbd-csi-nodeplugin-token-s4xt6 (ro)
  csi-rbdplugin:
    Container ID:  docker://06e500cae9e44154f5061bad88950bff60fff9a019530796251f647dccc65ca3
    Image:         bigtera/bigtera-csi:v2.0-canary
    Image ID:      docker-pullable://bigtera/bigtera-csi@sha256:047d608ee7a84e97625f980f7ebe9b9a6f03e46928262b6810d26d84cbfa356e
    Port:          <none>
    Host Port:     <none>
    Args:
      --nodeid=$(NODE_ID)
      --type=rbd
      --nodeserver=true
      --endpoint=$(CSI_ENDPOINT)
      --v=5
      --drivername=csi.block.bigtera.com
      --metricsport=8090
      --metricspath=/metrics
      --enablegrpcmetrics=false
    State:          Running
      Started:      Tue, 23 Jun 2020 11:04:45 +0800
    Ready:          True
    Restart Count:  0
    Environment:
      POD_IP:          (v1:status.podIP)
      NODE_ID:         (v1:spec.nodeName)
      POD_NAMESPACE:  default (v1:metadata.namespace)
      CSI_ENDPOINT:   unix:///csi/csi.sock
    Mounts:
      /csi from socket-dir (rw)
      /dev from host-dev (rw)
      /etc/ceph-csi-config/ from ceph-csi-config (rw)
      /lib/modules from lib-modules (ro)
      /run/mount from host-mount (rw)
      /sys from host-sys (rw)
      /tmp/csi/keys from keys-tmp-dir (rw)
      /var/lib/kubelet/plugins from plugin-dir (rw)
      /var/lib/kubelet/pods from mountpoint-dir (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from rbd-csi-nodeplugin-token-s4xt6 (ro)
  liveness-prometheus:
    Container ID:  docker://193036bcbc32ced9b950c15b84d55d26b68167134b90b4e7795b8b602a530cf2
    Image:         bigtera/bigtera-csi:v2.0-canary
    Image ID:      docker-pullable://bigtera/bigtera-csi@sha256:047d608ee7a84e97625f980f7ebe9b9a6f03e46928262b6810d26d84cbfa356e
    Port:          <none>
    Host Port:     <none>
    Args:
      --type=liveness
      --endpoint=$(CSI_ENDPOINT)
      --metricsport=8680
      --metricspath=/metrics
      --polltime=60s
      --timeout=3s
    State:          Running
      Started:      Tue, 23 Jun 2020 11:04:45 +0800
    Ready:          True
    Restart Count:  0
    Environment:
      CSI_ENDPOINT:  unix:///csi/csi.sock
      POD_IP:         (v1:status.podIP)
    Mounts:
      /csi from socket-dir (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from rbd-csi-nodeplugin-token-s4xt6 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  socket-dir:
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/kubelet/plugins/csi.block.bigtera.com
    HostPathType:  DirectoryOrCreate
  plugin-dir:
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/kubelet/plugins
    HostPathType:  Directory
  mountpoint-dir:
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/kubelet/pods
    HostPathType:  DirectoryOrCreate
  registration-dir:
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/kubelet/plugins_registry/
    HostPathType:  Directory
  host-dev:
    Type:          HostPath (bare host directory volume)
    Path:          /dev
    HostPathType:  
  host-sys:
    Type:          HostPath (bare host directory volume)
    Path:          /sys
    HostPathType:  
  host-mount:
    Type:          HostPath (bare host directory volume)
    Path:          /run/mount
    HostPathType:  
  lib-modules:
    Type:          HostPath (bare host directory volume)
    Path:          /lib/modules
    HostPathType:  
  ceph-csi-config:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      ceph-csi-config
    Optional:  false
  keys-tmp-dir:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     Memory
    SizeLimit:  <unset>
  rbd-csi-nodeplugin-token-s4xt6:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  rbd-csi-nodeplugin-token-s4xt6
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/disk-pressure:NoSchedule
                 node.kubernetes.io/memory-pressure:NoSchedule
                 node.kubernetes.io/network-unavailable:NoSchedule
                 node.kubernetes.io/not-ready:NoExecute
                 node.kubernetes.io/pid-pressure:NoSchedule
                 node.kubernetes.io/unreachable:NoExecute
                 node.kubernetes.io/unschedulable:NoSchedule
Events:          <none>
wsg@k8smater1:~$ kubectl get pod -o wide

```

#### 进入daemonset 中 container (csi-rbdplugin)

```
 kubectl exec -it csi-rbdplugin-8jzhg --container csi-rbdplugin -- /bin/bash


[root@k8smater1 bin]# ll -lh ceph*
-rwxr-xr-x 1 root root  44K Mar  2 18:09 ceph
-rwxr-xr-x 1 root root 187K Mar  2 18:59 ceph-authtool
-rwxr-xr-x 1 root root 8.2M Mar  2 18:59 ceph-bluestore-tool
-rwxr-xr-x 1 root root 1.5K Mar  2 17:49 ceph-clsinfo
-rwxr-xr-x 1 root root 179K Mar  2 18:59 ceph-conf
-rwxr-xr-x 1 root root 2.9K Mar  2 18:09 ceph-crash
-rwxr-xr-x 1 root root 9.6M Mar  2 19:01 ceph-dencoder.gz
-rwxr-xr-x 1 root root 1.6M Mar  2 18:59 ceph-fuse
-rwxr-xr-x 1 root root 7.9M Mar  2 18:59 ceph-kvstore-tool
-rwxr-xr-x 1 root root 5.5M Mar  2 18:59 ceph-mds
-rwxr-xr-x 1 root root 4.0M Mar  2 18:59 ceph-mgr
-rwxr-xr-x 1 root root 8.2M Mar  2 18:58 ceph-mon
-rwxr-xr-x 1 root root 5.0M Mar  2 18:59 ceph-monstore-tool
-rwxr-xr-x 1 root root  15M Mar  2 18:59 ceph-objectstore-tool
-rwxr-xr-x 1 root root  22M Mar  2 18:58 ceph-osd
-rwxr-xr-x 1 root root 4.9M Mar  2 18:59 ceph-osdomap-tool
-rwxr-xr-x 1 root root 4.1K Mar  2 18:09 ceph-post-file
-rwxr-xr-x 1 root root  275 Mar  2 17:49 ceph-rbdnamer
-rwxr-xr-x 1 root root  296 Mar  2 17:49 ceph-run
-rwxr-xr-x 1 root root 1.7M Mar  2 18:59 ceph-syn
-rwxr-xr-x 1 root root 6.2M Mar  2 19:00 cephfs-data-scan
-rwxr-xr-x 1 root root 6.3M Mar  2 19:00 cephfs-journal-tool
-rwxr-xr-x 1 root root 6.1M Mar  2 19:00 cephfs-table-tool
[root@k8smater1 bin]# pwd
/bin

```



查看pod 的磁盘情况，

？该pod是一个特权pod，看到了挂载点 其他的 sub mount （iscsi挂载），并且可以看到 rbd 是在 该

```
[root@k8smater1 /]# lsblk 
NAME                       MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
rbd0                       252:0    0     1G  0 disk /var/lib/kubelet/pods/527b1185-b9b1-47cb-ad35-6fc9352069a3/volumes/kubernetes.io~csi/pvc-7a4097ef-5d59-4c78-bd5d-d3e8a50b0765/mount
sdd                          8:48   0    20G  0 disk /var/lib/kubelet/plugins/kubernetes.io/iscsi/iface-default/172.17.73.37:3260-iqn.2020-02.com:wsg-lun-2
sdb                          8:16   0     1T  0 disk 
sr0                         11:0    1  1024M  0 rom  
sdc                          8:32   0   800G  0 disk /var/lib/kubelet/plugins/kubernetes.io/iscsi/iface-default/172.17.73.37:3260-iqn.2020-02.com:wsg-lun-1
sda                          8:0    0   500G  0 disk 
|-sda2                       8:2    0     1K  0 part 
|-sda5                       8:5    0 499.5G  0 part 
| |-ubuntu16wsg--vg-swap_1 253:1    0     8G  0 lvm  
| `-ubuntu16wsg--vg-root   253:0    0 491.5G  0 lvm  /var/lib/kubelet/plugins
`-sda1                       8:1    0   487M  0 part 
[root@k8smater1 /]# 

```

查看pod mount 信息

```
[root@k8smater1 /]# mount |grep rbd
tmpfs on /var/lib/kubelet/pods/691d0462-82a4-48d9-9f42-a475afee7c0f/volumes/kubernetes.io~secret/rbd-csi-provisioner-token-p6ttt type tmpfs (rw,relatime)
tmpfs on /var/lib/kubelet/pods/5d0e68b0-aeeb-4b2f-b32d-9f6e7412b20d/volumes/kubernetes.io~secret/rbd-csi-nodeplugin-token-s4xt6 type tmpfs (rw,relatime)
/dev/rbd0 on /var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-7a4097ef-5d59-4c78-bd5d-d3e8a50b0765/globalmount/0001-0024-bf27339d-a4fc-42db-a9af-3f3fed8e1f2c-000000000000000c-9333dd85-b506-11ea-953e-5235b1be5b28 type ext4 (rw,relatime,stripe=1024,data=ordered,_netdev)
/dev/rbd0 on /var/lib/kubelet/pods/527b1185-b9b1-47cb-ad35-6fc9352069a3/volumes/kubernetes.io~csi/pvc-7a4097ef-5d59-4c78-bd5d-d3e8a50b0765/mount type ext4 (rw,relatime,stripe=1024,data=ordered,_netdev)
[root@k8smater1 /]# 

```





### provisioner pod信息

pod的类型为replicaset

csi-provisioner 	quay.io/k8scsi/csi-provisioner:v1.4.0

csi-snapshotter:	quay.io/k8scsi/csi-snapshotter:v1.2.2

csi-attacher:	quay.io/k8scsi/csi-attacher:v2.1.0

csi-resizer:	quay.io/k8scsi/csi-resizer:v0.4.0

csi-rbdplugin:	bigtera/bigtera-csi:v2.0-canary

liveness-prometheus:	bigtera/bigtera-csi:v2.0-canary



完整的pod 信息

```
wsg@k8smater1:~$ kubectl describe pod csi-rbdplugin-provisioner-968f8d7c6-2kdj5
Name:         csi-rbdplugin-provisioner-968f8d7c6-2kdj5
Namespace:    default
Priority:     0
Node:         k8smater1/172.17.73.66
Start Time:   Tue, 23 Jun 2020 11:04:42 +0800
Labels:       app=csi-rbdplugin-provisioner
              pod-template-hash=968f8d7c6
Annotations:  cni.projectcalico.org/podIP: 192.168.129.59/32
              cni.projectcalico.org/podIPs: 192.168.129.59/32
Status:       Running
IP:           192.168.129.59
IPs:
  IP:           192.168.129.59
Controlled By:  ReplicaSet/csi-rbdplugin-provisioner-968f8d7c6
Containers:
  csi-provisioner:
    Container ID:  docker://8e8777af85b9d7e4f19bf4c6c79085ab5acac17ec0f73ee500bcb18d9b11ecf5
    Image:         quay.io/k8scsi/csi-provisioner:v1.4.0
    Image ID:      docker-pullable://quay.io/k8scsi/csi-provisioner@sha256:3a14f801f330d5eacee11f544d2c2c7cc4a733835e25b59887053358108cea69
    Port:          <none>
    Host Port:     <none>
    Args:
      --csi-address=$(ADDRESS)
      --v=5
      --timeout=150s
      --retry-interval-start=500ms
      --enable-leader-election=true
      --leader-election-type=leases
    State:          Running
      Started:      Tue, 23 Jun 2020 11:04:45 +0800
    Ready:          True
    Restart Count:  0
    Environment:
      ADDRESS:  unix:///csi/csi-provisioner.sock
    Mounts:
      /csi from socket-dir (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from rbd-csi-provisioner-token-p6ttt (ro)
  csi-snapshotter:
    Container ID:  docker://35c0b8cd8a835d07087e7cd028846790f890f838d89cd49a3592b5ed406068ca
    Image:         quay.io/k8scsi/csi-snapshotter:v1.2.2
    Image ID:      docker-pullable://quay.io/k8scsi/csi-snapshotter@sha256:fbef4d21ec79c9c5aced2e0e0396b9272e69ffe4cc718c3602ae6a466f682d41
    Port:          <none>
    Host Port:     <none>
    Args:
      --csi-address=$(ADDRESS)
      --v=5
      --timeout=150s
      --leader-election=true
    State:          Running
      Started:      Tue, 23 Jun 2020 11:04:50 +0800
    Ready:          True
    Restart Count:  0
    Environment:
      ADDRESS:  unix:///csi/csi-provisioner.sock
    Mounts:
      /csi from socket-dir (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from rbd-csi-provisioner-token-p6ttt (ro)
  csi-attacher:
    Container ID:  docker://2e1235e5ccf94cd908067ef883d0f58dd995d3062ec1ab83a4d84415fa7d6209
    Image:         quay.io/k8scsi/csi-attacher:v2.1.0
    Image ID:      docker-pullable://quay.io/k8scsi/csi-attacher@sha256:e2dd4c337f1beccd298acbd7179565579c4645125057dfdcbbb5beef977b779e
    Port:          <none>
    Host Port:     <none>
    Args:
      --v=5
      --csi-address=$(ADDRESS)
      --leader-election=true
    State:          Running
      Started:      Tue, 23 Jun 2020 11:04:50 +0800
    Ready:          True
    Restart Count:  0
    Environment:
      ADDRESS:        /csi/csi-provisioner.sock
      POD_NAMESPACE:  default (v1:metadata.namespace)
    Mounts:
      /csi from socket-dir (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from rbd-csi-provisioner-token-p6ttt (ro)
  csi-resizer:
    Container ID:  docker://f9ceef9dadc3b045ce51c620258432ccfece008dfae60cee78da5663609e3348
    Image:         quay.io/k8scsi/csi-resizer:v0.4.0
    Image ID:      docker-pullable://quay.io/k8scsi/csi-resizer@sha256:6883482c4c4bd0eabb83315b5a9e8d9c0f34357980489a679a56441ec74c24c9
    Port:          <none>
    Host Port:     <none>
    Args:
      --csi-address=$(ADDRESS)
      --v=5
      --csiTimeout=150s
      --leader-election
    State:          Running
      Started:      Tue, 23 Jun 2020 11:04:51 +0800
    Ready:          True
    Restart Count:  0
    Environment:
      ADDRESS:  unix:///csi/csi-provisioner.sock
    Mounts:
      /csi from socket-dir (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from rbd-csi-provisioner-token-p6ttt (ro)
  csi-rbdplugin:
    Container ID:  docker://025f85e36a92bdea63e49bdcb2d5d34dd3e4c756b4ae6492304376b7b74703b7
    Image:         bigtera/bigtera-csi:v2.0-canary
    Image ID:      docker-pullable://bigtera/bigtera-csi@sha256:047d608ee7a84e97625f980f7ebe9b9a6f03e46928262b6810d26d84cbfa356e
    Port:          <none>
    Host Port:     <none>
    Args:
      --nodeid=$(NODE_ID)
      --type=rbd
      --controllerserver=true
      --endpoint=$(CSI_ENDPOINT)
      --v=5
      --drivername=csi.block.bigtera.com
      --pidlimit=-1
      --metricsport=8090
      --metricspath=/metrics
      --enablegrpcmetrics=false
    State:          Running
      Started:      Tue, 23 Jun 2020 11:04:51 +0800
    Ready:          True
    Restart Count:  0
    Environment:
      POD_IP:         (v1:status.podIP)
      NODE_ID:        (v1:spec.nodeName)
      CSI_ENDPOINT:  unix:///csi/csi-provisioner.sock
    Mounts:
      /csi from socket-dir (rw)
      /dev from host-dev (rw)
      /etc/ceph-csi-config/ from ceph-csi-config (rw)
      /lib/modules from lib-modules (ro)
      /sys from host-sys (rw)
      /tmp/csi/keys from keys-tmp-dir (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from rbd-csi-provisioner-token-p6ttt (ro)
  liveness-prometheus:
    Container ID:  docker://50ee31ffaaa06843cc199f40b24c467d06c0028b84808f08e451837fcc54b87e
    Image:         bigtera/bigtera-csi:v2.0-canary
    Image ID:      docker-pullable://bigtera/bigtera-csi@sha256:047d608ee7a84e97625f980f7ebe9b9a6f03e46928262b6810d26d84cbfa356e
    Port:          <none>
    Host Port:     <none>
    Args:
      --type=liveness
      --endpoint=$(CSI_ENDPOINT)
      --metricsport=8680
      --metricspath=/metrics
      --polltime=60s
      --timeout=3s
    State:          Running
      Started:      Tue, 23 Jun 2020 11:04:51 +0800
    Ready:          True
    Restart Count:  0
    Environment:
      CSI_ENDPOINT:  unix:///csi/csi-provisioner.sock
      POD_IP:         (v1:status.podIP)
    Mounts:
      /csi from socket-dir (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from rbd-csi-provisioner-token-p6ttt (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  host-dev:
    Type:          HostPath (bare host directory volume)
    Path:          /dev
    HostPathType:  
  host-sys:
    Type:          HostPath (bare host directory volume)
    Path:          /sys
    HostPathType:  
  lib-modules:
    Type:          HostPath (bare host directory volume)
    Path:          /lib/modules
    HostPathType:  
  socket-dir:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     Memory
    SizeLimit:  <unset>
  ceph-csi-config:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      ceph-csi-config
    Optional:  false
  keys-tmp-dir:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     Memory
    SizeLimit:  <unset>
  rbd-csi-provisioner-token-p6ttt:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  rbd-csi-provisioner-token-p6ttt
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:          <none>
wsg@k8smater1:~$ 

```

#### 进入 provisioner pod

可以看到 和 daemonset pod的显示不一样， daemonset pod 的权限和 host主机类似，并且hostname 也是主机的hostname。

```
[root@csi-rbdplugin-provisioner-968f8d7c6-2kdj5 /]# lsblk 
NAME                       MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
rbd0                       252:0    0     1G  0 disk 
sdd                          8:48   0    20G  0 disk 
sdb                          8:16   0     1T  0 disk 
sr0                         11:0    1  1024M  0 rom  
sdc                          8:32   0   800G  0 disk 
sda                          8:0    0   500G  0 disk 
|-sda2                       8:2    0     1K  0 part 
|-sda5                       8:5    0 499.5G  0 part 
| |-ubuntu16wsg--vg-swap_1 253:1    0     8G  0 lvm  
| `-ubuntu16wsg--vg-root   253:0    0 491.5G  0 lvm  /etc/hosts
`-sda1                       8:1    0   487M  0 part 
[root@csi-rbdplugin-provisioner-968f8d7c6-2kdj5 /]# hostname
csi-rbdplugin-provisioner-968f8d7c6-2kdj5


```

查看pod 的mount 信息

```
[root@csi-rbdplugin-provisioner-968f8d7c6-2kdj5 /]# mount |grep rbd
[root@csi-rbdplugin-provisioner-968f8d7c6-2kdj5 /]# 

```







### host 的mount空间

```

mount 

/dev/rbd0 on /var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-7a4097ef-5d59-4c78-bd5d-d3e8a50b0765/globalmount/0001-0024-bf27339d-a4fc-42db-a9af-3f3fed8e1f2c-000000000000000c-9333dd85-b506-11ea-953e-5235b1be5b28 type ext4 (rw,relatime,stripe=1024,data=ordered,_netdev)
/dev/rbd0 on /var/lib/kubelet/pods/527b1185-b9b1-47cb-ad35-6fc9352069a3/volumes/kubernetes.io~csi/pvc-7a4097ef-5d59-4c78-bd5d-d3e8a50b0765/mount type ext4 (rw,relatime,stripe=1024,data=ordered,_netdev)

```

第一条 就是 node component 内map 的rbd，在host 空间也可以看到。







## csi插件开发探索

采用go 开发，需要定义的 代码如下

```
wsg@k8smaster2:~/ceph-csi/pkg/rbd$ ll
total 136
drwxrwxr-x 2 wsg wsg  4096 Jun  8 16:18 ./
drwxrwxr-x 7 wsg wsg  4096 Jun  8 16:18 ../
-rw-rw-r-- 1 wsg wsg 29017 Jun  8 16:18 controllerserver.go
-rw-rw-r-- 1 wsg wsg  5306 Jun  8 16:18 driver.go
-rw-rw-r-- 1 wsg wsg  2021 Jun  8 16:18 errors.go
-rw-rw-r-- 1 wsg wsg  1584 Jun  8 16:18 identityserver.go
-rw-rw-r-- 1 wsg wsg 26409 Jun  8 16:18 nodeserver.go
-rw-rw-r-- 1 wsg wsg 10107 Jun  8 16:18 rbd_attach.go
-rw-rw-r-- 1 wsg wsg  8861 Jun  8 16:18 rbd_journal.go
-rw-rw-r-- 1 wsg wsg 27697 Jun  8 16:18 rbd_util.go
wsg@k8smaster2:~/ceph-csi/pkg/rbd$ pwd
/home/wsg/ceph-csi/pkg/rbd
wsg@k8smaster2:~/ceph-csi/pkg/rbd$ 

CSI Identity 服务的实现，就定义在了 driver 目录下的 identity.go 文件里，用来获取 插件信息
CSI Controller 服务:  创建和删除volume
CSI Node 服务对应的，是 Volume 管理流程里的“Mount 阶段”
```





## kubernet 挂载rbd 后，存储端 观察



```
root@node3:~# rados -p rbd listwatchers rbd_header.8c2a34dd154fe
watcher=172.17.73.66:0/53298027 client.582445 cookie=18446462598732840961
root@node3:~# 

pod确实是被调度到了 73.66 上。
```





# 引申知识点记录



## mount propgation

Mount propagation allows for sharing volumes mounted by a Container to other Containers in the same Pod, or even to other Pods on the same node.

该配置项有3个值

- None（默认值）

  host 挂载点 里面如果有 新的sub mount，container里面不会看到

- HostToContainer

  host 挂载点 里面如果有 新的sub mount，container里面可以看到

- Bidirectional

  host 和container 互相可以看到

**Caution:** `Bidirectional` mount propagation can be dangerous. It can damage the host operating system and therefore it is allowed only in privileged Containers. Familiarity with Linux kernel behavior is strongly recommended. In addition, any volume mounts created by Containers in Pods must be destroyed (unmounted) by the Containers on terminatio

csi的 node plugin的yaml配置文件中,定义了 

```

99             - name: plugin-dir
100               mountPath: /var/lib/kubelet/plugins
101               mountPropagation: "Bidirectional"
102             - name: mountpoint-dir
103               mountPath: /var/lib/kubelet/pods
104               mountPropagation: "Bidirectional"


```





## 关于host 的/var/lib/kubelet 挂载

下面的挂载点 最终向上传递，逻辑抽象成 k8s的pv资源，供上层调用。。

正常pod挂载的volume格式如下，这些挂载点正是 host上的可持久化目录

```
/var/lib/kubelet/pods/<Pod 的 ID>/volumes/kubernetes.io~<Volume 类型 >/<Volume 名字 >

```



### iscsi 存储

```
/dev/sdd on /var/lib/kubelet/plugins/kubernetes.io/iscsi/iface-default/172.17.73.37:3260-iqn.2020-02.com:wsg-lun-2 type ext4 (rw,relatime,stripe=512,data=ordered)

/dev/sdd on /var/lib/kubelet/pods/9449c7d9-2f6e-4c2c-a53a-18ab15e1176d/volumes/kubernetes.io~iscsi/prometheus-grafana-pv type ext4 (rw,relatime,stripe=512,data=ordered)

```

第一条记录 是 host 主机通过 kubernet 自带的iscsi 存储插件挂载。

### nfs 存储

```
172.17.73.37:/vol/pve-kubernet-nfs/kubernet-nfs on /var/lib/kubelet/pods/8443e40e-df73-4e92-b64c-cbf2351d2ff8/volumes/kubernetes.io~nfs/jenkins-pv type nfs (rw,relatime,vers=3,rsize=1048576,wsize=1048576,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=172.17.73.37,mountvers=3,mountport=40084,mountproto=udp,local_lock=none,addr=172.17.73.37)


```

因为nfs 没有如块设备的mapp过程，

## pod 和docker container 对应关系

pod 信息

```
wsg@k8smaster2:~/ceph-csi/deploy/rbd/kubernetes$ kubectl get pod -n ci
NAME                      READY   STATUS    RESTARTS   AGE
jenkins-dc995bb85-zdbbc   1/1     Running   9          22d

```



docker  ps 信息

```
wsg@k8smaster2:~/ceph-csi/deploy/rbd/kubernetes$ sudo docker ps  |grep jenkins
93d12ca4f13b        jenkins/jenkins                                     "/sbin/tini -- /usr/…"   11 days ago         Up 11 days                              k8s_jenkins_jenkins-dc995bb85-zdbbc_ci_8443e40e-df73-4e92-b64c-cbf2351d2ff8_9
a75eac3431f9        registry.aliyuncs.com/google_containers/pause:3.2   "/pause"                 11 days ago         Up 11 days                              k8s_POD_jenkins-dc995bb85-zdbbc_ci_8443e40e-df73-4e92-b64c-cbf2351d2ff8_10


```

对比 k8s 和docker 层面信息，可以通过 dc995bb85-zdbbc，确定 pod 对应的 docker container

再进一步 根据docker inspect 信息

```


[
    {
        "Id": "93d12ca4f13bd04cce672a954ba8c1783a89a0ffeb3eb2945d5b1b0fbcf8351d",
        "Created": "2020-06-19T01:54:40.568612247Z",
        "Path": "/sbin/tini",
        "Args": [
            "--",
            "/usr/local/bin/jenkins.sh"
        ],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 6363,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2020-06-19T01:54:42.284984642Z",
            "FinishedAt": "0001-01-01T00:00:00Z"
        },
        "Image": "sha256:e43ba47cfa8bc4dffba5e5b202c777c28bd7ce24d42edad63b5a5c382a488b07",
        "ResolvConfPath": "/var/lib/docker/containers/a75eac3431f932c52377b6cbeee4cfddae5ffc77ff09b6a4298d870b33f08c00/resolv.conf",
        "HostnamePath": "/var/lib/docker/containers/a75eac3431f932c52377b6cbeee4cfddae5ffc77ff09b6a4298d870b33f08c00/hostname",
        "HostsPath": "/var/lib/kubelet/pods/8443e40e-df73-4e92-b64c-cbf2351d2ff8/etc-hosts",
        "LogPath": "/var/lib/docker/containers/93d12ca4f13bd04cce672a954ba8c1783a89a0ffeb3eb2945d5b1b0fbcf8351d/93d12ca4f13bd04cce672a954ba8c1783a89a0ffeb3eb2945d5b1b0fbcf8351d-json.log",
        "Name": "/k8s_jenkins_jenkins-dc995bb85-zdbbc_ci_8443e40e-df73-4e92-b64c-cbf2351d2ff8_9",
        "RestartCount": 0,
        "Driver": "overlay2",
        "Platform": "linux",

        "Mounts": [
            {
                "Type": "bind",
                "Source": "/var/lib/kubelet/pods/8443e40e-df73-4e92-b64c-cbf2351d2ff8/containers/jenkins/95b4c872",
                "Destination": "/dev/termination-log",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            },
            {
                "Type": "bind",
                "Source": "/var/lib/kubelet/pods/8443e40e-df73-4e92-b64c-cbf2351d2ff8/volumes/kubernetes.io~nfs/jenkins-pv",
                "Destination": "/var/jenkins_home",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"

```



可以看到 mount 信息 /var/lib/kubelet/pods/8443e40e-df73-4e92-b64c-cbf2351d2ff8/volumes/kubernetes.io~nfs/jenkins-pv

# 已知问题



## csi删除后，pvc 和pv的残留信息 需要手动删除



# 错误处理记录







```
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason                  Age                 From                     Message
  ----     ------                  ----                ----                     -------
  Normal   Scheduled               <unknown>           default-scheduler        Successfully assigned default/csi-rbd-demo-pod to k8smater1
  Normal   SuccessfulAttachVolume  13m                 attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-9e68cc33-f08f-4a72-96fa-e37720c0a4c8"
  Warning  FailedMount             4m15s               kubelet, k8smater1       Unable to attach or mount volumes: unmounted volumes=[mypvc], unattached volumes=[default-token-brqvs mypvc]: timed out waiting for the condition
  Warning  FailedMount             119s (x4 over 11m)  kubelet, k8smater1       Unable to attach or mount volumes: unmounted volumes=[mypvc], unattached volumes=[mypvc default-token-brqvs]: timed out waiting for the condition
  Warning  FailedMount             74s (x8 over 11m)   kubelet, k8smater1       MountVolume.MountDevice failed for volume "pvc-9e68cc33-f08f-4a72-96fa-e37720c0a4c8" : rpc error: code = Internal desc = rbd: map failed exit status 110, rbd output: rbd: sysfs write failed
In some cases useful info is found in syslog - try "dmesg | tail".
rbd: map failed: (110) Connection timed out




```



## ubuntu16.04 升级到ubuntu18



升级过程中 会提示要 重启 docker daemon

```

 │ If Docker is upgraded without restarting the Docker daemon, Docker will often have trouble starting new containers, and in some cases even maintaining the containers it is currently running. See                                    │  
 │ https://launchpad.net/bugs/1658691 for an example of this breakage.                               
 │ Normally, upgrading the package would simply restart the associated daemon(s). In the case of the Docker daemon, that would also imply stopping all running containers (which will only be restarted if they're part of a "service",  │  
 │ have an appropriate restart policy configured, or have some other means of being restarted such as an external systemd unit).                                                                                                                                                                        │  
 │ Automatically restart Docker daemon?                                                                                                                                                                                                            <Yes>                                                                           <No>       
```



升级步骤

```

wsg@k8smaster2:~$ sudo apt-get update && sudo apt-get upgrade -y && sudo apt-get dist-upgrade -y

然后reboot

sudo apt autoremove
会释放/boot 分区里面老内核，注意下 /boot分区使用率， 太高 会因为空间不足，无法执行下面的命令

sudo do-release-upgrade 
会有很多需要 手动输入 y 确认的步骤


```





## k8s 所在os的kernel版本 要求



当前k8s 部署在ubuntu16.04 上，内核版本

```
wsg@k8smater1:~/kube_deploy/jenkins$ uname -a
Linux k8smater1 4.4.0-184-generic #214-Ubuntu SMP Thu Jun 4 10:14:11 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
wsg@k8smater1:~/kube_deploy/jenkins$ cat /etc/issue
Ubuntu 16.04 LTS \n \l

```



pod 使用rbd时，会看到k8s 内核报错如下（antony 测试，与台湾的ceph cluster 对接）。

```
[84233.612090] libceph: mon1 10.1.15.185:6789 feature set mismatch, my 106b84a842a42 < server's 40106b84a842a42, missing 400000000000000
[84233.614930] libceph: mon1 10.1.15.185:6789 missing required protocol features
[84244.223766] libceph: mon1 10.1.15.185:6789 feature set mismatch, my 106b84a842a42 < server's 40106b84a842a42, missing 400000000000000
[84244.224407] libceph: mon1 10.1.15.185:6789 missing required protocol features
[84254.275506] libceph: mon1 10.1.15.185:6789 feature set mismatch, my 106b84a842a42 < server's 40106b84a842a42, missing 400000000000000
[84254.276286] libceph: mon1 10.1.15.185:6789 missing required protocol features
[84264.279257] libceph: mon1 10.1.15.185:6789 feature set mismatch, my 106b84a842a42 < server's 40106b84a842a42, missing 400000000000000
[84264.280249] libceph: mon1 10.1.15.185:6789 missing required protocol features
[84274.187687] libceph: mon1 10.1.15.185:6789 feature set mismatch, my 106b84a842a42 < server's 40106b84a842a42, missing 400000000000000
[84274.188850] libceph: mon1 10.1.15.185:6789 missing required protocol features
[84284.219750] libceph: mon1 10.1.15.185:6789 feature set mismatch, my 106b84a842a42 < server's 40106b84a842a42, missing 400000000000000
[84284.221050] libceph: mon1 10.1.15.185:6789 missing required protocol features
[84362.751767] libceph: mon1 10.1.15.185:6789 feature set mismatch, my 106b84a842a42 < server's 40106b84a842a42, missing 400000000000000
[84362.753281] libceph: mon1 10.1.15.185:6789 missing required protocol features
[84373.230687] libceph: mon1 10.1.15.185:6789 feature set mismatch, my 106b84a842a42 < server's 40106b84a842a42, missing 400000000000000
[84373.232313] libceph: mon1 10.1.15.185:6789 missing required protocol features
[84383.069946] libceph: mon0 10.1.15.185:3300 socket closed (con state CONNECTING)
[84393.054285] libceph: mon0 10.1.15.185:3300 socket closed (con state CONNECTING)
[84403.207664] libceph: mon1 10.1.15.185:6789 feature set mismatch, my 106b84a842a42 < server's 40106b84a842a42, missing 400000000000000

```





## ceph版本问题

> 需要特定的版本，才能和kubernets 正常对接

升级后的版本

```
root@node3:~# ceph mgr module ls
{
    "enabled_modules": [
        "balancer",
        "status",
        "virtualstor"
    ],
    "disabled_modules": [
        "dashboard",
        "influx",
        "localpool",
        "prometheus",
        "restful",
        "selftest",
        "volumes",
        "zabbix"
    ]
}
root@node3:~# ezs3-version 
v8.0-335 (ce8f568d5)

```



下面的版本 还无法和CSI对接

```
root@node3:~# ceph mgr module enable volumes
Error ENOENT: all mgr daemons do not support module 'volumes', pass --force to force enablement
root@node3:~# 
root@node3:~# ceph mgr module ls
{
    "enabled_modules": [
        "balancer",
        "status",
        "virtualstor"
    ],
    "disabled_modules": [
        "dashboard",
        "influx",
        "localpool",
        "prometheus",
        "restful",
        "selftest",
        "zabbix"
    ]
}


root@node3:~# ceph --version
ceph version 12.2.10-635-g01533dbfc6 (01533dbfc6fdeecc5199c8790f297d3141f61c6c) luminous (stable)
root@node3:~# 

root@node3:~# ezs3-version 
v8.0-230 (edd263c73)
Bigtera VirtualStor Scaler 8.0-203 - Release amd64 (201911192355~829bb855e)

```



# 参考资料



https://kubernetes-csi.github.io/docs/deploying.html

csi有专门的site

https://kubernetes-csi.github.io/

