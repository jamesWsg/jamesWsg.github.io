---
layout: post
title: k8s管理平台rancher实践
category: 技术
---

# 部署前规划

整个部署包括2个部分，一是管理集群部署，二是k8s集群部署。管理集群功能主要提供web界面方式管理k8s集群。正常情况，管理集群3个节点即可，k8s集群至少3个。本文以3节点管理集群，3节点k8s集群为例 说明部署过程

管理集群需要通过域名的方式访问，需要在访问客户端添加域名解析，示例配置的域名以及 节点IP规划如下：

管理集群访问域名:rancher.bigtera.com

| 节点功能  | 节点hostname | 节点管理IP    | 节点存储ip（访问ceph用） |
| --------- | ------------ | ------------- | ------------------------ |
| 管理节点1 | rancher1     | 172.17.73.161 |                          |
| 管理节点2 | rancher2     | 172.17.73.162 |                          |
| 管理节点3 | rancher3     | 172.17.73.163 |                          |

k8s集群可能需要对外开放 API调用，需要保证API server的高可用，所以需要给API server配置VIP，示例配置的VIP以及 节点IP规划如下：

k8s集群 api_server vip:172.17.73.154

| 节点功能 | 节点hostname | 节点管理IP    | 节点存储ip（访问ceph用） |
| -------- | ------------ | ------------- | ------------------------ |
| k8s节点1 | k8s1         | 172.17.73.151 | 10.10.101.151            |
| k8s节点2 | k8s2         | 172.17.73.152 | 10.10.101.152            |
| k8s节点3 | k8s3         | 172.17.73.153 | 10.10.101.153            |



除了上面的配置规划，需要考虑平台内部可能会提供容器镜像等其他服务，需要为平台内部的服务提供访问 入口，一般情况下，至少规划预留至少一个 IP，本文以 172.17.73.158，172.17.73.159 为例

有了上面的规划，就可以开始动手部署了。

# 部署步骤



##  1，管理集群部署（rancher）

### 克隆管理集群vm模板

根据实际情况，选择clone数目，一般3节点的集群，clone3个vm

### 配置vm

vm内置磁盘有2块，一块是作为OS，另一块存放一些应用数据。网卡也有2块，一块用作集群管理ip，一块用作连接外部 ceph存储，管理集群一般不需要连接ceph存储，可以只配置一个管理ip

vm内置账号 bigtera/1 ，登陆后可以看到默认挂载了 /rancher_deploy，里面有提前下载好的部署软件包。

```
bigtera@rancher1:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            2.0G     0  2.0G   0% /dev
tmpfs           396M  5.7M  390M   2% /run
/dev/sda1        98G  9.2G   84G  10% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/sdb1       493G  144M  467G   1% /rancher_deploy
tmpfs           396M     0  396M   0% /run/user/1001
bigtera@rancher1:~$ ll /rancher_deploy/
total 37016
drwxrwxrwx  5 root    root        4096 Nov 30 18:11 ./
drwxr-xr-x 24 root    root        4096 Nov 30 14:08 ../
-rwxr-xr-x  1 bigtera bigtera 37871616 Nov 30 15:41 helm*
drwxrwxr-x  2 bigtera bigtera     4096 Nov 30 18:48 helm_package/
drwx------  2 bigtera bigtera    16384 Nov 30 14:09 lost+found/
drwxrwxr-x  3 bigtera bigtera     4096 Dec  3 15:02 rke1.2.1/
bigtera@rancher1:~$ 
```

配置主要包括 ，配置vm hostname和ip

配置 hostname

```
hostnamectl set-hostname rancher1
```

配置vm ip

```
bigtera@rancher1:/rancher_deploy/rke1.2.1$ cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto ens18 ens19
iface ens18 inet static
    address 172.17.73.161
    netmask 255.255.252.0
    gateway 172.17.75.254
    #dns-nameservers 114.114.114.114
iface ens19 inet static
    address 10.10.101.161
    netmask 255.255.255.0

```





### 通过rke创建集群

所有管理集群的节点 配置完成后，在 节点1上开始 创建管理集群，因为创建过程会和其他管理节点通信，需要配置好 节点1 到 其他节点的免密登陆

- 配置免密登陆

  因为vm都是从同一个 vm clone出来的，.ssh/authorized_keys 中已经增加了 证书，只是第一次登陆需要输入yes 确认，如下

  ```
  bigtera@rancher1:/rancher_deploy/rke1.2.1$ ssh 172.17.73.163
  The authenticity of host '172.17.73.163 (172.17.73.163)' can't be established.
  ECDSA key fingerprint is SHA256:gp8xiVd/q4Qrfqj7Ie/lk5q3V3mnmMLYNIfFi2frI8I.
  Are you sure you want to continue connecting (yes/no)? yes
  ```

  依上面方法，登陆所有的节点（161 到163 节点）

  验证： 做完后，ssh 登陆3个节点，是否可以 免密登陆

- 创建集群

  完成免密配置后，生成集群的配置文件，如下

  ```
  bigtera@rancher1:/rancher_deploy/rke1.2.1$ cat cluster.yml 
  nodes:
    - address: 172.17.73.161
      user: bigtera
      role: ['controlplane', 'etcd', 'worker']
    - address: 172.17.73.162
      user: bigtera
      role: ['controlplane', 'etcd', 'worker']
    - address: 172.17.73.163
      user: bigtera
      role: ['controlplane', 'etcd', 'worker']
  bigtera@rancher1:/rancher_deploy/rke1.2.1$ 
  
  ```

  生成配置文件后，执行如下命令

  ```
  ./rke_linux-amd64 up --config cluster.yml
  
  执行完毕后，会看到命令 最后成功的输出如下
  INFO[0116] [ingress] ingress controller nginx deployed successfully 
  INFO[0116] [addons] Setting up user addons              
  INFO[0116] [addons] no user addons defined              
  INFO[0116] Finished building Kubernetes cluster successfully 
  bigtera@rancher1:/rancher_deploy/rke1.2.1$ 
  
  ```

  创建成功后，会在 目录下生成 如下管理集群的配置文件

  ```
  bigtera@rancher1:/rancher_deploy/rke1.2.1$ ls -l kube_config_cluster.yml 
  -rw-r----- 1 bigtera bigtera 5388 Dec  4 19:25 kube_config_cluster.yml
  bigtera@rancher1:/rancher_deploy/rke1.2.1$ 
  
  把该配置文件scp 到其他2个 节点（162和163）
  scp kube_config_cluster.yml 172.17.73.162:/rancher_deploy/rke1.2.1/
  
  scp kube_config_cluster.yml 172.17.73.163:/rancher_deploy/rke1.2.1/
  ```

  

  验证管理集群部署是否正常，可以看到集群中 3个节点的信息 显示 都是 ready状态

  ```
  bigtera@rancher1:/rancher_deploy/rke1.2.1$ kubectl get nodes
  NAME            STATUS   ROLES                      AGE     VERSION
  172.17.73.161   Ready    controlplane,etcd,worker   3m20s   v1.19.3
  172.17.73.162   Ready    controlplane,etcd,worker   3m20s   v1.19.3
  172.17.73.163   Ready    controlplane,etcd,worker   3m20s   v1.19.3
  bigtera@rancher1:/rancher_deploy/rke1.2.1$ 
  ```

  

### 安装rancher

按照上面步骤完成管理集群创建和配置后，就可以通过helm方式安装 rancher集群，已经预装了helm。

具体步骤如下：

rancher依赖cert-manger（用来管理自身的证书签发）

* install cert-manager

  ```
  
  kubectl create ns cert-manager
  
  bigtera@rancher1:/rancher_deploy/helm_package$ helm install   cert-manager ./cert-manager-v1.0.3.tgz   --namespace cert-manager    --set installCRDs=true
  
  
  
  
  ```

  

* install traefik

  ```
  bigtera@rancher1:/rancher_deploy/helm_package$ helm install traefik ./traefik-9.11.0.tgz
  
  ```
  

  
* install rancher

  其中 指定了文档开始 规划的 管理集群的域名 rancher.bigtera.com ，该域名会通过traefik模块 被解析到管理集群的每个节点.(即被解析到 172.17.73.161，172.17.73.162，172.17.73.163)

  ```
  kubectl create ns cattle-system
  
  bigtera@rancher1:/rancher_deploy/helm_package$ helm install rancher ./rancher-2.5.2.tgz --namespace cattle-system --set hostname=rancher.bigtera.com
  
  ```

  安装完成后，可以验证该域名的解析情况

  ```
  bigtera@rancher1:/rancher_deploy/helm_package$ kubectl get ingress --all-namespaces
  Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
  NAMESPACE       NAME      CLASS    HOSTS                 ADDRESS                                     PORTS     AGE
  cattle-system   rancher   <none>   rancher.bigtera.com   172.17.73.161,172.17.73.162,172.17.73.163   80, 443   6m1s
  bigtera@rancher1:/rancher_deploy/helm_package$ 
  
  ```

  此时，在客户端添加上面的 域名解析到 hosts配置文件，就可以访问管理集群了，以win10 为例

  添加如下到 C:\WINDOWS\system32\drivers\etc\hosts

  ```
  172.17.73.161 rancher.bigtera.com
  172.17.73.162 rancher.bigtera.com
  172.17.73.163 rancher.bigtera.com
  加入多个条目，目的是 当有管理节点down时，可以解析到其他正常节点。
  ```

  

  配置完成后，chrome 访问域名，如下，（第一次登陆后需要设置密码）

  ![image-20201204142047161](../img/容器管理平台/rancher-initial.png)

  

  ![image-20201204142251526](../img/容器管理平台/rancher-initial2.png)

  初始化完成后，效果如下：

  ![image-20201204142402457](../img/容器管理平台/rancher-initial3.png)





### 可能的问题处理



- 部署后，检查 pod 状态，发现metirc server pod 的状态不对

  会显示如下状态，

  ```
  bigtera@rancher1:/rancher_deploy/helm_package/rancher$ kubectl get pod --all-namespaces
  NAMESPACE       NAME                                       READY   STATUS             RESTARTS   AGE
  kube-system     coredns-autoscaler-79599b9dc6-pwm52        1/1     Running            0          147m
  kube-system     metrics-server-8449844bf-kpv4d             0/1     ImagePullBackOff   0          147m
  kube-system     rke-coredns-addon-deploy-job-qgvh6         0/1     Completed          0          147m
  kube-system     rke-ingress-controller-deploy-job-ptvc9    0/1     Completed          0          146m
  kube-system     rke-metrics-addon-deploy-job-n4z66         0/1     Completed          0          147m
  kube-system     rke-network-plugin-deploy-job-lm72k        0/1     Completed          0          147
  ```

  实际上 image已经 pull 下来，需要修改 对应deployment中的配置文件

  将下面的配置

  ```
  bigtera@rancher1:/rancher_deploy/helm_package/rancher$ kubectl edit deploy metrics-server  -n kube-system
  通过上面命令修改  
   51       containers:
   52       - command:
   53         - /metrics-server
   54         - --kubelet-insecure-tls
   55         - --kubelet-preferred-address-types=InternalIP
   56         - --logtostderr
   57         image: rancher/metrics-server:v0.3.6
   58         imagePullPolicy: Always
   59         name: metrics-server
   60         resources: {}
  
  ```

  修改为

  ```
   51       containers:
   52       - command:
   53         - /metrics-server
   54         - --kubelet-insecure-tls
   55         - --kubelet-preferred-address-types=InternalIP
   56         - --logtostderr
   57         image: rancher/metrics-server:v0.3.6
   58         imagePullPolicy: IfNotPresent
   59         name: metrics-server
   60         resources: {}
   61         terminationMessagePath: /dev/termination-log
   62         terminationMessagePolicy: File
   63       dnsPolicy: ClusterFirst
  
  ```

  修改完成后，再次检查 所有pod的状态，都正常了

  ```
  bigtera@rancher1:/rancher_deploy/helm_package/rancher$ kubectl get pod --all-namespaces
  NAMESPACE       NAME                                       READY   STATUS      RESTARTS   AGE
  cert-manager    cert-manager-556549df9-nxnn8               1/1     Running     0          127m
  cert-manager    cert-manager-cainjector-69d7cb5d4-4k5nv    1/1     Running     0          127m
  cert-manager    cert-manager-webhook-c5bdf945c-c6tpj       1/1     Running     0          127m
  default         traefik-77fdb5c487-42xqx                   1/1     Running     0          127m
  ingress-nginx   default-http-backend-65dd5949d9-wgsql      1/1     Running     0          162m
  ingress-nginx   nginx-ingress-controller-8f4jb             1/1     Running     0          162m
  ingress-nginx   nginx-ingress-controller-hnrrk             1/1     Running     0          162m
  ingress-nginx   nginx-ingress-controller-xlp44             1/1     Running     0          162m
  kube-system     calico-kube-controllers-649b7b795b-7zsbd   1/1     Running     0          163m
  kube-system     canal-jx5t2                                2/2     Running     0          163m
  kube-system     canal-vqbgw                                2/2     Running     0          163m
  kube-system     canal-wtk67                                2/2     Running     0          163m
  kube-system     coredns-6f85d5fb88-6g6v4                   1/1     Running     0          162m
  kube-system     coredns-6f85d5fb88-hfjkq                   1/1     Running     0          162m
  kube-system     coredns-autoscaler-79599b9dc6-pwm52        1/1     Running     0          162m
  kube-system     metrics-server-56f9f865f-q4rmf             1/1     Running     0          108s
  kube-system     rke-coredns-addon-deploy-job-qgvh6         0/1     Completed   0          163m
  kube-system     rke-ingress-controller-deploy-job-ptvc9    0/1     Completed   0          162m
  kube-system     rke-metrics-addon-deploy-job-n4z66         0/1     Completed   0          162m
  kube-system     rke-network-plugin-deploy-job-lm72k        0/1     Completed   0          163m
  bigtera@rancher1:/rancher_deploy/helm_package/rancher$ 
  
  ```

  



## 2，k8s集群部署



### 克隆k8s-vm模板

根据实际情况，选择clone数目，一般3节点的集群，clone3个vm

### 配置vm

vm内置磁盘有2块，一块是作为OS，另一块存放一些应用数据。网卡也有2块，一块用作集群管理ip，一块用作连接外部 ceph存储，管理集群一般不需要连接ceph存储，可以只配置一个管理ip

vm内置账号 bigtera/1 ，登陆后可以看到默认挂载了/k8s_deploy，里面有提前下载好的部署软件包。

```
bigtera@k8s1:/k8s_deploy$ ll
total 52
drwxrwxrwx  9 root    root     4096 Dec  2 18:08 ./
drwxr-xr-x 24 root    root     4096 Nov 30 14:10 ../
drwxrwxr-x  3 bigtera bigtera  4096 Dec  2 14:42 deploy/
drwxrwxr-x  6 bigtera bigtera  4096 Dec  2 17:52 harbor/
drwxrwxr-x  3 bigtera bigtera  4096 Dec  2 17:32 ingress/
-rw-r--r--  1 bigtera bigtera   628 Dec  2 12:09 keepalived.conf
drwx------  2 root    root    16384 Nov 30 14:10 lost+found/
drwxrwxr-x  2 bigtera bigtera  4096 Dec  2 17:46 metalLB/
drwxrwxr-x  2 bigtera bigtera  4096 Dec  2 18:09 rancher_agent/
drwxrwxr-x  2 bigtera bigtera  4096 Dec  2 14:11 tools/
bigtera@k8s1:/k8s_deploy$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            3.9G     0  3.9G   0% /dev
tmpfs           798M  664K  797M   1% /run
/dev/sda1        98G  9.4G   84G  11% /
tmpfs           3.9G     0  3.9G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sdb1       492G  284M  466G   1% /k8s_deploy
tmpfs           798M     0  798M   0% /run/user/1001
bigtera@k8s1:/k8s_deploy$ 

```

和管理集群配置一样，需要更改所有vm的hostname 和 ip（根据文档开始的规划做配置），因为k8s集群需要访问ceph，所以需要配置存储ip。k8s集群基于ubuntu18，网络配置文件路径有变化

```
bigtera@k8s1:/k8s_deploy$ cat /etc/hostname 
k8s1
bigtera@k8s1:/k8s_deploy$ cat /etc/netplan/01-netcfg.yaml 
# This file describes the network interfaces available on your system
# For more information, see netplan(5).
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      dhcp4: no
      addresses: [172.17.73.151/22]
      optional: true
      gateway4: 172.17.75.254
      #nameservers:
      #        addresses: [114.114.114.114,8.8.8.8]

    ens19:
      dhcp4: no
      addresses: [10.10.101.151/22]
      optional: true
bigtera@k8s1:/k8s_deploy$ 

```



根据规划，k8s 集群需要vip,这里采用 keep alived来实现 ， keepalived 配置文件如下，其中指定了73.154 为vip

```
bigtera@k8s1:/k8s_deploy$ cat /etc/keepalived/keepalived.conf 
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
    virtual_router_id 66
#   Set the value of priority lower on the backup server than on the master server
    priority 60   #need change,for cluster first node has top priority
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1066
    }
    virtual_ipaddress {
        172.17.73.154/22
    }
}

```

配置完成 ，检查 vip 是否已经正常启动，部署时，需要将vip 落到节点1上，因为节点1 会作为集群的第一个节点，如下：

```
bigtera@k8s1:/k8s_deploy$ ip add
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 0e:1b:2f:c1:df:78 brd ff:ff:ff:ff:ff:ff
    inet 172.17.73.151/22 brd 172.17.75.255 scope global ens18
       valid_lft forever preferred_lft forever
    inet 172.17.73.154/22 scope global secondary ens18
       valid_lft forever preferred_lft forever
    inet6 fe80::c1b:2fff:fec1:df78/64 scope link 
       valid_lft forever preferred_lft forever

```





### 创建k8s集群

> 如果想了解详细的k8s集群创建步骤，可以参考：https://jameswsg.github.io/2020-04-12-kubernets-ha-master-relate.html

- 1，修改集群配置文件

  模板中已经预置了集群的配置文件 kubeadm-config.yaml，只需要根据实际ip规划情况做修改。

  ```
  bigtera@k8s1:/k8s_deploy/deploy$ ll
  total 48
  drwxrwxr-x  3 bigtera bigtera  4096 Dec  2 14:42 ./
  drwxrwxrwx  9 root    root     4096 Dec  2 18:08 ../
  -rw-rw-r--  1 bigtera bigtera 20755 Dec  2 14:42 calico.yaml
  drwxrwxr-x 15 bigtera bigtera  4096 Dec  2 14:01 ceph-csi/
  -rw-rw-r--  1 bigtera bigtera   998 Dec  2 13:10 kubeadm-config.yaml
  ```

  一般需要修改如下几行配置

  ```
   11 localAPIEndpoint:
   12   advertiseAddress: 172.17.73.151    #集群第一个节点的ip
   
   26 controlPlaneEndpoint: "172.17.73.154:6443"	集群的 vip 配置
  
  ```

- 2，修改完成，节点1执行初始化集群命令

  ```
  sudo kubeadm init --config=kubeadm-config.yaml --upload-certs |tee kubeadm-init.log
  
  输出中关键的信息如下：
  To start using your cluster, you need to run the following as a regular user:
  
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
  
  
  You can now join any number of the control-plane node running the following command on each as root:
  
    kubeadm join 172.17.73.154:6443 --token abcdef.0123456789abcdef \
      --discovery-token-ca-cert-hash sha256:57eefb7284e74740bfc69e945d369d3f63b0dfeae8e39e3a61489fc3fd3a1726 \
      --control-plane --certificate-key fe75999962225a1627ebd69a5f4dca8c9af08caca5a3c2a4682e10e8bb4874d7
  
  ```

  初始化完成后，屏幕会有提示，需要 粘贴上面的命令来配置，来配置和 api-server通信证书，需要执行如下命令

  ```
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
  
  ```

  完成后，可以检测第一个节点初始化的情况，如下:(可以看到集群中 目前只有一个节点)

  ```
  bigtera@k8s1:/k8s_deploy/deploy$ kubectl get nodes
  NAME          STATUS     ROLES    AGE     VERSION
  k8snode1v18   NotReady   master   2m27s   v1.18.4
  
  ```

  

- 3，节点2 加入集群

  步骤2中的输出中，有 加入集群的命令提示如下：

  ```
  You can now join any number of the control-plane node running the following command on each as root:
  
    kubeadm join 172.17.73.154:6443 --token abcdef.0123456789abcdef \
      --discovery-token-ca-cert-hash sha256:57eefb7284e74740bfc69e945d369d3f63b0dfeae8e39e3a61489fc3fd3a1726 \
      --control-plane --certificate-key fe75999962225a1627ebd69a5f4dca8c9af08caca5a3c2a4682e10e8bb4874d7
  ```

  在节点2中，直接粘贴上面的命令(注意需要加入 sudo )：

  ```
  bigtera@k8s2:~$ sudo kubeadm join 172.17.73.154:6443 --token abcdef.0123456789abcdef     --discovery-token-ca-cert-hash sha256:57eefb7284e74740bfc69e945d369d3f63b0dfeae8e39e3a61489fc3fd3a1726     --control-plane --certificate-key fe75999962225a1627ebd69a5f4dca8c9af08caca5a3c2a4682e10e8bb4874d7
  
  ```

  命令结束后，根据屏幕提示，粘贴提示的命令 执行

  ```
  	mkdir -p $HOME/.kube
  	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  	sudo chown $(id -u):$(id -g) $HOME/.kube/config
  
  ```

  

- 4，节点3 加入集群

  节点3操作 和上面步骤一样

  ```
  bigtera@k8s3:~$ sudo kubeadm join 172.17.73.154:6443 --token abcdef.0123456789abcdef     --discovery-token-ca-cert-hash sha256:57eefb7284e74740bfc69e945d369d3f63b0dfeae8e39e3a61489fc3fd3a1726     --control-plane --certificate-key fe75999962225a1627ebd69a5f4dca8c9af08caca5a3c2a4682e10e8bb4874d7
  
  ```

  命令结束后，根据屏幕提示，粘贴提示的命令 执行

  ```
  	mkdir -p $HOME/.kube
  	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  	sudo chown $(id -u):$(id -g) $HOME/.kube/config
  
  ```

  验证 集群当前状态

  ```
  bigtera@k8s3:~$ kubectl get nodes
  NAME          STATUS     ROLES    AGE     VERSION
  k8s2          NotReady   master   7m42s   v1.18.4
  k8s3          NotReady   master   4m19s   v1.18.4
  k8snode1v18   NotReady   master   15m     v1.18.4
  
  ```

  

  



### 部署网络插件

```
bigtera@k8s1:/k8s_deploy/deploy$ kubectl apply -f calico.yaml 

```

部署完网络插件，等待一会，检查所有pod 状态，应该所有pod 都是 正常的running 状态，如下：（如果有 pod 状态不对，需要回头检查）

```
bigtera@k8s1:/k8s_deploy/deploy$ kubectl get pod --all-namespaces
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-5fbfc9dfb6-hrfpm   1/1     Running   0          2m32s
kube-system   calico-node-kcvv5                          1/1     Running   0          2m32s
kube-system   calico-node-p8l8d                          1/1     Running   0          2m32s
kube-system   calico-node-tmdw6                          1/1     Running   0          2m32s
kube-system   coredns-7ff77c879f-8hn24                   1/1     Running   0          22m
kube-system   coredns-7ff77c879f-ngfnh                   1/1     Running   0          22m
kube-system   etcd-k8s2                                  1/1     Running   0          14m
kube-system   etcd-k8s3                                  1/1     Running   0          10m
kube-system   etcd-k8snode1v18                           1/1     Running   0          22m
kube-system   kube-apiserver-k8s2                        1/1     Running   0          14m
kube-system   kube-apiserver-k8s3                        1/1     Running   0          10m
kube-system   kube-apiserver-k8snode1v18                 1/1     Running   0          22m
kube-system   kube-controller-manager-k8s2               1/1     Running   0          14m
kube-system   kube-controller-manager-k8s3               1/1     Running   0          10m
kube-system   kube-controller-manager-k8snode1v18        1/1     Running   1          22m
kube-system   kube-proxy-4k4nk                           1/1     Running   1          14m
kube-system   kube-proxy-52t9h                           1/1     Running   1          22m
kube-system   kube-proxy-hndbf                           1/1     Running   1          10m
kube-system   kube-scheduler-k8s2                        1/1     Running   0          14m
kube-system   kube-scheduler-k8s3                        1/1     Running   0          10m
kube-system   kube-scheduler-k8snode1v18                 1/1     Running   1          22m
bigtera@k8s1:/k8s_deploy/deploy$ 

```





### 部署bigtera csi 插件

> 注意：部署bigtera csi插件前，需要ceph 集群ready，并且 ceph需要做某些配置，具体可以参考：https://jameswsg.github.io/2020-06-20-kubernets-use-ceph-basedOn-bigtera-CSI.html

简单步骤整理如下：

- mater 节点执行taint 操作

  k8s集群中每个节点既是master节点，也是work节点。csi插件以容器方式运行，需要运行在每个节点，所以要taint，否则csi的pod无法调度到节点

  ```
   kubectl taint nodes --all node-role.kubernetes.io/master-
   
  ```

- 部署

  ```
  bigtera@k8sTemplate:/k8s_deploy/deploy/ceph-csi/examples/rbd$ pwd
  /k8s_deploy/deploy/ceph-csi/examples/rbd
  
  bigtera@k8s1:/k8s_deploy/deploy/ceph-csi/examples/rbd$ bash -x plugin-deploy.sh 
  
  执行完继续执行下面命令
  kubectl apply -f ../csi-config-map-sample.yaml 
  kubectl apply -f secret.yaml
  kubectl apply -f storageclass.yaml 
  kubectl apply -f pvc.yaml
  ```

- 部署后，需要更改deployment的配置中 ImagePullPolicy ，将always 修改为IfNotPresent (和管理集群部署里的问题处理类似)

  ```
  bigtera@k8s1:~$ kubectl edit deploy csi-rbdplugin-provisioner 
  deployment.apps/csi-rbdplugin-provisioner edited
  
  ```

  

- 确认部署是否正常

  可以看到，csi相关的pod 已经全部是 running状态了。

  ```
  bigtera@k8s1:~$ kubectl get pod --all-namespaces
  NAMESPACE     NAME                                         READY   STATUS    RESTARTS   AGE
  default       csi-rbdplugin-provisioner-86644c75d6-g8crh   6/6     Running   0          95s
  default       csi-rbdplugin-provisioner-86644c75d6-jmgpj   6/6     Running   0          99s
  default       csi-rbdplugin-provisioner-86644c75d6-wdgnq   6/6     Running   0          102s
default       csi-rbdplugin-rrbfm                          3/3     Running   0          15m
  default       csi-rbdplugin-tc5kb                          3/3     Running   0          15m
  default       csi-rbdplugin-wk2qn                          3/3     Running   0          15m
  kube-system   calico-kube-controllers-5fbfc9dfb6-hrfpm     1/1     Running   0          21m
  kube-system   calico-node-kcvv5                            1/1     Running   0          21m
  
  ```
  
  
  
  可以测试创建一个pvc，pvc 可以正常bound时 说明部署成功，第一次创建pvc后，可能 要等 几分钟，才能 bound成功
  
  ```
  bigtera@k8s1:/k8s_deploy/deploy/ceph-csi/examples/rbd$ kubectl get pvc
  NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
  rbd-pvc   Bound    pvc-a873ec2a-c2a7-4ea5-8115-bbf1aa28a650   1Gi        RWO            csi-rbd-sc-harbor   2m17s
  bigtera@k8s1:/k8s_deploy/deploy/ceph-csi/examples/rbd$ 
  
  
  ```
  
  

### 部署ingress-controller

为后续部署其他需要ingress的应用做准备，比如 harbor仓库服务

因为其中的nginx镜像默认在国外，pull比较慢，更换成了 aliyun的镜像

```
bigtera@k8s1:/k8s_deploy/ingress$ helm install ingress-controller ./ingress-nginx-loadbalancer/

```

- 确认ingress 部署是否正常

  ```
  bigtera@k8s1:/k8s_deploy/ingress$ helm list
  NAME              	NAMESPACE	REVISION	UPDATED                               	STATUS  	CHART              	APP VERSION
  ingress-controller	default  	1       	2020-12-05 03:06:10.72208391 +0800 CST	deployed	ingress-nginx-3.7.0	0.40.2     
  bigtera@k8s1:/k8s_deploy/ingress$ 

  ```

  上面可以看出 ingress-controller的helm包已经正常deploy， 可以创建一个 ingress 验证是否正常
  
  ```
  bigtera@k8sTemplate:/k8s_deploy/ingress$ kubectl apply -f test-ingress.yaml 
  deployment.apps/my-nginx created
  service/my-nginx created
  ingress.extensions/my-nginx created
  
  创建完成后，查看ingress
  bigtera@k8sTemplate:/k8s_deploy/ingress$ kubectl get ingress
  NAME       CLASS    HOSTS                 ADDRESS   PORTS   AGE
  my-nginx   <none>   ingress.bigtera.com             80      7s
  

  ```
  
  



### 部署metallb（load balancer）

> 需要为 负载均衡器 规划额外的ip，外部通过 该 ip 访问k8s 集群内服务。





```
bigtera@k8sTemplate:/k8s_deploy/metalLB$ pwd
/k8s_deploy/metalLB


kubectl apply -f namespace.yaml 
kubectl apply -f deploy.yaml 
kubectl apply -f config.yaml 

kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"

```



- 验证部署是否正常

  检查之前 创建的ingress，如果已经分配了 ip，说明部署成功。

  ```
  bigtera@k8sTemplate:/k8s_deploy/metalLB$ kubectl get ingress
  NAME       CLASS    HOSTS                 ADDRESS         PORTS   AGE
  my-nginx   <none>   ingress.bigtera.com   172.17.73.152   80      16m
  bigtera@k8sTemplate:/k8s_deploy/metalLB$ 
  
  ```

  



### 部署容器镜像服务harbor

需要考虑其中的自定义配置， 包括harbor的域名，harbor的持久化存储所用的storage class



修改安装包中 配置文件的域名配置

```
 bigtera@k8s1:/k8s_deploy/harbor$ vim values.yaml 
bigtera@k8s1:/k8s_deploy/harbor$ 

 34   ingress:
 35     hosts:
 36       core: harbor.bigtera.com
 37       notary: notary.bigtera.com



```

配置完成后，部署harbor

```
bigtera@k8s1:/k8s_deploy$ helm install harbor harbor/


```

检查harbor部署情况，harbor通过ingress 方式访问，确认 ingress 访问入口 是否已经设置

```
bigtera@k8s1:/k8s_deploy$ kubectl get ingress
NAME                           CLASS    HOSTS                 ADDRESS         PORTS     AGE
harbor-harbor-ingress          <none>   harbor.bigtera.com    172.17.73.158   80, 443   52s
harbor-harbor-ingress-notary   <none>   notary.bigtera.com    172.17.73.158   80, 443   52s
my-nginx                       <none>   ingress.bigtera.com   172.17.73.158   80        7m44s
bigtera@k8s1:/k8s_deploy$ 


```

可以看到，harbor的访问入口已经被正确设置到了 之前配置的 vip 73.158



## 3，管理集群添加k8s集群

管理集群可以同时管理多个k8s集群（k8s集群可以是自己创建的，也可以是共有云厂商提供的)

管理集群添加 k8s集群主要分2个步骤，第一步在 管理集群生成配置，第二步 在被管集群操作。因为被管集群需要通过域名方式与 管理集群通信，所以 被管集群的每个节点也需要 手动配置域名解析，如下：

```
bigtera@k8s1:~$ cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	k8s1
172.17.73.161 rancher.bigtera.com
172.17.73.162 rancher.bigtera.com
172.17.73.163 rancher.bigtera.com

```

- 步骤1： 管理集群 操作如下

  ![image-20201204164152294](../img/容器管理平台/rancher-add-cluster0.png)

  

  ![image-20201204164234413](../img/容器管理平台/rancher-add-cluster2.png)

  输入集群的名字后，弹出如下 窗口

  ![image-20201204163828046](../img/容器管理平台/rancher-add-cluster.png)





- 步骤2：被管集群操作 

  步骤1中 会生成 一条指令，粘贴 最后一条指令在 被管节点执行

  ```
  bigtera@k8s1:~$ curl --insecure -sfL https://rancher.bigtera.com/v3/import/hstv8hh92rtrrth6pqqh6wdg5n76q4q7vhwchfdnjv9jh8fqv8vspv.yaml | kubectl apply -f -
  
  输出如下：
  clusterrole.rbac.authorization.k8s.io/proxy-clusterrole-kubeapiserver created
  clusterrolebinding.rbac.authorization.k8s.io/proxy-role-binding-kubernetes-master created
  namespace/cattle-system created
  serviceaccount/cattle created
  clusterrolebinding.rbac.authorization.k8s.io/cattle-admin-binding created
  secret/cattle-credentials-c1b144d created
  clusterrole.rbac.authorization.k8s.io/cattle-admin created
  deployment.apps/cattle-cluster-agent created
  bigtera@k8s1:~$ 
  
  ```

  命令执行完，最终会在 被管集群中 创建 rancher-agent，通过该agent 和管理集群通信，可以检测 该agent 运行是否正常

  ```
  bigtera@k8s1:~$ kubectl get pod --all-namespaces
  NAMESPACE       NAME                                         READY   STATUS             RESTARTS   AGE
  cattle-system   cattle-cluster-agent-6d4b5cb548-5hvhg        0/1     CrashLoopBackOff   4          2m43s
  
  ```

  可以看到 ，该agent 状态 还没有正常， 需要执行 如下的配置命令

  

### rancher-agent 配置



```

###### 直接粘贴执行即可

kubectl -n cattle-system patch  deployments cattle-cluster-agent --patch '{
    "spec": {
        "template": {
            "spec": {
                "hostAliases": [
                    {
                      "hostnames":
                      [
                        "rancher.bigtera.com"
                      ],
                      "ip": "172.17.73.161"
                    }
                ]
            }
        }
    }
}'


为保证 agent和 管理集群 通信的高可用， 建议 3个 ip 都加入

kubectl -n cattle-system patch  deployments cattle-cluster-agent --patch '{
    "spec": {
        "template": {
            "spec": {
                "hostAliases": [
                    {
                      "hostnames":
                      [
                        "rancher.bigtera.com"
                      ],
                      "ip": "172.17.73.162"
                    }
                ]
            }
        }
    }
}'


kubectl -n cattle-system patch  deployments cattle-cluster-agent --patch '{
    "spec": {
        "template": {
            "spec": {
                "hostAliases": [
                    {
                      "hostnames":
                      [
                        "rancher.bigtera.com"
                      ],
                      "ip": "172.17.73.163"
                    }
                ]
            }
        }
    }
}'

```



### 可能的问题处理

- k8s 集群和 管理集群 时间不一致




#平台统一 访问入口





nginx 变量：

rancher_servers

rancher 域名

cone host ip







## 虚拟化管理平台

https://172.17.72.228:8096/

![image-20201209151646750](../img/容器管理平台/login-hypervisor.png)

## 容器管理平台

目前需要通过域名访问，需要在访问客户端添加域名解析

![image-20201209151939806](../img/容器管理平台/login-rancher.png)

## 容器镜像 管理平台

https://172.17.72.228:8097/

![image-20201209151503090](../img/容器管理平台/login-harbor.png)

## 容器管理的 api 接口（k8s api）

https://172.17.72.228:8098  或者通过 k8s cluster 的 VIP



## 其他内置 服务

除了通过 host的 nginx 代理，还可以通过 metallb的load balancer ip

# 附录

相关信息 供参考





## rke 自动部署的 nginx

job 中的定义：

```
wsg@wsgRKE1:~$ kubectl -n kube-system edit job rke-ingress-controller-deploy-job



volumes:
      - configMap:
          defaultMode: 420
          items:
          - key: rke-ingress-controller
            path: rke-ingress-controller.yaml
          name: rke-ingress-controller
        name: config-volume

```





## rancher server debug

```
wsg@wsgRKE1:~$ kubectl -n cattle-system get pods -l app=rancher --no-headers -o custom-columns=name:.metadata.name | while read rancherpod; do kubectl -n cattle-system exec $rancherpod -c rancher -- loglevel --set debug; done
OK
OK
OK
wsg@wsgRKE1:~$ 

```



## rancher server 的 nginx 代理



### websocket

```
Request URL: wss://rancher.rke.com/v3/clusters/c-f4t68/subscribe?sockId=2
Request Method: GET
Status Code: 101 Switching Protocols



## request header

Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Cache-Control: no-cache
Connection: Upgrade
Cookie: CSRF=dca22f2412; R_SESS=token-qh96n:g5dfzrlhmh7dwcsbvgt2m8cjqg5sb69p5vs8l9jfnjdwbzk4c7tjp2
Host: rancher.rke.com
Origin: https://rancher.rke.com
Pragma: no-cache
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
Sec-WebSocket-Key: nTgVv8ZyZH+RWYd3uFbY0Q==
Sec-WebSocket-Version: 13
Upgrade: websocket
User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.88 Safari/537.36
```





### 错误记录





````

2020/12/15 03:10:59 [ERROR] Error during subscribe websocket: the client is not using the websocket protocol: 'upgrade' token not found in 'Connection' header
2020/12/15 03:10:59 [ERROR] Unknown error: websocket: the client is not using the websocket protocol: 'upgrade' token not found in 'Connection' header

````







## rancher server 删除

需要下载专门工具



```

bigtera@rancher1:/rancher_deploy/helm_package/rancher$ helm uninstall fleet-agent  -n fleet-system
release "fleet-agent" uninstalled
bigtera@rancher1:/rancher_deploy/helm_package/rancher$ helm uninstall fleet  -n fleet-system
release "fleet" uninstalled
bigtera@rancher1:/rancher_deploy/helm_package/rancher$ helm uninstall fleet-crd  -n fleet-system
release "fleet-crd" uninstalled


bigtera@rancher1:/rancher_deploy/helm_package/rancher$ helm uninstall rancher-operator rancher-operator-crd  -n rancher-operator-system


```





##rancher agent删除

将部署时 用的 yaml 文件，改位 delete -f







## rancher 高可用部署

不同 OS系统 对待 hosts文件的处理方法不太一样，win10的表现是 同一个 域名，如果第一个 不通，会尝试第二个域名，直至 尝试完所有配置的条目。

但是ubuntu看起来，只会选择第一个，就算第一个不通，也不会切换到第二个条目。

所以，windows通过 域名来实现 高可用，没有问题。 但是 linux 不能。







rancher lab 

https://mp.weixin.qq.com/s/LxKPnEMbNUinaWr8Qf4pNw





### tls external termination，80端口 提供服务



https://rancher.com/docs/rancher/v2.x/en/installation/resources/chart-options/#external-tls-termination











## 虚机模板 生成





单节点 环境

k8s

ingress-controller

部署 harbor（得到 harbor 镜像）

部署rancher agent（得到 agent 镜像）







##rancher集群

> 管理平台的 应用pod持久化存储 用 ceph。
>
> 管理平台自身的元数据 放在 etcd

ubuntu16版本（RKE install）



节点依赖软件包

docker.io

kubectl



节点配置

关闭swap

ssh用户加入docker 组，（docker ps 可以正常执行）

```
 usermod -aG docker <user_name>
 
```







traefik ingress-controller install

````
helm repo add traefik https://helm.traefik.io/traefik

helm install traefik traefik/traefik

````



安装cert-manger，参考之前文档



安装 rancher

```
wsg@wsgRKE1:~$ helm install rancher rancher-stable/rancher   --namespace cattle-system   --set hostname=rancher.rke.com
NAME: rancher
LAST DEPLOYED: Sat Nov 28 10:53:29 2020
NAMESPACE: cattle-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Rancher Server has been installed.

NOTE: Rancher may take several minutes to fully initialize. Please standby while Certificates are being issued and Ingress comes up.

Check out our docs at https://rancher.com/docs/rancher/v2.x/en/

Browse to https://rancher.rke.com

Happy Containering!
wsg@wsgRKE1:~$ kubectl get 



服务暴露在 如下node 端口
wsg@wsgRKE2:~$ kubectl get ingress --all-namespaces
Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
NAMESPACE       NAME      CLASS    HOSTS             ADDRESS                                     PORTS     AGE
cattle-system   rancher   <none>   rancher.rke.com   172.17.73.137,172.17.73.138,172.17.73.139   80, 443   60s
wsg@wsgRKE2:~$ 

```





参考

https://github.com/traefik/traefik-helm-chart











##k8s 集群



api-server需要高可用（Vip），通过keepalive的实现 vip，使用中发现 需要 创建集群的第一个节点 keepalive 权重设置为最大， 这样在 多个节点异常，需要重启k8s 所有节点时，etcd集群初始化 保证在 第一个 节点上 有 vip。





k8s 集群

ubuntu18 版本（因为bigtera CSI需要更高版本的内核）



禁用swap

挂载数据盘

停用 ubuntu18 内置的时间同步服务器

```
bigtera@k8s1:~$ systemctl status systemd-timesyncd.service 
● systemd-timesyncd.service - Network Time Synchronization
   Loaded: loaded (/lib/systemd/system/systemd-timesyncd.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:systemd-timesyncd.service(8)


```











