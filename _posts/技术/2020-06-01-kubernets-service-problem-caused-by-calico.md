





# 背景



nodePort service 在k8s 集群重启后，发现有些 node上 无法访问服务（30080为服务端口，服务类型为 nodePort）。

```
73.65可以访问30080
73.66 无法访问


wsg@k8smater1:~$ netstat -atpn |grep 30080
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:30080           0.0.0.0:*               LISTEN      -               
wsg@k8smater1:~$ ip add|grep 73
    inet 172.17.73.66/22 brd 172.17.75.255 scope global ens18
    inet 172.17.73.63/22 scope global secondary ens18
wsg@k8smater1:~$ 



###
wsg@k8smaster2:~$ netstat -atpn |grep 30080
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:30080           0.0.0.0:*               LISTEN      -               
wsg@k8smaster2:~$ ip add |grep 73
    inet 172.17.73.65/22 brd 172.17.75.255 scope global ens18
wsg@k8smaster2:~$ 

```

看起来host上的端口都已经正常创建了（所以 kube-proxy应该正常）。所以怀疑点就落到了网络插件calico上。排查过程整理如下

# 排查过程





```
wsg@k8smater1:~$ kubectl get pod -n ci -o wide
NAME                      READY   STATUS    RESTARTS   AGE   IP               NODE         NOMINATED NODE   READINESS GATES
jenkins-dc995bb85-7w6t8   1/1     Running   1          28d   10.109.209.196   k8smaster2   <none>           <none>
wsg@k8smater1:~$ kubectl get svc -n ci -o wide
NAME          TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                        AGE   SELECTOR
jenkins-svc   NodePort   10.99.214.53   <none>        80:30080/TCP,50000:30945/TCP   28d   k8s-app=jenkins
wsg@k8smater1:~$ kubectl get svc -n ci -o wide

```

服务的 endpoint

```
wsg@k8smater1:~$ kubectl get ep jenkins-svc  -n ci -o wide
NAME          ENDPOINTS                                  AGE
jenkins-svc   10.109.209.196:8080,10.109.209.196:50000   28d
wsg@k8smater1:~$ 


```



## 问题点,node 状态notready

node状态没有ready

```
wsg@k8smaster2:~$ kubectl get nodes
NAME          STATUS     ROLES    AGE   VERSION
k8scephnode   NotReady   <none>   17d   v1.18.2
k8smaster2    Ready      master   33d   v1.18.2
k8smater1     Ready      master   33d   v1.18.2

```



有node 没有启动，但是 显示 pod (kube-proxy-4n5z8  )确正在运行在该pod上

```
wsg@k8smater1:~$ kubectl get pod  -o wide --all-namespaces
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
ci            jenkins-dc995bb85-7w6t8                    1/1     Running   1          28d   10.109.209.196   k8smaster2    <none>           <none>
ci2           jenkins-768d45bcb9-bhcmq                   0/1     Pending   0          10d   <none>           <none>        <none>           <none>
kube-system   calico-kube-controllers-5fc5dbfc47-4wkpw   1/1     Running   3          29d   10.109.169.8     k8smater1     <none>           <none>
kube-system   calico-node-96bpx                          0/1     Running   1          29d   172.17.73.65     k8smaster2    <none>           <none>
kube-system   calico-node-z85db                          0/1     Running   0          13d   172.17.73.60     k8scephnode   <none>           <none>
kube-system   calico-node-zhjt9                          0/1     Running   3          29d   172.17.73.66     k8smater1     <none>           <none>
kube-system   coredns-7ff77c879f-brsgl                   1/1     Running   3          29d   10.109.169.7     k8smater1     <none>           <none>
kube-system   coredns-7ff77c879f-kl8wl                   1/1     Running   1          29d   10.109.209.195   k8smaster2    <none>           <none>
kube-system   etcd-k8smaster2                            1/1     Running   12         29d   172.17.73.65     k8smaster2    <none>           <none>
kube-system   etcd-k8smater1                             1/1     Running   6          29d   172.17.73.66     k8smater1     <none>           <none>
kube-system   kube-apiserver-k8smaster2                  1/1     Running   16         29d   172.17.73.65     k8smaster2    <none>           <none>
kube-system   kube-apiserver-k8smater1                   1/1     Running   6          29d   172.17.73.66     k8smater1     <none>           <none>
kube-system   kube-controller-manager-k8smaster2         1/1     Running   27         29d   172.17.73.65     k8smaster2    <none>           <none>
kube-system   kube-controller-manager-k8smater1          1/1     Running   28         29d   172.17.73.66     k8smater1     <none>           <none>
kube-system   kube-proxy-4n5z8                           1/1     Running   1          13d   172.17.73.60     k8scephnode   <none>           <none>
kube-system   kube-proxy-74mvh                           1/1     Running   3          29d   172.17.73.65     k8smaster2    <none>           <none>
kube-system   kube-proxy-gxgc9                           1/1     Running   7          29d   172.17.73.66     k8smater1     <none>           <none>
kube-system   kube-scheduler-k8smaster2                  1/1     Running   24         29d   172.17.73.65     k8smaster2    <none>           <none>
kube-system   kube-scheduler-k8smater1                   1/1     Running   25         29d   172.17.73.66     k8smater1     <none>           <none>
wsg@k8smater1:~$ ping 172.17.73.60

```



进一步检查node状态

```
wsg@k8smaster2:~/helm/chart/prometheus-operator$ kubectl describe node k8scephnode
Name:               k8scephnode
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=k8scephnode
                    kubernetes.io/os=linux
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                    node.alpha.kubernetes.io/ttl: 0
                    projectcalico.org/IPv4Address: 10.0.0.63/24
                    projectcalico.org/IPv4IPIPTunnelAddr: 10.101.55.64
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Sat, 09 May 2020 18:35:16 +0800
Taints:             node.kubernetes.io/unreachable:NoExecute
                    node.kubernetes.io/unreachable:NoSchedule
Unschedulable:      false
Lease:
  HolderIdentity:  k8scephnode
  AcquireTime:     <unset>
  RenewTime:       Wed, 20 May 2020 18:39:34 +0800
Conditions:
  Type                 Status    LastHeartbeatTime                 LastTransitionTime                Reason              Message
  ----                 ------    -----------------                 ------------------                ------              -------
  NetworkUnavailable   False     Sat, 09 May 2020 18:36:29 +0800   Sat, 09 May 2020 18:36:29 +0800   CalicoIsUp          Calico is running on this node
  MemoryPressure       Unknown   Wed, 20 May 2020 18:37:28 +0800   Thu, 21 May 2020 09:46:38 +0800   NodeStatusUnknown   Kubelet stopped posting node status.
  DiskPressure         Unknown   Wed, 20 May 2020 18:37:28 +0800   Thu, 21 May 2020 09:46:38 +0800   NodeStatusUnknown   Kubelet stopped posting node status.
  PIDPressure          Unknown   Wed, 20 May 2020 18:37:28 +0800   Thu, 21 May 2020 09:46:38 +0800   NodeStatusUnknown   Kubelet stopped posting node status.
  Ready                Unknown   Wed, 20 May 2020 18:37:28 +0800   Thu, 21 May 2020 09:46:38 +0800   NodeStatusUnknown   Kubelet stopped posting node status.
Addresses:
  InternalIP:  172.17.73.60
  Hostname:    k8scephnode
Capacity:
  cpu:                4
  ephemeral-storage:  98299524Ki
  hugepages-2Mi:      0
  memory:             15027980Ki
  pods:               110
Allocatable:
  cpu:                4
  ephemeral-storage:  90592841169
  hugepages-2Mi:      0
  memory:             14925580Ki
  pods:               110
System Info:
  Machine ID:                 6a2886db0584453694c49ecd6da0c948
  System UUID:                A3285E0C-5CEC-4680-90DE-BC1719BA4FDB
  Boot ID:                    8ac4656b-ce20-40b3-b796-336a3fc8c8e9
  Kernel Version:             4.14.148-server
  OS Image:                   Ubuntu 16.04.6 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  docker://18.9.7
  Kubelet Version:            v1.18.2
  Kube-Proxy Version:         v1.18.2
Non-terminated Pods:          (3 in total)
  Namespace                   Name                                                          CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                   ----                                                          ------------  ----------  ---------------  -------------  ---
  default                     prometheus-operator-release-prometheus-node-exporter-9dk8c    0 (0%)        0 (0%)      0 (0%)           0 (0%)         15h
  kube-system                 calico-node-z85db                                             250m (6%)     0 (0%)      0 (0%)           0 (0%)         17d
  kube-system                 kube-proxy-4n5z8                                              0 (0%)        0 (0%)      0 (0%)           0 (0%)         17d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests   Limits
  --------           --------   ------
  cpu                250m (6%)  0 (0%)
  memory             0 (0%)     0 (0%)
  ephemeral-storage  0 (0%)     0 (0%)
  hugepages-2Mi      0 (0%)     0 (0%)
Events:              <none>
wsg@k8smaster2:~/helm/chart/prometheus-operator$ 

```



进一步检查node上的kubelet 进程

```
root@k8sCephNode:~# systemctl status kubelet.service 
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: activating (auto-restart) (Result: exit-code) since Wed 2020-05-27 10:37:34 CST; 3s ago
     Docs: https://kubernetes.io/docs/home/
  Process: 148906 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS (code=exited, status=255)
 Main PID: 148906 (code=exited, status=255)

May 27 10:37:34 k8sCephNode systemd[1]: kubelet.service: Unit entered failed state.
May 27 10:37:34 k8sCephNode systemd[1]: kubelet.service: Failed with result 'exit-code'.




正常节点的kubelet进程状态如下：
wsg@k8smaster2:~/helm/chart/prometheus-operator$ systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Thu 2020-05-21 09:45:25 CST; 6 days ago
     Docs: https://kubernetes.io/docs/home/
 Main PID: 8217 (kubelet)
    Tasks: 22
   Memory: 145.6M
      CPU: 7h 45min 58.361s
   CGroup: /system.slice/kubelet.service
           └─8217 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --cgroup-driver=cgroupfs --network-plugin=cni --pod-infra-c

May 27 10:07:28 k8smaster2 kubelet[8217]: W0527 10:07:28.875839    8217 volume_linux.go:49] Setting volume ownership for /var/lib/kubelet/pods/be551d56-f58b-45ba-bc03-6c9a0f6fa795/volumes/kubernetes.io~secret/config and fsGroup set. If 
May 27 10:07:28 k8smaster2 kubelet[8217]: W0527 10:07:28.876636    8217 volume_linux.go:49] Setting volume ownership for /var/lib/kubelet/pods/be551d56-f58b-45ba-bc03-
```



进一步检查kubelet 启动日志,通过 journalctl，

```
May 26 05:32:51 k8sCephNode systemd[1]: kubelet.service: Main process exited, code=exited, status=255/n/a
May 26 05:32:51 k8sCephNode systemd[1]: kubelet.service: Unit entered failed state.
May 26 05:32:51 k8sCephNode systemd[1]: kubelet.service: Failed with result 'exit-code'.
May 26 05:33:01 k8sCephNode systemd[1]: kubelet.service: Service hold-off time over, scheduling restart.
May 26 05:33:01 k8sCephNode systemd[1]: Stopped kubelet: The Kubernetes Node Agent.
May 26 05:33:01 k8sCephNode systemd[1]: Started kubelet: The Kubernetes Node Agent.
May 26 05:33:01 k8sCephNode kubelet[886002]: Flag --cgroup-driver has been deprecated, This parameter should be set via the config file specified by the Kubelet's --config flag. See https://kubernetes.io/docs/tasks/administer-cluster/ku
May 26 05:33:01 k8sCephNode kubelet[886002]: Flag --cgroup-driver has been deprecated, This parameter should be set via the config file specified by the Kubelet's --config flag. See https://kubernetes.io/docs/tasks/administer-cluster/ku
May 26 05:33:01 k8sCephNode kubelet[886002]: I0526 05:33:01.905628  886002 server.go:417] Version: v1.18.2
May 26 05:33:01 k8sCephNode kubelet[886002]: I0526 05:33:01.906023  886002 plugins.go:100] No cloud provider specified.
May 26 05:33:01 k8sCephNode kubelet[886002]: I0526 05:33:01.906055  886002 server.go:837] Client rotation is on, will bootstrap in background
May 26 05:33:01 k8sCephNode kubelet[886002]: I0526 05:33:01.923715  886002 certificate_store.go:130] Loading cert/key pair from "/var/lib/kubelet/pki/kubelet-client-current.pem".
May 26 05:33:01 k8sCephNode kubelet[886002]: I0526 05:33:01.976570  886002 server.go:646] --cgroups-per-qos enabled, but --cgroup-root was not specified.  defaulting to /
May 26 05:33:01 k8sCephNode kubelet[886002]: F0526 05:33:01.977085  886002 server.go:274] failed to run Kubelet: running with swap on is not supported, please disable swap! or set --fail-swap-on flag to false. /proc/swaps contained: [Fi
May 26 05:33:01 k8sCephNode systemd[1]: kubelet.service: Main process exited, code=exited, status=255/n/a
May 26 05:33:01 k8sCephNode systemd[1]: kubelet.service: Unit entered failed state.
May 26 05:33:01 k8sCephNode systemd[1]: kubelet.service: Failed with result 'exit-code'.


```



其实 syslog中已经体现了 journalctl中的日志,下面是syslog的日志

```

May 27 00:17:27 k8sCephNode kubelet[38152]: I0527 00:17:27.394073   38152 server.go:417] Version: v1.18.2
May 27 00:17:27 k8sCephNode kubelet[38152]: I0527 00:17:27.394422   38152 plugins.go:100] No cloud provider specified.
May 27 00:17:27 k8sCephNode kubelet[38152]: I0527 00:17:27.394455   38152 server.go:837] Client rotation is on, will bootstrap in background
May 27 00:17:27 k8sCephNode kubelet[38152]: I0527 00:17:27.411586   38152 certificate_store.go:130] Loading cert/key pair from "/var/lib/kubelet/pki/kubelet-client-current.pem".
May 27 00:17:27 k8sCephNode dockerd[1608]: time="2020-05-27T00:17:27.425337313+08:00" level=warning msg="failed to retrieve runc version: unknown output format: runc version spec: 1.0.1-dev\n"
May 27 00:17:27 k8sCephNode dockerd[1608]: time="2020-05-27T00:17:27.438915840+08:00" level=warning msg="failed to retrieve runc version: unknown output format: runc version spec: 1.0.1-dev\n"
May 27 00:17:27 k8sCephNode kubelet[38152]: I0527 00:17:27.448683   38152 server.go:646] --cgroups-per-qos enabled, but --cgroup-root was not specified.  defaulting to /
May 27 00:17:27 k8sCephNode kubelet[38152]: F0527 00:17:27.449453   38152 server.go:274] failed to run Kubelet: running with swap on is not supported, please disable swap! or set --fail-swap-on flag to false. /proc/swaps contained: [Filename                               Type            Size    Used    Priority /dev/sda3                               partition      27992060        0       -2]
May 27 00:17:27 k8sCephNode systemd[1]: kubelet.service: Main process exited, code=exited, status=255/n/a
May 27 00:17:27 k8sCephNode systemd[1]: kubelet.service: Unit entered failed state.
May 27 00:17:27 k8sCephNode systemd[1]: kubelet.service: Failed with result 'exit-code'.
May 27 00:17:37 k8sCephNode systemd[1]: kubelet.service: Service hold-off time over, scheduling restart.
May 27 00:17:37 k8sCephNode systemd[1]: Stopped kubelet: The Kubernetes Node Agent.
May 27 00:17:37 k8sCephNode systemd[1]: Started kubelet: The Kubernetes Node Agent.
May 27 00:17:37 k8sCephNode kubelet[38182]: Flag --cgroup-driver has been deprecated, This parameter should be set via the config file specified by the Kubelet's --config flag. See https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/ for more information.
May 27 00:17:37 k8sCephNode kubelet[38182]: Flag --cgroup-driver has been deprecated, This parameter should be set via the config file specified by the Kubelet's --config flag. See https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/ for more information.
May 27 00:17:37 k8sCephNode systemd[1]: Started Kubernetes systemd probe.

```



看起来和swap有关系，虽然在fstab中将swap分区 注释了，但是看起来 reboot时，还会启用swap分区，手动关闭 swap后，kubelet就正常启动了

```
wsg@k8smaster2:~/helm/chart$ kubectl get nodes -o wide
NAME          STATUS   ROLES    AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8scephnode   Ready    <none>   17d   v1.18.2   172.17.73.60   <none>        Ubuntu 16.04.6 LTS   4.14.148-server     docker://18.9.7
k8smaster2    Ready    master   33d   v1.18.2   172.17.73.65   <none>        Ubuntu 16.04 LTS     4.4.0-178-generic   docker://18.9.7
k8smater1     Ready    master   33d   v1.18.2   172.17.73.66   <none>        Ubuntu 16.04 LTS     4.4.0-178-generic   docker://18.9.7
wsg@k8smaster2:~/helm/chart$ 

```





## ipvs检查

https://segmentfault.com/a/1190000002609473









```
wsg@k8smaster2:~$ sudo ipvsadm -L
[sudo] password for wsg: 
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  localhost:30080 rr

  -> 10.109.209.196:http-alt      Masq    1      0          0         
TCP  localhost:30082 rr


TCP  172.17.73.65:30080 rr
^C
wsg@k8smaster2:~$ 


```



无法访问node节点

```

```



## 删掉问题node后，发现caclico-pod 异常

```
wsg@k8smater1:~$ kubectl get pod --all-namespaces -o wide |grep caclio
wsg@k8smater1:~$ kubectl get pod --all-namespaces -o wide
NAMESPACE     NAME                                                              READY   STATUS    RESTARTS   AGE   IP               NODE         NOMINATED NODE   READINESS GATES
ci            jenkins-dc995bb85-7w6t8                                           1/1     Running   1          32d   10.109.209.196   k8smaster2   <none>           <none>
ci2           jenkins-768d45bcb9-bhcmq                                          0/1     Pending   0          14d   <none>           <none>       <none>           <none>
default       alertmanager-prometheus-operator-releas-alertmanager-0            2/2     Running   0          17h   10.109.169.12    k8smater1    <none>           <none>
default       prometheus-operator-releas-operator-6bc9b8c695-fcrqz              2/2     Running   0          17h   10.109.169.11    k8smater1    <none>           <none>
default       prometheus-operator-release-grafana-98b9f4ccf-vvqv6               2/2     Running   0          17h   10.109.209.199   k8smaster2   <none>           <none>
default       prometheus-operator-release-kube-state-metrics-5d9989d9b7-z7x4t   1/1     Running   0          17h   10.109.169.10    k8smater1    <none>           <none>
default       prometheus-operator-release-prometheus-node-exporter-hd8lw        1/1     Running   0          17h   172.17.73.65     k8smaster2   <none>           <none>
default       prometheus-operator-release-prometheus-node-exporter-nvwm9        1/1     Running   0          17h   172.17.73.66     k8smater1    <none>           <none>
default       prometheus-prometheus-operator-releas-prometheus-0                3/3     Running   1          17h   10.109.209.201   k8smaster2   <none>           <none>
es            p8s-elastic-exporter-58756f7f9b-5tfts                             1/1     Running   0          42h   10.109.209.198   k8smaster2   <none>           <none>
kube-system   calico-kube-controllers-5fc5dbfc47-4wkpw                          1/1     Running   3          33d   10.109.169.8     k8smater1    <none>           <none>
kube-system   calico-node-96bpx                                                 0/1     Running   1          33d   172.17.73.65     k8smaster2   <none>           <none>
kube-system   calico-node-zhjt9                                                 0/1     Running   3          33d   172.17.73.66     k8smater1    <none>           <none>

```





## 检查calico-pod问题



```
    Type:        Secret (a volume populated by a Secret)
    SecretName:  calico-node-token-5cbcz
    Optional:    false
QoS Class:       Burstable
Node-Selectors:  beta.kubernetes.io/os=linux
Tolerations:     :NoSchedule
                 :NoExecute
                 CriticalAddonsOnly
                 node.kubernetes.io/disk-pressure:NoSchedule
                 node.kubernetes.io/memory-pressure:NoSchedule
                 node.kubernetes.io/network-unavailable:NoSchedule
                 node.kubernetes.io/not-ready:NoExecute
                 node.kubernetes.io/pid-pressure:NoSchedule
                 node.kubernetes.io/unreachable:NoExecute
                 node.kubernetes.io/unschedulable:NoSchedule
Events:
  Type     Reason     Age                      From                 Message
  ----     ------     ----                     ----                 -------
  Warning  Unhealthy  2m4s (x52551 over 6d2h)  kubelet, k8smaster2  (combined from similar events): Readiness probe failed: calico/node is not ready: BIRD is not ready: BGP not established with 10.0.0.662020-05-27 03:45:40.461 [INFO][17547] health.go 156: Number of node(s) with BGP peering established = 0
wsg@k8smater1:~$ 

```



## 检查每个node上 calico pod 



node 73.65 （正常节点）

````

root@k8smaster2:/# calico-node -h^C
root@k8smaster2:/# ps aux 
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   2308    60 ?        Ss   May21   0:07 /usr/bin/runsvdir -P /etc/service/enabled
root        54  0.0  0.0   2156    64 ?        Ss   May21   0:00 runsv bird6
root        55  0.0  0.0   2156    60 ?        Ss   May21   0:00 runsv bird
root        56  0.0  0.0   2156    68 ?        Ss   May21   0:00 runsv confd
root        57  0.0  0.0   2156    64 ?        Ss   May21   0:00 runsv felix
root        60  0.0  0.4 147792 32712 ?        Sl   May21   1:00 calico-node -confd
root        61  2.0  0.5 147792 41544 ?        Sl   May21 181:43 calico-node -felix
root       137  0.0  0.0    800     4 ?        S    May21   1:59 bird -R -s /var/run/calico/bird.ctl -d -c /etc/calico/confd/config/bird.cfg
root       138  0.0  0.0    744     4 ?        S    May21   1:19 bird6 -R -s /var/run/calico/bird6.ctl -d -c /etc/calico/confd/config/bird6.cfg
root     20071  0.0  0.0   4192  3392 pts/0    Ss   03:59   0:00 /bin/bash
root     20219  0.0  0.0   7832  2688 pts/0    R+   04:00   0:00 ps aux
root@k8smaster2:/# ^C
root@k8smaster2:/# ^C
root@k8smaster2:/# ping 10.109.209.196
PING 10.109.209.196 (10.109.209.196) 56(84) bytes of data.
64 bytes from 10.109.209.196: icmp_seq=1 ttl=64 time=0.158 ms
64 bytes from 10.109.209.196: icmp_seq=2 ttl=64 time=0.069 ms
64 bytes from 10.109.209.196: icmp_seq=3 ttl=64 time=0.071 ms
64 bytes from 10.109.209.196: icmp_seq=4 ttl=64 time=0.065 ms




````

node 73.66(问题节点)，发现和后端服务pod的 ip不通。

```
wsg@k8smater1:~$ kubectl -n kube-system -it calico-node-zhjt9 /bin/bash 
Error: unknown command "calico-node-zhjt9" for "kubectl"
Run 'kubectl --help' for usage.
wsg@k8smater1:~$ kubectl -n kube-system exec -it calico-node-zhjt9 /bin/bash 
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.
root@k8smater1:/# 
root@k8smater1:/# ping 10.109.209.196
PING 10.109.209.196 (10.109.209.196) 56(84) bytes of data.



```





## calico 版本

```
    Image:          calico/node:v3.9.5

```





## calico controller pod 也有报错

虽然 get pod的状态 显示 没有异常

```
wsg@k8smaster2:~/helm/chart/prometheus-operator$ kubectl -n kube-system describe pod calico-kube-controllers-5fc5dbfc47-4wkpw 
Name:                 calico-kube-controllers-5fc5dbfc47-4wkpw
Namespace:            kube-system
Priority:             2000000000
Priority Class Name:  system-cluster-critical
Node:                 k8smater1/172.17.73.66
Start Time:           Thu, 23 Apr 2020 15:42:47 +0800
Labels:               k8s-app=calico-kube-controllers
                      pod-template-hash=5fc5dbfc47
Annotations:          cni.projectcalico.org/podIP: 10.109.169.8/32
                      scheduler.alpha.kubernetes.io/critical-pod: 
Status:               Running
IP:                   10.109.169.8
IPs:
  IP:           10.109.169.8
Controlled By:  ReplicaSet/calico-kube-controllers-5fc5dbfc47
Containers:
  calico-kube-controllers:
    Container ID:   docker://71a86218c0119198ee2966d56117e54668c6b64c40f3a9cdad57b41efb35cc13
    Image:          calico/kube-controllers:v3.9.5
    Image ID:       docker-pullable://calico/kube-controllers@sha256:0fa9d599e2147a5ee6d951c7f18f59144a6459c2346864a4b31c6ab38668a393
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Thu, 21 May 2020 09:47:00 +0800
    Last State:     Terminated
      Reason:       Error
      Exit Code:    255
      Started:      Fri, 24 Apr 2020 15:58:31 +0800
      Finished:     Thu, 21 May 2020 09:40:42 +0800
    Ready:          True
    Restart Count:  3
    Readiness:      exec [/usr/bin/check-status -r] delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment:
      ENABLED_CONTROLLERS:  node
      DATASTORE_TYPE:       kubernetes
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from calico-kube-controllers-token-578zn (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  calico-kube-controllers-token-578zn:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  calico-kube-controllers-token-578zn
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  beta.kubernetes.io/os=linux
Tolerations:     CriticalAddonsOnly
                 node-role.kubernetes.io/master:NoSchedule
                 node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason     Age                 From                Message
  ----     ------     ----                ----                -------
  Warning  Unhealthy  59m (x7 over 2d2h)  kubelet, k8smater1  Readiness probe failed: Error verifying datastore: etcdserver: request timed out
wsg@k8smaster2:~/helm/chart/prometheus-operator$ 

```



检查pod 日志如下：

```
kubectl -n kube-system logs calico-kube-controllers-5fc5dbfc47-4wkpw 


t.go 255: Error getting cluster information config ClusterInformation="default" error=Get https://10.96.0.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default: context deadline exceeded
2020-05-25 10:33:47.237 [ERROR][1] main.go 202: Failed to verify datastore error=Get https://10.96.0.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default: context deadline exceeded
2020-05-25 10:33:50.292 [ERROR][1] main.go 233: Failed to reach apiserver error=Get https://10.96.0.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default: context deadline exceeded
2020-05-26 12:42:35.947 [ERROR][1] client.go 255: Error getting cluster information config ClusterInformation="default" error=etcdserver: leader changed
2020-05-26 12:42:35.947 [ERROR][1] main.go 202: Failed to verify datastore error=etcdserver: leader changed
2020-05-26 12:42:37.950 [ERROR][1] main.go 233: Failed to reach apiserver error=etcdserver: leader changed
2020-05-26 12:53:54.237 [ERROR][1] client.go 255: Error getting cluster information config ClusterInformation="default" error=Get https://10.96.0.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default: context deadline exceeded
2020-05-26 12:53:54.237 [ERROR][1] main.go 202: Failed to verify datastore error=Get https://10.96.0.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default: context deadline exceeded
2020-05-26 12:53:56.242 [ERROR][1] main.go 233: Failed to reach apiserver error=Get https://10.96.0.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default: context deadline exceeded
2020-05-26 12:56:14.306 [ERROR][1] client.go 255: Error getting cluster information config ClusterInformation="default" error=etcdserver: request timed out
2020-05-26 12:56:14.306 [ERROR][1] main.go 202: Failed to verify datastore error=etcdserver: request timed out
2020-05-27 03:17:18.790 [INFO][1] kdd.go 154: Cleaning up IPAM resources for deleted node node="k8scephnode"
2020-05-27 03:17:18.790 [INFO][1] ipam.go 1166: Releasing all IPs with handle 'ipip-tunnel-addr-k8scephnode'
2020-05-27 03:17:18.867 [INFO][1] ipam.go 1469: Node doesn't exist, no need to release affinity cidr=10.101.55.64/26 host="k8scephnode"
2020-05-27 03:17:18.978 [INFO][1] kdd.go 167: Node and IPAM data is in sync
2020-05-27 08:19:48.249 [ERROR][1] client.go 255: Error getting cluster information config ClusterInformation="default" error=etcdserver: request timed out
2020-05-27 08:19:48.249 [ERROR][1] main.go 202: Failed to verify datastore error=etcdserver: request timed out
wsg@k8smaster2:~/helm/chart/prometheus-operator$ kubectl -n kube-system logs calico-kube-controllers-5fc5dbfc47-4wkpw 

```





## 再次部署网络插件

```
wsg@k8smater1:~/kube_config$ kubectl apply -f calico.yaml 
configmap/calico-config unchanged
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org unchanged
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers unchanged
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers unchanged
clusterrole.rbac.authorization.k8s.io/calico-node unchanged
clusterrolebinding.rbac.authorization.k8s.io/calico-node unchanged
daemonset.apps/calico-node configured
serviceaccount/calico-node unchanged
deployment.apps/calico-kube-controllers unchanged
serviceaccount/calico-kube-controllers unchanged
wsg@k8smater1:~/kube_config$ 

```



## 更换calico版本到v3.14.0

更换后，calico node 还是无法恢复正常状态，descirbe pod还是无法正常建立BGP连接

# 问题总结



问题的本质： node1和 node2 上的pod 网络不通， 并且 calico pod的状态也不对。

进一步仔细检查发现，2个节点上，calico-node pod尝试建立BGP连接时，使用的ip不一样（node65 选择172.17.73.65，node66 选择了10.0.0.66）

原因：

calico node pod在启动时，会和其他节点建议BGP通信，node66上被加了一张网卡（10.0.0.66），不幸该网卡被用来和其他节点建立BGP连接，可是其他节点没有该网段。









## 正常后calicoctl的状态

node66

可以看到 给节点的peers，

```
wsg@k8smater1:~/kube_config$ DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config sudo ./calicoctl node status
[sudo] password for wsg: 
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 172.17.73.65 | node-to-node mesh | up    | 07:11:05 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
wsg@k8smater1:~/kube_config$ 

```



node65

```
wsg@k8smaster2:~$ sudo ./calicoctl node status
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 172.17.73.66 | node-to-node mesh | up    | 07:11:05 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

wsg@k8smaster2:~$ 

```





# 知识点记录



## calicoctl使用



```
带环境变量使用

wsg@k8smater1:~/kube_config$ DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config ./calicoctl get nodes
NAME         
k8smaster2   
k8smater1    

wsg@k8smater1:~/kube_config$ 


wsg@k8smater1:~/kube_config$ DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config sudo ./calicoctl node status
[sudo] password for wsg: 
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+------------+---------+
| PEER ADDRESS |     PEER TYPE     | STATE |   SINCE    |  INFO   |
+--------------+-------------------+-------+------------+---------+
| 172.17.73.65 | node-to-node mesh | start | 2020-05-21 | Passive |
+--------------+-------------------+-------+------------+---------+

IPv6 BGP status
No IPv6 peers found.

wsg@k8smater1:~/kube_config$ 


```





## calico configure

https://docs.projectcalico.org/reference/node/configuration





















