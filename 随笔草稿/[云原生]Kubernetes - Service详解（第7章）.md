# [云原生]Kubernetes - Service详解（第7章）

[toc]

## 一、Service介绍

在 Kubernetes 中，Pod是应用程序的载体，我们可以通过 Pod 的 IP 来访问应用程序，但是Pod的IP地址不是固定的，这就意味着不方便直接采用 Pod 的 IP 对服务进行访问。

为了解决这个问题，Kubernetes 提供了 Service 资源，Service 会对提供同一个服务的多个 Pod 进行聚合，并且提供一个统一的入口地址。通过访问 Service 的入口地址就能访问到后面的 Pod 服务。

![image-20220427145318151](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220427145318151.png)

Service 在很多情况下只是一个概念，真正起作用的其实是 kube-proxy 服务进程，每个 Node 节点上都运行着一个 kube-proxy 服务进程。

当创建 Service 的时候会通过 api--server 向 etcd 写入创建的 Service 的信息，而 kube-proxy会基于监听的机制发现这种 Service 的变动，然后**它会将最新的 Service 信息转换成对应的访问规则。**

![image-20220427145348620](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220427145348620.png)



```shell
# 10.97.97.97:80 是service提供的访问入口
# 当访问这个入口的时候，可以发现后面有三个pod的服务在等待调用，
# kube-proxy会基于rr（轮询）的策略，将请求分发到其中一个pod上去
# 这个规则会同时在集群内的所有节点上都生成，所以在任何一个节点上访问都可以。
[root@node1 ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.97.97.97:80 rr
  -> 10.244.1.39:80               Masq    1      0          0
  -> 10.244.1.40:80               Masq    1      0          0
  -> 10.244.2.33:80               Masq    1      0          0
```

kube-proxy目前支持三种工作模式：

**userspace 模式：**

userspace 模式下，kube-proxy 会为每一个 Service 创建一个监听端口，发向 Cluster IP 的请求被 Iptables 规则重定向到 kube-proxy监听的端口上，kube-proxy 根据 LB算法选择一个提供服务的 Pod 并和其建立链接，以将请求转发到 Pod 上。该模式下，kube-proxy充当了一个四层负载均衡器的角色。由于 kube-proxy 运行在 userspace 中，在进行转发处理时会增加内核和用户空间之间的数据拷贝，虽然比较稳定，但是效率比较低。

![image-20220427154132118](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220427154132118.png)

**iptables 模式：**

iptables 模式下，kube-proxy 为 Service 后端的每个 Pod 创建对应的 Iptables 规则，直接将发向Cluster IP的请求重定向到一个 Pod IP。该模式下 kube-proxy 不承担四层负载均衡器的角色，只负责创建 Iptables 规则。该模式的优点是较 userspace 模式效率更高，但不能提供灵活的LB策略，当后端 Pod 不可用时也就无法进行重试。

![image-20220427154142375](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220427154142375.png)

**ipvs 模式：**

ipvs 模式和 Iptables 类似，kube-proxy监控 Pod 的变化并创建相应的 ipvs 规则。ipvs 相对 Iptables 转发效率更高。除此之外，ipvs 支持更多的LB算法。

![image-20220427154146869](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220427154146869.png)

```shell
# 此模式必须安装 ipvs 内核模块，否则会降级为iptables
# 开启 ipvs
[root@k8s-master ~]# kubectl edit configmap kube-proxy -n kube-system
# 修改mode："type"
[root@k8s-master ~]# kubectl delete pod -l k8s-app=kube-proxy -n kube-system


[root@k8s-master k8s-yaml]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.1:443 rr
  -> 10.0.8.10:6443               Masq    1      0          0
TCP  10.96.0.10:53 rr
  -> 10.244.0.30:53               Masq    1      0          0
  -> 10.244.0.31:53               Masq    1      0          0
TCP  10.96.0.10:9153 rr
  -> 10.244.0.30:9153             Masq    1      0          0
  -> 10.244.0.31:9153             Masq    1      0          0
TCP  10.111.19.126:443 rr
UDP  10.96.0.10:53 rr
  -> 10.244.0.30:53               Masq    1      0          0
  -> 10.244.0.31:53               Masq    1      0          0
```



## 二、Service类型

Service的资源清单文件：

```yaml
kind: Service  # 资源类型
apiVersion: v1  # 资源版本
metadata: # 元数据
  name: service # 资源名称
  namespace: dev # 命名空间
spec: # 描述
  selector: # 标签选择器，用于确定当前service代理哪些pod
    app: nginx
  type: # Service类型，指定service的访问方式
  clusterIP:  # 虚拟服务的ip地址
  sessionAffinity: # session亲和性，支持ClientIP、None两个选项
  ports: # 端口信息
    - protocol: TCP 
      port: 3017  # service端口
      targetPort: 5003 # pod端口
      nodePort: 31122 # 主机端口
```

- ClusterIP：默认值，它是 Kubernetes 系统自动分配的虚拟IP，只能在集群内部访问
- NodePort：将 Service 通过指定的 Node 上的端口暴露给外部，通过此方法，就可以在集群外部访问服务
- LoadBalancer：使用外接负载均衡器完成到服务的负载分发，注意此模式需要外部云环境支持
- ExternalName：把集群外部的服务引入集群内部，直接使用



## 三、Service使用

### 3.1 实验环境准备

在使用 Service 之前，首先利用 Deployment 创建出3个 Pod，注意要为 Pod 设置 `app=nginx-pod`的标签

创建 deployment.yaml，内容如下：

```yaml
apiVersion: apps/v1
kind: Deployment      
metadata:
  name: pc-deployment
  namespace: dev
spec: 
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
```

```shell
[root@k8s-master k8s-yaml]# kubectl create -f deployment.yaml
deployment.apps/pc-deployment created

# 查看 Pod 详情
[root@k8s-master k8s-yaml]# kubectl get pods -n dev -o wide --show-labels
NAME                             READY   STATUS    RESTARTS   AGE   IP             NODE         NOMINATED NODE   READINESS GATES   LABELS
pc-deployment-6756f95949-5ch9k   1/1     Running   0          49s   10.244.1.165   k8s-node01   <none>           <none>            app=nginx-pod,pod-template-hash=6756f9                 5949
pc-deployment-6756f95949-f7md8   1/1     Running   0          49s   10.244.2.45    k8s-node02   <none>           <none>            app=nginx-pod,pod-template-hash=6756f9                 5949
pc-deployment-6756f95949-rnbvw   1/1     Running   0          49s   10.244.1.164   k8s-node01   <none>           <none>            app=nginx-pod,pod-template-hash=6756f9

# 为了方便后面的测试，修改三台 Nginx 的 index.html 页面（内容为其本身IP地址）
# kubectl exec -it pc-deployment-6756f95949-5ch9k -n dev -- bash

# 修改完毕之后，访问测试
[root@k8s-master k8s-yaml]# curl 10.244.1.165
10.244.1.165
[root@k8s-master k8s-yaml]# curl 10.244.2.45
10.244.2.45
[root@k8s-master k8s-yaml]# curl 10.244.1.164
10.244.1.164
```

### 3.2 ClusterIP类型的Service

创建 service-clusterip.yaml 文件

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-clusterip
  namespace: dev
spec:
  selector:
    app: nginx-pod
  clusterIP: 10.97.97.97 # service的ip地址，如果不写，默认会生成一个
  type: ClusterIP
  ports:
  - port: 80  # Service端口       
    targetPort: 80 # pod端口
```

```shell
# 创建service
[root@k8s-master k8s-yaml]# kubectl create -f service-clusterip.yaml
service/service-clusterip created

# 查看service
[root@k8s-master k8s-yaml]# kubectl get svc -n dev -o wide
NAME                TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service-clusterip   ClusterIP   10.97.97.97   <none>        80/TCP    9s    app=nginx-pod


# 查看service的详细信息
# 在这里有一个Endpoints列表，里面就是当前service可以负载到的服务入口
[root@k8s-master k8s-yaml]# kubectl describe svc service-clusterip -n dev
Name:              service-clusterip
Namespace:         dev
Labels:            <none>
Annotations:       <none>
Selector:          app=nginx-pod
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.97.97.97
IPs:               10.97.97.97
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.1.164:80,10.244.1.165:80,10.244.2.45:80
Session Affinity:  None
Events:            <none>


# 查看ipvs的映射规则
[root@k8s-master k8s-yaml]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.97.97.97:80 rr
  -> 10.244.1.164:80              Masq    1      0          0
  -> 10.244.1.165:80              Masq    1      0          0
  -> 10.244.2.45:80               Masq    1      0          0



# 访问10.97.97.97:80观察效果
[root@k8s-master k8s-yaml]# curl 10.97.97.97:80
10.244.2.45
```

**Endpoint**

Endpoint 是 Kubernetes 中的一个资源对象，存储在 etcd 中，用来记录一个 Service 对应的所有 Pod 的访问地址，它是根据 Service 配置文件中 selector 描述产生的。

一个 Service 由一组 Pod 组成，这些 Pod 通过 Endpoints 暴露出来，Endpoints是实现实际服务的端点集合，换句话说，Service 和 Pod 之间的联系是通过 endpoints 实现的。

![image-20220428153852720](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220428153852720.png)

**负载分发策略**

对 Service 的访问杯分发到了后端的 Pod 上去，目前 Kubernetes 提供了两种负载分发策略：

- 如果不定义，默认使用 kube-proxy 的策略，比如随机、轮询
- 基于客户端地址的会话保持模式，即来自同一个客户端发起的所有请求都会转发到固定的一个 Pod 上
  此模式可以使在 spec 中添加`sessionAffinity:ClientIP`选项

```shell
# 查看ipvs的映射规则【rr 轮询】
[root@k8s-master k8s-yaml]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.97.97.97:80 rr
  -> 10.244.1.164:80              Masq    1      0          0
  -> 10.244.1.165:80              Masq    1      0          0
  -> 10.244.2.45:80               Masq    1      0          0


# 循环访问测试
[root@k8s-master k8s-yaml]# while true;do curl 10.97.97.97:80; sleep 2; done;
10.244.2.45
10.244.1.165
10.244.1.164
10.244.2.45
10.244.1.165
10.244.1.164
10.244.2.45


# 修改分发策略 --- sessionAffinity: ClientIP
[root@k8s-master k8s-yaml]# vim service-clusterip.yaml
[root@k8s-master k8s-yaml]# kubectl delete -f service-clusterip.yaml
service "service-clusterip" deleted
[root@k8s-master k8s-yaml]# kubectl create -f service-clusterip.yaml


# 查看ipvs规则【persistent 代表持久】
[root@k8s-master k8s-yaml]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.97.97.97:80 rr persistent 10800
  -> 10.244.1.164:80              Masq    1      0          2
  -> 10.244.1.165:80              Masq    1      0          3
  -> 10.244.2.45:80               Masq    1      0          3


# 循环访问测试
[root@k8s-master k8s-yaml]# while true; do curl 10.97.97.97; sleep 2; done;
10.244.2.45
10.244.2.45
10.244.2.45


# 删除Service
[root@k8s-master k8s-yaml]# kubectl delete -f service-clusterip.yaml
service "service-clusterip" deleted
```

### 3.3 HeadLiness类型的Service

在某些场景中，开发人员可能不想使用 Service 提供的负载均衡功能，而希望自己来控制负载均衡策略，针对这种情况，Kubernetes 提供了HeadLiness Service，这类 Service 不会分配 Cluster IP，如果想要访问 Service，只能通过 Service 的域名进行查询。

创建 service-headliness.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-headliness
  namespace: dev
spec:
  selector:
    app: nginx-pod
  clusterIP: None # 将clusterIP设置为None，即可创建headliness Service
  type: ClusterIP
  ports:
  - port: 80    
    targetPort: 80
```

```shell
# 创建Service
[root@k8s-master k8s-yaml]# kubectl create -f service-headliness.yaml
service/service-headliness created

# 获取Service，发现CLUSTER-IP未分配
[root@k8s-master k8s-yaml]# kubectl get svc service-headliness -n dev -o wide
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service-headliness   ClusterIP   None         <none>        80/TCP    19s   app=nginx-pod

# 查看Service详情
[root@k8s-master k8s-yaml]# kubectl describe svc service-headliness -n dev
Name:              service-headliness
Namespace:         dev
Labels:            <none>
Annotations:       <none>
Selector:          app=nginx-pod
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                None
IPs:               None
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.1.164:80,10.244.1.165:80,10.244.2.45:80
Session Affinity:  None
Events:            <none>

# 查看域名的解析情况
root@pc-deployment-6756f95949-5ch9k:/# cat /etc/resolv.conf
nameserver 10.96.0.10
search dev.svc.cluster.local svc.cluster.local cluster.local
options ndots:5

# yum -y install bind-utils（内置dig命令）
# 
[root@k8s-master k8s-yaml]# dig @10.96.0.10 service-headliness.dev.svc.cluster.local

;; ANSWER SECTION:
service-headliness.dev.svc.cluster.local. 30 IN A 10.244.1.164
service-headliness.dev.svc.cluster.local. 30 IN A 10.244.2.45
service-headliness.dev.svc.cluster.local. 30 IN A 10.244.1.165
```

### 3.4 NodePort类型的Service

在之前的样中，创建的 Service 的 ip 地址只有集群内部才可以访问，如果希望将 Service 暴露给集群外部使用，那么就要使用到另外一种类型的 Service，称为 NodePort 类型。NodePort 的工作原理其实就是**将 Service 的端口映射到 Node 的一个端口上**，然后就可以通过`NodeIP:NodePort`来访问 Service 了。

![image-20220428163337149](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220428163337149.png)

创建service-nodeport.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-nodeport
  namespace: dev
spec:
  selector:
    app: nginx-pod
  type: NodePort # service类型
  ports:
  - port: 80
    nodePort: 30002 # 指定绑定的node的端口(默认的取值范围是：30000-32767), 如果不指定，会默认分配
    targetPort: 80
```

```shell
# 创建Service
[root@k8s-master k8s-yaml]# kubectl create -f service-nodeport.yaml
service/service-nodeport created

# 查看Service
[root@k8s-master k8s-yaml]# kubectl get svc -n dev -o wide
NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE     SELECTOR
service-headliness   ClusterIP   None             <none>        80/TCP         6d19h   app=nginx-pod
service-nodeport     NodePort    10.108.149.157   <none>        80:30002/TCP   15s     app=nginx-pod
```

接下来可以通过电脑主机的浏览器去访问集群中任意一个nodeip的30002端口，即可访问到 Pod：

```shell
[root@k8s-master k8s-yaml]# kubectl get node -o wide
NAME         STATUS   ROLES                  AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
k8s-master   Ready    control-plane,master   57d   v1.23.4   10.0.8.10     <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://20.10.12
k8s-node01   Ready    <none>                 57d   v1.23.4   10.0.8.20     <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://20.10.12
k8s-node02   Ready    <none>                 57d   v1.23.4   10.0.8.30     <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://20.10.12
```

![image-20220505112251743](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220505112251743.png)



### 3.5 LoadBalancer类型的Service

LoadBalancer 和 NodePort 很相似，目的都是向外部暴露一个端口，区别在于 LoadBalancer 会在集群的外部再来做一个负载均衡设备，而这个设备需要外部环境支持，外部服务发送到这个设备上的请求，会被设备负载之后转发到集群中。

![image-20220505112959257](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220505112959257.png)

### 3.6 ExternalName类型的Service

ExternalName 类型的 Service 用于引入集群外部的服务，它通过`externalName`属性指定外部一个服务的地址，然后再集群内部访问此 Service 就可以访问到外部的服务了。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-externalname
  namespace: dev
spec:
  type: ExternalName
  externalName: www.baidu.com
```

```shell
# 创建Service
[root@k8s-master k8s-yaml]# kubectl create -f service-externalname.yaml
service/service-externalname created

# 域名解析
[root@k8s-master k8s-yaml]# dig @10.96.0.10 service-externalname.dev.svc.cluster.local
;; ANSWER SECTION:
service-externalname.dev.svc.cluster.local. 30 IN CNAME www.baidu.com.
www.baidu.com.          30      IN      CNAME   www.a.shifen.com.
www.a.shifen.com.       30      IN      A       163.177.151.110
www.a.shifen.com.       30      IN      A       163.177.151.109
```





## 四、Ingress介绍

在前面课程中已经提到，Service 对集群之外暴露服务的主要方式有两种：NotePort 和 LoadBalancer，但是这两种方式，都有一定的缺点：

- NodePort 方式的缺点是会占用很多集群机器的端口，那么当集群服务变多的时候，这个缺点就愈发明显
- LB方式的缺点是每个 Service 需要一个 LB，浪费、麻烦、并且需要 Kubernetes 之外设备的支持

基于这种现状，Kubernetes 提供 Ingress 资源对象，Ingress 只需要一个 NodePort 或者一个 LB 就可以满足暴露多个 Service 的需求。工作机制大致如下图表示：

![image-20220428145355613](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220428145355613.png)

实际上，Ingress 相当于一个7层的负载均衡器，是 Kubernetes 对反向代理的一个抽象，它的工作原理类似于 Nginx，可以理解成在 **Ingress 里建立诸多映射规则，Ingress Controller 通过监听这些配置规则并转化成 Nginx 的反向代理配置，然后对外部提供服务。**在这里有两个核心概念：

- Ingress：Kubernetes 中的一个对象，作用是定义请求如何转发到 Service 的规则
- Ingress controller：具体实现反向代理及负载均衡的程序，对 Ingress 定义的规则进行解析，根据配置的规则来实现请求转发，实现方式有 

Ingress（以Nginx为例）的工作原理如下：

1. 用户编写 Ingress 规则，说明哪个域名对应 Kubernetes 集群中的哪个 Service
2. Ingress 控制器动态感知 Ingress 服务规则的变化，然后生成一段对应的 Nginx 反向代理配置
3. Ingress 控制器会将生成的 Nginx 配置写入到一个运行着的 Nginx 服务中，并动态更新
4. 到此为止，其实真正在工作的就是一个 Nginx了，内部配置了用户定义的请求转发规则

![image-20220428150018906](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220428150018906.png)







## 五、Ingress使用

## 5.1 环境准备

**搭建Ingress环境**

```shell
# 创建文件夹
[root@k8s-master ~]# mkdir ingress-controller
[root@k8s-master ~]# cd ingress-controller/

# 获取ingress-nginx，本次案例使用的是0.30版本
[root@k8s-master ingress-controller]# wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml
[root@k8s-master ingress-controller]# wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/provider/baremetal/service-nodeport.yaml

# 修改mandatory.yaml文件中的仓库
# 修改quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0
# 为quay-mirror.qiniu.com/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0
# 输入 sed -i 's#rbac.authorization.k8s.io/v1beta1#rbac.authorization.k8s.io/v1#' mandatory.yaml
# 输入 sed -i 's#quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0#quay-mirror.qiniu.com/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0#' mandatory.yaml
# 创建ingress-nginx
[root@k8s-master ingress-controller]# kubectl apply -f ./

# 查看ingress-nginx
[root@k8s-master ingress-controller]# kubectl get pod -n ingress-nginx
NAME                                           READY   STATUS    RESTARTS   AGE
pod/nginx-ingress-controller-fbf967dd5-4qpbp   1/1     Running   0          12h

# 查看service
[root@k8s-master ingress-controller]# kubectl get svc -n ingress-nginx
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.98.75.163   <none>        80:32240/TCP,443:31335/TCP   11h
```

**准备 Service 和 Pod**

![image-20220505141645495](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220505141645495.png)

创建tomcat-nginx.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deployment
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tomcat-pod
  template:
    metadata:
      labels:
        app: tomcat-pod
    spec:
      containers:
      - name: tomcat
        image: tomcat:8.5-jre10-slim
        ports:
        - containerPort: 8080

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: dev
spec:
  selector:
    app: nginx-pod
  clusterIP: None
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
  namespace: dev
spec:
  selector:
    app: tomcat-pod
  clusterIP: None
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
```

```shell
# 创建
[root@k8s-master ~]# kubectl create -f tomcat-nginx.yaml

# 查看
[root@k8s-master ~]# kubectl get svc -n dev
NAME             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
nginx-service    ClusterIP   None         <none>        80/TCP     48s
tomcat-service   ClusterIP   None         <none>        8080/TCP   48s
```

## 5.2 HTTP代理

创建 ingress-http.yaml

```yaml
[root@k8s-master k8s-yaml]# vim ingress-http.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-http
  namespace: dev
spec:
  rules:
  - host: nginx.skybiubiu.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: nginx-service
            port:
              number: 80
  - host: tomcat.skybiubiu.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: tomcat-service
            port:
              number: 8080
```

```shell
# 创建
[root@k8s-master k8s-yaml]# kubectl create -f ingress-http.yaml
ingress.networking.k8s.io/ingress-http created

# 查看
[root@k8s-master k8s-yaml]# kubectl get ing ingress-http -n dev
NAME           CLASS    HOSTS                                      ADDRESS   PORTS   AGE
ingress-http   <none>   nginx.skybiubiu.com,tomcat.skybiubiu.com             80      9s

# 查看详情
[root@k8s-master k8s-yaml]# kubectl describe ing ingress-http -n dev
Name:             ingress-http
Labels:           <none>
Namespace:        dev
Address:
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host                  Path  Backends
  ----                  ----  --------
  nginx.skybiubiu.com
                        /   nginx-service:80 (10.244.1.180:80,10.244.1.181:80,10.244.2.48:80)
  tomcat.skybiubiu.com
                        /   tomcat-service:8080 (10.244.1.179:8080,10.244.1.182:8080,10.244.2.49:8080)
Annotations:            <none>
Events:                 <none>

# 接下来，在本地电脑上配置hosts文件，解析上面的两个域名到master节点IP上
10.0.8.10 nginx.skybiubiu.com
10.0.8.10 tomcat.skybiubiu.com

# 查看service服务端口，HTTP是31701
[root@k8s-master k8s-yaml]# kubectl get svc -n ingress-nginx
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.110.48.135   <none>        80:31701/TCP,443:31855/TCP   108m
[root@k8s-master k8s-yaml]#

# 然后，就可以分别访问tomcat.skybiubiu.com:31701 和 nginx.skybiubiu.com:31701 查看效果
```









## 5.3 HTTPS代理



