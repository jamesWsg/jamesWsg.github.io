---
layout: post
title: kubernets高可用master配置相关
category: 技术
---



# 背景



当前的3节点 k8s 集群，只有一个master节点，基于kubeadm v1.14版本，信息如下

```
wsg@ubuntu16:~$ kubectl get node
NAME                  STATUS     ROLES    AGE    VERSION
ubuntu16              Ready      master   346d   v1.14.0
ubuntu16-template     Ready      <none>   199d   v1.14.0
wsg-virtual-machine   NotReady   <none>   190d   v1.14.0
wsg@ubuntu16:~$
```

想尝试增加master节点，了解到kubeadm1.14的版本下面，需要加额外的选项，kubeadm初始化时，需要增加选项 --experiment-upload-certs，所以集群创建好之后没法再来修改。

在摸索过程中又发现该集群突然无法用（详细现象记录在文章末尾），原因是部署时生成的证书有效期是1年，虽然网上有说kubeadm可以生成证书，尝试过几次，发现有些证书无法生成，并且发现kubeadm的不同版本，支持的命令不太一样。 幸运的是证书到期之后，虽然kubectl的命令无法正常输出，但是worknode节点上pod仍然可以用，所以上面的jenkins服务可以勉强继续提供服务，猜测一旦重启该worknode ，上面的pod应该就起不了了。

为了不影响业务，所以决定再搭建一套k8s集群，然后采用多个master的部署方式，相关步骤整理如下：



# k8s 高可用集群搭建过程

OS为Ubunt16.04，aliyun的源中有k8s的版本 1.18.2，准备直接用新的版本来测试

整个过程概括如下：

- 安装基础环境

- 为api-server 规划vip

- docker 的registry image 要更新为 国内的ali 

- kubeadm 的配置 修改 其中的 registry

详细步骤整理如下



## 基础环境安装

```
配置 源
# kubeadm及kubernetes组件安装源
deb https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main


安装 kube组件和docker
sudo apt-get install kubeadm docker.io 


```



## api-server vip规划

所有的请求都要经过api-server，当有多个 master节点时，为了给请求提供一个稳定的URL，就需要给api-server创建一个负载均衡器（SLB），如果是云环境，可以用云厂商提供的，lab环境中可以通过（keepalived+nigix）组合，实现一个类似的SLB（keepalived提供一个稳定的VIP，niginx负责把请求转发给后端的多个api-server）。本次测试过程中，为了进一步简化，省去了nginx的配置，只用了keepalived，然后kubeadm 初始化配置中，api-server的端口就和本地的端口相同，用的都是6443.（如果用nginx做请求转发的话，就需要用一个 新的端口，让nginx监听该端口，然后将请求转发到 locakAPI端口上。）

```
localAPIEndpoint:
  advertiseAddress: 172.17.73.66
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8smater1
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "172.17.73.63:6443"

```

> 注意：这种部署，因为缺少了 SLB对后端服务的健康检查能力，所以有可能 vip所在节点的 api-server挂了，导致请求无法相应。 虽然 keepalived自身也是可以通过脚本做到对后端服务健康检查的能力，但是这里偷懒，没有做。keepalive的完整配置文件详见文章末尾



## kubeadm配置文件修改

主要修改如下几处：

修改localAPIEndpoint，设置为该节点的ip

增加controlPlaneEndpoint: "172.17.73.63:6443" （73.63是我keepalive中设置的vip）

修改imageRepository: registry.aliyuncs.com/google_containers，采用阿里云的image，pull速度快

增加一个 ipvs的配置

```
kubeadm config print init-defaults > kubeadm-config.yaml

完整的配置文件如下：
wsg@k8smater1:~/kube_config$ cat kubeadm-config.yaml 
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 172.17.73.66
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8smater1
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "172.17.73.63:6443"
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.18.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
wsg@k8smater1:~/kube_config$ 

```



## 初始化第一个master节点

```
sudo kubeadm init --config=kubeadm-config.yaml --upload-certs |tee kubeadm-init.log

```

需要 增加 --upload-certs，该参数的说明如下：

集群初始化有个 --upload-certs选项，控制证书的分发，

The `--upload-certs` flag is used to upload the certificates that should be shared across all the control-plane instances to the cluster. If instead, you prefer to copy certs across control-plane nodes manually or using automation tools, please remove this flag and refer to [Manual certificate distribution](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#manual-certs) section below

也可以手动管理证书。

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#manual-certs

需要考虑的是，master节点的api-server 需要有负载均衡的ip，需要单独部署负载均衡组件（haproxy和keepalive），集群初始化时需要指定 该VIP，

可以参考

https://blog.csdn.net/networken/article/details/89599004



## 增加第二个master节点

```
wsg@k8smaster2:~$ sudo vim /etc/fstab 
wsg@k8smaster2:~$ sudo  kubeadm join 172.17.73.63:6443 --token abcdef.0123456789abcdef \
>     --discovery-token-ca-cert-hash sha256:96e89f70a308ad8a236703f0618d8c2c361c54088205b74adf406388a3c927b8 \
>     --control-plane --certificate-key 3824b2babc6e9ffe032c87b9575943d1515dc67021b863a454c0787c44425643
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks before initializing the new control plane instance
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[download-certs] Downloading the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8smaster2 localhost] and IPs [172.17.73.65 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8smaster2 localhost] and IPs [172.17.73.65 127.0.0.1 ::1]
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8smaster2 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 172.17.73.65 172.17.73.63]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[certs] Using the existing "sa" key
[kubeconfig] Generating kubeconfig files
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
W0423 15:15:50.918094    2969 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
W0423 15:15:50.929267    2969 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
W0423 15:15:50.931720    2969 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[check-etcd] Checking that the etcd cluster is healthy
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.18" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
[etcd] Announced new etcd member joining to the existing etcd cluster
[etcd] Creating static Pod manifest for "etcd"
[etcd] Waiting for the new etcd member to join the cluster. This can take up to 40s
{"level":"warn","ts":"2020-04-23T15:16:04.676+0800","caller":"clientv3/retry_interceptor.go:61","msg":"retrying of unary invoker failed","target":"passthrough:///https://172.17.73.65:2379","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = context deadline exceeded"}
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[mark-control-plane] Marking the node k8smaster2 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node k8smaster2 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]

This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Control plane (master) label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.
* A new etcd member was added to the local/stacked etcd cluster.

To start administering your cluster from this node, you need to run the following as a regular user:

	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.

wsg@k8smaster2:~$ mkdir -p $HOME/.kube
wsg@k8smaster2:~$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
wsg@k8smaster2:~$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
wsg@k8smaster2:~$ kubectl get nodes
NAME         STATUS     ROLES    AGE     VERSION
k8smaster2   NotReady   master   89s     v1.18.2
k8smater1    NotReady   master   4m18s   v1.18.2
wsg@k8smaster2:~$ 

```



## 增加网络插件

网络插件会单独写文章来整理，这里选用 功能和性能都兼具的calico网络（非overlay网络，少掉一层包的封装与解封）

```
wget https://docs.projectcalico.org/v3.9/manifests/calico.yaml
```

修改默认calico中的网段，默认是192.168的断，修改成和 kubeadm中一致的。

```
            # The default IPv4 pool to create on startup if none exists. Pod IPs will be
            # chosen from this range. Changing this value after installation will have
            # no effect. This should fall within `--cluster-cidr`.
            - name: CALICO_IPV4POOL_CIDR
              value: "10.96.0.0/12"

```



## 检查搭建完的效果

节点情况

```
wsg@k8smater1:~$ kubectl get nodes
NAME         STATUS   ROLES    AGE   VERSION
k8smaster2   Ready    master   19h   v1.18.2
k8smater1    Ready    master   19h   v1.18.2

```



pod情况

```
wsg@k8smaster2:~$ kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE    IP               NODE         NOMINATED NODE   READINESS GATES
kube-system   calico-kube-controllers-5fc5dbfc47-4wkpw   1/1     Running   0          152m   10.109.169.2     k8smater1    <none>           <none>
kube-system   calico-node-96bpx                          1/1     Running   0          152m   172.17.73.65     k8smaster2   <none>           <none>
kube-system   calico-node-zhjt9                          1/1     Running   0          152m   172.17.73.66     k8smater1    <none>           <none>
kube-system   coredns-7ff77c879f-brsgl                   1/1     Running   0          3h1m   10.109.169.1     k8smater1    <none>           <none>
kube-system   coredns-7ff77c879f-kl8wl                   1/1     Running   0          3h1m   10.109.209.193   k8smaster2   <none>           <none>
kube-system   etcd-k8smaster2                            1/1     Running   0          178m   172.17.73.65     k8smaster2   <none>           <none>
kube-system   etcd-k8smater1                             1/1     Running   0          3h1m   172.17.73.66     k8smater1    <none>           <none>
kube-system   kube-apiserver-k8smaster2                  1/1     Running   0          178m   172.17.73.65     k8smaster2   <none>           <none>
kube-system   kube-apiserver-k8smater1                   1/1     Running   0          3h1m   172.17.73.66     k8smater1    <none>           <none>
kube-system   kube-controller-manager-k8smaster2         1/1     Running   7          178m   172.17.73.65     k8smaster2   <none>           <none>
kube-system   kube-controller-manager-k8smater1          1/1     Running   7          3h1m   172.17.73.66     k8smater1    <none>           <none>
kube-system   kube-proxy-74mvh                           1/1     Running   1          178m   172.17.73.65     k8smaster2   <none>           <none>
kube-system   kube-proxy-gxgc9                           1/1     Running   1          3h1m   172.17.73.66     k8smater1    <none>           <none>
kube-system   kube-scheduler-k8smaster2                  1/1     Running   5          178m   172.17.73.65     k8smaster2   <none>           <none>
kube-system   kube-scheduler-k8smater1                   1/1     Running   7          3h1m   172.17.73.66     k8smater1    <none>           <none>

```



## 测试 高可用情况

2节点的master 集群，准备将其中的一个节点keepalived服务停掉，这样vip（172.17.73.63）就会漂移到另外一个节点，最终请求就会落到第二个master的 api-server

测试过程如下，在vip漂移期间，请求会会有短暂的超时，最终可以正常。（这也是前面预计到的情况）

```
wsg@k8smater1:~$ kubectl get nodes
NAME         STATUS   ROLES    AGE   VERSION
k8smaster2   Ready    master   19h   v1.18.2
k8smater1    Ready    master   19h   v1.18.2

wsg@k8smater1:~$ /etc/init.d/keepalived stop 
[....] Stopping keepalived (via systemctl): keepalived.service==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ===
Authentication is required to stop 'keepalived.service'.
Authenticating as: wsg,,, (wsg)
Password: 
==== AUTHENTICATION COMPLETE ===
. ok 

wsg@k8smater1:~$ kubectl get nodes
Unable to connect to the server: dial tcp 172.17.73.63:6443: i/o timeout
wsg@k8smater1:~$ kubectl get nodes
NAME         STATUS   ROLES    AGE   VERSION
k8smaster2   Ready    master   19h   v1.18.2
k8smater1    Ready    master   19h   v1.18.2
wsg@k8smater1:~$ 

```

# 知识点记录

## 遗留待解决？？

如何验证集群证书会自动 生成，防止证书过期的情况再次发生？

检查证书文件的生成日期？ 





## nfs client包如果是在 k8s 集群运行之后状态，需要重启才能生效

否则 观察pod时，会提示 mount nfs 挂载不了，虽然此时 host主机可以正常mount nfs。 重启主机就正常了。



## master 节点运行运行 pod

```
wsg@k8smater1:~/kube_deploy/jenkins$ kubectl taint nodes --all node-role.kubernetes.io/master-
node/k8smaster2 untainted
node/k8smater1 untainted



```





## v1.14.0和v1.18.2的api接口的不兼容

- v.14下面yaml可以正常部署，但是到v.1.18 报错如下

  ```
  wsg@k8smater1:~/kube_deploy/jenkins$ kubectl apply -f jenkins_nodePort.yml 
  clusterrolebinding.rbac.authorization.k8s.io/jenkins created
  unable to recognize "jenkins_nodePort.yml": no matches for kind "Deployment" in version "extensions/v1beta1"
  
  extensions/v1beta1  更改为apps/v1
  ```

  https://kubernetes.io/blog/2019/07/18/api-deprecations-in-1-16/

  

- 解决上面的api后，接着

  ```
  wsg@k8smater1:~/kube_deploy/jenkins$ kubectl apply -f jenkins_nodePort.yml 
  clusterrolebinding.rbac.authorization.k8s.io/jenkins unchanged
  error validating "jenkins_nodePort.yml": error validating data: ValidationError(Deployment.spec): missing required field "selector" in io.k8s.api.apps.v1.DeploymentSpec; if you choose to ignore these errors, turn validation off with --validate=false
  
  
  ```

  官方解释

  `spec.selector` is now required and immutable after creation; use the existing template labels as the selector for seamless upgrades

  





## master机制探索 

2个 节点的组成的master集群，如果down掉一个 节点，集群就不可用了。

看起来etcd的pod 因为RAFT机制，导致不可用，所以 应该至少3个节点组成master 集群，才能 抵挡 down节点的情况。etcd pod的log如下：

```
raft2020/04/24 03:15:01 INFO: 57020b4a848c0755 is starting a new election at term 463
raft2020/04/24 03:15:01 INFO: 57020b4a848c0755 became candidate at term 464
raft2020/04/24 03:15:01 INFO: 57020b4a848c0755 received MsgVoteResp from 57020b4a848c0755 at term 464
raft2020/04/24 03:15:01 INFO: 57020b4a848c0755 [logterm: 84, index: 210308] sent MsgVote request to 5c5054334836fd1a at term 464
raft2020/04/24 03:15:03 INFO: 57020b4a848c0755 is starting a new election at term 464
raft2020/04/24 03:15:03 INFO: 57020b4a848c0755 became candidate at term 465
raft2020/04/24 03:15:03 INFO: 57020b4a848c0755 received MsgVoteResp from 57020b4a848c0755 at term 465
raft2020/04/24 03:15:03 INFO: 57020b4a848c0755 [logterm: 84, index: 210308] sent MsgVote request to 5c5054334836fd1a at term 465
raft2020/04/24 03:15:04 INFO: 57020b4a848c0755 is starting a new election at term 465
raft2020/04/24 03:15:04 INFO: 57020b4a848c0755 became candidate at term 466
raft2020/04/24 03:15:04 INFO: 57020b4a848c0755 received MsgVoteResp from 57020b4a848c0755 at term 466
raft2020/04/24 03:15:04 INFO: 57020b4a848c0755 [logterm: 84, index: 210308] sent MsgVote request to 5c5054334836fd1a at term 466
2020-04-24 03:15:05.257901 E | etcdserver: publish error: etcdserver: request timed out
2020-04-24 03:15:05.258244 W | rafthttp: health check for peer 5c5054334836fd1a could not connect: dial tcp 172.17.73.66:2380: connect: no route to host
2020-04-24 03:15:05.258280 W | rafthttp: health check for peer 5c5054334836fd1a could not connect: dial tcp 172.17.73.66:2380: connect: no route to host
wsg@k8smaster2:~$ 

```

down掉的节点再次启动，可以自动恢复。



## master节点查看token信息

```
wsg@ubuntu16:~/helm/jenkins/aliyun$ kubeadm token list
TOKEN     TTL       EXPIRES   USAGES    DESCRIPTION   EXTRA GROUPS
wsg@ubuntu16:~/helm/jenkins/aliyun$ 


token是有时效的，上面命令看出，当前没有有效的token，所以手动生成一个token
wsg@ubuntu16:~/helm/jenkins/aliyun$ kubeadm token create --print-join-command
kubeadm join 172.17.73.68:6443 --token xwgmrz.a3bod9lk35ai8krx     --discovery-token-ca-cert-hash sha256:682c5cb5f0a0b3134ccdb66f7bdd03a6c2cb16c2e3cba3b10d9bf7577eee2e04 
wsg@ubuntu16:~/helm/jenkins/aliyun$ 


```



## 可以查看kubeadm的默认初始化参数

```
kubeadm config print init-defaults > kubeadm-config.yaml

```

可以通过上面导出的配置文件修改后，初始化时 指定该配置文件，达到定制化的目的。

##master节点的证书

```
wsg@ubuntu16:/etc/kubernetes/pki$ ll
total 68
drwxr-xr-x 3 root root 4096 Mar 29  2019 ./
drwxr-xr-x 4 root root 4096 Mar 30  2019 ../
-rw-r--r-- 1 root root 1220 Mar 29  2019 apiserver.crt
-rw-r--r-- 1 root root 1090 Mar 29  2019 apiserver-etcd-client.crt
-rw------- 1 root root 1679 Mar 29  2019 apiserver-etcd-client.key
-rw------- 1 root root 1675 Mar 29  2019 apiserver.key
-rw-r--r-- 1 root root 1099 Mar 29  2019 apiserver-kubelet-client.crt
-rw------- 1 root root 1679 Mar 29  2019 apiserver-kubelet-client.key
-rw-r--r-- 1 root root 1025 Mar 29  2019 ca.crt
-rw------- 1 root root 1675 Mar 29  2019 ca.key
drwxr-xr-x 2 root root 4096 Mar 29  2019 etcd/
-rw-r--r-- 1 root root 1038 Mar 29  2019 front-proxy-ca.crt
-rw------- 1 root root 1679 Mar 29  2019 front-proxy-ca.key
-rw-r--r-- 1 root root 1058 Mar 29  2019 front-proxy-client.crt
-rw------- 1 root root 1675 Mar 29  2019 front-proxy-client.key
-rw------- 1 root root 1679 Mar 29  2019 sa.key
-rw------- 1 root root  451 Mar 29  2019 sa.pub


```

里面 etcd 的证书

```
wsg@ubuntu16:/etc/kubernetes/pki$ ll etcd/
total 40
drwxr-xr-x 2 root root 4096 Mar 29  2019 ./
drwxr-xr-x 3 root root 4096 Mar 29  2019 ../
-rw-r--r-- 1 root root 1017 Mar 29  2019 ca.crt
-rw------- 1 root root 1679 Mar 29  2019 ca.key
-rw-r--r-- 1 root root 1094 Mar 29  2019 healthcheck-client.crt
-rw------- 1 root root 1675 Mar 29  2019 healthcheck-client.key
-rw-r--r-- 1 root root 1131 Mar 29  2019 peer.crt
-rw------- 1 root root 1675 Mar 29  2019 peer.key
-rw-r--r-- 1 root root 1131 Mar 29  2019 server.crt
-rw------- 1 root root 1679 Mar 29  2019 server.key
wsg@ubuntu16:/etc/kubernetes/pki$ 

```











# 测试过程其他相关记录

其他相关记录整理如下



## keepalived的完整配置

```
wsg@k8smater1:~$ cat /etc/keepalived/keepalived.conf 
global_defs {
   notification_email {
     root@mydomain.com
   }
   notification_email_from svr2@mydomain.com
   smtp_server localhost
   smtp_connect_timeout 30
}

vrrp_instance VRRP1 {
    state BACKUP
#   Specify the network interface to which the virtual address is assigned
    interface ens18
    virtual_router_id 86
#   Set the value of priority lower on the backup server than on the master server
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1066
    }
    virtual_ipaddress {
        172.17.73.63/22
    }
}

```



# kubeadm 1.18 init

## kubeadm 配置文件

```

controlPlaneEndpoint:

ipvs

imageRepository: registry.aliyuncs.com/google_containers



```





## 成功

需要 注意 api server vip的情况，

```
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
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

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 172.17.73.63:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:165cfb5b7b10db3a4e4b53b4963cf6a5c0239b1037bd9bbaacde86d3680f285c \
    --control-plane --certificate-key d53cf0283af9958d0bff03dd688c5e37769f7872ceb28f4eabc3d207bc472179

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.17.73.63:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:165cfb5b7b10db3a4e4b53b4963cf6a5c0239b1037bd9bbaacde86d3680f285c 
wsg@k8sMasterNode1:~/kube_config$ 


```







## 第一次报错

```
[certs] etcd/server serving cert is signed for DNS names [ubuntu16wsg localhost] and IPs [1.2.3.4 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [ubuntu16wsg localhost] and IPs [1.2.3.4 127.0.0.1 ::1]
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
W0423 12:34:42.271026    8515 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
W0423 12:34:42.272733    8515 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[kubelet-check] Initial timeout of 40s passed.

	Unfortunately, an error has occurred:
		timed out waiting for the condition

	This error is likely caused by:
		- The kubelet is not running
		- The kubelet is unhealthy due to a misconfiguration of the node in some way (required cgroups disabled)

	If you are on a systemd-powered system, you can try to troubleshoot the error with the following commands:
		- 'systemctl status kubelet'
		- 'journalctl -xeu kubelet'

	Additionally, a control plane component may have crashed or exited when started by the container runtime.
	To troubleshoot, list all containers using your preferred container runtimes CLI.

	Here is one example how you may list all Kubernetes containers running in docker:
		- 'docker ps -a | grep kube | grep -v pause'
		Once you have found the failing container, you can inspect its logs with:
		- 'docker logs CONTAINERID'

error execution phase wait-control-plane: couldn't initialize a Kubernetes cluster
To see the stack trace of this error execute with --v=5 or higher
wsg@ubuntu16wsg:~/kube_config$ 


```







# kubeadm 部署的单 master 集群 cert 过期



证书到期之后，etcd的pod 也会无法启动。所以api-server 也无法启动

## etcd pod log

```
2020-04-12 01:27:44.412402 I | raft: cd3d85ed5e2c6ebc received MsgVoteResp from cd3d85ed5e2c6ebc at term 5144
2020-04-12 01:27:44.412426 I | raft: cd3d85ed5e2c6ebc became leader at term 5144
2020-04-12 01:27:44.412436 I | raft: raft.node: cd3d85ed5e2c6ebc elected leader cd3d85ed5e2c6ebc at term 5144
2020-04-12 01:27:44.412736 I | embed: ready to serve client requests
2020-04-12 01:27:44.413142 I | etcdserver: published {Name:ubuntu16 ClientURLs:[https://172.17.73.68:2379]} to cluster 225495ed2e66c256
2020-04-12 01:27:44.413177 I | embed: ready to serve client requests
2020-04-12 01:27:44.414926 I | embed: serving client requests on 172.17.73.68:2379
2020-04-12 01:27:44.416500 I | embed: serving client requests on 127.0.0.1:2379
2020-04-12 01:27:44.428699 I | embed: rejected connection from "172.17.73.68:45158" (error "tls: failed to verify client's certificate: x509: certificate has expired or is not yet valid", ServerName "")
WARNING: 2020/04/12 01:27:44 Failed to dial 172.17.73.68:2379: connection error: desc = "transport: authentication handshake failed: remote error: tls: bad certificate"; please retry.
2020-04-12 01:27:44.431414 I | embed: rejected connection from "127.0.0.1:46146" (error "tls: failed to verify client's certificate: x509: certificate has expired or is not yet valid", ServerName "")
WARNING: 2020/04/12 01:27:44 Failed to dial 127.0.0.1:2379: connection error: desc = "transport: authentication handshake failed: remote error: tls: bad certificate"; please retry.
2020-04-12 01:28:01.070372 I | embed: rejected connection from "127.0.0.1:46930" (error "remote error: tls: bad certificate", ServerName "")
2020-04-12 01:28:11.075566 I | embed: rejected connection from "127.0.0.1:47390" (error "remote error: tls: bad certificate", ServerName "")
2020-04-12 01:28:21.070949 I | embed: rejected connection from "127.0.0.1:47850" (error "remote error: tls: bad certificate", ServerName "")
2020-04-12 01:28:31.063674 I | embed: rejected connection from "127.0.0.1:48310" (error "remote error: tls: bad certificate", ServerName "")
2020-04-12 01:28:41.073461 I | embed: rejected connection from "127.0.0.1:48770" (error "remote error: tls: bad certificate", ServerName "")
2020-04-12 01:28:51.070447 I | embed: rejected connection from "127.0.0.1:49230" (error "remote error: tls: bad certificate", ServerName "")
2020-04-12 01:29:01.059663 I | embed: rejected connection from "127.0.0.1:49684" (error "remote error: tls: bad certificate", ServerName "")
2020-04-12 01:29:11.080056 I | embed: rejected connection from "127.0.0.1:50162" (error "remote error: tls: bad certificate", ServerName "")
2020-04-12 01:29:13.128725 N | pkg/osutil: received terminated signal, shutting down...
2020-04-12 01:29:13.128896 I | etcdserver: skipped leadership transfer for single member cluster
wsg@ubuntu16:~$ 
wsg@ubuntu16:~$ 


```



etcd 启动的过程记录

```
看到 会有证书的加载信息。
2020-04-12 01:27:43.520341 I | embed: peerTLS: cert = /etc/kubernetes/pki/etcd/peer.crt, key = /etc/kubernetes/pki/etcd/peer.key, ca = , trusted-ca = /etc/kubernetes/pki/etcd/ca.crt, client-cert-auth = true, crl-file = 
2020-04-12 01:27:43.523862 I | embed: listening for peers on https://172.17.73.68:2380
2020-04-12 01:27:43.523946 I | embed: listening for client requests on 127.0.0.1:2379
2020-04-12 01:27:43.523992 I | embed: listening for client requests on 172.17.73.68:2379

2020-04-12 01:27:43.556380 I | etcdserver: advertise client URLs = https://172.17.73.68:2379

2020-04-12 01:27:44.412736 I | embed: ready to serve client requests
2020-04-12 01:27:44.413142 I | etcdserver: published {Name:ubuntu16 ClientURLs:[https://172.17.73.68:2379]} to cluster 225495ed2e66c256
2020-04-12 01:27:44.413177 I | embed: ready to serve client requests
2020-04-12 01:27:44.414926 I | embed: serving client requests on 172.17.73.68:2379
2020-04-12 01:27:44.416500 I | embed: serving client requests on 127.0.0.1:2379


进一步
```

