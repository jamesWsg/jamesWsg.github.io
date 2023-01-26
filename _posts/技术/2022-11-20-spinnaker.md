---
layout: post
title: spinnaker探索实践
category: 技术
---

# devops-spinnaker

# kind install k8s

[https://kind.sigs.k8s.io/docs/user/configuration/](https://kind.sigs.k8s.io/docs/user/configuration/)

[https://kind.sigs.k8s.io/docs/user/configuration/#kubeadm-config-patches](https://kind.sigs.k8s.io/docs/user/configuration/#kubeadm-config-patches)

[https://kind.sigs.k8s.io/docs/user/ingress/](https://kind.sigs.k8s.io/docs/user/ingress/)

## ubuntu20 install kind

```jsx
18  curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-linux-amd64
   19  ll
   20  chmod +x ./kind
   21  sudo mv ./kind /usr/local/bin/kind

```

- install
    
    ```jsx
    ubuntu@ip-172-31-47-249:~$ sudo kind create cluster --name wsg-cluster
    Creating cluster "wsg-cluster" ...
     ✓ Ensuring node image (kindest/node:v1.25.3) 🖼
     ✓ Preparing nodes 📦
     ✓ Writing configuration 📜
     ✓ Starting control-plane 🕹️
     ✓ Installing CNI 🔌
     ✓ Installing StorageClass 💾
    Set kubectl context to "kind-wsg-cluster"
    You can now use your cluster with:
    
    kubectl cluster-info --context kind-wsg-cluster
    
    Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
    ubuntu@ip-172-31-47-249:~$
    
    // use spec version
    
    ubuntu@ip-172-31-47-249:~/kind$ cat kind.yml
    kind: Cluster
    apiVersion: kind.x-k8s.io/v1alpha4
    nodes:
    - role: control-plane
      image: kindest/node:v1.23.13@sha256:ef453bb7c79f0e3caba88d2067d4196f427794086a7d0df8df4f019d5e336b61
    
    ```
    
    - check
        
        ```jsx
        ubuntu@ip-172-31-47-249:~$ sudo kubectl get nodes
        NAME                 STATUS   ROLES                  AGE   VERSION
        kind-control-plane   Ready    control-plane,master   11m   v1.23.13
        ubuntu@ip-172-31-47-249:~$
        ubuntu@ip-172-31-47-249:~$ sudo kubectl get pod --all-namespaces
        NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
        kube-system          coredns-64897985d-5njq4                      1/1     Running   0          11m
        kube-system          coredns-64897985d-kpzrm                      1/1     Running   0          11m
        kube-system          etcd-kind-control-plane                      1/1     Running   0          11m
        kube-system          kindnet-cjjb4                                1/1     Running   0          11m
        kube-system          kube-apiserver-kind-control-plane            1/1     Running   0          11m
        kube-system          kube-controller-manager-kind-control-plane   1/1     Running   0          11m
        kube-system          kube-proxy-jn5bl                             1/1     Running   0          11m
        kube-system          kube-scheduler-kind-control-plane            1/1     Running   0          11m
        local-path-storage   local-path-provisioner-58dc9cd8d9-xpffk      1/1     Running   0          11m
        ubuntu@ip-172-31-47-249:~$
        ```
        
    

- delete cluster
    
    ```jsx
    ubuntu@ip-172-31-47-249:~/kind$ sudo kind delete cluster --name wsg-cluster
    Deleting cluster "wsg-cluster" ...
    ubuntu@ip-172-31-47-249:~/kind$
    
    ```
    
- install kubectl
    
    [https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)
    
- install helm
    
    ```jsx
    
    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
    
    //国内环境下载可能有问题
    ```
    

## centos7 install kind

```jsx
18  curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-linux-amd64
   19  ll
   20  chmod +x ./kind
   21  sudo mv ./kind /usr/local/bin/kind

```

install docker

install kubectl

install helm

- check
    
    ```jsx
    [azureuser@shengguo1 kind]$ sudo kind create cluster --config kind.yml
    Creating cluster "kind" ...
     ✓ Ensuring node image (kindest/node:v1.23.13) 🖼
     ✓ Preparing nodes 📦
     ✓ Writing configuration 📜
     ✓ Starting control-plane 🕹️
     ✓ Installing CNI 🔌
     ✓ Installing StorageClass 💾
    Set kubectl context to "kind-kind"
    You can now use your cluster with:
    
    kubectl cluster-info --context kind-kind
    
    Have a nice day! 👋
    ```
    
- kubectl get
    
    ```jsx
    [azureuser@shengguo1 kind]$ sudo kubectl get nodes
    NAME                 STATUS   ROLES                  AGE     VERSION
    kind-control-plane   Ready    control-plane,master   4m49s   v1.23.13
    ```
    
- helm check
    
    ```jsx
    [azureuser@shengguo1 kind]$ sudo cp /usr/local/bin/helm /usr/sbin/
    [azureuser@shengguo1 kind]$ sudo helm list
    NAME	NAMESPACE	REVISION	UPDATED	STATUS	CHART	APP VERSION
    [azureuser@shengguo1 kind]$
    ```
    

## deploy

- create cluster
    
    ```jsx
    ➜  spinnaker git:(main) ✗ kind create cluster --config=kind-cluster.yml
    Creating cluster "kind" ...
     ✓ Ensuring node image (kindest/node:v1.25.3) 🖼
     ✓ Preparing nodes 📦
     ✓ Writing configuration 📜
     ✓ Starting control-plane 🕹️
     ✓ Installing CNI 🔌
     ✓ Installing StorageClass 💾
    Set kubectl context to "kind-kind"
    You can now use your cluster with:
    
    kubectl cluster-info --context kind-kind
    
    //check
    ➜  spinnaker git:(main) ✗ kubectl cluster-info --context kind-kind
    Kubernetes control plane is running at https://127.0.0.1:57670
    CoreDNS is running at https://127.0.0.1:57670/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
    
    To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
    ➜  spinnaker git:(main) ✗ kubectl get nodes
    NAME                 STATUS   ROLES           AGE     VERSION
    kind-control-plane   Ready    control-plane   5m54s   v1.25.3
    ➜  spinnaker git:(main) ✗
    ```
    

## kind 常用命令

```jsx

kind create cluster # Default cluster context name is `kind`.
...
kind create cluster --name kind-2

```

# prepare spinnaker

## k8s and ingress-controller

- create ingress-controller
    
    [https://kind.sigs.k8s.io/docs/user/ingress/](https://kind.sigs.k8s.io/docs/user/ingress/)
    
    ```jsx
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
    
    //check
    
    ```
    

## jenkins

## docker-registry

arkade

- install arkade
    
    ```jsx
    
    curl -sLS https://get.arkade.dev >get-arkade.sh
    
    ➜  spinnaker git:(main) ✗ bash -x  get-arkade.sh
    
    Creating alias 'ark' for 'arkade'.
    + /usr/local/bin/arkade version
                _             _
      __ _ _ __| | ____ _  __| | ___
     / _` | '__| |/ / _` |/ _` |/ _ \
    | (_| | |  |   < (_| | (_| |  __/
     \__,_|_|  |_|\_\__,_|\__,_|\___|
    
    Open Source Marketplace For Developer Tools
    
    Version: 0.8.52
    Git Commit: 2e201a1a48bf273cc7cd61111159a96aa3f28215
    
     🐳 arkade needs your support: https://github.com/sponsors/alexellis
    ```
    

- arkade install cert-manger
    
    ```jsx
    ➜  spinnaker git:(main) ✗ ark install cert-manger
    Error: no such app: cert-manger, run "arkade install --help" for a list of apps
    ➜  spinnaker git:(main) ✗ arkade install cert-manager
    Using Kubeconfig: /Users/wsg/.kube/config
    Client: x86_64, Darwin
    2022/12/20 12:08:53 User dir established as: /Users/wsg/.arkade/
    "jetstack" has been added to your repositories
    
    Hang tight while we grab the latest from your chart repositories...
    ...Successfully got an update from the "jetstack" chart repository
    ...Successfully got an update from the "twuni" chart repository
    Update Complete. ⎈Happy Helming!⎈
    
    VALUES values.yaml
    Command: /Users/wsg/.arkade/bin/helm [upgrade --install cert-manager jetstack/cert-manager --namespace cert-manager --values /var/folders/8t/sy8k6gbx7gq9jywkpzjwl76w0000gn/T/charts/cert-manager/values.yaml --set installCRDs=true]
    Release "cert-manager" does not exist. Installing it now.
    NAME: cert-manager
    LAST DEPLOYED: Tue Dec 20 12:08:57 2022
    NAMESPACE: cert-manager
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    NOTES:
    cert-manager v1.10.1 has been deployed successfully!
    
    In order to begin issuing certificates, you will need to set up a ClusterIssuer
    or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).
    
    More information on the different types of issuers and how to configure them
    can be found in our documentation:
    
    https://cert-manager.io/docs/configuration/
    
    For information on how to configure cert-manager to automatically provision
    Certificates for Ingress resources, take a look at the `ingress-shim`
    documentation:
    
    https://cert-manager.io/docs/usage/ingress/
    =======================================================================
    = cert-manager  has been installed.                                   =
    =======================================================================
    
    # Get started with cert-manager here:
    # https://docs.cert-manager.io/en/latest/tutorials/acme/http-validation.html
    
    🐳 arkade needs your support: https://github.com/sponsors/alexellis
    ➜  spinnaker git:(main) ✗
    ```
    
- arkade install docker-registry
    - install and output
        
        ```jsx
        
        ➜  spinnaker git:(main) ✗ ark install docker-registry
        Using Kubeconfig: /Users/wsg/.kube/config
        Client: x86_64, Darwin
        2022/12/20 11:54:05 User dir established as: /Users/wsg/.arkade/
        Downloading: https://get.helm.sh/helm-v3.9.3-darwin-amd64.tar.gz
        /var/folders/8t/sy8k6gbx7gq9jywkpzjwl76w0000gn/T/helm-v3.9.3-darwin-amd64.tar.gz written.
        2022/12/20 11:54:12 Extracted: /var/folders/8t/sy8k6gbx7gq9jywkpzjwl76w0000gn/T/helm
        2022/12/20 11:54:12 Copying /var/folders/8t/sy8k6gbx7gq9jywkpzjwl76w0000gn/T/helm to /Users/wsg/.arkade/bin/helm
        Downloaded to:  /Users/wsg/.arkade/bin/helm helm
        "twuni" has been added to your repositories
        
        Hang tight while we grab the latest from your chart repositories...
        ...Successfully got an update from the "twuni" chart repository
        Update Complete. ⎈Happy Helming!⎈
        
        Node architecture: "amd64"
        Chart path:  /var/folders/8t/sy8k6gbx7gq9jywkpzjwl76w0000gn/T/charts
        VALUES values.yaml
        Command: /Users/wsg/.arkade/bin/helm [upgrade --install docker-registry twuni/docker-registry --namespace default --values /var/folders/8t/sy8k6gbx7gq9jywkpzjwl76w0000gn/T/charts/docker-registry/values.yaml --set persistence.enabled=false --set secrets.htpasswd=admin:$2a$10$Ez7ZRNgwGqptcFDZAiFqK.POsgnAKNkljxpKo61RxTjd7JtIdx.rG
        ]
        Release "docker-registry" does not exist. Installing it now.
        NAME: docker-registry
        LAST DEPLOYED: Tue Dec 20 11:54:17 2022
        NAMESPACE: default
        STATUS: deployed
        REVISION: 1
        TEST SUITE: None
        NOTES:
        1. Get the application URL by running these commands:
          export POD_NAME=$(kubectl get pods --namespace default -l "app=docker-registry,release=docker-registry" -o jsonpath="{.items[0].metadata.name}")
          echo "Visit http://127.0.0.1:8080 to use your application"
          kubectl -n default port-forward $POD_NAME 8080:5000
        =======================================================================
        = docker-registry has been installed.                                 =
        =======================================================================
        
        # Your docker-registry has been configured
        
        kubectl logs deploy/docker-registry
        
        export IP="192.168.0.11" # Set to WiFI/ethernet adapter
        export PASSWORD="" # See below
        kubectl port-forward svc/docker-registry --address 0.0.0.0 5000 &
        
        docker login $IP:5000 --username admin --password $PASSWORD
        docker tag alpine:3.11 $IP:5000/alpine:3.11
        docker push $IP:5000/alpine:3.11
        
        # This chart is community maintained.
        # Find out more at:
        # https://github.com/twuni/docker-registry.helm
        # https://github.com/distribution/distribution
        
        🐳 arkade needs your support: https://github.com/sponsors/alexellis
        Registry credentials: admin 3B76uJdN05m5A2Xz63d3
        export PASSWORD=3B76uJdN05m5A2Xz63d3
        ➜  spinnaker git:(main) ✗
        ```
        
    - check pod
        
        ```jsx
        ➜  spinnaker git:(main) ✗ kubectl get pod --all-namespaces
        NAMESPACE            NAME                                         READY   STATUS      RESTARTS   AGE
        default              docker-registry-58f5d6d647-lppzt             1/1     Running     0          72s
        ingress-nginx        ingress-nginx-admission-create-5cx4b         0/1     Completed   0          16m
        ingress-nginx        ingress-nginx-admission-patch-ffjsg          0/1     Completed   0          16m
        ingress-nginx        ingress-nginx-controller-6bccc5966-hbct9     1/1     Running     0          16m
        kube-system          coredns-565d847f94-49smj                     1/1     Running     0          24m
        kube-system          coredns-565d847f94-kc7qg                     1/1     Running     0          24m
        kube-system          etcd-kind-control-plane                      1/1     Running     0          25m
        kube-system          kindnet-4zlkg                                1/1     Running     0          24m
        kube-system          kube-apiserver-kind-control-plane            1/1     Running     0          25m
        kube-system          kube-controller-manager-kind-control-plane   1/1     Running     0          25m
        kube-system          kube-proxy-cnnfm                             1/1     Running     0          24m
        kube-system          kube-scheduler-kind-control-plane            1/1     Running     0          25m
        local-path-storage   local-path-provisioner-684f458cdd-8t6v2      1/1     Running     0          24m
        ➜  spinnaker git:(main) ✗
        ```
        
    - self sign
        
        ```jsx
        export DOCKER_REGISTRY=docker.spinbook.local
        export DOCKER_EMAIL=docker@spinbook.local
        
        arkade install docker-registry-ingress --email $DOCKER_EMAIL --domain $DOCKER_REGISTRY
        ```
        
        - output
            
            ```jsx
            
            ➜  spinnaker git:(main) ✗
            export DOCKER_REGISTRY=docker.spinbook.local
            export DOCKER_EMAIL=docker@spinbook.local
            
            arkade install docker-registry-ingress --email $DOCKER_EMAIL --domain $DOCKER_REGISTRY
            
            Using Kubeconfig: /Users/wsg/.kube/config
            =======================================================================
            = Docker Registry Ingress and cert-manager Issuer have been installed =
            =======================================================================
            
            # You will need to ensure that your domain points to your cluster and is
            # accessible through ports 80 and 443.
            #
            # This is used to validate your ownership of this domain by LetsEncrypt
            # and then you can use https with your installation.
            
            # Ingress to your domain has been installed for the Registry
            # to see the ingress record run
            kubectl get -n <installed-namespace> ingress docker-registry
            
            # Check the cert-manager logs with:
            kubectl logs -n cert-manager deploy/cert-manager
            
            # A cert-manager Issuer has been installed into the provided
            # namespace - to see the resource run
            kubectl describe -n <installed-namespace> Issuer letsencrypt-prod-registry
            
            # To check the status of your certificate you can run
            kubectl describe -n <installed-namespace> Certificate docker-registry
            
            # It may take a while to be issued by LetsEncrypt, in the meantime a
            # self-signed cert will be installed
            
            🐳 arkade needs your support: https://github.com/sponsors/alexellis
            ➜  spinnaker git:(main) ✗
            ```
            
    - check ingress
        
        ```jsx
        
        ➜  ~ kubectl get ingress --all-namespaces
        NAMESPACE   NAME              CLASS    HOSTS                   ADDRESS     PORTS     AGE
        default     docker-registry   <none>   docker.spinbook.local   localhost   80, 443   11m
        ➜  ~
        ```
        
    - test docker login
        
        ```jsx
        
        ➜  spinnaker git:(main) ✗ docker login docker.spinbook.local
        Username: admin
        Password:
        Login Succeeded
        ➜  spinnaker git:(main) ✗
        
        ```
        
    - 
- 

## check service change

- berfore install cert-manager ,and
    
    ```jsx
    ➜  spinnaker git:(main) ✗ kubectl get svc --all-namespaces
    NAMESPACE       NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
    default         docker-registry                      ClusterIP   10.96.123.90    <none>        5000/TCP                     4m25s
    default         kubernetes                           ClusterIP   10.96.0.1       <none>        443/TCP                      28m
    ingress-nginx   ingress-nginx-controller             NodePort    10.96.164.149   <none>        80:32666/TCP,443:32197/TCP   19m
    ingress-nginx   ingress-nginx-controller-admission   ClusterIP   10.96.29.115    <none>        443/TCP                      19m
    kube-system     kube-dns                             ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP,9153/TCP       28m
    ```
    

## install halyard

```jsx
➜  spinnaker git:(main) ✗ docker run --name halyard \
-v ~/.hal:/home/spinnaker/.hal \
-v ~/.kind:/home/spinnaker/.kube \
-d --network kind \
gcr.io/spinnaker-marketplace/halyard:stable
Unable to find image 'gcr.io/spinnaker-marketplace/halyard:stable' locally
stable: Pulling from spinnaker-marketplace/halyard
9b794450f7b6: Pull complete
4604c29e94e4: Pull complete
681d52438c76: Pull complete
46616e4184c8: Pull complete
a977c2a1abca: Pull complete
9898a26465c7: Pull complete
af773600f4c1: Pull complete
098cf5eb8bd1: Pull complete
Digest: sha256:c948428fa99c76a45866dcd0b96d7e43c7e77ae893777570d6272e60553f92f3
Status: Downloaded newer image for gcr.io/spinnaker-marketplace/halyard:stable
d3ce75f858db53732f39e8518932536c29ee243ecb78a2268333625ec9cd29c1
➜  spinnaker git:(main) ✗ docker exec -it halyard bash
bash-5.0$
```

## halyard config provider

```jsx
hal config provider kubernetes enable
```

- full output
    
    ```jsx
    bash-5.0$ hal config provider kubernetes enable
    
    + Get current deployment
      Success
    + Edit the kubernetes provider
      Success
    Validation in default:
    - WARNING You have not yet selected a version of Spinnaker to
      deploy.
    ? Options include:
      - 1.29.2
      - 1.28.4
      - 1.27.3
    
    Validation in halconfig:
    - WARNING There is a newer version of Halyard available (1.54.0),
      please update when possible
    ? Run 'sudo apt-get update && sudo apt-get install
      spinnaker-halyard -y' to upgrade
    
    Validation in default.provider.kubernetes:
    - WARNING Provider kubernetes is enabled, but no accounts have been
      configured.
    
    + Successfully enabled kubernetes
    bash-5.0$
    ```
    

## halyard add kubernetes account

```jsx
bash-5.0$ CONTEXT=$(kubectl config current-context)
bash-5.0$ hal config provider kubernetes account add localk8s \
> --context=$CONTEXT
+ Get current deployment
  Success
+ Add the localk8s account
  Success
Validation in default:
- WARNING You have not yet selected a version of Spinnaker to
  deploy.
? Options include:
  - 1.29.2
  - 1.28.4
  - 1.27.3

Validation in halconfig:
- WARNING There is a newer version of Halyard available (1.54.0),
  please update when possible
? Run 'sudo apt-get update && sudo apt-get install
  spinnaker-halyard -y' to upgrade

+ Successfully added account localk8s for provider kubernetes.
bash-5.0$
```

## choose spinnaker run env

```jsx
bash-5.0$ hal config deploy edit \
> --account-name localk8s \
> --type distributed \
> --location spinnaker

+ Get current deployment
  Success
+ Get the deployment environment
  Success
+ Edit the deployment environment
  Success
Validation in default:
- WARNING You have not yet selected a version of Spinnaker to
  deploy.
? Options include:
  - 1.29.2
  - 1.28.4
  - 1.27.3

Validation in halconfig:
- WARNING There is a newer version of Halyard available (1.54.0),
  please update when possible
? Run 'sudo apt-get update && sudo apt-get install
  spinnaker-halyard -y' to upgrade

+ Successfully updated your deployment environment.
bash-5.0$
bash-5.0$
```

## choose persist storage(minio)

- install minio
    
    ```jsx
    
    ➜  ~ helm install --namespace minio --generate-name minio/minio
    NAME: minio-1671522366
    LAST DEPLOYED: Tue Dec 20 15:46:06 2022
    NAMESPACE: minio
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    NOTES:
    Minio can be accessed via port 9000 on the following DNS name from within your cluster:
    minio-1671522366.minio.svc.cluster.local
    
    To access Minio from localhost, run the below commands:
    
      1. export POD_NAME=$(kubectl get pods --namespace minio -l "release=minio-1671522366" -o jsonpath="{.items[0].metadata.name}")
    
      2. kubectl port-forward $POD_NAME 9000 --namespace minio
    
    Read more about port forwarding here: http://kubernetes.io/docs/user-guide/kubectl/kubectl_port-forward/
    
    You can now access Minio server on http://localhost:9000. Follow the below steps to connect to Minio server with mc client:
    
      1. Download the Minio mc client - https://docs.minio.io/docs/minio-client-quickstart-guide
    
      2. Get the ACCESS_KEY=$(kubectl get secret minio-1671522366 -o jsonpath="{.data.accesskey}" | base64 --decode) and the SECRET_KEY=$(kubectl get secret minio-1671522366 -o jsonpath="{.data.secretkey}" | base64 --decode)
    
      3. mc alias set minio-1671522366-local http://localhost:9000 "$ACCESS_KEY" "$SECRET_KEY" --api s3v4
    
      4. mc ls minio-1671522366-local
    
    Alternately, you can use your browser or the Minio SDK to access the server - https://docs.minio.io/categories/17
    ```
    
- check minio
    
    ```jsx
    
    ➜  ~ kubectl get pod -n minio
    NAME                               READY   STATUS    RESTARTS   AGE
    minio-1671522366-57988cd57-8pbtn   1/1     Running   0          13m
    
    ➜  ~ kubectl get secret
    NAME                                    TYPE                 DATA   AGE
    docker-registry-secret                  Opaque               4      4h9m
    docker-registry-sfnzw                   Opaque               1      3h54m
    letsencrypt-prod-issuer                 Opaque               1      3h51m
    sh.helm.release.v1.docker-registry.v1   helm.sh/release.v1   1      4h9m
    
    ➜  ~ kubectl get secret -n minio
    NAME                                     TYPE                 DATA   AGE
    minio-1671522366                         Opaque               2      17m
    sh.helm.release.v1.minio-1671522366.v1   helm.sh/release.v1   1      17m
    ➜  ~
    ```
    
    get ak,sk
    
    ```jsx
    
    ```
    

- config s3
    
    ```jsx
    bash-5.0$ hal config storage s3 edit --endpoint http://minio-1671522366.minio.svc.cluster.local:9000 \
    > --access-key-id 4H0GVTzaIS95gNDJqFuZ% \
    > --secret-access-key CFTm39nsvjIYGqkwwhEWYsxXMRCspFP0LlNEJGaM% \
    > --path-style-access true
    + Get current deployment
      Success
    + Get persistent store
      Success
    + Edit persistent store
      Success
    Validation in default.persistentStorage:
    - WARNING Your deployment will most likely fail until you configure
      and enable a persistent store.
    
    Validation in default:
    - WARNING You have not yet selected a version of Spinnaker to
      deploy.
    ? Options include:
      - 1.29.2
      - 1.28.4
      - 1.27.3
    
    Validation in halconfig:
    - WARNING There is a newer version of Halyard available (1.54.0),
      please update when possible
    ? Run 'sudo apt-get update && sudo apt-get install
      spinnaker-halyard -y' to upgrade
    
    + Successfully edited persistent store "s3".
    bash-5.0$
    ```
    
- set s3
    
    ```jsx
    bash-5.0$ hal config storage edit --type s3
    + Get current deployment
      Success
    + Get persistent storage settings
      Success
    + Edit persistent storage settings
      Success
    Validation in halconfig:
    - WARNING There is a newer version of Halyard available (1.54.0),
      please update when possible
    ? Run 'sudo apt-get update && sudo apt-get install
      spinnaker-halyard -y' to upgrade
    
    + Successfully edited persistent storage.
    ```
    

- config front50
    
    ```jsx
    ➜  profiles pwd
    /Users/wsg/.hal/default/profiles
    ➜  profiles cat front50-local.yml
    spinnaker.s3.versioning: false
    ➜  profiles
    ```
    

# deploy

- version list
    
    ```jsx
    bash-5.0$ helm version list
    bash: helm: command not found
    bash-5.0$ hal version list
    + Get current deployment
      Success
    + Get Spinnaker version
      Success
    + Get released versions
      Success
    + You are on version "", and the following are available:
     - 1.27.3 (v1.27.3):
       Changelog: https://spinnaker.io/changelogs/1.27.3-changelog/
       Published: Thu Dec 08 23:44:08 GMT 2022
       (Requires Halyard >= 1.45)
     - 1.28.4 (v1.28.4):
       Changelog: https://spinnaker.io/changelogs/1.28.4-changelog/
       Published: Thu Dec 08 23:54:03 GMT 2022
       (Requires Halyard >= 1.45)
     - 1.29.2 (v1.29.2):
       Changelog: https://spinnaker.io/changelogs/1.29.2-changelog/
       Published: Fri Dec 09 00:03:26 GMT 2022
       (Requires Halyard >= 1.45)
    ```
    
- choose version
    
    ```jsx
    bash-5.0$ hal config version edit --version 1.29.2
    + Get current deployment
      Success
    + Edit Spinnaker version
      Success
    Validation in halconfig:
    - WARNING There is a newer version of Halyard available (1.54.0),
      please update when possible
    ? Run 'sudo apt-get update && sudo apt-get install
      spinnaker-halyard -y' to upgrade
    
    + Spinnaker has been configured to update/install version "1.29.2".
      Deploy this version of Spinnaker with `hal deploy apply`.
    bash-5.0$
    ```
    
- deploy
    
    ```jsx
    bash-5.0$ hal deploy apply
    + Get current deployment
      Success
    + Prep deployment
      Success
    Validation in default.stats:
    - INFO Stats are currently ENABLED. Usage statistics are being
      collected. Thank you! These stats inform improvements to the product, and that
      helps the community. To disable, run `hal config stats disable`. To learn more
      about what and how stats data is used, please see
      https://www.spinnaker.io/community/stats.
    
    Validation in halconfig:
    - WARNING There is a newer version of Halyard available (1.54.0),
      please update when possible
    ? Run 'sudo apt-get update && sudo apt-get install
      spinnaker-halyard -y' to upgrade
    
    Validation in default.security:
    - WARNING Your UI or API domain does not have override base URLs
      set even though your Spinnaker deployment is a Distributed deployment on a
      remote cloud provider. As a result, you will need to open SSH tunnels against
      that deployment to access Spinnaker.
    ? We recommend that you instead configure an authentication
      mechanism (OAuth2, SAML2, or x509) to make it easier to access Spinnaker
      securely, and then register the intended Domain and IP addresses that your
      publicly facing services will be using.
    
    + Preparation complete... deploying Spinnaker
    + Get current deployment
      Success
    + Apply deployment
      Success
    + Deploy spin-redis
      Success
    + Deploy spin-clouddriver
      Success
    + Deploy spin-front50
      Success
    + Deploy spin-orca
      Success
    + Deploy spin-deck
      Success
    + Deploy spin-echo
      Success
    + Deploy spin-gate
      Success
    + Deploy spin-rosco
      Success
    + Run `hal deploy connect` to connect to Spinnaker.
    bash-5.0$
    ```
    

# 效果

- 需要将 deck的9000端口，和gate的8084 端口同时 port-forwarding
    
    ![Untitled](../../images/devops-spinnaker%20ba09c1f4a4d649048d5c22999859193d/Untitled.png)
    

![Untitled](../../images/devops-spinnaker%20ba09c1f4a4d649048d5c22999859193d/Untitled%201.png)

# 问题处理记录

## front pod 报错

```jsx
org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'serviceAccountsController' defined in URL [jar:file:/opt/front50/lib/front50-web-2.27.1.jar!/com/netflix/spinnaker/front50/controllers/ServiceAccountsController.class]: Unsatisfied dependency expressed through constructor parameter 0; nested exception is org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'serviceAccountsService' defined in URL [jar:file:/opt/front50/lib/front50-core-2.27.1.jar!/com/netflix/spinnaker/front50/ServiceAccountsService.class]: Unsatisfied dependency expressed through constructor parameter 0; nested exception is org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'serviceAccountDAO' defined in class path resource [com/netflix/spinnaker/front50/config/CommonStorageServiceDAOConfig.class]: Unsatisfied dependency expressed through method 'serviceAccountDAO' parameter 0; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 's3StorageService' defined in class path resource [com/netflix/spinnaker/front50/config/S3Config.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [com.netflix.spinnaker.front50.model.S3StorageService]: Factory method 's3StorageService' threw exception; nested exception is com.amazonaws.services.s3.model.AmazonS3Exception: Forbidden (Service: Amazon S3; Status Code: 403; Error Code: 403 Forbidden; Request ID: 173279D50628A01A; S3 Extended Request ID: null; Proxy: null), S3 Extended Request ID: null
	at org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(ConstructorResolver.java:799) ~[spring-beans-5.2.15.RELEASE.jar:5.2.15.RELEASE]
```

一行太长，

Failed to instantiate [com.netflix.spinnaker.front50.model.S3StorageService]: Factory method 's3StorageService' threw exception; nested exception is com.amazonaws.services.s3.model.AmazonS3Exception: Forbidden (Service: Amazon S3; Status Code: 403; Error Code: 403 Forbidden; Request ID: 173279D50628A01A; S3 Extended Request ID: null; Proxy: null), S3 Extended Request ID: null
at org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(ConstructorResolver.java:799) ~[spring-beans-5.2.15.RELEASE.jar:5.2.15.RELEASE]

### resolved by

重新配置了 minio的 s3配置， 原来的ak，sk 末尾 多了%，

增加了配置，禁用version

然后 hal deploy apply
