---
layout: post
title: 基于azure-vm的k8s网络性能测试
category: 技术
---

# k8s 网络性能测试

azure 环境，3台vm，自己安装k8s

```jsx

pulsar@k8s1:~$ kubectl get nodes -o wide
NAME   STATUS   ROLES                  AGE   VERSION    INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
k8s1   Ready    control-plane,master   92d   v1.20.15   10.2.0.15     <none>        Ubuntu 18.04.6 LTS   5.4.0-1089-azure   docker://20.10.7
k8s2   Ready    <none>                 92d   v1.20.15   10.2.0.16     <none>        Ubuntu 18.04.6 LTS   5.4.0-1089-azure   docker://20.10.7
k8s3   Ready    <none>                 92d   v1.20.15   10.2.0.17     <none>        Ubuntu 18.04.6 LTS   5.4.0-1089-azure   docker://20.10.7
pulsar@k8s1:~$

```

## 测试pod 准备

```jsx
pulsar@ansible-control:/opt/pulsar/DockerFile$ ls
Dockerfile  testpod.tar

pulsar@ansible-control:/opt/pulsar/DockerFile$ cat Dockerfile
FROM ubuntu:18.04
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && \
  apt-get install -y telnet && \
  apt install wget vim git curl sshpass  iperf3 hping3 -y
pulsar@ansible-control:/opt/pulsar/DockerFile$

2010  chown pulsar:pulsar testpod.tar
 2011  sudo chown pulsar:pulsar testpod.tar
 2012  scp -i ~/.ssh/my-mac-key/id_rsa testpod.tar pulsar@10.2.0.15:/home/pulsar
 2013  scp -i ~/.ssh/my-mac-key/id_rsa testpod.tar pulsar@10.2.0.16:/home/pulsar
 2014  scp -i ~/.ssh/my-mac-key/id_rsa testpod.tar pulsar@10.2.0.17:/home/pulsar
```

## vm 之间网络带宽

- iperf k8s1 to k8s3
    
    ```jsx
    pulsar@k8s3:~$ iperf3 -c 10.2.0.15 -i 1 -t 30
    Connecting to host 10.2.0.15, port 5201
    [  4] local 10.2.0.17 port 47578 connected to 10.2.0.15 port 5201
    [ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
    [  4]   0.00-1.00   sec   161 MBytes  1.35 Gbits/sec    0   1.43 MBytes
    [  4]   1.00-2.00   sec   158 MBytes  1.32 Gbits/sec    0   1.68 MBytes
    [  4]   2.00-3.00   sec   158 MBytes  1.33 Gbits/sec    0   1.98 MBytes
    [  4]   3.00-4.00   sec   158 MBytes  1.33 Gbits/sec    0   2.09 MBytes
    [  4]   4.00-5.00   sec   159 MBytes  1.33 Gbits/sec    0   2.30 MBytes
    [  4]   5.00-6.00   sec   159 MBytes  1.33 Gbits/sec    0   2.43 MBytes
    [  4]   6.00-7.00   sec   159 MBytes  1.33 Gbits/sec   22   1.22 MBytes
    [  4]   7.00-8.00   sec   157 MBytes  1.32 Gbits/sec    0   1.27 MBytes
    [  4]   8.00-9.00   sec   159 MBytes  1.33 Gbits/sec    0   1.35 MBytes
    [  4]   9.00-10.00  sec   158 MBytes  1.32 Gbits/sec    0   1.39 MBytes
    [  4]  10.00-11.00  sec   157 MBytes  1.31 Gbits/sec    0   1.41 MBytes
    [  4]  11.00-12.00  sec   159 MBytes  1.33 Gbits/sec    0   1.43 MBytes
    [  4]  12.00-13.00  sec   159 MBytes  1.33 Gbits/sec    0   1.44 MBytes
    [  4]  13.00-14.00  sec   159 MBytes  1.33 Gbits/sec    0   1.45 MBytes
    [  4]  14.00-15.00  sec   159 MBytes  1.33 Gbits/sec    0   1.45 MBytes
    [  4]  15.00-16.00  sec   158 MBytes  1.33 Gbits/sec    0   1.45 MBytes
    [  4]  16.00-17.00  sec   159 MBytes  1.33 Gbits/sec    8   1.07 MBytes
    [  4]  17.00-18.00  sec   159 MBytes  1.33 Gbits/sec    0   1.15 MBytes
    [  4]  18.00-19.00  sec   159 MBytes  1.33 Gbits/sec    0   1.26 MBytes
    [  4]  19.00-20.00  sec   158 MBytes  1.32 Gbits/sec    0   1.33 MBytes
    [  4]  20.00-21.00  sec   158 MBytes  1.33 Gbits/sec    0   1.37 MBytes
    [  4]  21.00-22.00  sec   159 MBytes  1.33 Gbits/sec    0   1.41 MBytes
    [  4]  22.00-23.00  sec   159 MBytes  1.33 Gbits/sec    0   1.43 MBytes
    [  4]  23.00-24.00  sec   159 MBytes  1.33 Gbits/sec    0   1.44 MBytes
    [  4]  24.00-25.00  sec   158 MBytes  1.32 Gbits/sec    0   1.45 MBytes
    [  4]  25.00-26.00  sec   158 MBytes  1.32 Gbits/sec    0   1.45 MBytes
    [  4]  26.00-27.00  sec   159 MBytes  1.33 Gbits/sec    0   1.45 MBytes
    [  4]  27.00-28.00  sec   159 MBytes  1.33 Gbits/sec    3   1.01 MBytes
    [  4]  28.00-29.00  sec   159 MBytes  1.33 Gbits/sec    0   1.13 MBytes
    [  4]  29.00-30.00  sec   159 MBytes  1.33 Gbits/sec    0   1.22 MBytes
    - - - - - - - - - - - - - - - - - - - - - - - - -
    [ ID] Interval           Transfer     Bandwidth       Retr
    [  4]   0.00-30.00  sec  4.64 GBytes  1.33 Gbits/sec   33             sender
    [  4]   0.00-30.00  sec  4.64 GBytes  1.33 Gbits/sec                  receiver
    
    iperf Done.
    ```
    
    - reverse direction, 一致
        
        ```jsx
        pulsar@k8s1:~$ iperf3 -c 10.2.0.17 -i 1 -t 30
        Connecting to host 10.2.0.17, port 5201
        [  4] local 10.2.0.15 port 39310 connected to 10.2.0.17 port 5201
        [ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
        [  4]   0.00-1.00   sec   161 MBytes  1.35 Gbits/sec   21   1.35 MBytes
        [  4]   1.00-2.00   sec   157 MBytes  1.32 Gbits/sec    0   1.49 MBytes
        [  4]   2.00-3.00   sec   158 MBytes  1.32 Gbits/sec    0   1.61 MBytes
        [  4]   3.00-4.00   sec   153 MBytes  1.29 Gbits/sec   11    866 KBytes
        [  4]   4.00-5.00   sec   157 MBytes  1.32 Gbits/sec   15    730 KBytes
        [  4]   5.00-6.00   sec   157 MBytes  1.31 Gbits/sec    0    796 KBytes
        [  4]   6.00-7.00   sec   157 MBytes  1.32 Gbits/sec    0    863 KBytes
        [  4]   7.00-8.00   sec   154 MBytes  1.29 Gbits/sec    4    703 KBytes
        [  4]   8.00-9.00   sec   157 MBytes  1.32 Gbits/sec    0    812 KBytes
        [  4]   9.00-10.00  sec   151 MBytes  1.27 Gbits/sec   55    509 KBytes
        [  4]  10.00-11.00  sec   155 MBytes  1.30 Gbits/sec    0    694 KBytes
        [  4]  11.00-12.00  sec   156 MBytes  1.31 Gbits/sec    7    642 KBytes
        [  4]  12.00-13.00  sec   156 MBytes  1.31 Gbits/sec    0    773 KBytes
        [  4]  13.00-14.00  sec   153 MBytes  1.28 Gbits/sec   40    700 KBytes
        [  4]  14.00-15.00  sec   155 MBytes  1.30 Gbits/sec    0    810 KBytes
        [  4]  15.00-16.00  sec   159 MBytes  1.33 Gbits/sec   19    696 KBytes
        [  4]  16.00-17.00  sec   158 MBytes  1.33 Gbits/sec   12    674 KBytes
        [  4]  17.00-18.00  sec   158 MBytes  1.32 Gbits/sec   13    572 KBytes
        [  4]  18.00-19.00  sec   156 MBytes  1.31 Gbits/sec    0    726 KBytes
        [  4]  19.00-20.00  sec   155 MBytes  1.30 Gbits/sec   17    658 KBytes
        [  4]  20.00-21.00  sec   157 MBytes  1.32 Gbits/sec   14    557 KBytes
        [  4]  21.00-22.00  sec   155 MBytes  1.30 Gbits/sec    0    722 KBytes
        [  4]  22.00-23.00  sec   158 MBytes  1.32 Gbits/sec    0    829 KBytes
        [  4]  23.00-24.00  sec   159 MBytes  1.33 Gbits/sec    0    890 KBytes
        [  4]  24.00-25.00  sec   156 MBytes  1.31 Gbits/sec    5    721 KBytes
        [  4]  25.00-26.00  sec   159 MBytes  1.33 Gbits/sec    0    775 KBytes
        [  4]  26.00-27.00  sec   158 MBytes  1.33 Gbits/sec    3    640 KBytes
        [  4]  27.00-28.00  sec   156 MBytes  1.31 Gbits/sec    4    594 KBytes
        [  4]  28.00-29.00  sec   155 MBytes  1.30 Gbits/sec    0    737 KBytes
        [  4]  29.00-30.00  sec   149 MBytes  1.25 Gbits/sec    0    834 KBytes
        - - - - - - - - - - - - - - - - - - - - - - - - -
        [ ID] Interval           Transfer     Bandwidth       Retr
        [  4]   0.00-30.00  sec  4.57 GBytes  1.31 Gbits/sec  240             sender
        [  4]   0.00-30.00  sec  4.57 GBytes  1.31 Gbits/sec                  receiver
        
        iperf Done.
        ```
        

- k8s2  k8s3
    
    ```jsx
    
    ```
    

- hping3 test k8s2 k8s3
    
    
    ```jsx
    
    pulsar@k8s2:~$ sudo hping3 -c 10 -S -p 22 10.2.0.17
    HPING 10.2.0.17 (eth0 10.2.0.17): S set, 40 headers + 0 data bytes
    len=44 ip=10.2.0.17 ttl=64 DF id=0 sport=22 flags=SA seq=0 win=64240 rtt=3.9 ms
    len=44 ip=10.2.0.17 ttl=64 DF id=0 sport=22 flags=SA seq=1 win=64240 rtt=3.7 ms
    len=44 ip=10.2.0.17 ttl=64 DF id=0 sport=22 flags=SA seq=2 win=64240 rtt=3.6 ms
    len=44 ip=10.2.0.17 ttl=64 DF id=0 sport=22 flags=SA seq=3 win=64240 rtt=7.5 ms
    len=44 ip=10.2.0.17 ttl=64 DF id=0 sport=22 flags=SA seq=4 win=64240 rtt=3.1 ms
    len=44 ip=10.2.0.17 ttl=64 DF id=0 sport=22 flags=SA seq=5 win=64240 rtt=2.9 ms
    len=44 ip=10.2.0.17 ttl=64 DF id=0 sport=22 flags=SA seq=6 win=64240 rtt=6.7 ms
    len=44 ip=10.2.0.17 ttl=64 DF id=0 sport=22 flags=SA seq=7 win=64240 rtt=6.5 ms
    len=44 ip=10.2.0.17 ttl=64 DF id=0 sport=22 flags=SA seq=8 win=64240 rtt=6.3 ms
    len=44 ip=10.2.0.17 ttl=64 DF id=0 sport=22 flags=SA seq=9 win=64240 rtt=2.2 ms
    
    --- 10.2.0.17 hping statistic ---
    10 packets transmitted, 10 packets received, 0% packet loss
    round-trip min/avg/max = 2.2/4.6/7.5 ms
    pulsar@k8s2:~$
    
    ```
    
    - reverse,
        
        ```jsx
        pulsar@k8s3:~$ sudo hping3 -c 10 -S -p 22 10.2.0.16
        HPING 10.2.0.16 (eth0 10.2.0.16): S set, 40 headers + 0 data bytes
        len=44 ip=10.2.0.16 ttl=64 DF id=0 sport=22 flags=SA seq=0 win=64240 rtt=7.8 ms
        len=44 ip=10.2.0.16 ttl=64 DF id=0 sport=22 flags=SA seq=1 win=64240 rtt=7.6 ms
        len=44 ip=10.2.0.16 ttl=64 DF id=0 sport=22 flags=SA seq=2 win=64240 rtt=3.3 ms
        len=44 ip=10.2.0.16 ttl=64 DF id=0 sport=22 flags=SA seq=3 win=64240 rtt=7.2 ms
        len=44 ip=10.2.0.16 ttl=64 DF id=0 sport=22 flags=SA seq=4 win=64240 rtt=7.1 ms
        len=44 ip=10.2.0.16 ttl=64 DF id=0 sport=22 flags=SA seq=5 win=64240 rtt=6.9 ms
        len=44 ip=10.2.0.16 ttl=64 DF id=0 sport=22 flags=SA seq=6 win=64240 rtt=6.6 ms
        len=44 ip=10.2.0.16 ttl=64 DF id=0 sport=22 flags=SA seq=7 win=64240 rtt=2.5 ms
        len=44 ip=10.2.0.16 ttl=64 DF id=0 sport=22 flags=SA seq=8 win=64240 rtt=2.3 ms
        len=44 ip=10.2.0.16 ttl=64 DF id=0 sport=22 flags=SA seq=9 win=64240 rtt=6.0 ms
        
        --- 10.2.0.16 hping statistic ---
        10 packets transmitted, 10 packets received, 0% packet loss
        round-trip min/avg/max = 2.3/5.7/7.8 ms
        pulsar@k8s3:~$
        ```
        

## pod 之间网络带宽

- 测试pod 运行
    
    
    ```jsx
    
    pulsar@k8s2:~$ sudo docker images
    REPOSITORY                                       TAG        IMAGE ID       CREATED          SIZE
    wsg/testpod                                      latest     10e2e83274da   19 minutes ago   267MB
    
    //run
    
    kubectl create deployment pingtest --image=busybox --replicas=3 -- sleep infinity
    
    kubectl create deployment testnginx --image=nginx  --replicas=3
    
    kubectl create deployment iperfdeploy --image=iper  --replicas=3
    
    kubectl create deployment wsgdeploy --image=wsg/testpod --image-pull-policy='never' --replicas=3 -- sleep infinity 
    ```
    
- pod 分布
    
    ```jsx
    pulsar@k8s1:~$ kubectl get pod -o wide
    NAME                        READY   STATUS    RESTARTS   AGE   IP            NODE   NOMINATED NODE   READINESS GATES
    wsgdeploy-f4d8f4587-9xpxv   1/1     Running   0          14s   10.244.2.15   k8s3   <none>           <none>
    wsgdeploy-f4d8f4587-hwvm2   1/1     Running   0          14s   10.244.1.20   k8s2   <none>           <none>
    wsgdeploy-f4d8f4587-q5h4c   1/1     Running   0          14s   10.244.2.16   k8s3   <none>           <none>
    pulsar@k8s1:~$
    ```
    

- iperf test pod in k8s2 ,k8s3
    
    k8s3 pod (10.244.2.15)
    
    ```jsx
    root@wsgdeploy-f4d8f4587-9xpxv:/# iperf3 -s
    -----------------------------------------------------------
    Server listening on 5201
    -----------------------------------------------------------
    ```
    
    k8s2 pod (10.244.1.20)
    
    ```jsx
    root@wsgdeploy-f4d8f4587-hwvm2:/# iperf3 -c 10.244.2.15 -t 1 -t 30
    Connecting to host 10.244.2.15, port 5201
    [  5] local 10.244.1.20 port 59890 connected to 10.244.2.15 port 5201
    [ ID] Interval           Transfer     Bitrate         Retr  Cwnd
    [  5]   0.00-1.00   sec   164 MBytes  1.38 Gbits/sec   65   1013 KBytes
    [  5]   1.00-2.00   sec   159 MBytes  1.33 Gbits/sec    0   1.10 MBytes
    [  5]   2.00-3.00   sec   159 MBytes  1.33 Gbits/sec    0   1.20 MBytes
    [  5]   3.00-4.00   sec   160 MBytes  1.34 Gbits/sec    0   1.29 MBytes
    [  5]   4.00-5.00   sec   159 MBytes  1.33 Gbits/sec    0   1.37 MBytes
    [  5]   5.00-6.00   sec   159 MBytes  1.33 Gbits/sec    0   1.45 MBytes
    [  5]   6.00-7.00   sec   159 MBytes  1.33 Gbits/sec    0   1.53 MBytes
    [  5]   7.00-8.00   sec   159 MBytes  1.33 Gbits/sec    0   1.60 MBytes
    [  5]   8.00-9.00   sec   159 MBytes  1.33 Gbits/sec    2   1.23 MBytes
    [  5]   9.00-10.00  sec   159 MBytes  1.33 Gbits/sec    0   1.34 MBytes
    [  5]  10.00-11.00  sec   159 MBytes  1.33 Gbits/sec    0   1.43 MBytes
    [  5]  11.00-12.00  sec   159 MBytes  1.33 Gbits/sec    0   1.50 MBytes
    [  5]  12.00-13.00  sec   155 MBytes  1.30 Gbits/sec    0   1.55 MBytes
    [  5]  13.00-14.00  sec   162 MBytes  1.36 Gbits/sec    4   1.12 MBytes
    [  5]  14.00-15.00  sec   159 MBytes  1.33 Gbits/sec    0   1.22 MBytes
    [  5]  15.00-16.00  sec   159 MBytes  1.33 Gbits/sec    0   1.31 MBytes
    [  5]  16.00-17.00  sec   160 MBytes  1.34 Gbits/sec    1   1.01 MBytes
    [  5]  17.00-18.00  sec   158 MBytes  1.32 Gbits/sec    8    845 KBytes
    [  5]  18.00-19.00  sec   160 MBytes  1.34 Gbits/sec    0    976 KBytes
    [  5]  19.00-20.00  sec   158 MBytes  1.32 Gbits/sec    0   1.06 MBytes
    [  5]  20.00-21.00  sec   160 MBytes  1.34 Gbits/sec    0   1.17 MBytes
    [  5]  21.00-22.00  sec   159 MBytes  1.33 Gbits/sec    0   1.26 MBytes
    [  5]  22.00-23.00  sec   159 MBytes  1.33 Gbits/sec   26    995 KBytes
    [  5]  23.00-24.00  sec   159 MBytes  1.33 Gbits/sec    0   1.08 MBytes
    [  5]  24.00-25.00  sec   159 MBytes  1.33 Gbits/sec    0   1.18 MBytes
    [  5]  25.00-26.00  sec   159 MBytes  1.33 Gbits/sec    0   1.27 MBytes
    [  5]  26.00-27.00  sec   159 MBytes  1.33 Gbits/sec   12    993 KBytes
    [  5]  27.00-28.00  sec   159 MBytes  1.33 Gbits/sec    0   1.08 MBytes
    [  5]  28.00-29.00  sec   160 MBytes  1.34 Gbits/sec    6    848 KBytes
    [  5]  29.00-30.00  sec   159 MBytes  1.33 Gbits/sec    0    978 KBytes
    - - - - - - - - - - - - - - - - - - - - - - - - -
    [ ID] Interval           Transfer     Bitrate         Retr
    [  5]   0.00-30.00  sec  4.66 GBytes  1.33 Gbits/sec  124             sender
    [  5]   0.00-30.00  sec  4.66 GBytes  1.33 Gbits/sec                  receiver
    
    iperf Done.
    ```
    
    - reverse test(looks ok)
        
        ```jsx
        
        root@wsgdeploy-f4d8f4587-9xpxv:/# iperf3 -c 10.244.1.20 -i 1 -t 30
        Connecting to host 10.244.1.20, port 5201
        [  5] local 10.244.2.15 port 57896 connected to 10.244.1.20 port 5201
        [ ID] Interval           Transfer     Bitrate         Retr  Cwnd
        [  5]   0.00-1.00   sec   132 MBytes  1.11 Gbits/sec    0   3.00 MBytes
        [  5]   1.00-2.00   sec   142 MBytes  1.20 Gbits/sec   37   3.15 MBytes
        [  5]   2.00-3.00   sec   145 MBytes  1.22 Gbits/sec   13   3.15 MBytes
        [  5]   3.00-4.00   sec   146 MBytes  1.23 Gbits/sec   23   3.15 MBytes
        [  5]   4.00-5.00   sec   144 MBytes  1.21 Gbits/sec   18   3.15 MBytes
        [  5]   5.00-6.00   sec   150 MBytes  1.26 Gbits/sec    0   3.15 MBytes
        [  5]   6.00-7.00   sec   140 MBytes  1.17 Gbits/sec    0   3.15 MBytes
        [  5]   7.00-8.00   sec   149 MBytes  1.25 Gbits/sec    0   3.15 MBytes
        [  5]   8.00-9.00   sec   150 MBytes  1.26 Gbits/sec    0   3.15 MBytes
        [  5]   9.00-10.00  sec   149 MBytes  1.25 Gbits/sec    0   3.15 MBytes
        [  5]  10.00-11.00  sec   148 MBytes  1.24 Gbits/sec    0   3.15 MBytes
        [  5]  11.00-12.00  sec   149 MBytes  1.25 Gbits/sec    0   3.15 MBytes
        [  5]  12.00-13.00  sec   150 MBytes  1.26 Gbits/sec    0   3.15 MBytes
        [  5]  13.00-14.00  sec   149 MBytes  1.25 Gbits/sec    0   3.15 MBytes
        [  5]  14.00-15.00  sec   154 MBytes  1.29 Gbits/sec    0   3.15 MBytes
        [  5]  15.00-16.00  sec   150 MBytes  1.26 Gbits/sec    0   3.15 MBytes
        [  5]  16.00-17.00  sec   149 MBytes  1.25 Gbits/sec    0   3.15 MBytes
        [  5]  17.00-18.00  sec   152 MBytes  1.28 Gbits/sec    0   3.15 MBytes
        [  5]  18.00-19.00  sec   151 MBytes  1.27 Gbits/sec    0   3.15 MBytes
        [  5]  19.00-20.00  sec   152 MBytes  1.28 Gbits/sec    0   3.15 MBytes
        [  5]  20.00-21.00  sec   148 MBytes  1.24 Gbits/sec    0   3.15 MBytes
        [  5]  21.00-22.00  sec   150 MBytes  1.26 Gbits/sec    0   3.15 MBytes
        [  5]  22.00-23.00  sec   150 MBytes  1.26 Gbits/sec    0   3.15 MBytes
        [  5]  23.00-24.00  sec   150 MBytes  1.26 Gbits/sec    0   3.15 MBytes
        [  5]  24.00-25.00  sec   148 MBytes  1.24 Gbits/sec    0   3.15 MBytes
        [  5]  25.00-26.00  sec   148 MBytes  1.24 Gbits/sec    0   3.15 MBytes
        [  5]  26.00-27.00  sec   149 MBytes  1.25 Gbits/sec    0   3.15 MBytes
        [  5]  27.00-28.00  sec   150 MBytes  1.26 Gbits/sec    0   3.15 MBytes
        [  5]  28.00-29.00  sec   149 MBytes  1.25 Gbits/sec    0   3.15 MBytes
        [  5]  29.00-30.00  sec   150 MBytes  1.26 Gbits/sec    0   3.15 MBytes
        - - - - - - - - - - - - - - - - - - - - - - - - -
        [ ID] Interval           Transfer     Bitrate         Retr
        [  5]   0.00-30.00  sec  4.34 GBytes  1.24 Gbits/sec   91             sender
        [  5]   0.00-30.00  sec  4.33 GBytes  1.24 Gbits/sec                  receiver
        
        iperf Done.
        root@wsgdeploy-f4d8f4587-9xpxv:/#
        ```
        
    

- hping3 test pod in k8s2,k8s3
    
    重新在deployment中增加了 80的ports 配置
    
    ```jsx
    
    pulsar@k8s1:~$ kubectl get pod -o wide
    NAME                        READY   STATUS    RESTARTS   AGE     IP            NODE   NOMINATED NODE   READINESS GATES
    wsgdeploy-7d94c4696-72zxt   1/1     Running   0          3m37s   10.244.1.21   k8s2   <none>           <none>
    wsgdeploy-7d94c4696-vfqlj   1/1     Running   0          3m34s   10.244.2.18   k8s3   <none>           <none>
    wsgdeploy-7d94c4696-xxzhg   1/1     Running   0          3m38s   10.244.2.17   k8s3   <none>           <none>
    pulsar@k8s1:~$
    
    root@wsgdeploy-7d94c4696-72zxt:/# hping3 -c 10 -S -p 80 10.244.2.18
    HPING 10.244.2.18 (eth0 10.244.2.18): S set, 40 headers + 0 data bytes
    len=44 ip=10.244.2.18 ttl=62 DF id=0 sport=80 flags=SA seq=0 win=64860 rtt=3.6 ms
    len=44 ip=10.244.2.18 ttl=62 DF id=0 sport=80 flags=SA seq=1 win=64860 rtt=11.3 ms
    len=44 ip=10.244.2.18 ttl=62 DF id=0 sport=80 flags=SA seq=2 win=64860 rtt=7.1 ms
    len=44 ip=10.244.2.18 ttl=62 DF id=0 sport=80 flags=SA seq=3 win=64860 rtt=2.8 ms
    len=44 ip=10.244.2.18 ttl=62 DF id=0 sport=80 flags=SA seq=4 win=64860 rtt=10.6 ms
    len=44 ip=10.244.2.18 ttl=62 DF id=0 sport=80 flags=SA seq=5 win=64860 rtt=2.4 ms
    len=44 ip=10.244.2.18 ttl=62 DF id=0 sport=80 flags=SA seq=6 win=64860 rtt=2.2 ms
    len=44 ip=10.244.2.18 ttl=62 DF id=0 sport=80 flags=SA seq=7 win=64860 rtt=10.1 ms
    len=44 ip=10.244.2.18 ttl=62 DF id=0 sport=80 flags=SA seq=8 win=64860 rtt=9.8 ms
    len=44 ip=10.244.2.18 ttl=62 DF id=0 sport=80 flags=SA seq=9 win=64860 rtt=5.6 ms
    
    --- 10.244.2.18 hping statistic ---
    10 packets transmitted, 10 packets received, 0% packet loss
    round-trip min/avg/max = 2.2/6.6/11.3 ms
    root@wsgdeploy-7d94c4696-72zxt:/#
    ```
    
    - pod 到自己的延迟,pod 里面无法测试
        
        ```jsx
        root@wsgdeploy-7d94c4696-72zxt:/# hping3 -c 10 -S -p 80 10.244.1.21
        HPING 10.244.1.21 (eth0 10.244.1.21): S set, 40 headers + 0 data bytes
        
        --- 10.244.1.21 hping statistic ---
        10 packets transmitted, 0 packets received, 100% packet loss
        round-trip min/avg/max = 0.0/0.0/0.0 ms
        ```
        
    

# 常用命令

## 运行测试pod

```jsx
pulsar@k8s1:~$ kubectl create deployment pingtest --image=busybox --replicas=3 -- sleep infinity
deployment.apps/pingtest created

pulsar@k8s1:~$ kubectl get pod -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP            NODE   NOMINATED NODE   READINESS GATES
pingtest-64f9cb6b84-d4f2n   1/1     Running   0          12s   10.244.1.15   k8s2   <none>           <none>
pingtest-64f9cb6b84-gvtz6   1/1     Running   0          12s   10.244.2.7    k8s3   <none>           <none>
pingtest-64f9cb6b84-mqxwz   1/1     Running   0          12s   10.244.2.8    k8s3   <none>           <none>
pulsar@k8s1:~$

```

## 测试pod 选择

busybox 里面apt 和yum 都不能用，无法安装自己想要的

ubuntu18.04 里面没有常驻进程，k8s 调度后，会导致pod 状态不对。

nginx 是基于 ubunut 系列，里面有常驻进程。

- check my own image(base on nginx)
    
    
    ```jsx
    pulsar@ansible-control:/opt/pulsar/DockerFile$ sudo docker inspect iperfpod
    [
        {
            "Id": "sha256:e391f4611583062170ceb08e5c68cceb608cf1471a226ff3a5191d547cd884b3",
            "RepoTags": [
                "iperfpod:latest"
            ],
            "RepoDigests": [],
            "Parent": "sha256:920e865e9a2211ac13d8839d359289c93481485d7f4439578f2c4bca01b4652d",
            "Comment": "",
            "Created": "2022-11-20T03:25:52.878285695Z",
            "Container": "876de6369009995fa0ab06cf7f9e50fe7b221b8ca23e6da0d3fae16e64ead6af",
            "ContainerConfig": {
    ],
                "Cmd": [
                    "nginx",
                    "-g",
                    "daemon off;"
                ],
                "Image": "sha256:920e865e9a2211ac13d8839d359289c93481485d7f4439578f2c4bca01b4652d",
                "Volumes": null,
                "WorkingDir": "",
                "Entrypoint": [
                    "/docker-entrypoint.sh"
                ],
    ```
    
    dockerfile
    
    ```jsx
    
    pulsar@ansible-control:/opt/pulsar/DockerFile$ cat Dockerfile
    FROM nginx
    ENV DEBIAN_FRONTEND=noninteractive
    RUN apt-get update && \
      apt-get install -y telnet && \
      apt install wget vim git curl sshpass  iperf3 hping3 -y
    
    ```
    

# 问题处理记录

## deployment 可以不配置ports，导致pod端口无法访问

- 无ports 配置效果
    
    ```jsx
    pulsar@k8s1:~$ kubectl describe pod wsgdeploy-f4d8f4587-9xpxv
    Name:         wsgdeploy-f4d8f4587-9xpxv
    Namespace:    default
    Priority:     0
    Node:         k8s3/10.2.0.17
    Start Time:   Sun, 20 Nov 2022 03:35:35 +0000
    Labels:       app=wsgdeploy
                  pod-template-hash=f4d8f4587
    Annotations:  <none>
    Status:       Running
    IP:           10.244.2.15
    IPs:
      IP:           10.244.2.15
    Controlled By:  ReplicaSet/wsgdeploy-f4d8f4587
    Containers:
      testpod:
        Container ID:   docker://d5993adc2d69d56eeeb14e30c50e63b0495dbe0069ca7a25a00ebbc176470c54
        Image:          iperfpod
        Image ID:       docker://sha256:e391f4611583062170ceb08e5c68cceb608cf1471a226ff3a5191d547cd884b3
        Port:           <none>
        Host Port:      <none>
        State:          Running
          Started:      Sun, 20 Nov 2022 03:35:36 +0000
        Ready:          True
        Restart Count:  0
        Environment:    <none>
        Mounts:
          /var/run/secrets/kubernetes.io/serviceaccount from default-token-fl9zj (ro)
    Conditions:
      Type              Status
      Initialized       True
      Ready             True
      ContainersReady   True
      PodScheduled      True
    Volumes:
      default-token-fl9zj:
        Type:        Secret (a volume populated by a Secret)
        SecretName:  default-token-fl9zj
        Optional:    false
    QoS Class:       BestEffort
    Node-Selectors:  <none>
    Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                     node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
    Events:
      Type    Reason     Age   From               Message
      ----    ------     ----  ----               -------
      Normal  Scheduled  15m   default-scheduler  Successfully assigned default/wsgdeploy-f4d8f4587-9xpxv to k8s3
      Normal  Pulled     15m   kubelet            Container image "iperfpod" already present on machine
      Normal  Created    15m   kubelet            Created container testpod
      Normal  Started    15m   kubelet            Started container testpod
    ```
    

- add ports config
    
    ```jsx
    pulsar@k8s1:~$ kubectl describe pod wsgdeploy-7d94c4696-vfqlj
    Name:         wsgdeploy-7d94c4696-vfqlj
    Namespace:    default
    Priority:     0
    Node:         k8s3/10.2.0.17
    Start Time:   Sun, 20 Nov 2022 04:03:10 +0000
    Labels:       app=wsgdeploy
                  pod-template-hash=7d94c4696
    Annotations:  <none>
    Status:       Running
    IP:           10.244.2.18
    IPs:
      IP:           10.244.2.18
    Controlled By:  ReplicaSet/wsgdeploy-7d94c4696
    Containers:
      testpod:
        Container ID:   docker://e2b375a4ee786f85933cad770f17eb61fede99f66d02a1f1dd080d3a1142b6e5
        Image:          iperfpod
        Image ID:       docker://sha256:e391f4611583062170ceb08e5c68cceb608cf1471a226ff3a5191d547cd884b3
        Port:           80/TCP
        Host Port:      0/TCP
        State:          Running
          Started:      Sun, 20 Nov 2022 04:03:11 +0000
        Ready:          True
        Restart Count:  0
        Environment:    <none>
        Mounts:
          /var/run/secrets/kubernetes.io/serviceaccount from default-token-fl9zj (ro)
    Conditions:
      Type              Status
      Initialized       True
      Ready             True
      ContainersReady   True
      PodScheduled      True
    Volumes:
      default-token-fl9zj:
        Type:        Secret (a volume populated by a Secret)
        SecretName:  default-token-fl9zj
        Optional:    false
    QoS Class:       BestEffort
    Node-Selectors:  <none>
    Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                     node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
    Events:
      Type    Reason     Age   From               Message
      ----    ------     ----  ----               -------
      Normal  Scheduled  21s   default-scheduler  Successfully assigned default/wsgdeploy-7d94c4696-vfqlj to k8s3
      Normal  Pulled     20s   kubelet            Container image "iperfpod" already present on machine
      Normal  Created    20s   kubelet            Created container testpod
      Normal  Started    20s   kubelet            Started container testpod
    ```
    

## k8s2 无法访问internet

- 正常节点
    
    
    ```jsx
    nameserver 127.0.0.53
    options edns0
    search 3k2eb4adn3qendpvi0gbixgu3a.bx.internal.cloudapp.net
    pulsar@k8s1:~$ cat /etc/resolv.conf
    
    nameserver 127.0.0.53
    options edns0
    search 3k2eb4adn3qendpvi0gbixgu3a.bx.internal.cloudapp.net
    pulsar@k8s3:~$
    ```
    

dns 解析有问题

```jsx
pulsar@k8s2:~$ nslookup www.google.com
Server:		127.0.0.53
Address:	127.0.0.53#53

** server can't find www.google.com: SERVFAIL
```

- other node
    
    ```jsx
    pulsar@k8s3:~$ nslookup www.google.com
    Server:		127.0.0.53
    Address:	127.0.0.53#53
    
    Non-authoritative answer:
    Name:	www.google.com
    Address: 172.253.115.104
    Name:	www.google.com
    Address: 172.253.115.106
    Name:	www.google.com
    Address: 172.253.115.105
    Name:	www.google.com
    Address: 172.253.115.99
    Name:	www.google.com
    Address: 172.253.115.103
    Name:	www.google.com
    Address: 172.253.115.147
    Name:	www.google.com
    Address: 2607:f8b0:4004:c1d::93
    Name:	www.google.com
    Address: 2607:f8b0:4004:c1d::6a
    ```
