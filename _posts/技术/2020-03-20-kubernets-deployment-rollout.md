---
layout: post
title: kubernets deployment回滚
category: 技术
---




# 背景



jenkins web界面的时间显示与现实相差8个小时，检查了jenkins所在pod的时间，时区配置(/etc/localtime)也确实 指向了UTC时区，所以想到应该在启动容器时，将host主机的该配置文件挂载给容器。

但是第一次更新yaml后，发现更改引入了错误，所以尝试将deployment 回滚，再次apply后，可以看到pod中时区已经更新了正确的。但是发现jenkins的时区并没有变。





# deployment 回滚



rollout的帮助文档

```
wsg@ubuntu16:~/helm/jenkins/aliyun$ kubectl rollout
Manage the rollout of a resource.
  
 Valid resource types include:

  *  deployments
  *  daemonsets
  *  statefulsets

Examples:
  # Rollback to the previous deployment
  kubectl rollout undo deployment/abc
  
  # Check the rollout status of a daemonset
  kubectl rollout status daemonset/foo

Available Commands:
  history     View rollout history
  pause       Mark the provided resource as paused
  resume      Resume a paused resource
  status      Show the status of the rollout
  undo        Undo a previous rollout

Usage:
  kubectl rollout SUBCOMMAND [options]

```





查看deployment的历史版本

```
wsg@ubuntu16:~/helm/jenkins/aliyun$ kubectl rollout history Deployment/jenkins -n ci
deployment.extensions/jenkins 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
wsg@ubuntu16:~/helm/jenkins/aliyun$ 

```



回滚到指定版本

```
kubectl rollout undo deployment/app --to-revision=2


wsg@ubuntu16:~/helm/jenkins/aliyun$ kubectl rollout undo Deployment/jenkins --to-revision=2  -n ci
deployment.extensions/jenkins rolled back
wsg@ubuntu16:~/helm/jenkins/aliyun$ 


```









###操作失误后的 deployment 状态

```
wsg@ubuntu16:~/helm/jenkins/aliyun$ kubectl describe deployment -n ci
Name:                   jenkins
Namespace:              ci
CreationTimestamp:      Tue, 11 Feb 2020 12:58:24 +0800
Labels:                 k8s-app=jenkins
Annotations:            deployment.kubernetes.io/revision: 3
                        kubectl.kubernetes.io/last-applied-configuration:
                          {"apiVersion":"extensions/v1beta1","kind":"Deployment","metadata":{"annotations":{},"name":"jenkins","namespace":"ci"},"spec":{"replicas":...
Selector:               k8s-app=jenkins
Replicas:               1 desired | 1 updated | 1 total | 0 available | 1 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
Pod Template:
  Labels:           k8s-app=jenkins
  Service Account:  jenkins
  Containers:
   jenkins:
    Image:        jenkins/jenkins:2.204
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:
      /etc/localtime from timezone-config (rw)
      /var/jenkins_home from home (rw)
  Volumes:
   home:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  jenkins
    ReadOnly:   false
   timezone-config:
    Type:          HostPath (bare host directory volume)
    Path:          /usr/share/zoneinfo/Asia/Shanghai
    HostPathType:  
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  jenkins-66f4c94f (1/1 replicas created)
NewReplicaSet:   <none>
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  10m   deployment-controller  Scaled up replica set jenkins-66f4c94f to 1
  Normal  ScalingReplicaSet  10m   deployment-controller  Scaled down replica set jenkins-bd446b4f to 0
wsg@ubuntu16:~/helm/jenkins/aliyun$ 

```



检查replicaSet

```
wsg@ubuntu16:~/helm/jenkins/aliyun$ kubectl describe replicaset jenkins-66f4c94f -n ci
Name:           jenkins-66f4c94f
Namespace:      ci
Selector:       k8s-app=jenkins,pod-template-hash=66f4c94f
Labels:         k8s-app=jenkins
                pod-template-hash=66f4c94f
Annotations:    deployment.kubernetes.io/desired-replicas: 1
                deployment.kubernetes.io/max-replicas: 2
                deployment.kubernetes.io/revision: 3
Controlled By:  Deployment/jenkins
Replicas:       1 current / 1 desired
Pods Status:    0 Running / 1 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:           k8s-app=jenkins
                    pod-template-hash=66f4c94f
  Service Account:  jenkins
  Containers:
   jenkins:
    Image:        jenkins/jenkins:2.204
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:
      /etc/localtime from timezone-config (rw)
      /var/jenkins_home from home (rw)
  Volumes:
   home:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  jenkins
    ReadOnly:   false
   timezone-config:
    Type:          HostPath (bare host directory volume)
    Path:          /usr/share/zoneinfo/Asia/Shanghai
    HostPathType:  
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  47m   replicaset-controller  Created pod: jenkins-66f4c94f-94nf5
wsg@ubuntu16:~/helm/jenkins/aliyun$ 

```



### rollout后的 deployment

确实发现deployment中 pvc是不一样的（从jenkins变成了jenkins-new）。

```
wsg@ubuntu16:~/helm/jenkins/aliyun$ kubectl describe deployment -n ci
Name:                   jenkins
Namespace:              ci
CreationTimestamp:      Tue, 11 Feb 2020 12:58:24 +0800
Labels:                 k8s-app=jenkins
Annotations:            deployment.kubernetes.io/revision: 4
                        kubectl.kubernetes.io/last-applied-configuration:
                          {"apiVersion":"extensions/v1beta1","kind":"Deployment","metadata":{"annotations":{},"name":"jenkins","namespace":"ci"},"spec":{"replicas":...
Selector:               k8s-app=jenkins
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
Pod Template:
  Labels:           k8s-app=jenkins
  Service Account:  jenkins
  Containers:
   jenkins:
    Image:        jenkins/jenkins:2.204
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:
      /var/jenkins_home from home (rw)
  Volumes:
   home:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  jenkins-new
    ReadOnly:   false
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   jenkins-bd446b4f (1/1 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  3m35s  deployment-controller  Scaled up replica set jenkins-bd446b4f to 1
  Normal  ScalingReplicaSet  3m35s  deployment-controller  Scaled down replica set jenkins-66f4c94f to 0
wsg@ubuntu16:~/helm/jenkins/aliyun$ 

```





# 知识点记录

## ubuntu的时区配置文件的区别

/etc/localtime 文件一般是个链接，对应的目标文件也是非纯文本的。注意和 /etc/timezone区别

```
wsg@ubuntu16:~/helm/jenkins/aliyun$ ll /etc/localtime 
lrwxrwxrwx 1 root root 33 Feb 28  2019 /etc/localtime -> /usr/share/zoneinfo/Asia/Shanghai
wsg@ubuntu16:~/helm/jenkins/aliyun$ cat /usr/share/zoneinfo/Asia/Shanghai
TZif2 
     ^	ꓽˊ껀л>㊻ӂ­䃢Ռ¿弿ֆfpםᘁ| i ~ !I}"g¡ #)_$G %|&'e &쎨G (рq־LMTCDTCSTTZif2 
                                                                                    ဿ~6C)ÿÿÿÿǙ^ÿÿÿÿ	ဿÿʓ½ÿÿÿÿˊဿÿʼ@ÿÿÿÿл>ဿÿӋ{ÿÿÿÿӂ­ဿÿԅ"ÿÿÿÿՌ¿ဿÿռ¿ÿÿÿÿֆfpÿÿÿÿם‿ÿÿ؁| i ~ !I}"g¡ #)_$G %|&'e &쎨G (рq־LMTCDTCST
CST-8
wsg@ubuntu16:~/helm/jenkins/aliyun$ XshellXshell

```

但是timezone是个纯文本的配置文件， 

```
wsg@ubuntu16:~/helm/jenkins/aliyun$ cat /etc/timezone 
Asia/Shanghai

```



```
http://172.17.73.67:30080/systemInfo
```

