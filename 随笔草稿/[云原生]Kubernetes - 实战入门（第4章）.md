# [云原生]Kubernetes - 实战入门（第4章）

[toc]

**参考：**

- [Kubernetes(K8S) 入门进阶实战完整教程，黑马程序员K8S全套教程（基础+高级）_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1Qv41167ck?spm_id_from=333.999.0.0)



本章节将介绍如何在Kubernetes集群中部署一个nginx服务，并且能够对其进行访问。

## 一、Namespace

Namespace是Kubernetes系统中的一种非常重要的资源，它的主要作用是用来实现**多套环境的资源隔离**或者**多租户的资源隔离**。

默认情况下，Kubernetes集群中的所有的Pod都是可以相互访问的。但是在实际中，可能不想让两个Pod之间进行互相的访问，那此时就可以将两个Pod划分到不同的namespace下。Kubernetes通过将集群内部的资源分配到不同的Namespace中，可以形成逻辑上的“组”，以方便不同的组的资源进行隔离使用和管理。

可以通过Kubernetes的授权机制，将不同的namespace交给不同租户进行管理，这样就实现了多租户的资源隔离。此时还能结合Kubernetes的资源配额机制，限定不同租户能占用的资源，例如CPU使用量、内存使用量等等，来实现租户可用资源的管理。

![image-20220309112132358](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220309112132358.png)

Kubernetes在集群启动之后，会默认创建几个namespace

```shell
[root@k8s-master ~]# kubectl get namespace
NAME              STATUS   AGE
default           Active   19h
kube-node-lease   Active   19h
kube-public       Active   19h
kube-system       Active   19h
```

主要有以下几个默认的namespace：

- default：所有未指定Namespace的对象都会被分配在default命名空间
- kube-node-lease：集群节点之间的心跳维护，v1.13开始引入
- kube-public：此命名空间下的资源可以被所有人访问（包括未认证用户）
- kube-system：所有由Kubernetes系统创建的资源都处于这个命名空间

下面来看namespace资源的具体操作：

**查看**

```shell
# 1. 查看所有的ns 命令：kubectl get ns
[root@k8s-master ~]# kubectl get ns
NAME              STATUS   AGE
default           Active   19h
dev               Active   60m
kube-node-lease   Active   19h
kube-public       Active   19h
kube-system       Active   19h

# 2. 查看指定的ns 命令：kubectl get ns ns名称
[root@k8s-master ~]# kubectl get ns default
NAME      STATUS   AGE
default   Active   19h

# 3. 指定输出格式 命令：kubectl get ns ns名称 -o 格式参数
# Kubernetes支持的格式有很多，比较常见的是wide、json、yaml
[root@k8s-master ~]# kubectl get ns default -o yaml
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: "2022-03-08T07:33:59Z"
  labels:
    kubernetes.io/metadata.name: default
  name: default
  resourceVersion: "206"
  uid: e97c05c1-1842-4ba7-86ff-5254b9e6896c
spec:
  finalizers:
  - kubernetes
status:
  phase: Active


# 4. 查看ns详情 命令：kubectl describe ns ns名称
[root@k8s-master ~]# kubectl describe ns default
Name:         default
Labels:       kubernetes.io/metadata.name=default
Annotations:  <none>
Status:       Active

# Resource Quota 针对namespace做资源限制
No resource quota.
# LimitRange    针对namespace中的每个组件做资源限制
No LimitRange resource.
```

**创建**

```shell
[root@k8s-master ~]# kubectl create ns dev
namespace/dev created
```

**删除**

```
[root@k8s-master ~]# kubectl delete ns dev
namespace "dev" deleted
```

**配置方式**

首先准备一个yaml文件：ns-dev.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

然后就可以执行对应的创建和删除命令了：

创建：`kubectl create -f ns-dev.yaml`

删除：`kubectl delete -f ns-dev.yaml`



## 二、Pod

Pod是Kubernetes集群进行管理的最小单元，程序要运行必须部署在容器中，而容器必须存在于Pod中。

Pod可以认为是容器的封装，一个Pod中可以存在一个或者多个容器。

![image-20220309113728112](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220309113728112.png)

Kubernetes在集群启动之后，集群中的各个组件也都是以Pod方式运行的。可以通过下面命令查看：

```shell
[root@k8s-master k8s-yaml]# kubectl get pod -n kube-system
NAME                                 READY   STATUS    RESTARTS       AGE
coredns-6d8c4cb4d-cj7bc              1/1     Running   3 (139m ago)   20h
coredns-6d8c4cb4d-h8x4t              1/1     Running   3 (139m ago)   20h
etcd-k8s-master                      1/1     Running   3 (139m ago)   20h
kube-apiserver-k8s-master            1/1     Running   3 (139m ago)   20h
kube-controller-manager-k8s-master   1/1     Running   3 (139m ago)   20h
kube-flannel-ds-5xs6m                1/1     Running   3 (139m ago)   19h
kube-flannel-ds-lz2wn                1/1     Running   3 (139m ago)   19h
kube-flannel-ds-nn4xs                1/1     Running   2 (139m ago)   19h
kube-proxy-j9h94                     1/1     Running   3 (139m ago)   19h
kube-proxy-tm4g7                     1/1     Running   2 (139m ago)   19h
kube-proxy-vxnj4                     1/1     Running   3 (139m ago)   20h
kube-scheduler-k8s-master            1/1     Running   3 (139m ago)   20h
```

**创建并运行**

```shell
# 命令格式： kubectl run (pod控制器名称) [参数] 
# --image  指定Pod的镜像
# --port   指定端口
# --namespace  指定namespace
[root@k8s-master k8s-yaml]# kubectl run nginx --image=nginx:latest --port=80 --namespace dev
pod/nginx created
```

**查看Pod信息**

```shell
# 查看Pod基本信息
[root@k8s-master k8s-yaml]# kubectl get pods -n dev
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          26s

# 查看Pod的详细信息
[root@k8s-master k8s-yaml]# kubectl describe pod nginx -n dev
Name:         nginx
Namespace:    dev
Priority:     0
Node:         k8s-node02/10.0.8.30
Start Time:   Wed, 09 Mar 2022 11:40:17 +0800
Labels:       run=nginx
Annotations:  <none>
Status:       Running
IP:           10.244.2.6
IPs:
  IP:  10.244.2.6
Containers:
  nginx:
    Container ID:   docker://2038972f9ff5edfd21586928f18256654da4fccac358900f4e0f0fd9b592c049
    Image:          nginx:latest
    Image ID:       docker-pullable://nginx@sha256:0d17b565c37bcbd895e9d92315a05c1c3c9a29f762b011a10c54a66cd53c9b31
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 09 Mar 2022 11:40:34 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-wtrb7 (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-wtrb7:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  38s   default-scheduler  Successfully assigned dev/nginx to k8s-node02
  Normal  Pulling    37s   kubelet            Pulling image "nginx:latest"
  Normal  Pulled     22s   kubelet            Successfully pulled image "nginx:latest" in 15.526055388s
  Normal  Created    21s   kubelet            Created container nginx
  Normal  Started    21s   kubelet            Started container nginx
```

**访问Pod**

```shell
# 获取Pod IP
[root@k8s-master k8s-yaml]# kubectl get pods -n dev -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE         NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          98s   10.244.2.6   k8s-node02   <none>           <none>

# 访问Pod
[root@k8s-master k8s-yaml]# curl http://10.244.2.6:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

**删除指定Pod**

```shell
[root@k8s-master k8s-yaml]# kubectl delete pod nginx -n dev
pod "nginx" deleted
```

**配置操作**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: dev
spec:
  containers:
  - image: nginx:latest
    name: pod
    ports:
    - name: nginx-port
      containerPort: 80
      protocol: TCP
```

然后就可以执行对应的创建和删除命令了：

创建：`kubectl create -f pod-nginx.yaml`

删除：`kubectl delete -f pod-nginx.yaml`



## 三、Label

Label是Kubernetes系统中第一个重要概念。它的作用就是在资源上添加标识，用来对它们进行区分和选择。

Label的特点：

- 一个 Label 会以 Key/value 键值对的形式附加到各种对象上，如 Node、Pod、Service 等等
- 一个资源对象可以定义任意数量的 Label，同一个 Label 也可以被添加到任意数量的资源对象上
- Label 通常在资源对象定义时确定，当然也可以在对象创建后动态添加或删除

可以通过 Label 实现资源的多维度分组，以便灵活、方便地进行资源分配、调度、配置、部署等管理工作。

> 一些常用的Label，示例如下：
>
> - 版本标签："version":"release"，"version":"stable" ...
> - 环境标签："environment":"dev"，"environment":"test"，"envrionment":"pro"
> - 架构标签："tier":"frontend"，"tier":"backend"

标签定义完毕之后，还要考虑到标签的选择，这就要使用到 Label Selector，即：

- Label 用于给某个资源对象

- Label Selector 用于查询和筛选拥有某些标签的资源对象

当前有两种 Label Selector：

- 基于等式的 Label Selector

  name = slave：选择所有包含 Label 中 key="name" 且  value="slave"的对象

  env != production：选择所有包含 Label 中的 key="env" 且 value不等于"frontend"的对象

- 基于集合的 Label Selector

  name in (master，slave)：选择所有包含 Label 中的 key="name" 且 value="master"或"slave"的对象

  name not in (frontend)：选择所有包含 Label 中的 key="name" 且 value不等于"frontend"的对象

标签的选择条件可以使用多个，此时将多个 Label Selector 进行组合，使用逗号 "," 进行分隔即可。例如：

- name=slave，env!=production

- name not in (frontend)，env!=production

**命令方式**

```shell
# 为Pod资源打标签
[root@k8s-master k8s-yaml]# kubectl label pod nginx-pod version=1.0 -n dev
pod/nginx-pod labeled

# 为Pod资源更新标签
[root@k8s-master k8s-yaml]# kubectl label pod nginx-pod version=2.0 -n dev --overwrite
pod/nginx-pod labeled

# 查看标签
[root@k8s-master k8s-yaml]# kubectl get pod nginx-pod -n dev --show-labels
NAME        READY   STATUS    RESTARTS   AGE     LABELS
nginx-pod   1/1     Running   0          7m16s   version=2.0

# 筛选标签
[root@k8s-master k8s-yaml]# kubectl get pod -n dev -l version=2.0 --show-labels
NAME        READY   STATUS    RESTARTS   AGE     LABELS
nginx-pod   1/1     Running   0          7m37s   version=2.0

[root@k8s-master k8s-yaml]# kubectl get pod -n dev -l version!=2.0 --show-labels
No resources found in dev namespace.

#删除标签（在后面加一个减号"-"）
[root@k8s-master k8s-yaml]# kubectl label pod nginx-pod version- -n dev
pod/nginx-pod unlabeled

```

**配置方式**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: dev
  labels:
   version: "3.0"
   env: "test"
spec:
  containers:
  - images: nginx:latest
    name: pod
    ports:
    - name: nginx-port
      containerPort: 80
      protocol: TCP
```

然后就可以执行对应的更新命令了： `kubectl apply -f pod-nginx.yaml`



## 四、Deployment

在Kubernetes中，Pod是最小的控制单元，但是Kubernetes很少直接控制Pod，一般都是通过Pod控制器来完成的。Pod控制器用于Pod的管理，确保Pod资源符合预期的状态，当Pod的资源出现故障时，会尝试进行重启或重建Pod。

在Kubernetes中Pod控制器的种类有很多，本章节只介绍其中一种：Deployment

![image-20220309154431765](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220309154431765.png)

**命令操作**

```shell
# 命令格式: kubectl create deployment 名称  [参数] 
# --image  指定pod的镜像
# --port   指定端口
# --replicas  指定创建pod数量
# --namespace  指定namespace

# 1. 通过Deployment创建Pod，三副本形式
[root@k8s-master ~]# kubectl create deploy nginx --image=nginx:latest --port=80 --replicas=3 -n dev
deployment.apps/nginx created

# 2. 查看步骤一中创建的Pod
[root@k8s-master ~]# kubectl get pods -n dev
NAME                    READY   STATUS    RESTARTS   AGE
nginx-8d545c96d-wf92k   1/1     Running   0          41s
nginx-8d545c96d-xv4j4   1/1     Running   0          41s
nginx-8d545c96d-zt6ks   1/1     Running   0          41s

# 3. 查看deployment的信息
[root@k8s-master ~]# kubectl get deploy -n dev
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   3/3     3            3           88s

[root@k8s-master ~]# kubectl get deploy -n dev -o wide
NAME    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
nginx   3/3     3            3           92s   nginx        nginx:latest   app=nginx

# UP-TO-DATE：成功升级的副本数量
# AVAILABLE：可用副本的数量

# 4. 查看deployment的详细信息
[root@k8s-master ~]# kubectl describe deploy nginx -n dev
Name:                   nginx
Namespace:              dev
CreationTimestamp:      Wed, 09 Mar 2022 16:11:53 +0800
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:latest
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-8d545c96d (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  2m3s  deployment-controller  Scaled up replica set nginx-8d545c96d to 3

# 5. 删除
[root@k8s-master ~]# kubectl delete deploy nginx -n dev
deployment.apps "nginx" deleted
```

备注：如果有Pod卡在Terminating状态，可以用 `kubectl delete pod [pod名字] --force --grace-period=0 -n [namespace] ` 强制删除。

**配置操作**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx:latest
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
```

然后就可以执行对应的创建和删除命令了：

创建：`kubectl create -f deploy-nginx.yaml`

删除：`kubectl delete -f deploy-nginx.yaml`



## 五、Service

上面学习了利用Deployment来创建一组Pod来提供具有高可用性的服务。

虽然每个Pod都会分配一个单独的Pod IP，然而却存在如下两个问题：

- Pod IP 会随着Pod的重建产生变化
- Pod IP 仅仅时集群内可见的虚拟IP，外部无法访问

这样对于访问这个服务带来了难度。因此，Kubernetes设计了Service来解决这个问题。

Service可以看作是一组同类Pod对外的访问接口。借助Service，应用可以方便地实现服务发现和负载均衡。

![image-20220309165623536](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220309165623536.png)

**操作一：创建集群内部可访问的Service**

```shell
# 1. 暴露service
kubectl expose deploy nginx --name=svc-nginx1 --type=ClusterIP --port=80 --target-port=80 -n dev

# 2. 查看service
[root@k8s-master ~]# kubectl get svc svc-nginx1 -n dev -o wide
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE   SELECTOR
svc-nginx1   ClusterIP   10.111.39.137   <none>        80/TCP    20s   app=nginx

# 这里产生了一个CLUSTER-IP，这就是service的IP，在service的生命周期中，这个地址是不变的

# 可以通过这个IP访问当前service对应的pod
[root@k8s-master ~]# curl 10.111.39.137
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

**操作二：创建集群外部也可访问的Service**

```shell
# 上面创建的Service的type类型为ClusterIP，这个ip地址只用集群内部可访问
# 如果需要创建外部也可以访问的Service，需要修改type为NodePort
[root@k8s-master ~]# kubectl expose deploy nginx --name=svc-nginx2 --type=NodePort --port=80 --target-port=80 -n dev
service/svc-nginx2 exposed

[root@k8s-master ~]# kubectl get svc svc-nginx2 -n dev -o wide
NAME         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE   SELECTOR
svc-nginx2   NodePort   10.99.160.158   <none>        80:32501/TCP   20s   app=nginx
```

接下来就可以通过集群外的主机访问 节点IP + 暴露的端口，来访问服务了

例如在电脑主机上通过浏览器访问下面的地址：

![image-20220309173733857](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220309173733857.png)

**删除Service**

```shell
[root@k8s-master ~]# kubectl delete svc svc-nginx1 -n dev
service "svc-nginx1" deleted
```

**配置方式**

 创建一个svc-nginx.yaml，内容如下：

```shell
apiVersion: v1
kind: Service
metadata:
  name: svc-nginx
  namespace: dev
spec:
  clusterIP: 10.109.179.231 #固定svc的内网ip
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
  type: ClusterIP
```

然后就可以执行对应的创建和删除命令了：

创建：`kubectl create -f svc-nginx.yaml`

删除：`kubectl delete -f svc-nginx.yaml`

> **小结：**至此，已经掌握了 Namespace、Pod、Deployment、Service 资源的基本操作，有了这些操作，就可以在 kubernetes 集群中实现一个服务的简单部署和访问了，但是如果想要更好的使用kubernetes，就需要深入学习这几种资源的细节和原理。