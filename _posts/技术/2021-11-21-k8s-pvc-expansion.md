---
layout: post
title: k8s-pvc/pv扩容记录
category: 技术
---


# 背景

一次聊天过程中，对方提及pvc的扩容，虽然有注意过 storageclass 有个AllowVolumeExpansion的配置（有些csi插件是不支持该配置的，比如local-volume-provisoner），但是没有实际用过，所以还是心里没底，所以抽时间 做了下扩容的操作。这里是基于 ceph-csi 插件，在rbd的使用方式下做的 扩容



# 原理



需要csi 插件的支持，更改pod的 pvc 申明的存储空间需求，csi插件便会自动对原来的rbd卷 做扩容操作

# 步骤



##检查当前storage class 支持的特性

wsg@k8snode1v18:~/k8s_deploy/ceph-csi/examples/rbd$ kubectl describe sc csi-rbd-sc

```
AllowVolumeExpansion:  True
MountOptions:
  discard
ReclaimPolicy:      Delete
VolumeBindingMode:  Immediate
Events:             <none>

```

> 可以看到，AllowVolumeExpansion 为true





## 创建test pod  并且 申明使用 

测试pod 定义如下：

```
wsg@k8snode1v18:~/k8s_deploy/ceph-csi/examples/rbd$ cat pod-test-pv-expansion.yaml 
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-test-pvc-expansion
spec:
  containers:
    - name: nginx-server
      image: nginx
      volumeMounts:
        - name: mypvc
          mountPath: /var/lib/www/html
  volumes:
    - name: mypvc
      persistentVolumeClaim:
        claimName: rbd-pvc
        readOnly: false

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







进入测试pod  检查 volume 挂载情况

wsg@k8snode1v18:~/k8s_deploy/ceph-csi/examples/rbd$ kubectl exec -it pod-test-pvc-expansion -- /bin/bash

```
/dev/rbd1 on /var/lib/www/html type ext4 (rw,relatime,stripe=16)

```

可以看到 声明的 1G rbd 磁盘已经 自动 mount 上了，并且采用 ext4 文件系统



写入测试数据

```
root@pod-test-pvc-expansion:/var/lib/www/html/wsg-inpod# md5sum before-expansion.txt >md5    
root@pod-test-pvc-expansion:/var/lib/www/html/wsg-inpod# md5sum before-expansion.txt 
0b4cfe43fc1f774cc0ed294515f626e1  before-expansion.txt
root@pod-test-pvc-expansion:/var/lib/www/html/wsg-inpod# cat md5 
0b4cfe43fc1f774cc0ed294515f626e1  before-expansion.txt

root@pod-test-pvc-expansion:/var/lib/www/html# md5sum test.txt |tee md5
798f8e998aa4c359f7ef66549866fb01  test.txt
root@pod-test-pvc-expansion:/var/lib/www/html# cat md5 
798f8e998aa4c359f7ef66549866fb01  test.txt


```





## 执行扩容（调整 pvc 声明的大小）



```
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: csi-rbd-sc
  volumeMode: Filesystem
  volumeName: pvc-abfad8c1-9d71-45fd-834c-60b8109c6b75


wsg@k8snode1v18:~$ kubectl edit pvc rbd-pvc
persistentvolumeclaim/rbd-pvc edited
wsg@k8snode1v18:~$ 



```



## 检查 pod 状态

```
##扩容中看到 pod 没有重启的记录（扩容前 kubectl exec的 会话 也是一直保持正常，没有中断）
pod-test-pvc-expansion                                         1/1     Running   0          25m

##进一步检查 pod 的event，发现已经有 成功扩容的记录
wsg@k8snode1v18:~$ kubectl describe  pod pod-test-pvc-expansion
Name:         pod-test-pvc-expansion
Namespace:    default
Priority:     0
Node:         k8snode1v18/172.17.73.53
Start Time:   Sat, 27 Nov 2021 14:05:46 +0800
Labels:       <none>
Annotations:  cni.projectcalico.org/podIP: 10.106.176.31/32
Status:       Running
IP:           10.106.176.31
IPs:
  IP:  10.106.176.31
Containers:
  nginx-server:
    Container ID:   docker://ce3e35282b5ff3dcd62848ab69efb751f592510e67aed9ad3183eaf817f54f21
    Image:          nginx
    Image ID:       docker-pullable://nginx@sha256:097c3a0913d7e3a5b01b6c685a60c03632fc7a2b50bc8e35bcaa3691d788226e
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sat, 27 Nov 2021 14:06:35 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/lib/www/html from mypvc (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-gc5x4 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  mypvc:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  rbd-pvc
    ReadOnly:   false
  default-token-gc5x4:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-gc5x4
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason                      Age        From                     Message
  ----     ------                      ----       ----                     -------
  Warning  FailedScheduling            <unknown>  default-scheduler        persistentvolumeclaim "rbd-pvc" not found
  Warning  FailedScheduling            <unknown>  default-scheduler        persistentvolumeclaim "rbd-pvc" not found
  Warning  FailedScheduling            <unknown>  default-scheduler        running "VolumeBinding" filter plugin for pod "pod-test-pvc-expansion": pod has unbound immediate PersistentVolumeClaims
  Normal   Scheduled                   <unknown>  default-scheduler        Successfully assigned default/pod-test-pvc-expansion to k8snode1v18
  Normal   SuccessfulAttachVolume      17m        attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-abfad8c1-9d71-45fd-834c-60b8109c6b75"
  Normal   Pulling                     17m        kubelet, k8snode1v18     Pulling image "nginx"
  Normal   Pulled                      16m        kubelet, k8snode1v18     Successfully pulled image "nginx"
  Normal   Created                     16m        kubelet, k8snode1v18     Created container nginx-server
  Normal   Started                     16m        kubelet, k8snode1v18     Started container nginx-server
  Normal   FileSystemResizeSuccessful  48s        kubelet, k8snode1v18     MountVolume.NodeExpandVolume succeeded for volume "pvc-abfad8c1-9d71-45fd-834c-60b8109c6b75"
wsg@k8snode1v18:~$ 

```



## 扩容前后 pvc 的对比

````
rbd-pvc                                                                                                  Bound    pvc-abfad8c1-9d71-45fd-834c-60b8109c6b75   1Gi        RWO            csi-rbd-sc          13m

//扩容后
rbd-pvc                                                                                                  Bound    pvc-abfad8c1-9d71-45fd-834c-60b8109c6b75   10Gi       RWO            csi-rbd-sc          19m
wsg@k8snode1v18:~$


````

可以看到 容量的变化，并且 绑定的 pv编号 也是没有变化的，说明是同一个pv



## 检查 pod 中的数据





```
root@pod-test-pvc-expansion:/var/lib/www/html/wsg-inpod# md5sum before-expansion.txt 
0b4cfe43fc1f774cc0ed294515f626e1  before-expansion.txt
root@pod-test-pvc-expansion:/var/lib/www/html/wsg-inpod# cat md5 
0b4cfe43fc1f774cc0ed294515f626e1  before-expansion.txt
root@pod-test-pvc-expansion:/var/lib/www/html/wsg-inpod# 

```







# 知识点 

## pod 可以看到 host mount namespce



```
root@pod-test-pvc-expansion:/# lsblk 
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0    7:0    0   2.5M  1 loop 
loop1    7:1    0  99.4M  1 loop 
loop2    7:2    0  61.8M  1 loop 
loop3    7:3    0   2.5M  1 loop 
loop4    7:4    0  61.8M  1 loop 
loop5    7:5    0 242.3M  1 loop 
loop6    7:6    0   548K  1 loop 
loop7    7:7    0     4K  1 loop 
loop8    7:8    0   704K  1 loop 
loop10   7:10   0   704K  1 loop 
loop11   7:11   0  65.1M  1 loop 
loop12   7:12   0 140.7M  1 loop 
loop13   7:13   0   548K  1 loop 
loop14   7:14   0   2.5M  1 loop 
loop15   7:15   0 247.9M  1 loop 
loop16   7:16   0   219M  1 loop 
loop17   7:17   0   2.5M  1 loop 
loop18   7:18   0  65.2M  1 loop 
loop19   7:19   0   219M  1 loop 
loop20   7:20   0  55.5M  1 loop 
loop21   7:21   0 140.7M  1 loop 
loop22   7:22   0  99.4M  1 loop 
loop23   7:23   0  55.5M  1 loop 
sda      8:0    0   500G  0 disk 
`-sda1   8:1    0   500G  0 part /etc/hosts
rbd0   252:0    0    60G  0 disk 
rbd1   252:16   0     1G  0 disk /var/lib/www/html
root@pod-test-pvc-expansion:/# 


```

