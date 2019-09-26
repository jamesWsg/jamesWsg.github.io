---
layout: post
title: kubernet学习记录
category: 技术
---



# 待解决问题

如何 定义 role，给 dashboard 所有权限？ 

如何用 证书的方式，创建 dashbaord，如何生成证书？



如何 编写 ymal， 启动 prometheus？ 如何 将 本地配置文件 关联到 容器。。

如何 在 github 上 dashboard的模板上 增加 tcp 的统计。。











## kubernet 组件



root@ubuntu16:~# dpkg -l |grep kuber
ii  kubernetes-cni                     0.7.5-00                                   amd64        Kubernetes CNI







## install kubeadm

ubuntu14 安装需要手动patch，官方源也没有

<https://stackoverflow.com/questions/44302071/how-to-install-latest-production-level-kubernetes-in-ubuntu-14>







ubuntu16

> kubernetes有单独的源

```
deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted
deb http://mirrors.aliyun.com/ubuntu/ xenial universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
deb http://mirrors.aliyun.com/ubuntu/ xenial multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
# kubeadm及kubernetes组件安装源
deb https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main
```



`root@ubuntu16:~# apt-get install kubeadm`



docker 环境需要单独安装，用上面的源 就可以。（ubuntu16的官方源没有找到docker）

```
root@ubuntu16:~# dpkg -l |grep docker
ii  docker.io                          18.09.2-0ubuntu1~16.04.1                   amd64        Linux container runtime
root@ubuntu16:~# 

```







## master 节点安装

init完之后，切换到普通用户，安装init输出提示，

```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```





```
wsg@ubuntu16:~$ kubectl get pods -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-fb8b8dccf-7tjxn            0/1     Pending   0          11h
coredns-fb8b8dccf-vdfrb            0/1     Pending   0          11h
etcd-ubuntu16                      1/1     Running   0          11h
kube-apiserver-ubuntu16            1/1     Running   0          11h
kube-controller-manager-ubuntu16   1/1     Running   24         11h
kube-proxy-rtzqx                   1/1     Running   0          11h
kube-scheduler-ubuntu16            1/1     Running   26         11h
wsg@ubuntu16:~$ 


```



* 安装网络插件

  ```
  kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

  ```

  ​



网络插件安装完，pending的pod进入 runing

```
wsg@ubuntu16:~$ kubectl get pods -n kube-system
NAME                               READY   STATUS              RESTARTS   AGE
coredns-fb8b8dccf-7tjxn            0/1     Pending             0          14h
coredns-fb8b8dccf-vdfrb            0/1     Pending             0          14h
etcd-ubuntu16                      1/1     Running             0          14h
kube-apiserver-ubuntu16            1/1     Running             0          14h
kube-controller-manager-ubuntu16   1/1     Running             43         14h
kube-proxy-rtzqx                   1/1     Running             0          14h
kube-scheduler-ubuntu16            1/1     Running             45         14h
weave-net-9qpps                    0/2     ContainerCreating   0          29s
wsg@ubuntu16:~$ 

```





> 单节点的 master，重启后，看起来 master 节点 服务可以自动起来。







### taint 操作

让master 节点可以 运行pod

```


CreationTimestamp:  Fri, 29 Mar 2019 22:45:16 +0800
Taints:             node-role.kubernetes.io/master:NoSchedule


wsg@ubuntu16:~$ kubectl taint nodes --all node-role.kubernetes.io/master-
node/ubuntu16 untainted
wsg@ubuntu16:~$ 


```





### 安装dashboard



???

```

http://<master-ip>:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

https://172.17.73.68:32605/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```







```
根据kubernet 官方推荐的dashboard
wsg@ubuntu16:~$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended/kubernetes-dashboard.yaml
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created
wsg@ubuntu16:~$ 

```

部署失败：

```
wsg@ubuntu16:~/kube_yaml$ kubectl get pods -n kube-system
NAME                                    READY   STATUS             RESTARTS 
kubernetes-dashboard-5f7b999d65-xt5r4   0/1     ImagePullBackOff   0          23m


打开yaml 文件，可以看到 其中用的image
    spec:
      containers:
      - name: kubernetes-dashboard
        image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1


mirrorgooglecontainers/kubernetes-dashboard-amd64
```



describer检查出错原因

```
wsg@ubuntu16:~/kube_yaml$ kubectl describe pod kubernetes-dashboard-5f7b999d65-xt5r4 -n kube-system
Name:               kubernetes-dashboard-5f7b999d65-xt5r4
Namespace:          kube-system
Priority:           0
PriorityClassName:  <none>
Node:               kube-worknode/172.17.73.67
Start Time:         Thu, 18 Apr 2019 14:03:53 +0800
Labels:             k8s-app=kubernetes-dashboard
                    pod-template-hash=5f7b999d65
Annotations:        <none>
Status:             Pending
IP:                 10.32.0.2
Controlled By:      ReplicaSet/kubernetes-dashboard-5f7b999d65
Containers:
  kubernetes-dashboard:
    Container ID:  
    Image:         k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
    Image ID:      
    Port:          8443/TCP
    Host Port:     0/TCP
    Args:
      --auto-generate-certificates
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Liveness:       http-get https://:8443/ delay=30s timeout=30s period=10s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /certs from kubernetes-dashboard-certs (rw)
      /tmp from tmp-volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kubernetes-dashboard-token-v4nkl (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  kubernetes-dashboard-certs:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  kubernetes-dashboard-certs
    Optional:    false
  tmp-volume:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     
    SizeLimit:  <unset>
  kubernetes-dashboard-token-v4nkl:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  kubernetes-dashboard-token-v4nkl
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node-role.kubernetes.io/master:NoSchedule
                 node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason          Age                    From                    Message
  ----     ------          ----                   ----                    -------
  Normal   Scheduled       36m                    default-scheduler       Successfully assigned kube-system/kubernetes-dashboard-5f7b999d65-xt5r4 to kube-worknode
  Normal   SandboxChanged  36m                    kubelet, kube-worknode  Pod sandbox changed, it will be killed and re-created.
  Warning  Failed          35m (x3 over 36m)      kubelet, kube-worknode  Failed to pull image "k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1": rpc error: code = Unknown desc = Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
  Warning  Failed          35m (x3 over 36m)      kubelet, kube-worknode  Error: ErrImagePull
  Normal   Pulling         34m (x4 over 36m)      kubelet, kube-worknode  Pulling image "k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1"
  Normal   BackOff         6m37s (x119 over 36m)  kubelet, kube-worknode  Back-off pulling image "k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1"
  Warning  Failed          88s (x140 over 36m)    kubelet, kube-worknode  Error: ImagePullBackOff
wsg@ubuntu16:~/kube_yaml$ 

```

解决方法：

```
wsg@ubuntu16:~/kube_yaml$ sudo docker pull mirrorgooglecontainers/kubernetes-dashboard-amd64:v1.10.1
[sudo] password for wsg: 
v1.10.1: Pulling from mirrorgooglecontainers/kubernetes-dashboard-amd64
63926ce158a6: Pull complete 
Digest: sha256:d6b4e5d77c1cdcb54cd5697a9fe164bc08581a7020d6463986fe1366d36060e8
Status: Downloaded newer image for mirrorgooglecontainers/kubernetes-dashboard-amd64:v1.10.1
wsg@ubuntu16:~/kube_yaml$ docker tag mirrorgooglecontainers/kubernetes-dashboard-amd64:v1.10.1 k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Post http://%2Fvar%2Frun%2Fdocker.sock/v1.39/images/mirrorgooglecontainers/kubernetes-dashboard-amd64:v1.10.1/tag?repo=k8s.gcr.io%2Fkubernetes-dashboard-amd64&tag=v1.10.1: dial unix /var/run/docker.sock: connect: permission denied
wsg@ubuntu16:~/kube_yaml$ sudo docker tag mirrorgooglecontainers/kubernetes-dashboard-amd64:v1.10.1 k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
wsg@ubuntu16:~/kube_yaml$ 

```

当执行了 kubectl -f  后，会创建pod，虽让创建 失败，但是 删除 之后，该pod 又会创建出来，可以 从另一个 侧面 看出 pod 的自动 创建 机制。从 上面 的 describe看出，该 pod 创建失败 是在 work-node 节点，所以 要在 work-node节点 pull dashboard image，然后再 删除 失败pod，触发 创建 新 pod

```
wsg@ubuntu16:~$ kubectl delete pod kubernetes-dashboard-5f7b999d65-xt5r4 -n kube-system
pod "kubernetes-dashboard-5f7b999d65-xt5r4" deleted
wsg@ubuntu16:~$ 

wsg@ubuntu16:~$ kubectl get pods -n kube-system
NAME                                    READY   STATUS    RESTARTS   AGE
coredns-fb8b8dccf-jd6hp                 1/1     Running   0          5d
coredns-fb8b8dccf-zptqw                 1/1     Running   0          5d
etcd-ubuntu16                           1/1     Running   0          5d
kube-apiserver-ubuntu16                 1/1     Running   0          5d
kube-controller-manager-ubuntu16        1/1     Running   0          5d
kube-proxy-ljg8p                        1/1     Running   0          5d
kube-proxy-zgkg2                        1/1     Running   0          5d
kube-scheduler-ubuntu16                 1/1     Running   0          5d
kubernetes-dashboard-5f7b999d65-pdqbs   1/1     Running   0          16s
weave-net-vv5mr                         2/2     Running   4          5d
weave-net-z28hj                         2/2     Running   0          5d
wsg@ubuntu16:~$ 

```



dashboard 如果不用 cert 证书，即使可以登陆还是 不能展示

![1555828258771](img/kube-dashboard.png)



#### dashboard 操作

```
wsg@ubuntu16:~/kube_ymal$ kubectl get service -n kube-system
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
kube-dns               ClusterIP   10.96.0.10     <none>        53/UDP,53/TCP,9153/TCP   22d
kubernetes-dashboard   NodePort    10.103.12.41   <none>        443:32605/TCP            111s
wsg@ubuntu16:~/kube_ymal$ kubectl get ep
NAME         ENDPOINTS           AGE
kubernetes   172.17.73.68:6443   22d
wsg@ubuntu16:~/kube_ymal$ kubectl get ep -n kube-system
NAME                      ENDPOINTS                                            AGE
kube-controller-manager   <none>                                               22d
kube-dns                  10.32.0.4:53,10.32.0.5:53,10.32.0.4:53 + 3 more...   22d
kube-scheduler            <none>                                               22d
kubernetes-dashboard      10.32.0.6:8443                                       2m29s
wsg@ubuntu16:~/kube_ymal$ 


```







### kubectl 命令



```
wsg@ubuntu16:~$ kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
ubuntu16   Ready    master   15h   v1.14.0
wsg@ubuntu16:~$ 


```



```
wsg@ubuntu16:/etc/kubernetes/pki$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://172.17.73.68:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
wsg@ubuntu16:/etc/kubernetes/pki$ 

wsg@ubuntu16:~/kube_docker_img$ kubectl get namespace
NAME              STATUS   AGE
default           Active   19h
kube-node-lease   Active   19h
kube-public       Active   19h
kube-system       Active   19h
wsg@ubuntu16:~/kube_docker_img$ 


```



检查节点的详细情况

```
wsg@ubuntu16:~/kube_docker_img$ kubectl describe node kube-worknode

```







## worker节点安装

kubernetes 的 Worker 节点跟 Master 节点几乎是相同的，它们运行着的都是一个 kubelet 组件。唯一的区别在于，在 kubeadm init 的过程中，kubelet 启动后，Master 节点上还会自动运行 kube-apiserver、kube-scheduler、kube-controller-manger 这三个系统 Pod。



```
root@ubuntu16:~# apt-get install kubeadm

wsg@kube-worknode:~$ dpkg -l |grep docker
ii  docker.io                          18.09.2-0ubuntu1~16.04.1                   amd64        Linux container runtime
wsg@kube-worknode:~$ dpkg -l |grep kub
ii  kubeadm                            1.14.0-00                                  amd64        Kubernetes Cluster Bootstrapping Tool
ii  kubectl                            1.14.0-00                                  amd64        Kubernetes Command Line Tool
ii  kubelet                            1.14.0-00                                  amd64        Kubernetes Node Agent
ii  kubernetes-cni                     0.7.5-00                                   amd64        Kubernetes CNI
wsg@kube-worknode:~$ 



```



加入后

```
wsg@ubuntu16:~/kube_docker_img$ kubectl get nodes
NAME            STATUS     ROLES    AGE     VERSION
kube-worknode   NotReady   <none>   7m51s   v1.14.0
ubuntu16        Ready      master   10m     v1.14.0
wsg@ubuntu16:~/kube_docker_img$ 


```

通过 journalctl 查看日志发现，work-node 也在尝试拉取 镜像， 手动加载 除 master 节点需要的 3个 kube image。。

```
   53  sudo docker load -i coredns.tar
   54  sudo docker load -i etcd.tar
   55  sudo docker load -i kube-proxy.tar
   56  sudo docker load -i weave-kube.tar
   57  sudo docker load -i weave-npc.tar

```



镜像加载后，等待一段时间。

```
wsg@ubuntu16:~$ kubectl get nodes
NAME            STATUS   ROLES    AGE   VERSION
kube-worknode   Ready    <none>   20h   v1.14.0
ubuntu16        Ready    master   20h   v1.14.0
wsg@ubuntu16:~$ 

```





## prometheus

官方的 docker image 没有 开启http reload 选项，导致修改 /etc/prometheus.yaml 无法生效

需要重新build prometheus镜像，但是 docker file 无法build 成功,因为网络原因

```
root@wsg:~/docker-file/promethus# docker build .

error pulling image configuration: Get https://d3uo42mtx6z2cr.cloudfront.net/sha256/6f/6f7f5431a5516717f91f4c018dbbe4a8bc4c730c8955dc634f2cd6e8dcfe0ddb?Expires=1558679818&Signature=RJ9wOqacmyTwbFA27yVhWTZflz17icgzwyhqsckjonAfHC46JfTOg5iIJZndQtxWXueRDiryWAYHUButdrupnhVDYdIqbILr47hMQ6K~OHDXoxMjWe-4m2zNbGbBi8BXO8d3WtBq6raS06LrdMxPJ0kJ6Ceh6eqkS3Uf2iSBWe~e8oiKpx3vaZAGw-GdTwX8DeSM~ND0rvcKtG8KBELlc2oF0Hm16~i8km1z~4bIj2Uw8kmEvG8nchrM7zBgRVdUG2Kkco1lGrQ2m9dXLMMbGXoFKwPBqSNsCMbJSxDDSzfafB0qWrClc4zi1R4dwFB09Xl9Mk7RzNIvGJs51wDsFw__&Key-Pair-Id=APKAJ67PQLWGCSP66DGA: dial tcp: lookup d3uo42mtx6z2cr.cloudfront.net on 114.114.114.114:53: read udp 172.17.73.69:56303->114.114.114.114:53: i/o timeout
root@wsg:~/docker-file/promethus# 

```



web reload

Alternatively, you can send a HTTP POST to the Prometheus web server:

```
curl -X POST http://localhost:9090/-/reload
```

Note that as of Prometheus 2.0, the `--web.enable-lifecycle` command line flag must be passed for HTTP reloading to work.



### prometheus

docker inspect 检查官方镜像的信息。

```
sudo docker pull prom/prometheus


            "Volumes": {
                "/prometheus": {}
            },
            "WorkingDir": "/prometheus",
            "Entrypoint": [
                "/bin/prometheus"





```



运行

```
sudo docker run --name wsg-prometheus -d -p 9090:9090 -v /mnt/prom_data/prometheus.yml:/etc/prometheus/prometheus.yml  prom/prometheus


其中
wsg@ubuntu16:/mnt/prom_data$ cat prometheus.yml 
global:
  scrape_interval:     10s
  evaluation_interval: 10s

scrape_configs:
  - job_name: 'code-vm'
    static_configs:
      - targets: ['172.17.73.70:9100']


  - job_name: 'cvm-124'
    static_configs:
      - targets: ['172.17.75.124:9100']
  - job_name: 'cvm-125'
    static_configs:
      - targets: ['172.17.75.125:9100']
  - job_name: 'phs-246'
    static_configs:
      - targets: ['172.17.73.246:9100']
  - job_name: 'phs-247'
    static_configs:
      - targets: ['172.17.73.247:9100']
wsg@ubuntu16:/mnt/prom_data$ 

```





###dashboard grafana

```
https://grafana.com/grafana/download

wsg@ubuntu16:~/prometheus_dashboard$ dpkg -l |grep grafan
ii  grafana                            6.2.0                                      amd64        Grafana



wsg@ubuntu16:~/prometheus_dashboard$ sudo /etc/init.d/grafana-server start
[ ok ] Starting grafana-server (via systemctl): grafana-server.service.



wsg@ubuntu16:~/kube_ymal$ sudo netstat -atpn |grep grafa
tcp6       0      0 :::3000                 :::*                    LISTEN      28138/grafana-serve
wsg@ubuntu16:~/kube_ymal$ 


```





### node exporter

```
https://prometheus.io/download/

```



```
curl -OL https://github.com/prometheus/node_exporter/releases/download/v0.15.2/node_exporter-0.15.2.darwin-amd64.tar.gz
tar -xzf node_exporter-0.15.2.darwin-amd64.tar.gz
```

运行node exporter:

```
cd node_exporter-0.15.2.darwin-amd64
cp node_exporter-0.15.2.darwin-amd64/node_exporter /usr/local/bin/
node_exporter

##启用 disable 指标
--collector.perf  无法启用，估计要 搭配其他 的配置
root@converger-124:~/wsg/node_exporter-0.18.0.linux-amd64# ./node_exporter  --collector.processes --collector.tcpstat


```

启动成功后，可以看到以下输出：

```
INFO[0000] Listening on :9100                            source="node_exporter.go:7
```







## create deployment yaml







# 知识整理



## statefulset



statefulset 是改良的 deployment

管理拓扑状态和存储状态









# 问题处理记录

## kubeadm init

```
root@ubuntu16:~# kubeadm init
[init] Using Kubernetes version: v1.14.0
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR Swap]: running with swap on is not supported. Please disable swap
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
```

处理方法:

关闭swap `root@ubuntu16:~#  sudo swapoff -a`





## kubeadm init gcr.io

```
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/kube-apiserver:v1.14.0: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/kube-controller-manager:v1.14.0: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/kube-scheduler:v1.14.0: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/kube-proxy:v1.14.0: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/pause:3.1: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/etcd:3.3.10: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/coredns:1.3.1: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`

```

解决方法

> 参考：<https://blog.csdn.net/jinguangliu/article/details/82792617>
>
> 

gcr 在docker.io 上有镜像

```
提前拉取docker hub上的镜像

docker pull mirrorgooglecontainers/kube-apiserver:v1.14.0
docker pull mirrorgooglecontainers/kube-controller-manager:v1.14.0
docker pull mirrorgooglecontainers/kube-scheduler:v1.14.0
docker pull mirrorgooglecontainers/kube-proxy:v1.14.0
docker pull mirrorgooglecontainers/pause:3.1
docker pull mirrorgooglecontainers/etcd:3.3.10
docker pull coredns/coredns:1.3.1        ##注意仓库名和前面不一样


拉取完成后，需要更改tag
docker tag mirrorgooglecontainers/kube-apiserver:v1.14.0 k8s.gcr.io/kube-apiserver:v1.14.0
docker tag mirrorgooglecontainers/kube-controller-manager:v1.14.0 k8s.gcr.io/kube-controller-manager:v1.14.0
docker tag mirrorgooglecontainers/kube-scheduler:v1.14.0 k8s.gcr.io/kube-scheduler:v1.14.0
docker tag mirrorgooglecontainers/kube-proxy:v1.14.0 k8s.gcr.io/kube-proxy:v1.14.0
docker tag mirrorgooglecontainers/pause:3.1 k8s.gcr.io/pause:3.1
docker tag mirrorgooglecontainers/etcd:3.3.10 k8s.gcr.io/etcd:3.3.10
docker tag coredns/coredns:1.3.1 k8s.gcr.io/coredns:1.3.1
 
 
```

将这些 image 保存到 nas, 后续安装 master 节点节省时间。。











## kubeadm init 正常记录

```
root@ubuntu16:~# kubeadm init
I0329 22:44:51.067111    7566 version.go:96] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.txt": Get https://storage.googleapis.com/kubernetes-release/release/stable-1.txt: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
I0329 22:44:51.067262    7566 version.go:97] falling back to the local client version: v1.14.0
[init] Using Kubernetes version: v1.14.0
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [ubuntu16 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 172.17.73.68]
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [ubuntu16 localhost] and IPs [172.17.73.68 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [ubuntu16 localhost] and IPs [172.17.73.68 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 20.027611 seconds
[upload-config] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.14" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --experimental-upload-certs
[mark-control-plane] Marking the node ubuntu16 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node ubuntu16 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: e6lqeh.6fq6eabr5lvm47m7
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.17.73.68:6443 --token e6lqeh.6fq6eabr5lvm47m7 \
    --discovery-token-ca-cert-hash sha256:682c5cb5f0a0b3134ccdb66f7bdd03a6c2cb16c2e3cba3b10d9bf7577eee2e04 
root@ubuntu16:~# 

```



## 恢复快照 ，重新 init

提前 准备好 init过程中需要的 image

```
sudo docker save -o kube-proxy.tar k8s.gcr.io/kube-proxy                 
sudo docker save -o kube-apiserver.tar k8s.gcr.io/kube-apiserver             
sudo docker save -o kube-scheduler.tar k8s.gcr.io/kube-scheduler             
sudo docker save -o kube-controller-manager.tar k8s.gcr.io/kube-controller-manager    
sudo docker save -o weave-npc.tar weaveworks/weave-npc                  
sudo docker save -o weave-kube.tar weaveworks/weave-kube                 
sudo docker save -o coredns.tar k8s.gcr.io/coredns                    
sudo docker save -o etcd.tar k8s.gcr.io/etcd                       
sudo docker save -o pause.tar k8s.gcr.io/pause                      


sudo docker load -i  kube-proxy.tar             
sudo docker load -i  kube-apiserver.tar           
sudo docker load -i  kube-scheduler.tar         
sudo docker load -i  kube-controller-manager.tar
sudo docker load -i  weave-npc.tar             
sudo docker load -i  weave-kube.tar           
sudo docker load -i  coredns.tar              
sudo docker load -i  etcd.tar                  
sudo docker load -i  pause.tar 
```







完整init 输出如下：

```
wsg@ubuntu16:~/kube_docker_img$ sudo kubeadm init
I0413 16:15:24.936996    2979 version.go:96] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.txt": Get https://dl.k8s.io/release/stable-1.txt: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
I0413 16:15:24.937099    2979 version.go:97] falling back to the local client version: v1.14.0
[init] Using Kubernetes version: v1.14.0
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [ubuntu16 localhost] and IPs [172.17.73.68 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [ubuntu16 localhost] and IPs [172.17.73.68 127.0.0.1 ::1]
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [ubuntu16 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 172.17.73.68]
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 18.006672 seconds
[upload-config] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.14" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --experimental-upload-certs
[mark-control-plane] Marking the node ubuntu16 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node ubuntu16 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: 5s6jut.7k67cvl6g60cajj3
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.17.73.68:6443 --token 5s6jut.7k67cvl6g60cajj3 \
    --discovery-token-ca-cert-hash sha256:ba2732918ee36538bfbf80cb2514c7cbf830bafbe9f3d3ba68810c1e07df4755 
wsg@ubuntu16:~/kube_docker_img$ 

```



然后 立即 worknode 加入,这次 没有 出现 timeout 情况，

```
wsg@kube-worknode:~$ sudo kubeadm join 172.17.73.68:6443 --token 5s6jut.7k67cvl6g60cajj3     --discovery-token-ca-cert-hash sha256:ba2732918ee36538bfbf80cb2514c7cbf830bafbe9f3d3ba68810c1e07df4755 
[sudo] password for wsg: 
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.14" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

wsg@kube-worknode:~$ 

```



master节点查看

```
wsg@ubuntu16:~/kube_docker_img$ kubectl get nodes
NAME            STATUS     ROLES    AGE     VERSION
kube-worknode   NotReady   <none>   4m46s   v1.14.0
ubuntu16        Ready      master   7m26s   v1.14.0


```





## worknode 加入 错误



1. 编辑 service

2. 报错如下：

   ```
   wsg@kube-worknode:~$ sudo kubeadm join 172.17.73.68:6443 --token e6lqeh.6fq6eabr5lvm47m7     --discovery-token-ca-cert-hash sha256:682c5cb5f0a0b3134ccdb66f7bdd03a6c2cb16c2e3cba3b10d9bf7577eee2e04
   sudo: unable to resolve host kube-worknode
   [preflight] Running pre-flight checks
   	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
   	[WARNING Hostname]: hostname "kube-worknode" could not be reached
   	[WARNING Hostname]: hostname "kube-worknode": lookup kube-worknode on 114.114.114.114:53: no such host
   error execution phase preflight: [preflight] Some fatal errors occurred:
   	[ERROR Swap]: running with swap on is not supported. Please disable swap
   [preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
   wsg@kube-worknode:~$ 
   ```

   原因： 手动更改了 /etc/hostname ，但是/etc/hosts 中还是用了 原来的 hostname。 更改 /etc/hosts  为 正确的hostname 。 引出 下面的问题：

3. 报错如下：

   ```
   wsg@kube-worknode:~$ sudo kubeadm join 172.17.73.68:6443 --token e6lqeh.6fq6eabr5lvm47m7     --discovery-token-ca-cert-hash sha256:682c5cb5f0a0b3134ccdb66f7bdd03a6c2cb16c2e3cba3b10d9bf7577eee2e04
   [preflight] Running pre-flight checks
   	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
   error execution phase preflight: [preflight] Some fatal errors occurred:
   	[ERROR Swap]: running with swap on is not supported. Please disable swap
   [preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
   wsg@kube-worknode:~$ 

   ```

   和master节点一样，需要 关闭swap，接着遇到下面的报错

4. 报错如下

   ```
   wsg@kube-worknode:~$ sudo kubeadm join 172.17.73.68:6443 --token e6lqeh.6fq6eabr5lvm47m7     --discovery-token-ca-cert-hash sha256:682c5cb5f0a0b3134ccdb66f7bdd03a6c2cb16c2e3cba3b10d9bf7577eee2e04
   [preflight] Running pre-flight checks
   	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
   error execution phase preflight: couldn't validate the identity of the API Server: abort connecting to API servers after timeout of 5m0s
   wsg@kube-worknode:~$ 

   ```

   join的命令 是在第一次 创建master节点时 ，console 自动 吐出来的。 然后 该 master节点有发生过 强制重启。

5. dashboard 






## work node 被自动踢出后，再加入

worknode 上有残留 配置信息。



```
wsg@kube-worknode:~$ kubeadm join 172.17.73.68:6443 --token vmle7l.r7z3d3rxexm5rkun     --discovery-token-ca-cert-hash sha256:682c5cb5f0a0b3134ccdb66f7bdd03a6c2cb16c2e3cba3b10d9bf7577eee2e04 
[preflight] Running pre-flight checks
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR IsPrivilegedUser]: user is not running as root
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
wsg@kube-worknode:~$ sudo kubeadm join 172.17.73.68:6443 --token vmle7l.r7z3d3rxexm5rkun     --discovery-token-ca-cert-hash sha256:682c5cb5f0a0b3134ccdb66f7bdd03a6c2cb16c2e3cba3b10d9bf7577eee2e04 
[sudo] password for wsg: 
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR FileAvailable--etc-kubernetes-kubelet.conf]: /etc/kubernetes/kubelet.conf already exists
	[ERROR FileAvailable--etc-kubernetes-bootstrap-kubelet.conf]: /etc/kubernetes/bootstrap-kubelet.conf already exists
	[ERROR Swap]: running with swap on is not supported. Please disable swap
	[ERROR FileAvailable--etc-kubernetes-pki-ca.crt]: /etc/kubernetes/pki/ca.crt already exists
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
wsg@kube-worknode:~$ ll
total 351368


```



## master 节点生成token后，worknode 节点加入还是报错



```
wsg@ubuntu16-template:~$ sudo kubeadm join 172.17.73.68:6443 --token uz7fhd.p8kvwrrimnlbg070     --discovery-token-ca-cert-hash sha256:682c5cb5f0a0b3134ccdb66f7bdd03a6c2cb16c2e3cba3b10d9bf7577eee2e04 
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.15" ConfigMap in the kube-system namespace
error execution phase kubelet-start: configmaps "kubelet-config-1.15" is forbidden: User "system:bootstrap:uz7fhd" cannot get resource "configmaps" in API group "" in the namespace "kube-system"
wsg@ubuntu16-template:~$ 

worknode
wsg@ubuntu16-template:~$ sudo dpkg -l |grep kube
ii  kubeadm                            1.15.3-00                                       amd64        Kubernetes Cluster Bootstrapping Tool
ii  kubectl                            1.15.3-00                                       amd64        Kubernetes Command Line Tool
ii  kubelet                            1.15.3-00                                       amd64        Kubernetes Node Agent
ii  kubernetes-cni                     0.7.5-00                                        amd64        Kubernetes CNI
wsg@ubuntu16-template:~$ 





```

发现 官方源中 版本已经更新了，手动 安装和 master节点一致的 kube 版本

```
 sudo apt-get install kubeadm=1.14.0-00 kubelet=1.14.0-00 kubectl=1.14.0-00

wsg@ubuntu16-template:~$ dpkg -l |grep kube
ii  kubeadm                            1.14.0-00                                       amd64        Kubernetes Cluster Bootstrapping Tool
ii  kubectl                            1.14.0-00                                       amd64        Kubernetes Command Line Tool
ii  kubelet                            1.14.0-00                                       amd64        Kubernetes Node Agent
ii  kubernetes-cni                     0.7.5-00                                        amd64        Kubernetes CNI
wsg@ubuntu16-template:~$ dpkg -l |grep docker
wsg@ubuntu16-template:~$ 


本次docker 版本和 master 节点有点 小版本区别

wsg@ubuntu16-template:~$ dpkg -l |grep docker
ii  docker.io                          18.09.7-0ubuntu1~16.04.5                        amd64        Linux container runtime
wsg@ubuntu16-template:~$ 

```



没有细看错误，删错文件

```
sg@ubuntu16-template:~$ sudo kubeadm join 172.17.73.68:6443 --token uz7fhd.p8kvwrrimnlbg070     --discovery-token-ca-cert-hash sha256:682c5cb5f0a0b3134ccdb66f7bdd03a6c2cb16c2e3cba3b10d9bf7577eee2e04 
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR FileAvailable--etc-kubernetes-bootstrap-kubelet.conf]: /etc/kubernetes/bootstrap-kubelet.conf already exists
	[ERROR Swap]: running with swap on is not supported. Please disable swap
	[ERROR FileAvailable--etc-kubernetes-pki-ca.crt]: /etc/kubernetes/pki/ca.crt already exists
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
wsg@ubuntu16-template:~$ rm -rf /etc/kubernetes/

```





## swap 开启的状况下，worknode 节点启动失败



journal 查看日志后，发现 提示 swap on， swapoff -a 



永久 关闭swap







# 常用命令



## 创建pv

```
wsg@ubuntu16:~/kube_ymal/elastic-search$ kubectl apply -f pv.yaml 
persistentvolume/elasticsearch-data created

wsg@ubuntu16:~/kube_ymal/elastic-search$ kubectl get pv
NAME                 CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
elasticsearch-data   100Gi      RWX            Retain           Available           manual                  7s
wsg@ubuntu16:~/kube_ymal/elastic-search$ 



```



查看pv

```
wsg@ubuntu16:~/kube_ymal/elastic-search$ kubectl describe pv elasticsearch-data
Name:            elasticsearch-data
Labels:          <none>
Annotations:     kubectl.kubernetes.io/last-applied-configuration:
                   {"apiVersion":"v1","kind":"PersistentVolume","metadata":{"annotations":{},"name":"elasticsearch-data"},"spec":{"accessModes":["ReadWriteMa...
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    manual
Status:          Available
Claim:           
Reclaim Policy:  Retain
Access Modes:    RWX
VolumeMode:      Filesystem
Capacity:        100Gi
Node Affinity:   <none>
Message:         
Source:
    Type:      NFS (an NFS mount that lasts the lifetime of a pod)
    Server:    172.17.75.128
    Path:      /var/share/ezfs/shareroot/share/shengguo/kube_share
    ReadOnly:  false
Events:        <none>
wsg@ubuntu16:~/kube_ymal/elastic-search$ 


```





## 查看 token（secret）

> 后面es的token，没有数据？？

```
wsg@ubuntu16:~$ kubectl get secret
NAME                                   TYPE                                  DATA   AGE
default-token-qpmqt                    kubernetes.io/service-account-token   3      148d
quickstart-es-2cwwv2t5rn-certs         Opaque                                0      17h
quickstart-es-2cwwv2t5rn-config        Opaque                                1      17h
quickstart-es-elastic-user             Opaque                                1      17h
quickstart-es-http-ca-internal         Opaque                                2      17h
quickstart-es-http-certs-internal      Opaque                                2      17h
quickstart-es-http-certs-public        Opaque                                1      17h
quickstart-es-internal-users           Opaque                                3      17h
quickstart-es-transport-ca-internal    Opaque                                2      17h
quickstart-es-transport-certs-public   Opaque                                1      17h
quickstart-es-xpack-file-realm         Opaque                                3      17h
wsg@ubuntu16:~$ kubectl describe secret default-token-qpmqt
Name:         default-token-qpmqt
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: default
              kubernetes.io/service-account.uid: 4a2119fb-5231-11e9-a74d-005056a74747

Type:  kubernetes.io/service-account-token

Data
====
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZmF1bHQtdG9rZW4tcXBtcXQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVmYXVsdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjRhMjExOWZiLTUyMzEtMTFlOS1hNzRkLTAwNTA1NmE3NDc0NyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.yM98bm5vxW3D_I6n7dPCkxiLNayMo08NYofbTHmkUerx04Am_p20G-SfXpFAdPpQj2Us7zw4vUMvGczP9FouyMZ2PjwHKhUE5vipZU_fMqFIZzp12YjBnNPLmRBB4IXORtJI-KBvybhet-rycZMkyZi-sOLUaNlDYIvBTLjLITx3QlriDkuKVxKvPQMD_3-pUHiwlOG-OTUC8k_izIW7ZGG4Jn0Xv_mCLkGjERrSxIF056i7LErOJ8vQsOc3YmGMpEZ2315a6rl7-XiUpafkaHMk6TqywUoZZiEE-1Cwf-DRerZX8apWLSbofBg_OkQF3CRYqfEtlexWGkysoR5Xig
ca.crt:     1025 bytes
namespace:  7 bytes
wsg@ubuntu16:~$ kubectl describe secret quickstart-es-2cwwv2t5rn-certs
Name:         quickstart-es-2cwwv2t5rn-certs
Namespace:    default
Labels:       certificates.elasticsearch.k8s.elastic.co/type=transport
              elasticsearch.k8s.elastic.co/cluster-name=quickstart
              elasticsearch.k8s.elastic.co/pod-name=quickstart-es-2cwwv2t5rn
Annotations:  <none>

Type:  Opaque

Data
====
wsg@ubuntu16:~$ kubectl describe secret quickstart-es-elastic-user 
Name:         quickstart-es-elastic-user
Namespace:    default
Labels:       common.k8s.elastic.co/type=elasticsearch
              elasticsearch.k8s.elastic.co/cluster-name=quickstart
Annotations:  <none>

Type:  Opaque

Data
====
elastic:  24 bytes
wsg@ubuntu16:~$ 

```





## 查看集群内置用户

```
wsg@ubuntu16:~$ kubectl get sa
NAME      SECRETS   AGE
default   1         148d

wsg@ubuntu16:~$ kubectl get sa --all-namespaces
NAMESPACE         NAME                                 SECRETS   AGE
default           default                              1         148d
elastic-system    default                              1         18h
elastic-system    elastic-operator                     1         18h
kube-node-lease   default                              1         148d
kube-public       default                              1         148d
kube-system       attachdetach-controller              1         148d
kube-system       bootstrap-signer                     1         148d
kube-system       certificate-controller               1         148d


```





## 查看集群配置信息



```
wsg@ubuntu16:~$ kubectl -n kube-system get cm kubeadm-config -oyaml
apiVersion: v1
data:
  ClusterConfiguration: |
    apiServer:
      extraArgs:
        authorization-mode: Node,RBAC
      timeoutForControlPlane: 4m0s
    apiVersion: kubeadm.k8s.io/v1beta1
    certificatesDir: /etc/kubernetes/pki
    clusterName: kubernetes
    controlPlaneEndpoint: ""
    controllerManager: {}
    dns:
      type: CoreDNS
    etcd:
      local:
        dataDir: /var/lib/etcd
    imageRepository: k8s.gcr.io
    kind: ClusterConfiguration
    kubernetesVersion: v1.14.0
    networking:
      dnsDomain: cluster.local
      podSubnet: ""
      serviceSubnet: 10.96.0.0/12
    scheduler: {}
  ClusterStatus: |
    apiEndpoints:
      ubuntu16:
        advertiseAddress: 172.17.73.68
        bindPort: 6443
    apiVersion: kubeadm.k8s.io/v1beta1
    kind: ClusterStatus
kind: ConfigMap
metadata:
  creationTimestamp: "2019-03-29T14:45:19Z"
  name: kubeadm-config
  namespace: kube-system
  resourceVersion: "160"
  selfLink: /api/v1/namespaces/kube-system/configmaps/kubeadm-config
  uid: 464d4609-5231-11e9-a74d-005056a74747

```



## kubeadm reset







## 删除节点



```
wsg@ubuntu16:/etc/kubernetes$ kubectl get nodes
NAME                STATUS     ROLES    AGE    VERSION
ubuntu16            Ready      master   147d   v1.14.0
ubuntu16-template   NotReady   <none>   155m   v1.14.0
wsg@ubuntu16:/etc/kubernetes$ kubectl delete node ubuntu16-template
node "ubuntu16-template" deleted
wsg@ubuntu16:/etc/kubernetes$ kubectl get nodes
NAME       STATUS   ROLES    AGE    VERSION
ubuntu16   Ready    master   147d   v1.14.0
wsg@ubuntu16:/etc/kubernetes$ 


```



网上看到，在删除节点之前还有个 drain命令，不知作用是啥。。

```
kubectl drain k8s-node2 --delete-local-data --force --ignore-daemonsets
kubectl delete node k8s-node2
```



## add node



```
wsg@ubuntu16:~$ kubeadm token list
TOKEN     TTL       EXPIRES   USAGES    DESCRIPTION   EXTRA GROUPS
wsg@ubuntu16:~$ 

wsg@ubuntu16:~$ kubeadm token create --print-join-command
kubeadm join 172.17.73.68:6443 --token vmle7l.r7z3d3rxexm5rkun     --discovery-token-ca-cert-hash sha256:682c5cb5f0a0b3134ccdb66f7bdd03a6c2cb16c2e3cba3b10d9bf7577eee2e04 
wsg@ubuntu16:~$ 


wsg@ubuntu16:~$ kubeadm token list
TOKEN                     TTL       EXPIRES                     USAGES                   DESCRIPTION   EXTRA GROUPS
vmle7l.r7z3d3rxexm5rkun   23h       2019-08-20T18:36:56+08:00   authentication,signing   <none>        system:bootstrappers:kubeadm:default-node-token
wsg@ubuntu16:~$ 






```







## 内置用户，service account

```
wsg@ubuntu16:~/kube_ymal/no_https_dashboard$ kubectl get sa -n kube-system
NAME                                 SECRETS   AGE
attachdetach-controller              1         22d
bootstrap-signer                     1         22d
certificate-controller               1         22d
clusterrole-aggregation-controller   1         22d


查看 具体某个sa 的 信息
wsg@ubuntu16:~$ kubectl describe sa kubernetes-dashboard -n kube-system
Name:                kubernetes-dashboard
Namespace:           kube-system
Labels:              k8s-app=kubernetes-dashboard
Annotations:         kubectl.kubernetes.io/last-applied-configuration:
                       {"apiVersion":"v1","kind":"ServiceAccount","metadata":{"annotations":{},"labels":{"k8s-app":"kubernetes-dashboard"},"name":"kubernetes-das...
Image pull secrets:  <none>
Mountable secrets:   kubernetes-dashboard-token-pd5sm
Tokens:              kubernetes-dashboard-token-pd5sm
Events:              <none>
wsg@ubuntu16:~$ 


wsg@ubuntu16:~$ kubectl get clusterrole -n kube-system
NAME                                                                   AGE
admin                                                                  22d
cluster-admin                                                          22d
edit                                                                   22d
system:aggregate-to-admin                                              22d


wsg@ubuntu16:~$ kubectl get roles -n kube-system
NAME                                             AGE
extension-apiserver-authentication-reader        22d
kube-proxy                                       22d
kubeadm:kubelet-config-1.14                      22d
kubeadm:nodes-kubeadm-config                     22d
kubernetes-dashboard-minimal                     4h46m
system::leader-locking-kube-controller-manager   22d
system::leader-locking-kube-scheduler            22d
system:controller:bootstrap-signer               22d


wsg@ubuntu16:~$ kubectl get RoleBinding -n kube-system
NAME                                                AGE
kube-proxy                                          22d
kubeadm:kubelet-config-1.14                         22d
kubeadm:nodes-kubeadm-config                        22d
kubernetes-dashboard-minimal                        4h47m
system::extension-apiserver-authentication-reader   22d
system::leader-locking-kube-controller-manager      22d
system::leader-locking-kube-scheduler               22d
system:controller:bootstrap-signer                  22d

# 产看具体rolebinding
wsg@ubuntu16:~$ kubectl describe RoleBinding kubernetes-dashboard-minimal  -n kube-system
Name:         kubernetes-dashboard-minimal
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"rbac.authorization.k8s.io/v1","kind":"RoleBinding","metadata":{"annotations":{},"name":"kubernetes-dashboard-minimal","name...
Role:
  Kind:  Role
  Name:  kubernetes-dashboard-minimal
Subjects:
  Kind            Name                  Namespace
  ----            ----                  ---------
  ServiceAccount  kubernetes-dashboard  kube-system
wsg@ubuntu16:~$ 

```





## 查看node的详细信息

```
wsg@ubuntu16:~/kube_docker_img$ kubectl get nodes
NAME            STATUS     ROLES    AGE   VERSION
kube-worknode   NotReady   <none>   19h   v1.14.0
ubuntu16        Ready      master   19h   v1.14.0
wsg@ubuntu16:~/kube_docker_img$ kubectl describe node kube-worknode
Name:               kube-worknode
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=kube-worknode
                    kubernetes.io/os=linux
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Sat, 13 Apr 2019 16:18:25 +0800
Taints:             node.kubernetes.io/not-ready:NoExecute
                    node.kubernetes.io/not-ready:NoSchedule
Unschedulable:      false
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Sun, 14 Apr 2019 11:27:16 +0800   Sat, 13 Apr 2019 16:18:25 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Sun, 14 Apr 2019 11:27:16 +0800   Sat, 13 Apr 2019 16:18:25 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Sun, 14 Apr 2019 11:27:16 +0800   Sat, 13 Apr 2019 16:18:25 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            False   Sun, 14 Apr 2019 11:27:16 +0800   Sat, 13 Apr 2019 16:18:25 +0800   KubeletNotReady              runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
Addresses:
  InternalIP:  172.17.73.67
  Hostname:    kube-worknode
Capacity:
 cpu:                4
 ephemeral-storage:  26208124Ki
 hugepages-2Mi:      0
 memory:             4046472Ki
 pods:               110
Allocatable:
 cpu:                4
 ephemeral-storage:  24153407039
 hugepages-2Mi:      0
 memory:             3944072Ki
 pods:               110
System Info:
 Machine ID:                 3dcb62efe8adf5cc8efbe4005a657162
 System UUID:                4222ACAE-6323-10DB-5175-0063D69B9D4B
 Boot ID:                    78be4fab-1471-4cb9-b524-0d0507d5a3e8
 Kernel Version:             4.4.0-21-generic
 OS Image:                   Ubuntu 16.04 LTS
 Operating System:           linux
 Architecture:               amd64
 Container Runtime Version:  docker://18.9.2
 Kubelet Version:            v1.14.0
 Kube-Proxy Version:         v1.14.0
Non-terminated Pods:         (2 in total)
  Namespace                  Name                CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                  ----                ------------  ----------  ---------------  -------------  ---
  kube-system                kube-proxy-ljg8p    0 (0%)        0 (0%)      0 (0%)           0 (0%)         19h
  kube-system                weave-net-vv5mr     20m (0%)      0 (0%)      0 (0%)           0 (0%)         19h
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests  Limits
  --------           --------  ------
  cpu                20m (0%)  0 (0%)
  memory             0 (0%)    0 (0%)
  ephemeral-storage  0 (0%)    0 (0%)
Events:              <none>
wsg@ubuntu16:~/kube_docker_img$ 

```



## 检查报错日志

> 日志中发现，kube 采用的是 go 语言

```
 journalctl -xefu kubelet


```





## 查看所有pod

```
wsg@ubuntu16:~$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                    READY   STATUS    RESTARTS   AGE
kube-system   coredns-fb8b8dccf-7tjxn                 1/1     Running   9          147d
kube-system   coredns-fb8b8dccf-vdfrb                 1/1     Running   9          147d
kube-system   etcd-ubuntu16                           1/1     Running   5          147d
kube-system   kube-apiserver-ubuntu16                 1/1     Running   6          147d
kube-system   kube-controller-manager-ubuntu16        1/1     Running   462        147d
kube-system   kube-proxy-bzsfk                        1/1     Running   0          55m
kube-system   kube-proxy-rtzqx                        1/1     Running   4          147d
kube-system   kube-scheduler-ubuntu16                 1/1     Running   460        147d
kube-system   kubernetes-dashboard-54fb766c84-sxwrv   1/1     Running   3          124d
kube-system   weave-net-9qpps                         2/2     Running   9          147d
kube-system   weave-net-pkhsx                         2/2     Running   0          55m
wsg@ubuntu16:~$ ll

```



## 查看pod所在 节点

```
wsg@ubuntu16:~$  kubectl get pod -n kube-system -o wide
NAME                                    READY   STATUS    RESTARTS   AGE    IP             NODE            NOMINATED NODE   READINESS GATES
coredns-fb8b8dccf-jd6hp                 1/1     Running   0          5d1h   10.40.0.1      ubuntu16        <none>           <none>
coredns-fb8b8dccf-zptqw                 1/1     Running   0          5d1h   10.40.0.2      ubuntu16        <none>           <none>
etcd-ubuntu16                           1/1     Running   0          5d1h   172.17.73.68   ubuntu16        <none>           <none>
kube-apiserver-ubuntu16                 1/1     Running   0          5d1h   172.17.73.68   ubuntu16        <none>           <none>
kube-controller-manager-ubuntu16        1/1     Running   0          5d1h   172.17.73.68   ubuntu16        <none>           <none>
kube-proxy-ljg8p                        1/1     Running   0          5d1h   172.17.73.67   kube-worknode   <none>           <none>
kube-proxy-zgkg2                        1/1     Running   0          5d1h   172.17.73.68   ubuntu16        <none>           <none>
kube-scheduler-ubuntu16                 1/1     Running   0          5d1h   172.17.73.68   ubuntu16        <none>           <none>
kubernetes-dashboard-5f7b999d65-pdqbs   1/1     Running   0          60m    10.32.0.2      kube-worknode   <none>           <none>
weave-net-vv5mr                         2/2     Running   4          5d1h   172.17.73.67   kube-worknode   <none>           <none>
weave-net-z28hj                         2/2     Running   0          5d1h   172.17.73.68   ubuntu16        <none>           <none>
wsg@ubuntu16:~$ 

```



## 查看 service和删除deployment



```
wsg@ubuntu16:~/kube_yaml$ kubectl get service
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   5d17h

wsg@ubuntu16:~/kube_yaml$ kubectl get service -n kube-system
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
kube-dns               ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP   5d17h
kubernetes-dashboard   NodePort    10.103.226.110   <none>        443:32678/TCP            20h
wsg@ubuntu16:~/kube_yaml$ 



wsg@ubuntu16:~/kube_yaml$ kubectl get deployment -n kube-system
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
coredns                2/2     2            2           5d18h
kubernetes-dashboard   1/1     1            1           20h
wsg@ubuntu16:~/kube_yaml$ 


wsg@ubuntu16:~/kube_yaml$ kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended/kubernetes-dashboard.yaml
secret "kubernetes-dashboard-certs" deleted
secret "kubernetes-dashboard-csrf" deleted
serviceaccount "kubernetes-dashboard" deleted
role.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" deleted
rolebinding.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" deleted
deployment.apps "kubernetes-dashboard" deleted
service "kubernetes-dashboard" deleted
wsg@ubuntu16:~/kube_yaml$ kubectl get deployment -n kube-system
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   2/2     2            2           5d18h
wsg@ubuntu16:~/kube_yaml$ 


```

