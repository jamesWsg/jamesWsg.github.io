---
layout: post
title: azure k8s AKS探索实践
category: 技术
---

# azure k8s

# 背景

在azure AKS上测试产品相关操作时整理的一些知识点

## bookie

```jsx

Volumes:
  journal-0:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  journal-0-fujun-test-sn-platform-bookie-0
    ReadOnly:   false
  ledgers-0:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  ledgers-0-fujun-test-sn-platform-bookie-0
    ReadOnly:   false
  kube-api-access-rtl45:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
```

check pvc

```jsx
➜  ~ kubectl get pvc -n pulsar
NAME                                                                       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-fujun-test-sn-platform-zookeeper-0                                    Bound    pvc-fb2af620-cdbc-4ef0-924c-57252623a221   50Gi       RWO            default        14d
data-fujun-test-sn-platform-zookeeper-1                                    Bound    pvc-281470a3-ae62-4a54-9b5e-954eb75abd97   50Gi       RWO            default        14d
data-fujun-test-sn-platform-zookeeper-2                                    Bound    pvc-9ba198cb-e8dd-4fbe-9b9b-2ede99b55958   50Gi       RWO            default        14d
fujun-test-sn-platform-grafana-data                                        Bound    pvc-cebbcac9-ba38-4c6c-9504-7a05cf9ac643   10Gi       RWO            default        14d
fujun-test-sn-platform-prometheus-data                                     Bound    pvc-5a0cdb98-4559-4a68-a2a9-9576bd5119b6   10Gi       RWO            default        14d
fujun-test-sn-platform-streamnative-console-data                           Bound    pvc-bc13b4ee-4740-497b-b129-291d392f35e6   10Gi       RWO            default        14d
fujun-test-sn-platform-vault-vault-volume-fujun-test-sn-platform-vault-0   Bound    pvc-bd049872-ac6f-4494-9528-b3e0dd507cc3   10Gi       RWO            default        111d
fujun-test-sn-platform-vault-vault-volume-fujun-test-sn-platform-vault-1   Bound    pvc-2bddfbd2-c2fe-49a6-9cf3-d421a7cdf2d5   10Gi       RWO            default        111d
fujun-test-sn-platform-vault-vault-volume-fujun-test-sn-platform-vault-2   Bound    pvc-6ada15e0-f425-4794-ab89-44c96767b860   10Gi       RWO            default        111d
journal-0-fujun-test-sn-platform-bookie-0                                  Bound    pvc-2c809928-d1b7-4e34-b338-a3512728d18d   10Gi       RWO            default        14d
journal-0-fujun-test-sn-platform-bookie-1                                  Bound    pvc-ee067a35-df1b-45c0-9a79-890f31596ca6   10Gi       RWO            default        14d
journal-0-fujun-test-sn-platform-bookie-2                                  Bound    pvc-b6401801-8844-40d9-a542-00be214d8f18   10Gi       RWO            default        14d
ledgers-0-fujun-test-sn-platform-bookie-0                                  Bound    pvc-fbcc2b48-2f5e-475e-8b9b-6612ff69437c   50Gi       RWO            default        14d
ledgers-0-fujun-test-sn-platform-bookie-1                                  Bound    pvc-fd15a9d0-3d8d-43ca-887c-c74f251b3444   50Gi       RWO            default        14d
ledgers-0-fujun-test-sn-platform-bookie-2                                  Bound    pvc-9b7cc73d-2cd0-4738-a6cc-e8392595b18e   50Gi       RWO            default        14d
➜  ~
```

check storage class

```jsx
➜  ~ kubectl get sc
NAME                    PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
azurefile               file.csi.azure.com   Delete          Immediate              true                   111d
azurefile-csi           file.csi.azure.com   Delete          Immediate              true                   111d
azurefile-csi-premium   file.csi.azure.com   Delete          Immediate              true                   111d
azurefile-premium       file.csi.azure.com   Delete          Immediate              true                   111d
default (default)       disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   111d
managed                 disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   111d
managed-csi             disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   111d
managed-csi-premium     disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   111d
managed-premium         disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   111d
```

# pod 信息中可以看到 node 信息

- bookie-0 pod
    
    ```jsx
    ➜  ~ kubectl describe pod fujun-test-sn-platform-bookie-0 -n pulsar
    Name:         fujun-test-sn-platform-bookie-0
    Namespace:    pulsar
    Priority:     0
    Node:         aks-agentpool-13502177-vmss000000/10.224.0.4
    Start Time:   Mon, 19 Dec 2022 16:34:27 +0800
    Labels:       app=sn-platform
                  cloud.streamnative.io/app=pulsar
                  cloud.streamnative.io/cluster=fujun-test-sn-platform-bookie
                  cloud.streamnative.io/component=bookie
                  cluster=fujun-test-sn-platform
                  component=bookie
                  controller-revision-hash=fujun-test-sn-platform-bookie-748c7c75c7
                  release=fujun-test
                  statefulset.kubernetes.io/pod-name=fujun-test-sn-platform-bookie-0
    Annotations:  cloud.streamnative.io/checksum-config: 4FD8E7DB3214951E
                  operator-sdk/primary-resource: pulsar/fujun-test-sn-platform-bookie
                  operator-sdk/primary-resource-type: pod
                  prometheus.io/port: 8000
                  prometheus.io/scrape: true
    Status:       Running
    IP:           10.244.0.28
    IPs:
      IP:           10.244.0.28
    Controlled By:  StatefulSet/fujun-test-sn-platform-bookie
    Containers:
      bookie:
        Container ID:  containerd://1eaa7fe7209242249d9cc206ffd34ac2ecc9f42ddf85ccc686f7ffcbe76f2b75
        Image:         streamnative/sn-platform:2.10.1.10
    ```
    

- bookie-1 pod
    
    ```jsx
    
    ➜  ~ kubectl describe pod fujun-test-sn-platform-bookie-1 -n pulsar
    Name:         fujun-test-sn-platform-bookie-1
    Namespace:    pulsar
    Priority:     0
    Node:         aks-agentpool-13502177-vmss000000/10.224.0.4
    Start Time:   Mon, 19 Dec 2022 16:34:28 +0800
    Labels:       app=sn-platform
                  cloud.streamnative.io/app=pulsar
                  cloud.streamnative.io/cluster=fujun-test-sn-platform-bookie
                  cloud.streamnative.io/component=bookie
                  cluster=fujun-test-sn-platform
                  component=bookie
                  controller-revision-hash=fujun-test-sn-platform-bookie-748c7c75c7
                  release=fujun-test
                  statefulset.kubernetes.io/pod-name=fujun-test-sn-platform-bookie-1
    Annotations:  cloud.streamnative.io/checksum-config: 4FD8E7DB3214951E
                  operator-sdk/primary-resource: pulsar/fujun-test-sn-platform-bookie
                  operator-sdk/primary-resource-type: pod
                  prometheus.io/port: 8000
                  prometheus.io/scrape: true
    Status:       Running
    IP:           10.244.0.29
    IPs:
      IP:           10.244.0.29
    Controlled By:  StatefulSet/fujun-test-sn-platform-bookie
    ```
    

- bookie-2 pod
    
    ```jsx
    ➜  ~ kubectl describe pod fujun-test-sn-platform-bookie-2 -n pulsar
    Name:         fujun-test-sn-platform-bookie-2
    Namespace:    pulsar
    Priority:     0
    Node:         aks-agentpool-13502177-vmss000000/10.224.0.4
    Start Time:   Mon, 19 Dec 2022 16:34:28 +0800
    Labels:       app=sn-platform
                  cloud.streamnative.io/app=pulsar
                  cloud.streamnative.io/cluster=fujun-test-sn-platform-bookie
                  cloud.streamnative.io/component=bookie
                  cluster=fujun-test-sn-platform
                  component=bookie
                  controller-revision-hash=fujun-test-sn-platform-bookie-748c7c75c7
                  release=fujun-test
                  statefulset.kubernetes.io/pod-name=fujun-test-sn-platform-bookie-2
    Annotations:  cloud.streamnative.io/checksum-config: 4FD8E7DB3214951E
                  operator-sdk/primary-resource: pulsar/fujun-test-sn-platform-bookie
                  operator-sdk/primary-resource-type: pod
                  prometheus.io/port: 8000
                  prometheus.io/scrape: true
    Status:       Running
    IP:           10.244.0.30
    IPs:
      IP:           10.244.0.30
    Controlled By:  StatefulSet/fujun-test-sn-platform-bookie
    Containers:
      bookie:
        Container ID:  containerd://b99efae1a688a89fc0cce1b78a2e981b531ec5c4a6ef9eb21aaa7dc0183f9b11
        Image:         streamnative/sn-platform:2.10.1.10
    
    ```
    
    # create k8s
    
    - create summary
        
        ![Untitled](azure%20k8s%20ff660755eb0340c99d90fd15053a3a8c/Untitled.png)
        
        ![Untitled](azure%20k8s%20ff660755eb0340c99d90fd15053a3a8c/Untitled%201.png)
        
    
    ## select nodePool
    
    ![Untitled](azure%20k8s%20ff660755eb0340c99d90fd15053a3a8c/Untitled%202.png)
    
    - primary node pool 的az 看起来 和 master节点一致，可以增加自定义的 node pool
        
        ![Untitled](azure%20k8s%20ff660755eb0340c99d90fd15053a3a8c/Untitled%203.png)
        
    
    ## networking
    
    ![Untitled](azure%20k8s%20ff660755eb0340c99d90fd15053a3a8c/Untitled%204.png)
    
    # install snp
    
    ## before install
    
    ```jsx
    ➜  aks kubectl get pod --all-namespaces
    NAMESPACE     NAME                                  READY   STATUS    RESTARTS   AGE
    kube-system   ama-logs-qz2pl                        2/2     Running   0          30m
    kube-system   ama-logs-rs-587f9c87bf-kz2l7          1/1     Running   0          30m
    kube-system   azure-ip-masq-agent-b56xg             1/1     Running   0          30m
    kube-system   cloud-node-manager-c2dwh              1/1     Running   0          30m
    kube-system   coredns-autoscaler-5589fb5654-x2skp   1/1     Running   0          13m
    kube-system   coredns-b4854dd98-548ff               1/1     Running   0          28m
    kube-system   coredns-b4854dd98-mn8rj               1/1     Running   0          13m
    kube-system   csi-azuredisk-node-np9vm              3/3     Running   0          30m
    kube-system   csi-azurefile-node-tb5ps              3/3     Running   0          30m
    kube-system   konnectivity-agent-554c4499fc-fc6ng   1/1     Running   0          101s
    kube-system   konnectivity-agent-554c4499fc-lb2x4   1/1     Running   0          100s
    kube-system   kube-proxy-8lltz                      1/1     Running   0          30m
    kube-system   metrics-server-f77b4cd8-2hkqg         1/1     Running   0          30m
    kube-system   metrics-server-f77b4cd8-d9hr4         1/1     Running   0          13m
    ```
    
    ## after install snp
    
    ### svc
    
    ```jsx
    
    ➜  ~ kubectl get svc -n pulsar
    NAME                                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                                        AGE
    cert-manager                              ClusterIP   10.0.14.245    <none>        9402/TCP                                       27m
    cert-manager-webhook                      ClusterIP   10.0.75.74     <none>        443/TCP                                        27m
    function-mesh-admission-webhook-service   ClusterIP   10.0.248.103   <none>        443/TCP                                        26m
    vault-operator                            ClusterIP   10.0.40.133    <none>        80/TCP,8383/TCP                                27m
    wsg-sn-platform-alert-manager             ClusterIP   10.0.31.159    <none>        9093/TCP                                       15m
    wsg-sn-platform-bookie                    ClusterIP   None           <none>        3181/TCP,8000/TCP                              8m58s
    wsg-sn-platform-broker                    ClusterIP   10.0.168.197   <none>        6650/TCP,8080/TCP,9092/TCP                     15m
    wsg-sn-platform-broker-headless           ClusterIP   None           <none>        6650/TCP,8080/TCP,9092/TCP                     15m
    wsg-sn-platform-grafana                   ClusterIP   None           <none>        3000/TCP                                       15m
    wsg-sn-platform-prometheus                ClusterIP   None           <none>        9090/TCP                                       15m
    wsg-sn-platform-proxy-headless            ClusterIP   None           <none>        6650/TCP,8080/TCP                              15m
    wsg-sn-platform-pulsar-detector           ClusterIP   10.0.9.227     <none>        9000/TCP                                       15m
    wsg-sn-platform-recovery                  ClusterIP   None           <none>        3181/TCP,8000/TCP                              8m58s
    wsg-sn-platform-streamnative-console      ClusterIP   None           <none>        9527/TCP,7750/TCP                              15m
    wsg-sn-platform-toolset                   ClusterIP   None           <none>        <none>                                         15m
    wsg-sn-platform-vault                     ClusterIP   10.0.138.49    <none>        8200/TCP,8201/TCP,9091/TCP,9102/TCP            15m
    wsg-sn-platform-vault-0                   ClusterIP   10.0.185.98    <none>        8200/TCP,8201/TCP,9091/TCP                     15m
    wsg-sn-platform-vault-1                   ClusterIP   10.0.107.6     <none>        8200/TCP,8201/TCP,9091/TCP                     15m
    wsg-sn-platform-vault-2                   ClusterIP   10.0.28.148    <none>        8200/TCP,8201/TCP,9091/TCP                     15m
    wsg-sn-platform-zookeeper                 ClusterIP   None           <none>        2181/TCP,2888/TCP,3888/TCP,8000/TCP,9990/TCP   15m
    ➜  ~
    ```
    
    ### pvc
    
    ```jsx
    ➜  ~ kubectl get pvc -n pulsar
    NAME                                                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    data-wsg-sn-platform-zookeeper-0                             Bound    pvc-c4454763-5530-4f82-b071-de00806c52f1   50Gi       RWO            default        15m
    data-wsg-sn-platform-zookeeper-1                             Bound    pvc-c1d12b3c-b493-432a-8e55-b0fba08fe399   50Gi       RWO            default        15m
    data-wsg-sn-platform-zookeeper-2                             Bound    pvc-0341c27d-3401-4610-9b56-33310575bd2c   50Gi       RWO            default        15m
    journal-0-wsg-sn-platform-bookie-0                           Bound    pvc-a5d16e1c-0c61-460b-a233-854eade7aecc   10Gi       RWO            default        8m29s
    journal-0-wsg-sn-platform-bookie-1                           Bound    pvc-7872b09f-34c6-42a7-96f1-bad7b90544a6   10Gi       RWO            default        8m29s
    journal-0-wsg-sn-platform-bookie-2                           Bound    pvc-2da870ac-b1b9-4c40-a4ae-1e172b7dc82c   10Gi       RWO            default        8m29s
    ledgers-0-wsg-sn-platform-bookie-0                           Bound    pvc-6527eb7b-46b1-4b4b-ad7b-aeae3458b8fe   50Gi       RWO            default        8m29s
    ledgers-0-wsg-sn-platform-bookie-1                           Bound    pvc-33e05ffb-1c42-482c-b172-9c5e09d4c309   50Gi       RWO            default        8m29s
    ledgers-0-wsg-sn-platform-bookie-2                           Bound    pvc-76a557fd-b2d0-4205-adc5-4beecd8466cc   50Gi       RWO            default        8m29s
    wsg-sn-platform-grafana-data                                 Bound    pvc-6a2c061f-94bd-41b1-aed6-c393a2b31943   10Gi       RWO            default        15m
    wsg-sn-platform-prometheus-data                              Bound    pvc-d971c89c-c78e-49ad-a7a7-37ba13ca9294   10Gi       RWO            default        15m
    wsg-sn-platform-streamnative-console-data                    Bound    pvc-b2a1361c-3d8f-4288-91ec-ec9124a7b349   10Gi       RWO            default        15m
    wsg-sn-platform-vault-vault-volume-wsg-sn-platform-vault-0   Bound    pvc-415fc03f-55cf-4b96-8a38-3c97ddd9048e   10Gi       RWO            default        15m
    wsg-sn-platform-vault-vault-volume-wsg-sn-platform-vault-1   Bound    pvc-05e76d73-6844-45d2-945e-2b78e800bdd1   10Gi       RWO            default        11m
    wsg-sn-platform-vault-vault-volume-wsg-sn-platform-vault-2   Bound    pvc-e9f0d00d-de8d-4e2b-8fb1-b9d535db2db3   10Gi       RWO            default        10m
    ```
    
    ## perf
    
    ```jsx
    oduced: 2898472 msg ---  10006.3 msg/s ---     78.2 Mbit/s  --- failure      0.0 msg/s --- Latency: mean:  11.345 ms - med:  10.397 - 95pct:  18.542 - 99pct:  24.808 - 99.9pct:  32.215 - 99.99pct:  39.591 - Max:  39.593
    10:58:11.176 [main] INFO  org.apache.pulsar.testclient.PerformanceProducer - Throughput produced: 2998547 msg ---   9996.7 msg/s ---     78.1 Mbit/s  --- failure      0.0 msg/s --- Latency: mean:  11.394 ms - med:  10.423 - 95pct:  18.785 - 99pct:  25.631 - 99.9pct:  32.515 - 99.99pct:  35.776 - Max:  35.783
    10:58:11.302 [pulsar-client-io-2-1] INFO  org.apache.pulsar.client.impl.ConnectionPool - [[id: 0x96a5c05e, L:/10.244.1.25:43328 - R:wsg-sn-platform-broker.pulsar.svc.cluster.local/10.0.168.197:6650]] Connected to server
    10:58:21.187 [main] INFO  org.apache.pulsar.testclient.PerformanceProducer - Throughput produced: 3098677 msg ---  10002.2 msg/s ---     78.1 Mbit/s  --- failure      0.0 msg/s --- Latency: mean:  11.277 ms - med:  10.585 - 95pct:  17.273 - 99pct:  23.326 - 99.9pct:  27.705 - 99.99pct:  30.013 - Max:  30.020
    10:58:31.200 [main] INFO  org.apache.pulsar.testclient.PerformanceProducer - Throughput produced: 3198801 msg ---  10000.6 msg/s ---     78.1 Mbit/s  --- failure      0.0 msg/s --- Latency: mean:  11.447 ms - med:  10.488 - 95pct:  18.777 - 99pct:  26.807 - 99.9pct:  36.903 - 99.99pct:  43.360 - Max:  43.375
    10:58:41.211 [main] INFO  org.apache.pulsar.testclient.PerformanceProducer - Throughput produced: 3298887 msg ---   9997.1 msg/s ---     78.1 Mbit/s  --- failure      0.0 msg/s --- Latency: mean:  11.455 ms - med:  10.547 - 95pct:  18.817 - 99pct:  25.789 - 99.9pct:  30.322 - 99.99pct:  32.221 - Max:  32.242
    ^C10:58:43.280 [Thread-0] INFO  org.apache.pulsar.testclient.PerformanceProducer - Aggregated throughput stats --- 3319675 records sent --- 9981.507 msg/s --- 77.981 Mbit/s
    10:58:43.304 [Thread-0] INFO  org.apache.pulsar.testclient.PerformanceProducer - Aggregated latency stats --- Latency: mean:  19.601 ms - med:  10.592 - 95pct:  20.823 - 99pct:  37.681 - 99.9pct: 1802.839 - 99.99pct: 2068.023 - 99.999pct: 2095.071 - Max: 2098.223
    root@wsg-sn-platform-toolset-0:/pulsar# ^C
    ```
    

## expose service

```jsx

```

# add bookie

modify value.yml. ,bookie replica to 4

then

```jsx
➜  aks helm upgrade -f values.yaml $RELEASENAME streamnative/sn-platform -n $NAMESPACE
Release "wsg" has been upgraded. Happy Helming!
NAME: wsg
LAST DEPLOYED: Tue Jan  3 19:46:58 2023
NAMESPACE: pulsar
STATUS: deployed
REVISION: 3
TEST SUITE: None

```

trigger aks add k8s node

```jsx
Type     Reason                  Age                    From                     Message
  ----     ------                  ----                   ----                     -------
  Normal   TriggeredScaleUp        4m28s                  cluster-autoscaler       pod triggered scale-up: [{aks-agentpool-36826358-vmss 3->4 (max: 5)}]
  Warning  FailedScheduling        3m38s (x2 over 4m40s)  default-scheduler        0/3 nodes are available: 1 Insufficient cpu, 2 node(s) exceed max volume count.
  Warning  FailedScheduling        3m35s                  default-scheduler        0/4 nodes are available: 1 Insufficient cpu, 1 node(s) had taint {node.cloudprovider.kubernetes.io/uninitialized: true}, that the pod didn't tolerate, 2 node(s) exceed max volume count.
  Normal   Scheduled               2m35s                  default-scheduler        Successfully assigned pulsar/wsg-sn-platform-bookie-3 to aks-agentpool-36826358-vmss000005
  Normal   SuccessfulAttachVolume  2m23s                  attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-600fced1-bc07-49c9-ba69-4d33622583ce"
  Normal   SuccessfulAttachVolume  117s                   attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-3fc6f168-46ed-45b5-bce7-88bb7755aaae"
  Normal   Pulling                 115s                   kubelet                  Pulling image "streamnative/sn-platform:2.10.1.11"

```

check pod 

```jsx

wsg-sn-platform-bookie-0                                         1/1     Running             0             65m
wsg-sn-platform-bookie-1                                         1/1     Running             0             65m
wsg-sn-platform-bookie-2                                         1/1     Running             0             65m
wsg-sn-platform-bookie-3                                         0/1     ContainerCreating   0             2m42s
```

check

```jsx
wsg-sn-platform-bookie-0                                         1/1     Running   0             70m
wsg-sn-platform-bookie-1                                         1/1     Running   0             70m
wsg-sn-platform-bookie-2                                         1/1     Running   0             70m
wsg-sn-platform-bookie-3                                         1/1     Running   0             7m46s

```

- operator log
    
    ```jsx
    {"severity":"info","timestamp":"2023-01-03T11:53:40Z","logger":"controllers.BookKeeperCluster","message":"Reconciling BookKeeperCluster","Request.Namespace":"pulsar","Request.Name":"wsg-sn-platform-bookie"}
    {"severity":"info","timestamp":"2023-01-03T11:53:40Z","logger":"controllers.BookKeeperCluster","message":"Updating the status for the BookKeeperCluster","Namespace":"pulsar","Name":"wsg-sn-platform-bookie","Status":{"observedGeneration":4,"replicas":4,"readyReplicas":4,"updatedReplicas":4,"labelSelector":"cloud.streamnative.io/app=pulsar,cloud.streamnative.io/cluster=wsg-sn-platform-bookie,cloud.streamnative.io/component=bookie","conditions":[{"type":"AutoRecovery","status":"True","reason":"Deployed","message":"Ready","lastTransitionTime":"2023-01-03T10:48:43Z"},{"type":"Bookie","status":"True","reason":"Ready","message":"Bookies are ready","lastTransitionTime":"2023-01-03T11:47:58Z"},{"type":"Initialization","status":"True","reason":"Initialization","message":"Initialization succeeded","lastTransitionTime":"2023-01-03T10:45:35Z"},{"type":"Ready","status":"True","reason":"Ready","lastTransitionTime":"2023-01-03T11:47:58Z"}]}}
    {"severity":"error","timestamp":"2023-01-03T11:53:40Z","logger":"controller.bookkeepercluster","message":"Reconciler error","reconciler group":"bookkeeper.streamnative.io","reconciler kind":"BookKeeperCluster","name":"wsg-sn-platform-bookie","namespace":"pulsar","error":"Operation cannot be fulfilled on bookkeeperclusters.bookkeeper.streamnative.io \"wsg-sn-platform-bookie\": the object has been modified; please apply your changes to the latest version and try again","stacktrace":"sigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).Start.func2.2\n\t/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.10.3/pkg/internal/controller/controller.go:227"}
    {"severity":"info","timestamp":"2023-01-03T11:53:40Z","logger":"controllers.BookKeeperCluster","message":"Reconciling BookKeeperCluster","Request.Namespace":"pulsar","Request.Name":"wsg-sn-platform-bookie"}
    {"severity":"info","timestamp":"2023-01-03T11:53:40Z","logger":"controllers.BookKeeperCluster","message":"Updating the status for the BookKeeperCluster","Namespace":"pulsar","Name":"wsg-sn-platform-bookie","Status":{"observedGeneration":4,"replicas":4,"readyReplicas":4,"updatedReplicas":4,"labelSelector":"cloud.streamnative.io/app=pulsar,cloud.streamnative.io/cluster=wsg-sn-platform-bookie,cloud.streamnative.io/component=bookie","conditions":[{"type":"AutoRecovery","status":"True","reason":"Deployed","message":"Ready","lastTransitionTime":"2023-01-03T10:48:43Z"},{"type":"Bookie","status":"True","reason":"Ready","message":"Bookies are ready","lastTransitionTime":"2023-01-03T11:47:58Z"},{"type":"Initialization","status":"True","reason":"Initialization","message":"Initialization succeeded","lastTransitionTime":"2023-01-03T10:45:35Z"},{"type":"Ready","status":"True","reason":"Ready","lastTransitionTime":"2023-01-03T11:47:58Z"}]}}
    Logs from 03/01/2023, 18:28:20
    Show timestamps
    ```
    

# delete bookie

- prepare
    
    ```jsx
    root@wsg-sn-platform-toolset-0:/pulsar# pulsar-admin topics list public/default
    persistent://public/default/__sla
    persistent://public/default/perf1-partition-0
    persistent://public/default/__aliveness
    root@wsg-sn-platform-toolset-0:/pulsar# pulsar-admin topics partitioned-stats public/default/perf1
    {
      "msgRateIn" : 0.0,
      "msgThroughputIn" : 0.0,
      "msgRateOut" : 0.0,
      "msgThroughputOut" : 0.0,
      "bytesInCounter" : 3454020180,
      "msgInCounter" : 3321311,
      "bytesOutCounter" : 0,
      "msgOutCounter" : 0,
      "averageMsgSize" : 0.0,
      "msgChunkPublished" : false,
      "storageSize" : 3457199601,
      "backlogSize" : 0,
      "publishRateLimitedTimes"
    ```
    
- modify value
    
    
    - 可以看到 bookie-decommission，
        
        看起来是 删除哪个bookieid，就会生成对应的 decommission pod
        
        ```jsx
        
        wsg-sn-platform-bookie-0                                         1/1     Running       0             77m
        wsg-sn-platform-bookie-1                                         1/1     Running       0             77m
        wsg-sn-platform-bookie-2                                         1/1     Running       0             77m
        wsg-sn-platform-bookie-3                                         0/1     Terminating   0             15m
        wsg-sn-platform-bookie-3-decommission-t4f62                      1/1     Running       0             28s
        wsg-sn-platform-broker-0                                         1/1     Running       0             77m
        wsg-sn-platform-grafana-0                                        1/1     Running       0             84m
        wsg-sn-platform-node-exporter-5sbcc                              1/1     Running       0             84m
        ```
        
- 但是 很快，该pod 就会 自动删除
    
    ```jsx
    
    wsg-sn-platform-bookie-0                                         1/1     Running   0             78m
    wsg-sn-platform-bookie-1                                         1/1     Running   0             78m
    wsg-sn-platform-bookie-2                                         1/1     Running   0             78m
    wsg-sn-platform-broker-0                                         1/1     Running   0             78m
    wsg-sn-platform-grafana-0                                        1/1     Running   0             8
    ```
    

- check bookie operator log
    
    
    - detect
        
        ```jsx
        
        2023-01-03T19:53:40+08:00 {"severity":"info","timestamp":"2023-01-03T11:53:40Z","logger":"controllers.BookKeeperCluster","message":"Updating the status for the BookKeeperCluster","Namespace":"pulsar","Name":"wsg-sn-platform-bookie","Status":{"observedGeneration":4,"replicas":4,"readyReplicas":4,"updatedReplicas":4,"labelSelector":"cloud.streamnative.io/app=pulsar,cloud.streamnative.io/cluster=wsg-sn-platform-bookie,cloud.streamnative.io/component=bookie","conditions":[{"type":"AutoRecovery","status":"True","reason":"Deployed","message":"Ready","lastTransitionTime":"2023-01-03T10:48:43Z"},{"type":"Bookie","status":"True","reason":"Ready","message":"Bookies are ready","lastTransitionTime":"2023-01-03T11:47:58Z"},{"type":"Initialization","status":"True","reason":"Initialization","message":"Initialization succeeded","lastTransitionTime":"2023-01-03T10:45:35Z"},{"type":"Ready","status":"True","reason":"Ready","lastTransitionTime":"2023-01-03T11:47:58Z"}]}}
        2023-01-03T20:02:36+08:00 {"severity":"info","timestamp":"2023-01-03T12:02:36Z","logger":"controllers.BookKeeperCluster","message":"Reconciling BookKeeperCluster","Request.Namespace":"pulsar","Request.Name":"wsg-sn-platform-bookie"}
        2023-01-03T20:02:36+08:00 {"severity":"info","timestamp":"2023-01-03T12:02:36Z","logger":"StatefulSetReconciler","message":"Scaling BookKeeper","Namespace":"pulsar","Name":"wsg-sn-platform-bookie","Namespace":"pulsar","Name":"wsg-sn-platform-bookie","From":4,"To":4,"Desired":3}
        2023-01-03T20:02:36+08:00 {"severity":"info","timestamp":"2023-01-03T12:02:36Z","logger":"StatefulSetReconciler","message":"Updated resource","Namespace":"pulsar","Name":"wsg-sn-platform-bookie"}
        2023-01-03T20:02:36+08:00 {"severity":"info","timestamp":"2023-01-03T12:02:36Z","logger":"controllers.BookKeeperCluster","message":"Updating the status for the BookKeeperCluster","Namespace":"pulsar","Name":"wsg-sn-platform-bookie","Status":{"observedGeneration":5,"replicas":4,"readyReplicas":4,"updatedReplicas":4,"labelSelector":"cloud.streamnative.io/app=pulsar,cloud.streamnative.io/cluster=wsg-sn-platform-bookie,cloud.streamnative.io/component=bookie","conditions":[{"type":"AutoRecovery","status":"True","reason":"Deployed","message":"Ready","lastTransitionTime":"2023-01-03T10:48:43Z"},{"type":"Bookie","status":"True","reason":"Ready","message":"Bookies are ready","lastTransitionTime":"2023-01-03T11:47:58Z"},{"type":"Initialization","status":"True","reason":"Initialization","message":"Initialization succeeded","lastTransitionTime":"2023-01-03T10:45:35Z"},{"type":"Ready","status":"False","reason":"Ready","message":"Scaling cluster","lastTransitionTime":"2023-01-03T12:02:36Z"}]}}
        ```
        
    

# delete bookie pod

- prepare check pvc
    
    ```jsx
    journal-0-wsg-sn-platform-bookie-0                             Bound    pvc-a5d16e1c-0c61-460b-a233-854eade7aecc   10Gi       RWO            default        40h
    journal-0-wsg-sn-platform-bookie-1                             Bound    pvc-7872b09f-34c6-42a7-96f1-bad7b90544a6   10Gi       RWO            default        40h
    journal-0-wsg-sn-platform-bookie-2                             Bound    pvc-2da870ac-b1b9-4c40-a4ae-1e172b7dc82c   10Gi       RWO            default        40h
    
    //
    ledgers-0-wsg-sn-platform-bookie-0                             Bound    pvc-6527eb7b-46b1-4b4b-ad7b-aeae3458b8fe   50Gi       RWO            default        40h
    ledgers-0-wsg-sn-platform-bookie-1                             Bound    pvc-33e05ffb-1c42-482c-b172-9c5e09d4c309   50Gi       RWO            default        40h
    ledgers-0-wsg-sn-platform-bookie-2                             Bound    pvc-76a557fd-b2d0-4205-adc5-4beecd8466cc   50Gi       RWO            default        40h
    ```
    
- check topics
    
    
    ```jsx
    root@wsg-sn-platform-toolset-0:/pulsar# pulsar-admin topics list public/default
    persistent://public/default/__sla
    persistent://public/default/perf1-partition-0
    persistent://public/default/perf2-partition-0
    persistent://public/default/__aliveness
    root@wsg-sn-platform-toolset-0:/pulsar#
    ```
    

- check bookie pod on which node
    
    ```jsx
    ➜  aks kubectl get pod -n pulsar -o wide
    NAME                                                             READY   STATUS             RESTARTS          AGE   IP            NODE                                NOMINATED NODE   READINESS GATES
    cert-manager-7599c44747-b5fqt                                    1/1     Running            0                 43h   10.244.1.14   aks-agentpool-36826358-vmss000001   <none>           <none>
    cert-manager-cainjector-7d9466748-znk97                          1/1     Running            0                 43h   10.244.1.15   aks-agentpool-36826358-vmss000001   <none>           <none>
    cert-manager-webhook-d77bbf4cb-gv5d7                             1/1     Running            0                 43h   10.244.1.16   aks-agentpool-36826358-vmss000001   <none>           <none>
    function-mesh-controller-manager-8848bd5b8-hjcks                 1/1     Running            0                 43h   10.244.1.21   aks-agentpool-36826358-vmss000001   <none>           <none>
    pulsar-operator-bookkeeper-controller-manager-669b58bfb4-vpqlf   1/1     Running            0                 43h   10.244.1.19   aks-agentpool-36826358-vmss000001   <none>           <none>
    pulsar-operator-pulsar-controller-manager-5f9d7fdc54-v4zxs       1/1     Running            0                 43h   10.244.1.18   aks-agentpool-36826358-vmss000001   <none>           <none>
    pulsar-operator-zookeeper-controller-manager-84744d7bb4-qcbjk    1/1     Running            0                 43h   10.244.1.20   aks-agentpool-36826358-vmss000001   <none>           <none>
    vault-operator-558cf7bd5b-ws6ph                                  1/1     Running            0                 43h   10.244.1.13   aks-agentpool-36826358-vmss000001   <none>           <none>
    wsg-sn-platform-alert-manager-8676999f9b-krcx9                   2/2     Running            0                 42h   10.244.1.23   aks-agentpool-36826358-vmss000001   <none>           <none>
    wsg-sn-platform-bookie-0                                         1/1     Running            0                 42h   10.244.4.7    aks-agentpool-36826358-vmss000004   <none>           <none>
    wsg-sn-platform-bookie-1                                         1/1     Running            0                 42h   10.244.3.14   aks-agentpool-36826358-vmss000003   <none>           <none>
    wsg-sn-platform-bookie-2                                         1/1     Running            0                 42h   10.244.4.8    aks-agentpool-36826358-vmss000004   <none>           <none>
    wsg-sn-platform-broker-0                                         1/1     Running            0                 42h   10.244.4.6    aks-agentpool-36826358-vmss000004   <none>           <none>
    wsg-sn-platform-grafana-0                                        1/1     Running            0                 42h   10.244.3.9    aks-agentpool-36826358-vmss000003   <none>           <none>
    wsg-sn-platform-node-exporter-5sbcc                              1/1     Running            0                 42h   10.224.0.5    aks-agentpool-36826358-vmss000001   <none>           <none>
    wsg-sn-platform-node-exporter-8pzfq                              1/1     Running            0                 25h   10.224.0.7    aks-agentpool-36826358-vmss000006   <none>           <none>
    wsg-sn-platform-node-exporter-vwng5                              1/1     Running            0                 42h   10.224.0.4    aks-agentpool-36826358-vmss000003   <none>           <none>
    wsg-sn-platform-node-exporter-wvbzs                              1/1     Running            0                 42h   10.224.0.6    aks-agentpool-36826358-vmss000004   <none>           <none>
    wsg-sn-platform-prometheus-0                                     2/2     Running            0                 42h   10.244.3.13   aks-agentpool-36826358-vmss000003   <none>           <none>
    wsg-sn-platform-proxy-0                                          1/1     Running            0                 42h   10.244.3.3    aks-agentpool-36826358-vmss000003   <none>           <none>
    wsg-sn-platform-pulsar-detector-7576d858b9-n9ztq                 1/1     Running            6 (42h ago)       42h   10.244.1.22   aks-agentpool-36826358-vmss000001   <none>           <none>
    wsg-sn-platform-recovery-0                                       1/1     Running            0                 42h   10.244.3.15   aks-agentpool-36826358-vmss000003   <none>           <none>
    wsg-sn-platform-streamnative-console-0                           1/1     Running            0                 42h   10.244.1.24   aks-agentpool-36826358-vmss000001   <none>           <none>
    wsg-sn-platform-toolset-0                                        1/1     Running            0                 42h   10.244.1.25   aks-agentpool-36826358-vmss000001   <none>           <none>
    ```
    
    ```jsx
    wsg-sn-platform-bookie-0                                         1/1     Running            0                 42h   10.244.4.7    aks-agentpool-36826358-vmss000004   <none>           <none>
    wsg-sn-platform-bookie-1                                         1/1     Running            0                 42h   10.244.3.14   aks-agentpool-36826358-vmss000003   <none>           <none>
    wsg-sn-platform-bookie-2                                         1/1     Running            0                 42h   10.244.4.8    aks-agentpool-36826358-vmss000004   <none>           <none>
    wsg-sn-platform-broker-0                                         1/1     Running            0                 42h   10.244.4.6    aks-agentpool-36826358-vmss000004   <none>           <none>
    wsg-sn-platform-grafana-0  
    ```
    
    node info
    
    ```jsx
    ➜  aks kubectl get node -o wide
    NAME                                STATUS   ROLES   AGE   VERSION    INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
    aks-agentpool-36826358-vmss000001   Ready    agent   43h   v1.23.12   10.224.0.5    <none>        Ubuntu 18.04.6 LTS   5.4.0-1098-azure   containerd://1.6.4+azure-4
    aks-agentpool-36826358-vmss000003   Ready    agent   42h   v1.23.12   10.224.0.4    <none>        Ubuntu 18.04.6 LTS   5.4.0-1098-azure   containerd://1.6.4+azure-4
    aks-agentpool-36826358-vmss000004   Ready    agent   42h   v1.23.12   10.224.0.6    <none>        Ubuntu 18.04.6 LTS   5.4.0-1098-azure   containerd://1.6.4+azure-4
    aks-agentpool-36826358-vmss000006   Ready    agent   25h   v1.23.12   10.224.0.7    <none>        Ubuntu 18.04.6 LTS   5.4.0-1098-azure   containerd://1.6.4+azure-4
    ➜  aks 
    ```
    
- then delete bookie-0
    
    ```jsx
    wsg-sn-platform-bookie-0                                         1/1     Running            0                 99s   10.244.4.11   aks-agentpool-36826358-vmss000004   <none>           <none>
    wsg-sn-platform-bookie-1                                         1/1     Running            0                 43h   10.244.3.14   aks-agentpool-36826358-vmss000003   <none>           <none>
    wsg-sn-platform-bookie-2                                         1/1     Running            0                 43h   10.244.4.8    aks-agentpool-36826358-vmss000004   <none>           <none>
    wsg-sn-platform-broker-0                                         1/1     Running            0                 43h   10.244.4.6    aks-agentpool-36826358-vmss000004   <none>           <none>
    ```
    
    bookie-0 还是调度到了 原来的节点，
    
    - check pvc
        
        uuid 和bookie-0 之前一样
        
        ```jsx
        journal-0-wsg-sn-platform-bookie-0                             Bound    pvc-a5d16e1c-0c61-460b-a233-854eade7aecc   10Gi       RWO            default        43h
        journal-0-wsg-sn-platform-bookie-1                             Bound    pvc-7872b09f-34c6-42a7-96f1-bad7b90544a6   10Gi       RWO            default        43h
        journal-0-wsg-sn-platform-bookie-2                             Bound    pvc-2da870ac-b1b9-4c40-a4ae-1e172b7dc82c   10Gi       RWO            default        43h
        
        ledgers-0-wsg-sn-platform-bookie-0                             Bound    pvc-6527eb7b-46b1-4b4b-ad7b-aeae3458b8fe   50Gi       RWO            default        43h
        ledgers-0-wsg-sn-platform-bookie-1                             Bound    pvc-33e05ffb-1c42-482c-b172-9c5e09d4c309   50Gi       RWO            default        43h
        ledgers-0-wsg-sn-platform-bookie-2                             Bound    pvc-76a557fd-b2d0-4205-adc5-4beecd8466cc   50Gi       RWO            default        43h
        
        ```
        
    - check recovery pod log
        
        ```jsx
        2023-01-05T05:52:25,128+0000 [main-EventThread] INFO  org.apache.bookkeeper.discover.ZKRegistrationClient - Invalidate cache for wsg-sn-platform-bookie-0.wsg-sn-platform-bookie.pulsar.svc.cluster.local:3181
        2023-01-05T05:52:25,130+0000 [main-EventThread] INFO  org.apache.bookkeeper.discover.ZKRegistrationClient - Invalidate cache for wsg-sn-platform-bookie-0.wsg-sn-platform-bookie.pulsar.svc.cluster.local:3181
        2023-01-05T05:52:25,132+0000 [BookKeeperClientScheduler-OrderedScheduler-0-0] INFO  org.apache.bookkeeper.net.NetworkTopologyImpl - Removing a node: /default-rack/wsg-sn-platform-bookie-0.wsg-sn-platform-bookie.pulsar.svc.cluster.local:3181
        2023-01-05T05:52:25,133+0000 [BookKeeperClientScheduler-OrderedScheduler-0-0] INFO  org.apache.bookkeeper.net.NetworkTopologyImpl - Removing a node: /default-rack/wsg-sn-platform-bookie-0.wsg-sn-platform-bookie.pulsar.svc.cluster.local:3181
        2023-01-05T05:52:25,136+0000 [AuditorBookie-wsg-sn-platform-recovery-0.wsg-sn-platform-bookie.pulsar.svc.cluster.local:3181] INFO  org.apache.bookkeeper.replication.Auditor - Delaying bookie audit by 60 secs for [wsg-sn-platform-bookie-0.wsg-sn-platform-bookie.pulsar.svc.cluster.local:3181]
        2023-01-05T05:52:37,558+0000 [main-EventThread] INFO  org.apache.bookkeeper.discover.ZKRegistrationClient - Update BookieInfoCache (writable bookie) wsg-sn-platform-bookie-0.wsg-sn-platform-bookie.pulsar.svc.cluster.local:3181 -> BookieServiceInfo{properties={}, endpoints=[EndpointInfo{id=httpserver, port=8000, host=0.0.0.0, protocol=http, auth=[], extensions=[]}, EndpointInfo{id=bookie, port=3181, host=wsg-sn-platform-bookie-0.wsg-sn-platform-bookie.pulsar.svc.cluster.local, protocol=bookie-rpc, auth=[], extensions=[]}]}
        2023-01-05T05:52:37,564+0000 [main-EventThread] INFO  org.apache.bookkeeper.discover.ZKRegistrationClient - Update BookieInfoCache (writable bookie) wsg-sn-platform-bookie-0.wsg-sn-platform-bookie.pulsar.svc.cluster.local:3181 -> BookieServiceInfo{properties={}, endpoints=[EndpointInfo{id=httpserver, port=8000, host=0.0.0.0, protocol=http, auth=[], extensions=[]}, EndpointInfo{id=bookie, port=3181, host=wsg-sn-platform-bookie-0.wsg-sn-platform-bookie.pulsar.svc.cluster.local, protocol=bookie-rpc, auth=[], extensions=[]}]}
        2023-01-05T05:52:37,576+0000 [BookKeeperClientScheduler-OrderedScheduler-0-0] WARN  org.apache.bookkeeper.client.TopologyAwareEnsemblePlacementPolicy - Failed to resolve network location for wsg-sn-platform-bookie-0.wsg-sn-platform-bookie.pulsar.svc.cluster.local, using default rack for it : /default-rack.
        2023-01-05T05:52:37,576+0000 [BookKeeperClientScheduler-OrderedScheduler-0-0] WARN  org.apache.bookkeeper.client.TopologyAwareEnsemblePlacementPolicy - Failed to resolve network location for wsg-sn-platform-bookie-0.wsg-sn-platform-bookie.pulsar.svc.cluster.local, using default rack for it : /default-rack.
        2023-01-05T05:52:37,581+0000 [BookKeeperClientScheduler-OrderedScheduler-0-0] INFO  org.apache.bookkeeper.net.NetworkTopologyImpl - Adding a new node: /default-rack/wsg-sn-platform-bookie-0.wsg-sn-platform-bookie.pulsar.svc.cluster.local:3181
        2023-01-05T05:52:37,581+0000 [BookKeeperClientScheduler-OrderedScheduler-0-0] INFO  org.apache.bookkeeper.net.NetworkTopologyImpl - Adding a new node: /default-rack/wsg-sn-platform-bookie-0.wsg-sn-platform-bookie.pulsar.svc.clu
        ```
        

# delete bookie pod + node cordon

manual expand k8s node,but bookie-0 can not schedule to other node 

- describe bookie-0
    
    ```jsx
    ➜  aks kubectl describe pod wsg-sn-platform-bookie-0 -n pulsar
    Name:           wsg-sn-platform-bookie-0
    Namespace:      pulsar
    Priority:       0
    Node:           <none>
    Labels:         app=sn-platform
                    cloud.streamnative.io/app=pulsar
                    cloud.streamnative.io/cluster=wsg-sn-platform-bookie
                    cloud.streamnative.io/component=bookie
                    cluster=wsg-sn-platform
                    component=bookie
                    controller-revision-hash=wsg-sn-platform-bookie-f9bd7db65
                    release=wsg
                    statefulset.kubernetes.io/pod-name=wsg-sn-platform-bookie-0
    Annotations:    cloud.streamnative.io/checksum-config: 07CD171855291DAD
                    operator-sdk/primary-resource: pulsar/wsg-sn-platform-bookie
                    operator-sdk/primary-resource-type: pod
                    prometheus.io/port: 8000
                    prometheus.io/scrape: true
    Status:         Pending
    IP:
    IPs:            <none>
    Controlled By:  StatefulSet/wsg-sn-platform-bookie
    Containers:
      bookie:
        Image:       streamnative/sn-platform:2.10.1.11
        Ports:       8000/TCP, 3181/TCP
        Host Ports:  0/TCP, 0/TCP
        Command:
          bash
          -c
        Args:
          bin/apply-config-from-env.py conf/bookkeeper.conf &&
                if [ -x scripts/run-bookie.sh ];then exec scripts/run-bookie.sh;else exec bin/pulsar bookie; fi
        Requests:
          cpu:      200m
          memory:   512Mi
        Liveness:   http-get http://:http/api/v1/bookie/state delay=30s timeout=5s period=3s #success=1 #failure=10
        Readiness:  http-get http://:http/api/v1/bookie/is_ready delay=30s timeout=5s period=3s #success=1 #failure=10
        Startup:    http-get http://:http/api/v1/bookie/is_ready delay=0s timeout=5s period=3s #success=1 #failure=200
        Environment Variables from:
          wsg-sn-platform-bookie  ConfigMap  Optional: false
        Environment:              <none>
        Mounts:
          /pulsar/data/bookkeeper/journal-0 from journal-0 (rw)
          /pulsar/data/bookkeeper/ledgers-0 from ledgers-0 (rw)
          /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-l2sgj (ro)
    Conditions:
      Type           Status
      PodScheduled   False
    Volumes:
      journal-0:
        Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
        ClaimName:  journal-0-wsg-sn-platform-bookie-0
        ReadOnly:   false
      ledgers-0:
        Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
        ClaimName:  ledgers-0-wsg-sn-platform-bookie-0
        ReadOnly:   false
      kube-api-access-l2sgj:
        Type:                    Projected (a volume that contains injected data from multiple sources)
        TokenExpirationSeconds:  3607
        ConfigMapName:           kube-root-ca.crt
        ConfigMapOptional:       <nil>
        DownwardAPI:             true
    QoS Class:                   Burstable
    Node-Selectors:              <none>
    Tolerations:                 node.kubernetes.io/memory-pressure:NoSchedule op=Exists
                                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
    Events:
      Type     Reason             Age                 From                Message
      ----     ------             ----                ----                -------
      Warning  FailedScheduling   17m                 default-scheduler   0/4 nodes are available: 1 node(s) had volume node affinity conflict, 1 node(s) were unschedulable, 2 Insufficient cpu.
      Warning  FailedScheduling   17m                 default-scheduler   0/4 nodes are available: 1 node(s) had volume node affinity conflict, 1 node(s) were unschedulable, 2 Insufficient cpu.
      Warning  FailedScheduling   10m                 default-scheduler   0/4 nodes are available: 1 node(s) had volume node affinity conflict, 1 node(s) were unschedulable, 2 Insufficient cpu.
      Warning  FailedScheduling   9m12s               default-scheduler   0/5 nodes are available: 1 node(s) had taint {node.kubernetes.io/network-unavailable: }, that the pod didn't tolerate, 1 node(s) had volume node affinity conflict, 1 node(s) were unschedulable, 2 Insufficient cpu.
      Warning  FailedScheduling   9m2s                default-scheduler   0/5 nodes are available: 1 node(s) had taint {node.kubernetes.io/network-unavailable: }, that the pod didn't tolerate, 1 node(s) had volume node affinity conflict, 1 node(s) were unschedulable, 2 Insufficient cpu.
      Warning  FailedScheduling   7m32s               default-scheduler   0/5 nodes are available: 1 node(s) were unschedulable, 2 Insufficient cpu, 2 node(s) had volume node affinity conflict.
      Normal   NotTriggerScaleUp  12m (x31 over 17m)  cluster-autoscaler  pod didn't trigger scale-up: 1 node(s) had volume node affinity conflict
    ➜  aks
    ```
    

# delete bookie pod (node pool in one az)

pv

- prepare check pvc and pv
    
    ```jsx
    ➜  aks kubectl get pvc -A |grep -v wsg6
    NAMESPACE   NAME                                                           STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    pulsar      data-wsg-sn-platform-zookeeper-0                               Bound     pvc-7b54b5da-43ee-4dee-a4f9-4459756b459a   50Gi       RWO            default        10h
    pulsar      journal-0-wsg-sn-platform-bookie-0                             Bound     pvc-aa7b07ca-cdcb-48cb-88ad-07db297aef05   10Gi       RWO            default        10h
    pulsar      journal-0-wsg-sn-platform-bookie-1                             Bound     pvc-e1cfbbbc-5701-4ed5-8651-4a96befdb46a   10Gi       RWO            default        10h
    pulsar      journal-0-wsg-sn-platform-bookie-2                             Bound     pvc-5812122b-852c-4459-841f-06b00a11c695   10Gi       RWO            default        10h
    pulsar      ledgers-0-wsg-sn-platform-bookie-0                             Bound     pvc-5379d9b8-c612-4bef-b757-c01ddae6cbde   50Gi       RWO            default        10h
    pulsar      ledgers-0-wsg-sn-platform-bookie-1                             Bound     pvc-5a4d22e6-7a23-4c5e-9aba-4909dc6292d4   50Gi       RWO            default        10h
    pulsar      ledgers-0-wsg-sn-platform-bookie-2                             Bound     pvc-4203ed69-1dd9-47a8-9de0-0bdf5b61a54b   50Gi       RWO            default        10h
    pulsar      wsg-sn-platform-grafana-data                                   Bound     pvc-df9b2c9b-0664-45a4-93f9-397637b2c16c   10Gi       RWO            default        10h
    pulsar      wsg-sn-platform-prometheus-data                                Bound     pvc-aff30c6e-9bcb-49d6-be40-a1543ed5f2b3   10Gi       RWO            default        10h
    pulsar      wsg-sn-platform-streamnative-console-data                      Bound     pvc-f6ae828e-4f88-4102-858a-10b0c2bb3e8a   10Gi       RWO            default        10h
    pulsar      wsg-sn-platform-vault-vault-volume-wsg-sn-platform-vault-0     Bound     pvc-ed1478f6-cd70-4838-918b-f01b8f3ac361   10Gi       RWO            default        10h
    pulsar      wsg-sn-platform-vault-vault-volume-wsg-sn-platform-vault-1     Bound     pvc-1a8795b3-7e6a-4991-8575-efbae0e8ab1d   10Gi       RWO            default        10h
    pulsar      wsg-sn-platform-vault-vault-volume-wsg-sn-platform-vault-2     Bound     pvc-8a2b38ae-7f16-4c10-8df1-31a94eec0760   10Gi       RWO            default        10h
    pulsar
    
    //pv
    ➜  aks kubectl get pv |grep -v wsg6
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                                 STORAGECLASS   REASON   AGE
    pvc-1a8795b3-7e6a-4991-8575-efbae0e8ab1d   10Gi       RWO            Delete           Bound    pulsar/wsg-sn-platform-vault-vault-volume-wsg-sn-platform-vault-1     default                 10h
    pvc-4203ed69-1dd9-47a8-9de0-0bdf5b61a54b   50Gi       RWO            Delete           Bound    pulsar/ledgers-0-wsg-sn-platform-bookie-2                             default                 10h
    pvc-5379d9b8-c612-4bef-b757-c01ddae6cbde   50Gi       RWO            Delete           Bound    pulsar/ledgers-0-wsg-sn-platform-bookie-0                             default                 10h
    pvc-5812122b-852c-4459-841f-06b00a11c695   10Gi       RWO            Delete           Bound    pulsar/journal-0-wsg-sn-platform-bookie-2                             default                 10h
    pvc-5a4d22e6-7a23-4c5e-9aba-4909dc6292d4   50Gi       RWO            Delete           Bound    pulsar/ledgers-0-wsg-sn-platform-bookie-1                             default                 10h
    pvc-7b54b5da-43ee-4dee-a4f9-4459756b459a   50Gi       RWO            Delete           Bound    pulsar/data-wsg-sn-platform-zookeeper-0                               default                 10h
    pvc-8a2b38ae-7f16-4c10-8df1-31a94eec0760   10Gi       RWO            Delete           Bound    pulsar/wsg-sn-platform-vault-vault-volume-wsg-sn-platform-vault-2     default                 10h
    pvc-aa7b07ca-cdcb-48cb-88ad-07db297aef05   10Gi       RWO            Delete           Bound    pulsar/journal-0-wsg-sn-platform-bookie-0                             default                 10h
    pvc-aff30c6e-9bcb-49d6-be40-a1543ed5f2b3   10Gi       RWO            Delete           Bound    pulsar/wsg-sn-platform-prometheus-data                                default                 10h
    pvc-df9b2c9b-0664-45a4-93f9-397637b2c16c   10Gi       RWO            Delete           Bound    pulsar/wsg-sn-platform-grafana-data                                   default                 10h
    pvc-e1cfbbbc-5701-4ed5-8651-4a96befdb46a   10Gi       RWO            Delete           Bound    pulsar/journal-0-wsg-sn-platform-bookie-1                             default                 10h
    pvc-ed1478f6-cd70-4838-918b-f01b8f3ac361   10Gi       RWO            Delete           Bound    pulsar/wsg-sn-platform-vault-vault-volume-wsg-sn-platform-vault-0     default                 10h
    pvc-f6ae828e-4f88-4102-858a-10b0c2bb3e8a   10Gi       RWO            Delete           Bound    pulsar/wsg-sn-platform-streamnative-console-data                      default                 10h
    ➜  aks
    ```
    

# 问题处理记录

## vault 会创建多个 pvc

```jsx
➜  aks kubectl get pvc -n pulsar
NAME                                                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
wsg-sn-platform-vault-vault-volume-wsg-sn-platform-vault-0   Bound    pvc-3ddd6221-3121-4f0b-a8e0-5507846c1f25   10Gi       RWO            default        48m
wsg-sn-platform-vault-vault-volume-wsg-sn-platform-vault-1   Bound    pvc-e3389230-e96b-4746-9daa-91f99ac849a8   10Gi       RWO            default        7m18s
wsg-sn-platform-vault-vault-volume-wsg-sn-platform-vault-2   Bound    pvc-a78c3011-c398-4440-b19d-5f99834bdfd8   10Gi       RWO            default        6m42s
```

此时，zk初始化失败了。

正常如下

```jsx
➜  aks kubectl get pvc -A
NAMESPACE   NAME                                                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pulsar      data-wsg-sn-platform-zookeeper-0                             Bound    pvc-6b1ae559-18fa-4c54-b91d-1f5ab8c008a2   50Gi       RWO            default        4m21s
pulsar      wsg-sn-platform-grafana-data                                 Bound    pvc-384a5098-9bea-48e1-9b01-1dd3e2763369   10Gi       RWO            default        4m26s
pulsar      wsg-sn-platform-prometheus-data                              Bound    pvc-ccd2dc74-3734-4391-af2c-40ed8dd086c3   10Gi       RWO            default        4m26s
pulsar      wsg-sn-platform-streamnative-console-data                    Bound    pvc-9c5d6510-48f4-4c55-8390-7766e7b7abed   10Gi       RWO            default        4m26s
pulsar      wsg-sn-platform-vault-vault-volume-wsg-sn-platform-vault-0   Bound    pvc-bba5b6d5-5376-40f6-977d-fc3208f9e46b   10Gi       RWO            default        4m21s
➜  aks
```

# 知识点记录

## azure 创建自定义storage class

[https://github.com/Azure/AKS/issues/1075](https://github.com/Azure/AKS/issues/1075)

-o export ,then delete , apply

```jsx

➜  aks cat azure-sc-default.yaml
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  labels:
    addonmanager.kubernetes.io/mode: EnsureExists
    kubernetes.io/cluster-service: "true"
  name: default
  resourceVersion: "402"
  uid: 3ad50ab9-e427-4e02-be1b-749062fd6807
parameters:
  skuname: StandardSSD_LRS
provisioner: disk.csi.azure.com
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
➜  aks
```

## 删除namespace ，会自动删除里面的资源

包括pvc

```jsx
pulsar        wsg-sn-platform-bookie-0              0/1     Terminating   0          124m
pulsar        wsg-sn-platform-bookie-1              0/1     Terminating   0          45h
pulsar        wsg-sn-platform-bookie-2              0/1     Terminating   0          45h
pulsar        wsg2-sn-platform-bookie-0             0/1     Terminating   0          27h
➜  aks kubectl get ns
NAME              STATUS        AGE
default           Active        46h
kube-node-lease   Active        46h
kube-public       Active        46h
kube-system       Active        46h
pulsar            Terminating   45h
➜  aks kubectl get pvc -n pulsar
NAME                                  STATUS        VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
journal-0-wsg-sn-platform-bookie-1    Terminating   pvc-7872b09f-34c6-42a7-96f1-bad7b90544a6   10Gi       RWO            default        45h
journal-0-wsg-sn-platform-bookie-2    Terminating   pvc-2da870ac-b1b9-4c40-a4ae-1e172b7dc82c   10Gi       RWO            default        45h
journal-0-wsg2-sn-platform-bookie-0   Terminating   pvc-6c4c9d2f-125e-4903-8031-e2e7ca3ae81b   10Gi       RWO            default        27h
ledgers-0-wsg-sn-platform-bookie-1    Terminating   pvc-33e05ffb-1c42-482c-b172-9c5e09d4c309   50Gi       RWO            default        45h
ledgers-0-wsg-sn-platform-bookie-2    Terminating   pvc-76a557fd-b2d0-4205-adc5-4beecd8466cc   50Gi       RWO            default        45h
ledgers-0-wsg2-sn-platform-bookie-0   Terminating   pvc-21976ced-297b-4612-ba28-ccf5c15c7d49   50Gi       RWO            default        27h
➜  aks
```

## AKS 默认会在 worknode 安装的pod

```jsx

➜  aks kubectl get pod --all-namespaces -o wide
NAMESPACE     NAME                                  READY   STATUS        RESTARTS   AGE    IP            NODE                                NOMINATED NODE   READINESS GATES
kube-system   ama-logs-5kl9n                        2/2     Running       0          45h    10.244.4.2    aks-agentpool-36826358-vmss000004   <none>           <none>
kube-system   ama-logs-f78lc                        2/2     Running       0          111m   10.244.7.2    aks-agentpool-36826358-vmss000007   <none>           <none>
kube-system   ama-logs-mq4q6                        2/2     Running       0          45h    10.244.3.2    aks-agentpool-36826358-vmss000003   <none>           <none>
kube-system   ama-logs-qz2pl                        2/2     Running       0          46h    10.244.1.2    aks-agentpool-36826358-vmss000001   <none>           <none>
kube-system   ama-logs-rs-587f9c87bf-2zrhk          1/1     Running       0          18m    10.244.3.30   aks-agentpool-36826358-vmss000003   <none>           <none>
kube-system   ama-logs-tpclj                        2/2     Running       0          27h    10.244.6.2    aks-agentpool-36826358-vmss000006   <none>           <none>
kube-system   azure-ip-masq-agent-6v8j9             1/1     Running       0          45h    10.224.0.4    aks-agentpool-36826358-vmss000003   <none>           <none>
kube-system   azure-ip-masq-agent-b56xg             1/1     Running       0          46h    10.224.0.5    aks-agentpool-36826358-vmss000001   <none>           <none>
kube-system   azure-ip-masq-agent-cnnl2             1/1     Running       0          111m   10.224.0.8    aks-agentpool-36826358-vmss000007   <none>           <none>
kube-system   azure-ip-masq-agent-qz5s4             1/1     Running       0          27h    10.224.0.7    aks-agentpool-36826358-vmss000006   <none>           <none>
kube-system   azure-ip-masq-agent-s8wmr             1/1     Running       0          45h    10.224.0.6    aks-agentpool-36826358-vmss000004   <none>           <none>
kube-system   cloud-node-manager-c2dwh              1/1     Running       0          46h    10.224.0.5    aks-agentpool-36826358-vmss000001   <none>           <none>
kube-system   cloud-node-manager-cn8fd              1/1     Running       0          27h    10.224.0.7    aks-agentpool-36826358-vmss000006   <none>           <none>
kube-system   cloud-node-manager-mkwlq              1/1     Running       0          111m   10.224.0.8    aks-agentpool-36826358-vmss000007   <none>           <none>
kube-system   cloud-node-manager-s8sq2              1/1     Running       0          45h    10.224.0.6    aks-agentpool-36826358-vmss000004   <none>           <none>
kube-system   cloud-node-manager-tkl5g              1/1     Running       0          45h    10.224.0.4    aks-agentpool-36826358-vmss000003   <none>           <none>
kube-system   coredns-autoscaler-5589fb5654-g76xv   1/1     Running       0          18m    10.244.3.24   aks-agentpool-36826358-vmss000003   <none>           <none>
kube-system   coredns-b4854dd98-7grzh               1/1     Running       0          18m    10.244.3.28   aks-agentpool-36826358-vmss000003   <none>           <none>
kube-system   coredns-b4854dd98-bgpfh               1/1     Running       0          17m    10.244.3.31   aks-agentpool-36826358-vmss000003   <none>           <none>
kube-system   csi-azuredisk-node-k5h2s              3/3     Running       0          45h    10.224.0.4    aks-agentpool-36826358-vmss000003   <none>           <none>
kube-system   csi-azuredisk-node-n2858              3/3     Running       0          27h    10.224.0.7    aks-agentpool-36826358-vmss000006   <none>           <none>
kube-system   csi-azuredisk-node-np9vm              3/3     Running       0          46h    10.224.0.5    aks-agentpool-36826358-vmss000001   <none>           <none>
kube-system   csi-azuredisk-node-rzgsj              3/3     Running       0          45h    10.224.0.6    aks-agentpool-36826358-vmss000004   <none>           <none>
kube-system   csi-azuredisk-node-zxhwj              3/3     Running       0          111m   10.224.0.8    aks-agentpool-36826358-vmss000007   <none>           <none>
kube-system   csi-azurefile-node-2qn8t              3/3     Running       0          45h    10.224.0.4    aks-agentpool-36826358-vmss000003   <none>           <none>
kube-system   csi-azurefile-node-628b5              3/3     Running       0          111m   10.224.0.8    aks-agentpool-36826358-vmss000007   <none>           <none>
kube-system   csi-azurefile-node-pfq9d              3/3     Running       0          45h    10.224.0.6    aks-agentpool-36826358-vmss000004   <none>           <none>
kube-system   csi-azurefile-node-tb5ps              3/3     Running       0          46h    10.224.0.5    aks-agentpool-36826358-vmss000001   <none>           <none>
kube-system   csi-azurefile-node-x8h6h              3/3     Running       0          27h    10.224.0.7    aks-agentpool-36826358-vmss000006   <none>           <none>
kube-system   konnectivity-agent-554c4499fc-gmztp   1/1     Running       0          18m    10.244.3.21   aks-agentpool-36826358-vmss000003   <none>           <none>
kube-system   konnectivity-agent-554c4499fc-r2hgs   1/1     Running       0          18m    10.244.3.27   aks-agentpool-36826358-vmss000003   <none>           <none>
kube-system   kube-proxy-82zsj                      1/1     Running       0          45h    10.224.0.4    aks-agentpool-36826358-vmss000003   <none>           <none>
kube-system   kube-proxy-8lltz                      1/1     Running       0          46h    10.224.0.5    aks-agentpool-36826358-vmss000001   <none>           <none>
kube-system   kube-proxy-mkd4p                      1/1     Running       0          45h    10.224.0.6    aks-agentpool-36826358-vmss000004   <none>           <none>
kube-system   kube-proxy-z6z8c                      1/1     Running       0          27h    10.224.0.7    aks-agentpool-36826358-vmss000006   <none>           <none>
kube-system   kube-proxy-zzkqp                      1/1     Running       0          111m   10.224.0.8    aks-agentpool-36826358-vmss000007   <none>           <none>
kube-system   metrics-server-f77b4cd8-2t2gb         1/1     Running       0          18m    10.244.3.29   aks-agentpool-36826358-vmss000003   <none>           <none>
kube-system   metrics-server-f77b4cd8-58tmm         1/1     Running       0          16m    10.244.3.32   aks-agentpool-36826358-vmss000003   <none>           <none>
```

## AKS nodePool 操作

一个k8s集群 可以有多个 node pool， 所有node pool中的节点 都属于 work node ，如下，2个 node pool

```jsx

➜  .kube kubectl get nodes
NAME                                STATUS   ROLES   AGE   VERSION
aks-agentpool-32276248-vmss000002   Ready    agent   85m   v1.23.12
aks-wsgnode-32276248-vmss000001     Ready    agent   85m   v1.23.12
aks-wsgnode-32276248-vmss000002     Ready    agent   85m   v1.23.12

➜  .kube kubectl get pod -A -o wide
NAMESPACE     NAME                                  READY   STATUS    RESTARTS   AGE   IP            NODE                                NOMINATED NODE   READINESS GATES
kube-system   ama-logs-mjzqb                        2/2     Running   0          87m   10.244.5.2    aks-wsgnode-32276248-vmss000002     <none>           <none>
kube-system   ama-logs-rs-86f6f7bf7f-4pwh4          1/1     Running   0          87m   10.244.0.8    aks-agentpool-32276248-vmss000002   <none>           <none>
kube-system   ama-logs-vmcc2                        2/2     Running   0          87m   10.244.0.2    aks-agentpool-32276248-vmss000002   <none>           <none>
kube-system   ama-logs-zf2xd                        2/2     Running   0          87m   10.244.2.2    aks-wsgnode-32276248-vmss000001     <none>           <none>
kube-system   azure-ip-masq-agent-hnq24             1/1     Running   0          87m   10.224.0.8    aks-wsgnode-32276248-vmss000001     <none>           <none>
kube-system   azure-ip-masq-agent-lwzvb             1/1     Running   0          87m   10.224.0.9    aks-wsgnode-32276248-vmss000002     <none>           <none>
kube-system   azure-ip-masq-agent-vsfhm             1/1     Running   0          87m   10.224.0.6    aks-agentpool-32276248-vmss000002   <none>           <none>
kube-system   cloud-node-manager-ft4vn              1/1     Running   0          87m   10.224.0.9    aks-wsgnode-32276248-vmss000002     <none>           <none>
kube-system   cloud-node-manager-gt97g              1/1     Running   0          87m   10.224.0.6    aks-agentpool-32276248-vmss000002   <none>           <none>
kube-system   cloud-node-manager-jrlcd
```

- 可以看到 每个 node pool 配置的 az
    
    ![Untitled](azure%20k8s%20ff660755eb0340c99d90fd15053a3a8c/Untitled%205.png)
    

ui上，可以将 user node pool 转换为 syste node pool

删除自带的 system node pool

```jsx
➜  .kube kubectl get nodes
NAME                              STATUS   ROLES   AGE    VERSION
aks-wsgnode-32276248-vmss000001   Ready    agent   111m   v1.23.12
aks-wsgnode-32276248-vmss000002   Ready    agent   111m   v1.23.12
➜  .kube
```

### nodePool 删除节点

从5个node 减少到1个

![Untitled](azure%20k8s%20ff660755eb0340c99d90fd15053a3a8c/Untitled%206.png)

- check node
    
    ```jsx
    ➜  aks kubectl get nodes
    NAME                                STATUS                     ROLES   AGE   VERSION
    aks-agentpool-36826358-vmss000001   Ready,SchedulingDisabled   agent   45h   v1.23.12
    aks-agentpool-36826358-vmss000003   Ready                      agent   45h   v1.23.12
    aks-agentpool-36826358-vmss000004   Ready,SchedulingDisabled   agent   45h   v1.23.12
    aks-agentpool-36826358-vmss000006   Ready,SchedulingDisabled   agent   27h   v1.23.12
    aks-agentpool-36826358-vmss000007   Ready,SchedulingDisabled   agent   95m   v1.23.12
    ```
    

- pod in pending state
    
    ```jsx
    Events:
      Type     Reason            Age    From               Message
      ----     ------            ----   ----               -------
      Warning  FailedScheduling  3m55s  default-scheduler  0/5 nodes are available: 1 Insufficient cpu, 4 node(s) were unschedulable.
      Warning  FailedScheduling  3m53s  default-scheduler  0/5 nodes are available: 1 Insufficient cpu, 4 node(s) were unschedulable.
    ➜  aks kubectl describe pod pulsar-operator-pulsar-controller-manager-5f9d7fdc54-99mrq -n pulsar
    ```
    

### system node pools vs user node pools

System node pools serve the primary purpose of hosting critical system pods such as `CoreDNS`
 and `metrics-server`
. User node pools serve the primary purpose of hosting your application pods.

system node pool中会部署一些 核心服务，比如metirc server。 集群中至少需要一个 system node pool。

## check pv locate on az

```jsx
➜  aks kubectl describe pv pvc-a5d16e1c-0c61-460b-a233-854eade7aecc
Name:              pvc-a5d16e1c-0c61-460b-a233-854eade7aecc
Labels:            <none>
Annotations:       pv.kubernetes.io/provisioned-by: disk.csi.azure.com
Finalizers:        [kubernetes.io/pv-protection external-attacher/disk-csi-azure-com]
StorageClass:      default
Status:            Bound
Claim:             pulsar/journal-0-wsg-sn-platform-bookie-0
Reclaim Policy:    Delete
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          10Gi
Node Affinity:
  Required Terms:
    Term 0:        topology.disk.csi.azure.com/zone in [eastus-3]
Message:
Source:
    Type:              CSI (a Container Storage Interface (CSI) volume source)
    Driver:            disk.csi.azure.com
    FSType:
    VolumeHandle:      /subscriptions/de21133c-6144-44e8-91f8-fbcc36eec165/resourceGroups/mc_shengguo_group_shengguo-k8s_eastus/providers/Microsoft.Compute/disks/pvc-a5d16e1c-0c61-460b-a233-854eade7aecc
    ReadOnly:          false
    VolumeAttributes:      csi.storage.k8s.io/pv/name=pvc-a5d16e1c-0c61-460b-a233-854eade7aecc
                           csi.storage.k8s.io/pvc/name=journal-0-wsg-sn-platform-bookie-0
                           csi.storage.k8s.io/pvc/namespace=pulsar
                           requestedsizegib=10
                           skuname=StandardSSD_LRS
                           storage.kubernetes.io/csiProvisionerIdentity=1672739727180-8081-disk.csi.azure.com
Events:                <none>
➜  aks
```

## node locate on diff az

➜  aks kubectl get nodes
NAME                                STATUS                     ROLES   AGE   VERSION
aks-agentpool-36826358-vmss000001   Ready                      agent   44h   v1.23.12
aks-agentpool-36826358-vmss000003   Ready                      agent   44h   v1.23.12
aks-agentpool-36826358-vmss000004   Ready,SchedulingDisabled   agent   44h   v1.23.12
aks-agentpool-36826358-vmss000006   Ready                      agent   26h   v1.23.12
aks-agentpool-36826358-vmss000007   Ready                      agent   28m   v1.23.12
➜  aks

- node4
    
    ```jsx
    ➜  aks kubectl describe node aks-agentpool-36826358-vmss000004
    Name:               aks-agentpool-36826358-vmss000004
    Roles:              agent
    Labels:             agentpool=agentpool
                        beta.kubernetes.io/arch=amd64
                        beta.kubernetes.io/instance-type=Standard_DS2_v2
                        beta.kubernetes.io/os=linux
                        failure-domain.beta.kubernetes.io/region=eastus
                        failure-domain.beta.kubernetes.io/zone=eastus-3
    ```
    
- node6
    
    ```jsx
    ➜  aks kubectl describe node aks-agentpool-36826358-vmss000006
    Name:               aks-agentpool-36826358-vmss000006
    Roles:              agent
    Labels:             agentpool=agentpool
                        beta.kubernetes.io/arch=amd64
                        beta.kubernetes.io/instance-type=Standard_DS2_v2
                        beta.kubernetes.io/os=linux
                        failure-domain.beta.kubernetes.io/region=eastus
                        failure-domain.beta.kubernetes.io/zone=eastus-1
    ```
    
- node7
    
    ```jsx
    ➜  aks kubectl describe node aks-agentpool-36826358-vmss000007
    Name:               aks-agentpool-36826358-vmss000007
    Roles:              agent
    Labels:             agentpool=agentpool
                        beta.kubernetes.io/arch=amd64
                        beta.kubernetes.io/instance-type=Standard_DS2_v2
                        beta.kubernetes.io/os=linux
                        failure-domain.beta.kubernetes.io/region=eastus
                        failure-domain.beta.kubernetes.io/zone=eastus-2
                        kubernetes.azure.com/agentpool=agentpool
                        kubernetes.azure.com/cluster=MC_shengguo_group_shengguo-k8s_eastus
                        kubernetes.azure.com/kubelet-identity-client-id=2f507fdb-a0d5-4132-bb66-3757a7f69a5f
                        kubernetes.azure.com/mode=system
                        kubernetes.azure.com/node-image-version=AKSUbuntu-1804gen2containerd-2022.12.15
                        kubernetes.azure.com/os-sku=Ubuntu
                        kubernetes.azure.com/role=agent
                        kubernetes.azure.com/storageprofile=managed
                        kubernetes.azure.com/storagetier=Premium_LRS
                        kubernetes.io/arch=amd64
                        kubernetes.io/hostname=aks-agentpool-36826358-vmss000007
                        kubernetes.io/os=linux
                        kubernetes.io/role=agent
    ```
    
- node1
    
    ```jsx
    
    ➜  aks kubectl describe node aks-agentpool-36826358-vmss000001
    Name:               aks-agentpool-36826358-vmss000001
    Roles:              agent
    Labels:             agentpool=agentpool
                        beta.kubernetes.io/arch=amd64
                        beta.kubernetes.io/instance-type=Standard_DS2_v2
                        beta.kubernetes.io/os=linux
                        failure-domain.beta.kubernetes.io/region=eastus
                        failure-domain.beta.kubernetes.io/zone=eastus-2
                        kubernetes.azure.com/agentpool=agentpool
                        kubernetes.azure.com/cluster=MC_shengguo_group_shengguo-k8s_eastus
                        kubernetes.azure.com/kubelet-identity-client-id=2f507fdb-a0d5-4132-bb66-3757a7f69a5f
                        kubernetes.azure.com/mode=system
                        kubernetes.azure.com/node-image-version=AKSUbuntu-1804gen2containerd-2022.12.15
                        kubernetes.azure.com/os-sku=Ubuntu
                        kubernetes.azure.com/role=agent
                        kubernetes.azure.com/storageprofile=manage
    ```
    
- node3
    
    ```jsx
    ➜  aks kubectl describe node aks-agentpool-36826358-vmss000003
    Name:               aks-agentpool-36826358-vmss000003
    Roles:              agent
    Labels:             agentpool=agentpool
                        beta.kubernetes.io/arch=amd64
                        beta.kubernetes.io/instance-type=Standard_DS2_v2
                        beta.kubernetes.io/os=linux
                        failure-domain.beta.kubernetes.io/region=eastus
                        failure-domain.beta.kubernetes.io/zone=eastus-1
                        kubernetes.azure.com/agentpool=agentpool
                        kubernetes.azure.com/cluster=MC_shengguo_grou
    ```
    
- 

```jsx

```

## drain pod

```jsx

kubectl drain --ignore-daemonsets=false aks-agentpool-36826358-vmss000004

```

## 设置 node 不允许调度pod

```jsx
➜  aks kubectl cordon aks-agentpool-36826358-vmss000004
node/aks-agentpool-36826358-vmss000004 cordoned

//describe pod 
Taints:             node.kubernetes.io/unschedulable:NoSchedule
Unschedulable:      true

```

check

```jsx
➜  aks kubectl get nodes
NAME                                STATUS                     ROLES   AGE   VERSION
aks-agentpool-36826358-vmss000001   Ready                      agent   44h   v1.23.12
aks-agentpool-36826358-vmss000003   Ready                      agent   43h   v1.23.12
aks-agentpool-36826358-vmss000004   Ready,SchedulingDisabled   agent   43h   v1.23.12
aks-agentpool-36826358-vmss000006   Ready                      agent   26h   v1.23.12
➜  aks
```

## node describe

可以看到node 资源消耗

```jsx

➜  aks kubectl describe node aks-agentpool-36826358-vmss000004
Name:               aks-agentpool-36826358-vmss000004
Roles:              agent
Labels:             agentpool=agentpool
                    beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=Standard_DS2_v2
                    beta.kubernetes.io/os=linux
                    failure-domain.beta.kubernetes.io/region=eastus
                    failure-domain.beta.kubernetes.io/zone=eastus-3
                    kubernetes.azure.com/agentpool=agentpool
                    kubernetes.azure.com/cluster=MC_shengguo_group_shengguo-k8s_eastus
                    kubernetes.azure.com/kubelet-identity-client-id=2f507fdb-a0d5-4132-bb66-3757a7f69a5f
                    kubernetes.azure.com/mode=system
                    kubernetes.azure.com/node-image-version=AKSUbuntu-1804gen2containerd-2022.12.15
                    kubernetes.azure.com/os-sku=Ubuntu
                    kubernetes.azure.com/role=agent
                    kubernetes.azure.com/storageprofile=managed
                    kubernetes.azure.com/storagetier=Premium_LRS
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=aks-agentpool-36826358-vmss000004
                    kubernetes.io/os=linux
                    kubernetes.io/role=agent
                    node-role.kubernetes.io/agent=
                    node.kubernetes.io/instance-type=Standard_DS2_v2
                    storageprofile=managed
                    storagetier=Premium_LRS
                    topology.disk.csi.azure.com/zone=eastus-3
                    topology.kubernetes.io/region=eastus
                    topology.kubernetes.io/zone=eastus-3
Annotations:        csi.volume.kubernetes.io/nodeid:
                      {"disk.csi.azure.com":"aks-agentpool-36826358-vmss000004","file.csi.azure.com":"aks-agentpool-36826358-vmss000004"}
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Tue, 03 Jan 2023 18:40:20 +0800
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  aks-agentpool-36826358-vmss000004
  AcquireTime:     <unset>
  RenewTime:       Thu, 05 Jan 2023 13:46:08 +0800
Conditions:
  Type                          Status  LastHeartbeatTime                 LastTransitionTime                Reason                          Message
  ----                          ------  -----------------                 ------------------                ------                          -------
  NetworkUnavailable            False   Tue, 03 Jan 2023 18:42:00 +0800   Tue, 03 Jan 2023 18:42:00 +0800   RouteCreated                    RouteController created a route
  FrequentContainerdRestart     False   Thu, 05 Jan 2023 13:41:15 +0800   Tue, 03 Jan 2023 18:40:20 +0800   NoFrequentContainerdRestart     containerd is functioning properly
  FilesystemCorruptionProblem   False   Thu, 05 Jan 2023 13:41:15 +0800   Tue, 03 Jan 2023 18:40:20 +0800   FilesystemIsOK                  Filesystem is healthy
  FrequentUnregisterNetDevice   False   Thu, 05 Jan 2023 13:41:15 +0800   Tue, 03 Jan 2023 18:40:20 +0800   NoFrequentUnregisterNetDevice   node is functioning properly
  KubeletProblem                False   Thu, 05 Jan 2023 13:41:15 +0800   Tue, 03 Jan 2023 18:40:20 +0800   KubeletIsUp                     kubelet service is up
  KernelDeadlock                False   Thu, 05 Jan 2023 13:41:15 +0800   Tue, 03 Jan 2023 18:40:20 +0800   KernelHasNoDeadlock             kernel has no deadlock
  ReadonlyFilesystem            False   Thu, 05 Jan 2023 13:41:15 +0800   Tue, 03 Jan 2023 18:40:20 +0800   FilesystemIsNotReadOnly         Filesystem is not read-only
  VMEventScheduled              False   Thu, 05 Jan 2023 13:41:15 +0800   Tue, 03 Jan 2023 18:41:10 +0800   NoVMEventScheduled              VM has no scheduled event
  FrequentKubeletRestart        False   Thu, 05 Jan 2023 13:41:15 +0800   Tue, 03 Jan 2023 18:40:20 +0800   NoFrequentKubeletRestart        kubelet is functioning properly
  FrequentDockerRestart         False   Thu, 05 Jan 2023 13:41:15 +0800   Tue, 03 Jan 2023 18:40:20 +0800   NoFrequentDockerRestart         docker is functioning properly
  ContainerRuntimeProblem       False   Thu, 05 Jan 2023 13:41:15 +0800   Tue, 03 Jan 2023 18:40:20 +0800   ContainerRuntimeIsUp            container runtime service is up
  MemoryPressure                False   Thu, 05 Jan 2023 13:45:13 +0800   Tue, 03 Jan 2023 18:40:20 +0800   KubeletHasSufficientMemory      kubelet has sufficient memory available
  DiskPressure                  False   Thu, 05 Jan 2023 13:45:13 +0800   Tue, 03 Jan 2023 18:40:20 +0800   KubeletHasNoDiskPressure        kubelet has no disk pressure
  PIDPressure                   False   Thu, 05 Jan 2023 13:45:13 +0800   Tue, 03 Jan 2023 18:40:20 +0800   KubeletHasSufficientPID         kubelet has sufficient PID available
  Ready                         True    Thu, 05 Jan 2023 13:45:13 +0800   Tue, 03 Jan 2023 18:40:20 +0800   KubeletReady                    kubelet is posting ready status. AppArmor enabled
Addresses:
  InternalIP:  10.224.0.6
  Hostname:    aks-agentpool-36826358-vmss000004
Capacity:
  cpu:                2
  ephemeral-storage:  129886128Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             7116264Ki
  pods:               110
Allocatable:
  cpu:                1900m
  ephemeral-storage:  119703055367
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             4670952Ki
  pods:               110
System Info:
  Machine ID:                 e09d0cd563d34fa098e4ebb4ab714886
  System UUID:                c119e69e-f34f-4056-bf98-2764b3893d8a
  Boot ID:                    3a313aa6-ed78-42c3-b713-7b007b2c0f6c
  Kernel Version:             5.4.0-1098-azure
  OS Image:                   Ubuntu 18.04.6 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.6.4+azure-4
  Kubelet Version:            v1.23.12
  Kube-Proxy Version:         v1.23.12
PodCIDR:                      10.244.4.0/24
PodCIDRs:                     10.244.4.0/24
ProviderID:                   azure:///subscriptions/de21133c-6144-44e8-91f8-fbcc36eec165/resourceGroups/mc_shengguo_group_shengguo-k8s_eastus/providers/Microsoft.Compute/virtualMachineScaleSets/aks-agentpool-36826358-vmss/virtualMachines/4
Non-terminated Pods:          (16 in total)
  Namespace                   Name                                    CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                    ------------  ----------  ---------------  -------------  ---
  kube-system                 ama-logs-5kl9n                          150m (7%)     1 (52%)     550Mi (12%)      1774Mi (38%)   43h
  kube-system                 azure-ip-masq-agent-s8wmr               100m (5%)     500m (26%)  50Mi (1%)        250Mi (5%)     43h
  kube-system                 cloud-node-manager-s8sq2                50m (2%)      0 (0%)      50Mi (1%)        512Mi (11%)    43h
  kube-system                 csi-azuredisk-node-rzgsj                30m (1%)      0 (0%)      60Mi (1%)        400Mi (8%)     43h
  kube-system                 csi-azurefile-node-pfq9d                30m (1%)      0 (0%)      60Mi (1%)        600Mi (13%)    43h
  kube-system                 kube-proxy-mkd4p                        100m (5%)     0 (0%)      0 (0%)           0 (0%)         43h
  pulsar                      wsg-sn-platform-bookie-0                200m (10%)    0 (0%)      512Mi (11%)      0 (0%)         43h
  pulsar                      wsg-sn-platform-bookie-2                200m (10%)    0 (0%)      512Mi (11%)      0 (0%)         43h
  pulsar                      wsg-sn-platform-broker-0                200m (10%)    0 (0%)      512Mi (11%)      0 (0%)         43h
  pulsar                      wsg-sn-platform-node-exporter-wvbzs     0 (0%)        0 (0%)      0 (0%)           0 (0%)         43h
  pulsar                      wsg-sn-platform-vault-0                 200m (10%)    400m (21%)  320Mi (7%)       640Mi (14%)    43h
  pulsar                      wsg-sn-platform-vault-1                 200m (10%)    400m (21%)  320Mi (7%)       640Mi (14%)    43h
  pulsar                      wsg-sn-platform-vault-2                 200m (10%)    400m (21%)  320Mi (7%)       640Mi (14%)    43h
  pulsar                      wsg2-sn-platform-grafana-0              100m (5%)     0 (0%)      250Mi (5%)       0 (0%)         25h
  pulsar                      wsg2-sn-platform-node-exporter-cv9kt    0 (0%)        0 (0%)      0 (0%)           0 (0%)         25h
  pulsar                      wsg2-sn-platform-toolset-0              100m (5%)     0 (0%)      256Mi (5%)       0 (0%)         25h
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests      Limits
  --------           --------      ------
  cpu                1860m (97%)   2700m (142%)
  memory             3772Mi (82%)  5456Mi (119%)
  ephemeral-storage  0 (0%)        0 (0%)
  hugepages-1Gi      0 (0%)        0 (0%)
  hugepages-2Mi      0 (0%)        0 (0%)
Events:              <none>
➜  aks
```

- nodepool delete node
    
    ```jsx
    
    6826358-vmss/virtualMachines/3
    Non-terminated Pods:          (26 in total)
      Namespace                   Name                                                CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
      ---------                   ----                                                ------------  ----------  ---------------  -------------  ---
      kube-system                 ama-logs-mq4q6                                      150m (7%)     1 (52%)     550Mi (12%)      1774Mi (38%)   45h
      kube-system                 ama-logs-rs-587f9c87bf-2zrhk                        150m (7%)     1 (52%)     250Mi (5%)       1280Mi (28%)   8m37s
      kube-system                 azure-ip-masq-agent-6v8j9                           100m (5%)     500m (26%)  50Mi (1%)        250Mi (5%)     45h
      kube-system                 cloud-node-manager-tkl5g                            50m (2%)      0 (0%)      50Mi (1%)        512Mi (11%)    45h
      kube-system                 coredns-autoscaler-5589fb5654-g76xv                 20m (1%)      200m (10%)  10Mi (0%)        500Mi (10%)    8m38s
      kube-system                 coredns-b4854dd98-7grzh                             100m (5%)     3 (157%)    70Mi (1%)        500Mi (10%)    8m38s
      kube-system                 coredns-b4854dd98-bgpfh                             100m (5%)     3 (157%)    70Mi (1%)        500Mi (10%)    7m52s
      kube-system                 csi-azuredisk-node-k5h2s                            30m (1%)      0 (0%)      60Mi (1%)        400Mi (8%)     45h
      kube-system                 csi-azurefile-node-2qn8t                            30m (1%)      0 (0%)      60Mi (1%)        600Mi (13%)    45h
      kube-system                 konnectivity-agent-554c4499fc-gmztp                 20m (1%)      1 (52%)     20Mi (0%)        1Gi (22%)      8m38s
      kube-system                 konnectivity-agent-554c4499fc-r2hgs                 20m (1%)      1 (52%)     20Mi (0%)        1Gi (22%)      8m32s
      kube-system                 kube-proxy-82zsj                                    100m (5%)     0 (0%)      0 (0%)           0 (0%)         45h
      kube-system                 metrics-server-f77b4cd8-2t2gb                       44m (2%)      1 (52%)     55Mi (1%)        2000Mi (43%)   8m38s
      kube-system                 metrics-server-f77b4cd8-58tmm                       44m (2%)      1 (52%)     55Mi (1%)        2000Mi (43%)   6m41s
      pulsar                      cert-manager-7599c44747-xccnj                       0 (0%)        0 (0%)      0 (0%)           0 (0%)         8m38s
      pulsar                      cert-manager-cainjector-7d9466748-8dpth             0 (0%)        0 (0%)      0 (0%)           0 (0%)         8m38s
      pulsar                      cert-manager-webhook-d77bbf4cb-7rrnj                0 (0%)        0 (0%)      0 (0%)           0 (0%)         8m36s
      pulsar                      function-mesh-controller-manager-8848bd5b8-whn4q    80m (4%)      0 (0%)      50Mi (1%)        0 (0%)         8m38s
      pulsar                      wsg-sn-platform-bookie-1                            200m (10%)    0 (0%)      512Mi (11%)      0 (0%)         45h
      pulsar                      wsg-sn-platform-grafana-0                           100m (5%)     0 (0%)      250Mi (5%)       0 (0%)         45h
      pulsar                      wsg-sn-platform-node-exporter-vwng5                 0 (0%)        0 (0%)      0 (0%)           0 (0%)         45h
      pulsar                      wsg-sn-platform-proxy-0                             200m (10%)    0 (0%)      128Mi (2%)       0 (0%)         45h
      pulsar                      wsg-sn-platform-zookeeper-0                         100m (5%)     0 (0%)      256Mi (5%)       0 (0%)         45h
      pulsar                      wsg-sn-platform-zookeeper-1                         100m (5%)     0 (0%)      256Mi (5%)       0 (0%)         45h
      pulsar                      wsg-sn-platform-zookeeper-2                         100m (5%)     0 (0%)      256Mi (5%)       0 (0%)         45h
      pulsar                      wsg2-sn-platform-recovery-0                         50m (2%)      0 (0%)      64Mi (1%)        0 (0%)         6m10s
    Allocated resources:
      (Total limits may be over 100 percent, i.e., overcommitted.)
      Resource           Requests      Limits
      --------           --------      ------
      cpu                1888m (99%)   12700m (668%)
      memory             3092Mi (67%)  12364Mi (271%)
      ephemeral-storage  0 (0%)        0 (0%)
      hugepages-1Gi      0 (0%)        0 (0%)
      hugepages-2Mi      0 (0%)        0 (0%)
    Events:              <none>
    ➜  aks kubectl describe node aks-agentpool-36826358-vmss000003
    ```
    

## check bookie list

```jsx
root@wsg-sn-platform-toolset-0:/pulsar# pulsar-admin bookies list-bookies
{
  "bookies" : [ {
    "bookieId" : "wsg-sn-platform-bookie-0.wsg-sn-platform-bookie.pulsar.svc.cluster.local:3181"
  }, {
    "bookieId" : "wsg-sn-platform-bookie-1.wsg-sn-platform-bookie.pulsar.svc.cluster.local:3181"
  }, {
    "bookieId" : "wsg-sn-platform-bookie-2.wsg-sn-platform-bookie.pulsar.svc.cluster.local:3181"
  } ]
}
root@wsg-sn-platform-toolset-0:/pulsar#
```

## bookie operator 关键log

- add or delete pod
    
    replica from 3 to 4
    
    ```jsx
    
    2023-01-03T11:47:58.211723235Z {"severity":"info","timestamp":"2023-01-03T11:47:58Z","logger":"StatefulSetReconciler","message":"Scaling BookKeeper","Namespace":"pulsar","Name":"wsg-sn-platform-bookie","Namespace":"pulsar","Name":"wsg-sn-platform-bookie","From":3,"To":3,"Desired":4}
    ```
    
    replica from 4 to 3
    
    ```jsx
    2023-01-03T12:02:36.130084183Z {"severity":"info","timestamp":"2023-01-03T12:02:36Z","logger":"StatefulSetReconciler","message":"Scaling BookKeeper","Namespace":"pulsar","Name":"wsg-sn-platform-bookie","Namespace":"pulsar","Name":"wsg-sn-platform-bookie","From":4,"To":4,"Desired":3}
    ```
    
- create and delete decomission pod
    
    ```jsx
    2023-01-03T12:02:36.251275962Z {"severity":"info","timestamp":"2023-01-03T12:02:36Z","logger":"JobReconciler","message":"Created resource","Namespace":"pulsar","Name":"wsg-sn-platform-bookie-3-decommission"}
    
    //delete decomission pod
    2023-01-03T12:03:11.666626984Z {"severity":"info","timestamp":"2023-01-03T12:03:11Z","logger":"JobReconciler","message":"Deleted resource","Namespace":"pulsar","Name":"wsg-sn-platform-bookie-3-decommission"}
    ```
    

- 

## 查看 bookie 存储使用

enter bookie pod

```jsx
I have no name!@wsg-sn-platform-bookie-0:/pulsar$ df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay         124G   36G   89G  29% /
tmpfs            64M     0   64M   0% /dev
tmpfs           3.4G     0  3.4G   0% /sys/fs/cgroup
/dev/sda1       124G   36G   89G  29% /etc/hosts
shm              64M     0   64M   0% /dev/shm
/dev/sdf        9.8G  1.6G  8.3G  16% /pulsar/data/bookkeeper/journal-0
/dev/sdg         49G  3.4G   46G   7% /pulsar/data/bookkeeper/ledgers-0
tmpfs           4.5G   12K  4.5G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs           3.4G     0  3.4G   0% /proc/acpi
tmpfs           3.4G     0  3.4G   0% /proc/scsi
tmpfs           3.4G     0  3.4G   0% /sys/firmware
I have no name!@wsg-sn-platform-bookie-0:/pulsar$

```

## azure k8s work node 自动扩容

```jsx
Events:
  Type     Reason            Age                  From                Message
  ----     ------            ----                 ----                -------
  Normal   TriggeredScaleUp  2m4s                 cluster-autoscaler  pod triggered scale-up: [{aks-agentpool-36826358-vmss 1->2 (max: 5)}]
  Warning  FailedScheduling  72s (x2 over 2m13s)  default-scheduler   0/1 nodes are available: 1 Insufficient cpu.
  Warning  FailedScheduling  12s                  default-scheduler   0/3 nodes are available: 1 Insufficient cpu, 2 node(s) had taint {node.kubernetes.io/network-unavailable: }, that the pod didn't tolerate.
  Normal   Scheduled         4s                   default-scheduler   Successfully assigned pulsar/wsg-sn-platform-zookeeper-0 to aks-agentpool-36826358-vmss000003
➜  aks

//cpu资源不够，触发了自动扩容

➜  aks kubectl get nodes
NAME                                STATUS   ROLES   AGE   VERSION
aks-agentpool-36826358-vmss000001   Ready    agent   45m   v1.23.12
aks-agentpool-36826358-vmss000003   Ready    agent   82s   v1.23.12
aks-agentpool-36826358-vmss000004   Ready    agent   77s   v1.23.12

➜  aks kubectl get nodes -o wide
NAME                                STATUS   ROLES   AGE   VERSION    INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
aks-agentpool-36826358-vmss000001   Ready    agent   45m   v1.23.12   10.224.0.5    <none>        Ubuntu 18.04.6 LTS   5.4.0-1098-azure   containerd://1.6.4+azure-4
aks-agentpool-36826358-vmss000003   Ready    agent   95s   v1.23.12   10.224.0.4    <none>        Ubuntu 18.04.6 LTS   5.4.0-1098-azure   containerd://1.6.4+azure-4
aks-agentpool-36826358-vmss000004   Ready    agent   90s   v1.23.12   10.224.0.6    <none>        Ubuntu 18.04.6 LTS   5.4.0-1098-azure   containerd://1.6.4+azure-4
```

## 默认的storage class

```jsx
➜  ~ kubectl get sc
NAME                    PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
azurefile               file.csi.azure.com   Delete          Immediate              true                   57m
azurefile-csi           file.csi.azure.com   Delete          Immediate              true                   57m
azurefile-csi-premium   file.csi.azure.com   Delete          Immediate              true                   57m
azurefile-premium       file.csi.azure.com   Delete          Immediate              true                   57m
default (default)       disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   57m
managed                 disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   57m
managed-csi             disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   57m
managed-csi-premium     disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   57m
managed-premium         disk.csi.azure.com   Delete          WaitForFirstConsumer   true                   57m
```
