# [云原生]Kubernetes - Pod详解（第5章）

[toc]

## 一、Pod介绍

### 1.1 Pod结构

![image-20220310152016040](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220310152016040.png)

每个Pod中都可以包含一个或者多个容器，这些容器可以分为两类：

- 用户程序所在的容器，数量可多可少
- Pause容器，这是每个Pod都会有的一个**根容器**，它的作用有两个：
  - 可以以它为依据，评估整个Pod的健康状态。
  - 可以在根容器上设置IP地址，其它容器都用此IP（Pod IP），以实现Pod内部的网络通讯。（这里是Pod内部的通讯，Pod之间的通讯采用虚拟二层网络技术来实现，当前K8S环境用的是Flannel）

### 1.2 Pod定义

下面是Pod的资源清单：

```yaml
apiVersion: v1				#必选，版本号，例如：v1
kind: Pod					#必选，资源类型，例如：Pod
metadata:					#必选，元数据
  name: string				#必选，Pod名称
  namespace: string 		#Pod所属的命名空间，默认为"default"
  labels:					#自定义标签列表
    - name: string
spec:						#必选，Pod中容器的详细定义
  containers:				#必选，Pod中容器列表
  - name: string			#必选，容器名称
    image: string			#必选，容器的镜像名称
    imagePullPolicy: [ Always|Never|IfNotPresent ]	#获取镜像的策略
    command: [string]		#容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args: [string]			#容器的启动命令参数列表
    workingDir: string		#容器的工作目录
    volumeMounts:			#挂载到容器内部的存储卷配置
    - name: string			#引用Pod定义的共享存储卷的名称，需要volumes[]部分定义的卷名
      mountPath: string		#存储卷在容器内mount的绝对路径，应少于512字符
      readOnly: boolean		#是否为只读模式
    ports:					#需要暴露的端口号列表
    - name: string			#端口的名称
      containerPort: int	#容器需要监听的端口号
      hostPort: int			#容器所在主机需要监听的端口号，默认与Container相同
      protocol: string		#端口协议，支持TCP和UDP，默认TCP
    env:					#容器运行前需设置的环境变量列表
    - name: string			#环境变量名称
      value: string			#环境变量的值
    resources:				#资源限制和请求的设置
      limits:				#资源限制的设置
        cpu: string			#CPU的限制，单位为Core数，将用于docker run --cpu-shares参数
        memory: string		#内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
      requests:				#资源请求的设置
        cpu: string			#CPU请求，容量启动的初始可用CPU数量
        memory: string		#内存请求，容器启动的初始可用内存数量
      lifecycle:			#生命周期钩子
        postStart:			#容器启动后立即执行此钩子，如果执行失败，会根据重启策略进行重启
        preStop:			#容器终止前执行此钩子，无论结果如何，容器都会终止
      livenessProbe:		#对Pod内各容器健康检查的设置，当探测无响应几次后将自动重启该容器
        exec:				#对Pod内容内检查方式设置为exec方式
          command: [string]	#exec方式需要指定的命令或脚本
        httpGet:			#对Pod内各容器健康检查的方式设置为httpGet，需要指定Path、Port
          path: string		
          port: number
          host: string
          scheme: string
          HttpHeaders:
          - name: string
            value: string
       tcpSocket:				#对Pod内各容器健康检查的方式设置为tcpSocket方式
         port: number
        initialDelaySeconds: 0	#容器启动完成后首次探测的时间，单位为秒
        timeoutSeconds: 0		#对容器健康检查探测等待响应的超时时间，单位为秒，默认1秒
        periodSeconds: 0		#对容器监控检查的定期探测时间设置，单位为秒，默认10秒一次
        successThreshold: 0		
        failureThreshold: 0
        securityContext:
          privileged: false
  restartPolicy: [ Always|Never|Onfailure ] 	#Pod的重启策略
  nodeName: <string>		#设置nodeName表示将该Pod调度到指定名称的node节点上
  nodeSelector: object		#设置nodeSelector表示将该Pod调度到包含这个label的node上
  imagePullSecrets:			#Pull镜像时使用的secret名称，以key:secretkey格式指定
  - name: string
  hostNetwork: false		#是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
  volumes:					#在该Pod上定义共享存储卷列表
  - name: string			#共享存储卷名称（volumes类型有很多种）
    emptyDir: {}			#类型为emptyDir的存储卷，在与Pod同生命周期的一个临时目录，为空值
    hostPath: string		#类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
      path: string			#Pod所在宿主机的目录，将被用于同期中mount的目录
    secret:					#类型为secret的存储卷，挂载集群与定义的secret对象到容器内部
      scretname: string
      items:
      - key: string
        path: string
    configMap:				#类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
      name: string
      items: 
      - key: string
        path: string
    
```

```yaml
# 小提示：
#  在这里，可以通过一个命令来查看每种资源的可配置项
#  kubectl explain 资源类型					查看某种资源可以配置的一级属性
#  kubectl explain 资源类型.属性			   查看属性的子属性
[root@k8s-master ~]# kubectl explain pod
KIND:     Pod
VERSION:  v1
FIELDS:
   apiVersion   <string>
   kind <string>
   metadata     <Object>
   spec <Object>
   status       <Object>

[root@k8s-master ~]# kubectl explain pod.metadata
KIND:     Pod
VERSION:  v1
RESOURCE: metadata <Object>
FIELDS:
   annotations  <map[string]string>
   clusterName  <string>
   creationTimestamp    <string>
   deletionGracePeriodSeconds   <integer>
   deletionTimestamp    <string>
   finalizers   <[]string>
   generateName <string>
   generation   <integer>
   labels       <map[string]string>
   managedFields        <[]Object>
   name <string>
   namespace    <string>
   ownerReferences      <[]Object>
   resourceVersion      <string>
   selfLink     <string>
   uid  <string>
```

在 Kubernetes 中基本所有资源的**一级属性**都是一样的，主要包括5部分：

- apiVersion <string> 版本，由 Kubernetes 内部定义，可以用 `kubectl api-versions` 查询
- kind <string> 类型，由 Kubernetes 内部定义，可以用 `kubectl api-resources` 查询
- metadata <Object> 元数据，主要是资源标识和说明，常用的有 name、namespace、labels 等
- spec <Object> 描述，这是配置中最重要的一部分，里面是对各种资源配置的详细描述
- status <Object> 状态信息，里面的内容不需要定义，由 Kubernetes 自动生成

在上面的一级属性中，spec 是接下来研究的重点，继续看下它的常见子属性：

- containers <[]Object> 容器列表，用于定义容器的详细信息
- nodeName <String> 根据 nodeName 的值，将Pod调度到指定的 Node节点上
- nodeSelector <map[]> 根据 nodeSelector 中定义的信息选择将该 Pod 调度到包含这些 label 的 Node 上
- hostNetwork <boolean> 是否使用主机网络模式，默认为 false， 如果设置为 true，表示使用宿主机网络
- volumes <[]Object> 存储卷，用于定义Pod上面挂载的存储信息
- restartPolicy <string> 重启策略，表示Pod在遇到故障的时候的处理策略



## 二、Pod配置

本小节主要来研究 `pod.spec.containers` 属性，这也是Pod配置中最为关键的一项配置。

```shell
[root@k8s-master ~]# kubectl explain pod.spec.containers
KIND:     Pod
VERSION:  v1
RESOURCE: containers <[]Object>   # 数组，代表可以有多个容器
FIELDS:
   name  <string>     # 容器名称
   image <string>     # 容器需要的镜像地址
   imagePullPolicy  <string> # 镜像拉取策略 
   command  <[]string> # 容器的启动命令列表，如不指定，使用打包时使用的启动命令
   args     <[]string> # 容器的启动命令需要的参数列表
   env      <[]Object> # 容器环境变量的配置
   ports    <[]Object>     # 容器需要暴露的端口号列表
   resources <Object>      # 资源限制和资源请求的设置
```

### 2.1 基本配置

创建pod-base.yaml文件，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-base
  namespace: dev
  labels:
    user: skybiubiu
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  - name: busybox
    image: busybox:1.30
```
上面定义了一个比较简单的 Pod 的配置，里面有两个容器：

- nginx: 用1.17.1版本的 nginx 镜像创建（一个轻量级的Web服务器）
- busybox：用1.30版本的 busybox 镜像创建 （busybox是一个小巧的Linux命令集合）



但是，创建之后出现了如下的状况：STATUS 显示为 CrashLoopBackOff（在 2.3 启动命令 中会解释这个问题）


```shell
[root@k8s-master k8s-yaml]# kubectl create -f pod-base.yaml
pod/pod-base created

[root@k8s-master k8s-yaml]# kubectl get pods -n dev
NAME       READY   STATUS             RESTARTS     AGE
pod-base   1/2     CrashLoopBackOff   1 (3s ago)   22s

```

### 2.2 镜像拉取

创建 pod-imagePullPolicy.yaml文件，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-imagepullpolicy
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.2
    imagePullPolicy: Never # 用于设置镜像拉取策略
  - name: busybox
    image: busybox:1.30
```

由于拉取策略设置为Never，所以会出现如下的状况（一直无法拉取）

```shell
[root@k8s-master k8s-yaml]# kubectl create -f pod-imagePullPolicy.yaml
pod/pod-imagepullpolicy created

[root@k8s-master k8s-yaml]# kubectl get pods -n dev
NAME                  READY   STATUS             RESTARTS     AGE
pod-imagepullpolicy   0/2     CrashLoopBackOff   1 (4s ago)   5s
```

**imagePullPolicy**，用于设置镜像拉取策略，Kubernetes支持配置三种拉取策略：

- **Always**：总是从远程仓库拉取镜像（一直远程下载）
- **IfNotPresent**：本地有则使用本地镜像，本地没有则从远程仓库拉取镜像（本地有就本地 本地没远程下载）
- **Never：**只使用本地镜像，从不去远程仓库拉取，本地没有就报错 （一直使用本地）

> 默认值说明：
>
> 如果镜像tag为具体版本号，默认策略是：IfNotPresent
>
> 如果镜像tag为latest（最新版本），默认策略是：Always

```shell
# 创建Pod
[root@k8s-master pod]# kubectl create -f pod-imagepullpolicy.yaml
pod/pod-imagepullpolicy created

# 查看Pod详情
# 此时明显可以看到nginx镜像有一步Pulling image "nginx:1.17.2"的过程
[root@k8s-master pod]# kubectl describe pod pod-imagepullpolicy -n dev
......
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  <unknown>         default-scheduler  Successfully assigned dev/pod-imagePullPolicy to node1
  Normal   Pulling    32s               kubelet, node1     Pulling image "nginx:1.17.2"
  Normal   Pulled     26s               kubelet, node1     Successfully pulled image "nginx:1.17.2"
  Normal   Created    26s               kubelet, node1     Created container nginx
  Normal   Started    25s               kubelet, node1     Started container nginx
  Normal   Pulled     7s (x3 over 25s)  kubelet, node1     Container image "busybox:1.30" already present on machine
  Normal   Created    7s (x3 over 25s)  kubelet, node1     Created container busybox
  Normal   Started    7s (x3 over 25s)  kubelet, node1     Started container busybox
```

### 2.3 启动命令

在前面的案例中，一直有一个问题没有解决，就是busybox容器一直没有成功运行，是因为什么呢？

解释：busybox并不是一个程序，而是类似于一个工具类的集合，Kubernetes 集群启动管理后，它会自动关闭。

而解决方法就是让其一直在运行，这就用到了command配置。

创建 pod-command.yaml 文件，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-command
  namespace: dev
spec: 
  containers:
  - name: nginx
    image: nginx:1.17.1
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","touch /tmp/hello.txt;while true;do /bin/echo $(date +%T) >> /tmp/hello.txt;sleep 3;done;"]
```

这样子就能正常跑起来了

```shell
[root@k8s-master k8s-yaml]# kubectl get pod -n dev
NAME          READY   STATUS    RESTARTS   AGE
pod-command   2/2     Running   0          8s
```

> 原理：command，用于在Pod中的容器初始化完成之后运行一个命令。
>
> 然后解释下上面的命令含义：
>
> "/bin/sh"，"-c"，使用sh执行命令
>
> touch /tmp/hello.txt; 创建一个/tmp/hello.txt 文件
>
> while true;do /bin/echo $(date +%T) >> /tmp/hello.txt; sleep 3; done; 每隔3秒向文件中写入当前时间

```shell
# 进入Pod中的busybox容器，查看文件内容
# 补充一个命令：kubectl exec pod名称 -n 命名空间 -it -c 容器名称 /bin/sh 在容器内部执行命令
# 使用这个命令就可以进入某个容器的内部，然后进行相关操作了
# 比如，可以查看txt文件的内容
[root@k8s-master k8s-yaml]# kubectl exec pod-command -n dev -it -c busybox /bin/sh

/ #
/ # tail -f /tmp/hello.txt
08:51:50
08:51:53
08:51:56
08:51:59
```

**特别说明：**

通过上面发现command已经可以完成启动命令和传递参数的功能，为什么这里还要提供一个 args 选项，用于传递参数呢?这其实跟 docker 有点关系，kubernetes 中的 command、args 两项其实是实现覆盖 Dockerfile 中 ENTRYPOINT 的功能。

1. 如果 command 和 args 均没有写，那么用 Dockerfile 的配置。
2. 如果 command 写了，但 args 没有写，那么 Dockerfile 默认的配置会被忽略，执行输入的 command。
3. 如果 command 没写，但 args 写了，那么 Dockerfile 中配置的 ENTRYPOINT 的命令会被执行，使用当前 args 的参数。
4. 如果 command 和 args 都写了，那么 Dockerfile 的配置被忽略，执行 command 并追加上 args 参数。

### 2.4 环境变量

创建pod-env.yaml文件，内容如下：

```shell
apiVersion: v1
kind: Pod
metadata:
  name: pod-env
  namespace: dev
spec:
  containers:
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","while true;do /bin/echo $(date +%T);sleep 60;done;"]
    env:
    - name: "username"
      value: "admin"
    - name: "password"
      value: "123456"

```

env，环境变量，用于在Pod中的容器设置环境变量。

```shell
[root@k8s-master k8s-yaml]# kubectl create -f pod-env.yaml
pod/pod-env created

[root@k8s-master k8s-yaml]# kubectl get pod -n dev
NAME      READY   STATUS    RESTARTS   AGE
pod-env   1/1     Running   0          3s


[root@k8s-master k8s-yaml]# kubectl exec pod-env -n dev -c busybox -it /bin/sh
/ # echo $username,$password
admin,123456

```

备注：现在进入容器推荐使用`kubectl exec -it <Pod名称> -n <Namespace> -- sh `

这种方式不是很推荐，推荐将这些配置单独存储在配置文件中，这种方式将在后面介绍。

### 2.5 端口设置

本小节来介绍容器的端口设置，也就是containers的ports选项。

首先来看下ports支持的子选项：

```shell
[root@k8s-master ~]# kubectl explain pod.spec.containers.ports
KIND:     Pod
VERSION:  v1
RESOURCE: ports <[]Object>
FIELDS:
   name         <string>  # 端口名称，如果指定，必须保证name在pod中是唯一的		
   containerPort<integer> # 容器要监听的端口(0<x<65536)
   hostPort     <integer> # 容器要在主机上公开的端口，如果设置，主机上只能运行容器的一个副本(一般省略) 
   hostIP       <string>  # 要将外部端口绑定到的主机IP(一般省略)
   protocol     <string>  # 端口协议。必须是UDP、TCP或SCTP。默认为“TCP”。
```

接下来，编写一个测试案例，创建pod-ports.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-ports
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports: #设置容器暴露的端口列表
    - name: nginx-port
      containerPort: 80
      protocol: TCP
```

```shell
# 创建Pod
[root@k8s-master k8s-yaml]# kubectl create -f pod-ports.yaml
pod/pod-ports created

[root@k8s-master k8s-yaml]# kubectl get pod -n dev
NAME        READY   STATUS    RESTARTS   AGE
pod-env     1/1     Running   0          11m
pod-ports   1/1     Running   0          6s

# 查看Pod
# 在下面可以明显看到配置信息
[root@k8s-master k8s-yaml]# kubectl get pod pod-ports -n dev -o yaml
......
spec:
  containers:
  - image: nginx:1.17.1
    imagePullPolicy: IfNotPresent
    name: nginx
    ports:
    - containerPort: 80
      name: nginx-port
      protocol: TCP
......
```

访问容器中的程序需要使用的是`PodIP：ContainerPort`

### 2.6 资源配额

容器中的程序要运行，肯定需要占用一定的资源，比如CPU和内存，如果不对某个容器的资源做限制，那么它就可能吃掉大量的资源，导致其它容器无法运行。针对这种情况，Kubernetes提供了对内存和CPU的资源进行配额的机制，这种机制主要通过resources选项实现，它有两个子选项：

- **limits：**用于限制运行时容器的最大占用资源，当容器占用资源超过limits时会被终止，并进行重启。
- **requests：**用于设置容器需要的最小资源，如果环境资源不够，容器将无法启动。

可以通过上面两个选项设置资源的上下限

接下来，编写一个测试案例，创建pod-resources.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-resources
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    resources:            #资源配额
      limits:             #限制资源（上限）
        cpu: "2"          #CPU限制，单位是Core数
        memory: "10Gi"    #内存限制
      requests:			  #请求资源（下限）
        cpu: "1"		  #CPU限制，单位是Core数
        memory: "10Mi"	  #内存限制
```

在这里对CPU和Memory的单位做一个说明：

- cpu：core数，可以为整数或小数
- memory： 内存大小，可以使用Gi、Mi、G、M等形式

```shell
# 运行Pod
[root@k8s-master k8s-yaml]# kubectl create -f pod-resources.yaml
pod/pod-resources created

# 查看发现Pod运行正常
[root@k8s-master k8s-yaml]# kubectl get pod pod-resources -n dev
NAME            READY   STATUS    RESTARTS   AGE
pod-resources   1/1     Running   0          4s

# 接下来，停止Pod
[root@k8s-master k8s-yaml]# kubectl delete -f pod-resources.yaml

# 编辑Pod，修改resources.requests.memory的值为10Gi
[root@k8s-master k8s-yaml]# vim pod-resources.yaml

# 再次启动Pod
[root@k8s-master k8s-yaml]# kubectl create -f pod-resources.yaml

# 查看Pod状态，发现Pod启动失败
[root@k8s-master k8s-yaml]# kubectl get pod pod-resources -n dev
NAME            READY   STATUS    RESTARTS   AGE
pod-resources   0/1     Pending   0          3s

# 查看Pod详情会发现，如下提示
[root@k8s-master k8s-yaml]# kubectl describe pod pod-resources -n dev
......
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  70s   default-scheduler  0/3 nodes are available: 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate, 2 Insufficient memory（内存不足）.
```



## 三、Pod生命周期

我们一般将Pod对象从创建至终的这段时间范围称为Pod的生命周期，它主要包含下面的过程：

- Pod创建过程
- 运行初始化容器（init container）过程
- 运行主容器（main container）
  - 容器启动后钩子（post start）、容器终止前钩子（pre stop）
  - 容器的存活性探测（liveness probe）、就绪性探测（readiness probe）
- Pod终止过程

![image-20220311153356405](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220311153356405.png)

在整个生命周期中，Pod会出现5种**状态（相位）**，分别如下：

- 挂起（Pending）：apiServer 已经创建了 Pod 资源对象，但它尚未被调度完成或者仍处于下载镜像的过程。

- 运行（Running）：Pod 已经被调度至某节点，并且所有容器都已经被 Kubelet 创建完成。

- 成功（Succeeded）：Pod 中的所有容器都已经成功终止并且不会被重启。
- 失败（Failed）：所有容器都已经终止，但至少有一个容器终止失败，即容器返回了非0值的退出状态。

- 未知（Unknown）：apiServer 无法正常获取到 Pod 对象的状态信息，通常由网路通信失败所导致。

### 3.1 创建和终止

**Pod的创建过程：**

1. 用户通过 kubectl 或其它 api客户端提交需要创建的 Pod 信息给 apiServer；
2. apiServer 开始生成 Pod 对象的信息，并将信息存入 Etcd，然后返回确认信息至客户端；
3. apiServer 开始反应 Etcd 中的 Pod 对象的变化，其它组件使用watch机制来跟踪检查 apiServer 上的变动；
4. Scheduler 发现有新的 Pod 对象要创建，开始为 Pod 分配主机并将结果信息更新至 apiServer；
5. Node 节点上的 kubectl 发现有 Pod 调度过来，尝试调用 docker 启动容器，并将结果回送至apiServer；
6. apiServer 将接收到的 Pod 状态信息存入 Etcd 中。

![image-20220311154952897](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220311154952897.png)



**Pod的终止过程：**

1. 用户向 apiServer 发送删除 Pod 对象的命令；
2. apiServer 中的 Pod 对象信息会随着时间的推移而更新，在宽限期内（默认30S），Pod 被视为 dead状态；
3. 将 Pod 标记为 terminating 状态；
4. kubelet 在监控到 Pod 对象转为 terminating 状态的同时启动 Pod 关闭过程；
5. 端点控制器监控到 Pod 对象的关闭行为时将其从所有匹配到此端点的 Service 资源的端点列表中移除；
6. 如果当前 Pod 对象定义了 Pre Stop 钩子处理器，则在其标记为 terminating 后即会以同步的方式启动执行；
7. Pod 对象中的容器进程收到停止信号；
8. 宽限期结束后，若 Pod 中还存在仍在运行的进程，那么 Pod 对象会收到立即终止的信号；
9. kubelet 请求 apiServer 将此 Pod 资源的宽限期设置为0，从而完成删除操作，此时 Pod 对于用户已不可见。

### 3.2 初始化容器

初始化容器是在 Pod 的主容器启动之前要运行的容器，主要是做一些主容器的前置工作，它具备两大特征：

1. 初始化容器必须运行完成直至结束，若某初始化容器运行失败，那么 Kubernetes 需要重启它直到成功完成。
2. 初始化容器必须按照定义的顺序执行，当且仅当一个成功之后，后面的一个才能运行。

初始化容器有很多的应用场景，下面列出的是最常见的几个：

- 提供主容器镜像中不具备的工具程序或自定义代码
- 初始化容器要先于应用容器串行启动并运行完成，因此可用于延后应用容器的启动直至其依赖的条件得到满足

接下来做一个案例，模拟下面这个需求：

假设要以主容器来运行 Nginx，但是要求在运行 Nginx 之前先要能够连接上 Mysql 和 Redis 所在服务器，为了简化测试，事先规定好 Mysql（192.168.5.14）和 Redis（192.168.5.15）服务器的地址。

创建 pod-initcontainer.yaml，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-initcontainer
  namespace: dev
spec:
  containers:
  - name: main-container
    image: nginx:1.17.1
    ports:
    - name: nginx-port
      containerPort: 80
  initContainers:
  - name: test-mysql
    image: busybox:1.30
    command: ["sh","-c","until ping 192.168.5.14 -c 1 ; do echo waiting for mysql...; sleep 2; done;"]
  - name: test-redis
    image: busybox:1.30
    command: ["sh","-c","until ping 192.168.5.15 -c 1 ; do echo waiting for redis...; sleep 2; done;"]
```

```shell
# 创建Pod
[root@k8s-master k8s-yaml]# kubectl create -f pod-initcontainer.yaml
pod/pod-initcontainer created

# 查看Pod状态
# 发现Pod卡在启动第一个初始化容器过程中，后面的容器不会运行
[root@k8s-master k8s-yaml]# kubectl describe pod pod-initcontainer -n dev

......
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  6m13s  default-scheduler  Successfully assigned dev/pod-initcontainer to k8s-node01
  Normal  Pulled     6m13s  kubelet            Container image "busybox:1.30" already present on machine
  Normal  Created    6m13s  kubelet            Created container test-mysql
  Normal  Started    6m13s  kubelet            Started container test-mysql


# 动态查看Pod
[root@k8s-master ~]# kubectl get pods pod-initcontainer -n dev -w
NAME                             READY   STATUS     RESTARTS   AGE
pod-initcontainer                0/1     Init:0/2   0          15s
pod-initcontainer                0/1     Init:1/2   0          52s
pod-initcontainer                0/1     Init:1/2   0          53s
pod-initcontainer                0/1     PodInitializing   0          89s
pod-initcontainer                1/1     Running           0          90s

# 接下来新开一个Shell，为当前服务器新增两个IP，观察Pod的变化
[root@k8s-master ~]# ifconfig ens33:1 192.168.5.14 netmask 255.255.255.0 up
[root@k8s-master ~]# ifconfig ens33:2 192.168.5.15 netmask 255.255.255.0 up
```

### 3.3 钩子函数

钩子函数能够感知自身生命周期中的事件，并在响应的时刻到来时运行用户指定的程序代码。

Kubernetes 在主容器启动之后和停止之前，提供了两个钩子函数：

- postStart：在容器创建之后执行，如果失败了会重启容器。
- preStop：容器终止之前执行，执行完成之后容器将成功终止，在其完成之前会阻塞删除容器的操作。

钩子处理器支持使用下面三种方式定义动作：

- exec命令：在容器内执行一次命令

```yaml
……
  lifecycle:
    postStart: 
      exec:
        command:
        - cat
        - /tmp/healthy
……
```

- tcpSocket：在当前容器尝试访指定的socket

```yaml
……      
  lifecycle:
    postStart:
      tcpSocket:
        port: 8080
……
```

- httpGet：在当前容器中向某 url 发起 http 请求

```yaml
……
  lifecycle:
    postStart:
      httpGet:
        path: / #URI地址
        port: 80 #端口号
        host: 192.168.5.3 #主机地址
        scheme: HTTP #支持的协议，http或者https
……
```

接下来，以 exec 方式为例，演示下钩子函数的使用，创建 pod-hook-exec.yaml 文件，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: pod-hook-exec
  namespace: dev
spec:
  containers:
  - name: main-container
    image: nginx:1.17.1
    ports:
    - name: nginx-port
      containerPort: 80
    lifecycle:
      postStart:
        exec:  # 在容器启动的时候执行一个命令，修改掉nginx的默认首页内容
          command: ["/bin/sh","-c","echo postStart... > /usr/share/nginx/html/index.html"]
      preStop:
        exec: # 在容器停止之前停止nginx服务
          command: ["/usr/sbin/nginx","-s","quit"]
```

```shell
# 创建Pod
[root@k8s-master k8s-yaml]# kubectl create -f pod-hook-exec.yaml
pod/pod-hook-exec created

# 查看Pod
[root@k8s-master k8s-yaml]# kubectl get pods pod-hook-exec -n dev -o wide
NAME            READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
pod-hook-exec   1/1     Running   0          14s   10.244.1.27   k8s-node01   <none>           <none>

# 访问Pod
[root@k8s-master k8s-yaml]# curl 10.244.1.27
postStart...
```

### 3.4 容器探测

容器探测用于检测容器中的应用实例是否正常工作，是保障业务可用性的一种传统机制。如果经过探测，实例的状态不符合预期，那么 Kubernetes 就会把该问题实例 ”摘除“，不承担业务流量。

Kubernetes 提供了两种探针来实现容器探测，分别是：

- livenessProbe：存活性探针，用于检测应用实例当前是否处于正常运行状态，如果不是，K8S会重启容器。
- readinessProbe：就绪性探针，用于检测应用实例当前是否可以接收请求，如果不能，K8S不会转发流量。

> livenessProbe，决定是否重启容器
>
> readinessProbe，决定是否将请求转发给容器

上面两种探针目前均支持三种探测方式：

- exec命令：在容器内执行一次命令，如果命令执行的退出码为0，则认为程序正常，否则不正常。

  ```yaml
  ……
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
  ……
  ```

- tcpSocker：将会尝试访问一个用户容器的端口，如果能够建立这条连接，则认为程序正常，否则不正常。

  ```yaml
  ……      
    livenessProbe:
      tcpSocket:
        port: 8080
  ……
  ```

- httpGet：调用容器内Web应用的URL，如果返回

  ```yaml
  ……
    livenessProbe:
      httpGet:
        path: / 			#URI地址
        port: 80 			#端口号
        host: 127.0.0.1	#主机地址
        scheme: HTTP 		#支持的协议，http或者https
  ……
  ```

下面以livenessProbe为例，做几个演示：

**方式一：exec**

创建 pod-liveness-exec.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-exec
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports: 
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      exec:
        command: ["/bin/cat","/tmp/hello.txt"] # 执行一个查看文件的命令
```

创建 Pod，观察效果

```shell
# 创建Pod
[root@k8s-master k8s-yaml]# kubectl create -f pod-liveness-exec.yaml

# 查看Pod详情
[root@k8s-master k8s-yaml]# kubectl describe pods pod-liveness-exec -n dev
......
Events:
  Type     Reason     Age   From               Message
  ----     ------     ----  ----               -------
  Normal   Scheduled  19s   default-scheduler  Successfully assigned dev/pod-liveness-exec to k8s-node01
  Normal   Pulled     19s   kubelet            Container image "nginx:1.17.1" already present on machine
  Normal   Created    19s   kubelet            Created container nginx
  Normal   Started    19s   kubelet            Started container nginx
  Warning  Unhealthy  10s   kubelet            Liveness probe failed: /bin/cat: /tmp/hello.txt: No such file or directory

# 观察上面的信息就会发现nginx容器启动之后就进行了健康检查
# 检查失败之后，容器被kill掉，然后尝试进行重启（这是重启策略的作用，后面讲解）
# 稍等一会之后，再观察pod信息，就可以看到RESTARTS不再是0，而是一直在增长
[root@k8s-master k8s-yaml]# kubectl get pods pod-liveness-exec -n dev
NAME                READY   STATUS             RESTARTS       AGE
pod-liveness-exec   0/1     CrashLoopBackOff   6 (104s ago)   6m4s

# 当然接下来，可以修改成一个存在的文件，比如/tmp/hello.txt，再试，结果就正常了......
```

**方式二：tcpSocket**

创建 pod-liveness-tcpsocket.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-tcpsocket
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports: 
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      tcpSocket:
        port: 8080 			# 尝试访问8080端口
```

创建 pod，观察效果

```shell
# 创建Pod
[root@k8s-master k8s-yaml]# kubectl create -f pod-liveness-tcpsocket.yaml
pod/pod-liveness-tcpsocket created

# 查看Pod详情
[root@k8s-master k8s-yaml]# kubectl create -f pod-liveness-tcpsocket.yaml
......
Events:
  Type     Reason     Age   From               Message
  ----     ------     ----  ----               -------
  Normal   Scheduled  15s   default-scheduler  Successfully assigned dev/pod-liveness-tcpsocket to k8s-node01
  Normal   Pulled     14s   kubelet            Container image "nginx:1.17.1" already present on machine
  Normal   Created    14s   kubelet            Created container nginx
  Normal   Started    14s   kubelet            Started container nginx
  Warning  Unhealthy  5s    kubelet            Liveness probe failed: dial tcp 10.244.1.29:8080: connect: connection refused


# 观察上面的信息，发现尝试访问8080端口，但是失败了
# 稍等一会之后，再观察Pod信息，就可以看到RESTARTS不再是0，而是一直增长
[root@k8s-master k8s-yaml]# kubectl get pods pod-liveness-tcpsocket -n dev
NAME                     READY   STATUS    RESTARTS      AGE
pod-liveness-tcpsocket   1/1     Running   2 (13s ago)   73s

# 当然接下来，可以修改成一个可访问的端口，比如80，再试，结果就正常了...
```

**方式三：httpGet**

创建 pod-liveness-httpget.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-httpget
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      httpGet:  		# 其实就是访问http://127.0.0.1:80/hello  
        scheme: HTTP 	# 支持的协议，http或者https
        port: 80 		# 端口号
        path: /hello	# URI地址
```

创建 Pod，观察效果

```shell
# 创建Pod
[root@k8s-master ~]# kubectl create -f pod-liveness-httpget.yaml
pod/pod-liveness-httpget created

# 查看Pod详情
[root@k8s-master k8s-yaml]# kubectl describe pod pod-liveness-httpget -n dev
......
Events:
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  13m                   default-scheduler  Successfully assigned dev/pod-liveness-httpget to k8s-node01
  Normal   Created    12m (x4 over 13m)     kubelet            Created container nginx
  Normal   Started    12m (x4 over 13m)     kubelet            Started container nginx
  Normal   Killing    12m (x3 over 13m)     kubelet            Container nginx failed liveness probe, will be restarted
  Warning  Unhealthy  11m (x10 over 13m)    kubelet            Liveness probe failed: HTTP probe failed with statuscode: 404
  Normal   Pulled     8m34s (x7 over 13m)   kubelet            Container image "nginx:1.17.1" already present on machine
  Warning  BackOff    3m34s (x30 over 11m)  kubelet            Back-off restarting failed container

# 观察上面信息，尝试访问路径，但是未找到，出现404错误
# 稍等一会之后，再观察Pod信息，就可以看到RESTARTS不再是0，而是一直增长
[root@k8s-master k8s-yaml]# kubectl get pod pod-liveness-httpget -n dev
NAME                   READY   STATUS    RESTARTS     AGE
pod-liveness-httpget   1/1     Running   1 (4s ago)   34s


# 当然接下来，可以修改成一个可以访问的路径path，比如/，再试，结果就正常了......
```

至此，已经使用livenessProbe演示了三种探测方式，但是查看livenessProbe的子属性，会发现除了这三种方式，还有一些其他的配置，在这里一并解释下：

```shell
[root@k8s-master ~]# kubectl explain pod.spec.containers.livenessProbe
FIELDS:
   exec <Object>  
   tcpSocket    <Object>
   httpGet      <Object>
   initialDelaySeconds  <integer>  # 容器启动后等待多少秒执行第一次探测
   timeoutSeconds       <integer>  # 探测超时时间。默认1秒，最小1秒
   periodSeconds        <integer>  # 执行探测的频率。默认是10秒，最小1秒
   failureThreshold     <integer>  # 连续探测失败多少次才被认定为失败。默认是3。最小值是1
   successThreshold     <integer>  # 连续探测成功多少次才被认定为成功。默认是1
```

下面稍微配置两个，演示下效果即可：

```yaml
[root@k8s-master ~]# more pod-liveness-httpget.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-httpget
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      httpGet:
        scheme: HTTP
        port: 80 
        path: /
      initialDelaySeconds: 30 	# 容器启动后30s开始探测
      timeoutSeconds: 5 		# 探测超时时间为5s
```

### 3.5 重启策略

在上一节中，一旦容器探测出现了问题，Kubernetes 就会对容器所在的 Pod 进行重启，其实这是由 Pod 的重启策略决定的，Pod 的重启策略有 3 种，分别如下：

- Always：容器失效时，自动重启该容器，这也是默认值。
- OnFailure：容器终止运行且退出码不为0时重启。
- Never：不论状态为何，都不重启该容器。

重启策略适用于 Pod 对象种的所有容器，首次需要重启的容器，将在其需要时立即进行重启，随后再次现需要重启的操作将由 kubelet 延迟一段时间后进行，且反复的重启操作的延迟市场以此为10s、20s、40s、80s、160s和300s，300s是最大延迟时长。

创建 pod-restartpolicy.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-restartpolicy
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      httpGet:
        scheme: HTTP
        port: 80
        path: /hello
  restartPolicy: Never # 设置重启策略为Never
```

运行 Pod 测试

```shell
# 创建Pod
[root@k8s-master ~]# kubectl create -f pod-restartpolicy.yaml
pod/pod-restartpolicy created

# 查看Pod详情，发现nginx容器启动失败
[root@k8s-master ~]# kubectl  describe pods pod-restartpolicy  -n dev
......
Events:
  Type     Reason       Age               From               Message
  ----     ------       ----              ----               -------
  Normal   Scheduled    34s               default-scheduler  Successfully assigned dev/pod-restartpolicy to k8s-node01
  Normal   Pulled       34s               kubelet            Container image "nginx:1.17.1" already present on machine
  Normal   Created      34s               kubelet            Created container nginx
  Normal   Started      34s               kubelet            Started container nginx
  Warning  Unhealthy    5s (x3 over 25s)  kubelet            Liveness probe failed: HTTP probe failed with statuscode: 404
  Normal   Killing      5s                kubelet            Stopping container nginx
  Warning  FailedMount  0s (x4 over 4s)   kubelet            MountVolume.SetUp failed for volume "kube-api-access-x45zc" : object "dev"/"kube-root-ca.crt" not registered

# 多等一会，再观察pod的重启次数，发现一直是0，并未重启，并且是完成态
[root@k8s-master k8s-yaml]# kubectl get pods pod-restartpolicy -n dev
NAME                READY   STATUS      RESTARTS   AGE
pod-restartpolicy   0/1     Completed   0          35s
```



## 四、Pod调度

在默认情况下，一个 Pod 在哪个 Node 节点上运行，是由 Scheduler 组件采用相应的算法计算出来的，这个过程是不受人工控制的。但是在实际使用中，这并不满足需求，因为很多情况下，我们想控制某些 Pod 到达某些节点上，那么应该怎么做呢？这就要求了解 Kubernetes 对 Pod 的调度规则， Kubernetes 提供了四大类调度方式：

- 自动调度：运行在哪个节点上完全由 Scheduler 经过一系列的算法计算得出
- 定向调度：NodeName、NodeSelector
- 亲和性调度：NodeAffinity、PodAffinity、PodAntiAffinity
- 污点（容忍）调度：Taints、Toleration

### 4.1 定向调度

定向调度，指的是利用在 Pod 上声明 nodeName 或者 nodeSelector，以此将 Pod 调度到期望的 node 节点上。注意，这里的调度是强制的，这就意味着即使要调度的目标 Node 不存在，也会向上面进行调度，只不过 Pod 运行失败而已。

**NodeName**

NodeName 用于强制约束将 Pod 调度指定的 Name 的 Node 节点上。这种方式，其实是直接跳过 Scheduler 的调度逻辑，直接将 Pod 调度到指定名称的节点。

接下来，实验一下：创建一个 pod-nodename.yaml 文件

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodename
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  nodeName: node1 # 指定调度到node1节点上
```

```shell
# 创建Pod
[root@k8s-master k8s-yaml]# kubectl apply -f pod-nodename.yaml
pod/pod-nodename created

# 查看Pod调度到Node属性，确实是调度到了node1节点上
[root@k8s-master k8s-yaml]# kubectl get pods -n dev -o wide
NAME           READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
pod-nodename   1/1     Running   0          6s    10.244.1.35   k8s-node01   <none>           <none>

# 接下来，删除Pod，修改nodeName的值为node3（并没有node3节点）
# 再次查看，发现已经向Node3节点调度，但是由于不存在node3节点，所以pod无法正常运行
[root@k8s-master k8s-yaml]# kubectl get pods -n dev -o wide
NAME           READY   STATUS    RESTARTS   AGE   IP       NODE         NOMINATED NODE   READINESS GATES
pod-nodename   0/1     Pending   0          5s    <none>   k8s-node03   <none>           <none>
```

**NodeSelector**

NodeSelector 用于将 Pod 调度到添加了指定标签的 node 节点上。它是通过 Kubernetes 的 Label-Selector 机制实现的，也就是说，在 Pod 创建之前，会由 Scheduler 使用 MatchNodeSelector 调度策略进行 Label 匹配，找出目标 Node，然后将 Pod 调度到目标节点，该匹配规则是强制约束。

接下来，实验一下：

1. 首先分别为 k8s-node01 节点添加标签

```shell
[root@k8s-master k8s-yaml]# kubectl label nodes k8s-node01 nodeenv=pro
node/k8s-node01 labeled
[root@k8s-master k8s-yaml]# kubectl label nodes k8s-node02 nodeenv=test
node/k8s-node02 labeled

[root@k8s-master k8s-yaml]# kubectl get node --show-labels
NAME         STATUS   ROLES                  AGE   VERSION   LABELS
k8s-master   Ready    control-plane,master   11d   v1.23.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node.kubernetes.io/exclude-from-external-load-balancers=
k8s-node01   Ready    <none>                 11d   v1.23.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node01,kubernetes.io/os=linux,nodeenv=pro
k8s-node02   Ready    <none>                 11d   v1.23.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node02,kubernetes.io/os=linux,nodeenv=test
```

2. 创建一个 pod-nodeselector.yaml 文件，并使用它创建 Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeselector
  namespace: dev
spec: 
  containers:
  - name: nginx
    image: nginx:1.17.1
  nodeSelector:
    nodeenv: pro # 指定调度到具有nodeenv=pro 标签的节点上
```

```shell
# 创建Pod
[root@k8s-master k8s-yaml]# kubectl apply -f pod-nodeselector.yaml
pod/pod-nodeselector created

# 查看Pod调度到Node属性，确实是调度到了node1节点上
[root@k8s-master k8s-yaml]# kubectl get pods pod-nodeselector -n dev -o wide
NAME               READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
pod-nodeselector   1/1     Running   0          3s    10.244.1.36   k8s-node01   <none>           <none>

# 接下来，删除Pod，修改nodeSelector的值为nodeenv: xxxx （不存在打有此标签的节点）
# 再次查看，发现Pod无法正常运行
[root@k8s-master k8s-yaml]# kubectl get pods pod-nodeselector -n dev -o wide
NAME               READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
pod-nodeselector   0/1     Pending   0          4s    <none>   <none>   <none>           <none>

# 查看详情，发现node selector匹配失败的提示
[root@k8s-master k8s-yaml]# kubectl describe pod pod-nodeselector -n dev
......
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  52s   default-scheduler  0/3 nodes are available: 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate, 2 node(s) didn't match Pod's node affinity/selector.
```

### 4.2 亲和性调度

上一节，介绍了两种定向调度的方式，使用起来非常方便，但是也有一定的问题，那就是如果没有满足条件的 Node，那么 Pod 将不会被运行，即使在集群中还有可用 Node 列表也不行，这就限制了它的使用场景。

基于上面的问题，Kubernetes还提供了一种亲和性调度（Affinity）。它在 NodeSelector 的基础之上进行了扩展，可以通过配置的形式，实现优先选择满足条件的 Node 进行调度，如果没有，也可以调度到不满足条件的节点上，使调度更加灵活。

Affinity主要分为三类：

- nodeAffinity（Node亲和性）：以 Node 为目标，解决 Pod 可以调度到哪些 node 的问题。
- podAffinity（Pod亲和性）：以 Pod 为目标，解决 Pod 可以和哪些已存在的 Pod 部署在同一个拓扑域中的问题。
- podAntiAffinity（Pod反亲和性）：以 Pod 为目标，解决 Pod 不能和哪些已存在 Pod 部署在同一个拓扑域中的问题。

> 关于亲和性（反亲和性）使用场景的说明：
>
> **亲和性：**如果两个应用频繁交互，那就有必要利用亲和性让两个应用尽可能的靠近，这样就可以减少因网络通信而带来的性能损耗。
>
> **反亲和性：**当应用采用多副本部署时，有必要次啊用反亲和性让各个应用实例打散分布在各个 Node 上，这样可以提高服务的高可用性。

**NodeAffinity**

首先来看一下`NodeAffinity`的可配置项：

```shell
pod.spec.affinity.nodeAffinity
  requiredDuringSchedulingIgnoredDuringExecution  Node节点必须满足指定的所有规则才可以，相当于硬限制
    nodeSelectorTerms  节点选择列表
      matchFields   按节点字段列出的节点选择器要求列表
      matchExpressions   按节点标签列出的节点选择器要求列表(推荐)
        key    键
        values 值
        operator 关系符 支持Exists, DoesNotExist, In, NotIn, Gt, Lt
  preferredDuringSchedulingIgnoredDuringExecution 优先调度到满足指定的规则的Node，相当于软限制 (倾向)
    preference   一个节点选择器项，与相应的权重相关联
      matchFields   按节点字段列出的节点选择器要求列表
      matchExpressions   按节点标签列出的节点选择器要求列表(推荐)
        key    键
        values 值
        operator 关系符 支持In, NotIn, Exists, DoesNotExist, Gt, Lt
	weight 倾向权重，在范围1-100。
```

```yaml
## 关系符的使用说明:
- matchExpressions:
  - key: nodeenv              # 匹配存在标签的key为nodeenv的节点
    operator: Exists
  - key: nodeenv              # 匹配标签的key为nodeenv,且value是"xxx"或"yyy"的节点
    operator: In
    values: ["xxx","yyy"]
  - key: nodeenv              # 匹配标签的key为nodeenv,且value大于"xxx"的节点
    operator: Gt
    values: "xxx"
```

接下来首先演示一下`requiredDuringSchedulingIgnoreDuringExecution`

创建 pod-nodeaffinity-required.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeaffinity-required
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  affinity:  # 亲和性设置
    nodeAffinity: # 设置node亲和性
      requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
        nodeSelectorTerms:
        - matchExpressions: # 匹配env的值在["xxx","yyy"]中的标签
          - key: nodeenv
            operator: In
            values: ["xxx","yyy"]
```

```shell
# 创建Pod
[root@k8s-master k8s-yaml]# kubectl create -f pod-nodeaffinity-required.yaml
pod/pod-nodeaffinity-required created

# 查看Pod状态（运行失败）
[root@k8s-master k8s-yaml]# kubectl get pods pod-nodeaffinity-required -n dev -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
pod-nodeaffinity-required   0/1     Pending   0          41s   <none>   <none>   <none>           <none>

# 查看Pod的详情
# 发现调度失败，提示node选择失败
[root@k8s-master k8s-yaml]# kubectl describe pod pod-nodeaffinity-required -n dev
......
Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  12s (x2 over 81s)  default-scheduler  0/3 nodes are available: 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate, 2 node(s) didn't match Pod's node affinity/selector.

# 接下来，停止Pod
[root@k8s-master k8s-yaml]# kubectl delete -f pod-nodeaffinity-required.yaml
pod "pod-nodeaffinity-required" deleted

# 修改文件，将values: ["xxx","yyy"] -------> ["pro","yyy"]
# 再次启动
[root@k8s-master k8s-yaml]# kubectl create -f pod-nodeaffinity-required.yaml
pod/pod-nodeaffinity-required created

# 此时查看，发现调度成功，已经将Pod调度到了node1上
[root@k8s-master k8s-yaml]# kubectl get pods pod-nodeaffinity-required -n dev -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
pod-nodeaffinity-required   1/1     Running   0          5s    10.244.1.37   k8s-node01   <none>           <none>
```

接下来再演示一下`preferredDuringSchedulingIgnoredDuringExecution`

创建 pod-nodeaffinity-preferred.yaml

```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: pod-nodeaffinity-preferred
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  affinity:  # 亲和性设置
    nodeAffinity:  # 设置node亲和性
      preferredDuringSchedulingIgnoredDuringExecution: # 软限制
      - weight: 1
        preference: 
          matchExpressions: # 匹配env的值再["xxx","yyy"]中的标签（当前环境没有）
          - key: nodeenv
            operator: In
            values: ["xxx","yyy"]
```

```shell
# 创建Pod
[root@k8s-master k8s-yaml]# kubectl create -f pod-nodeaffinity-preferred.yaml
pod/pod-nodeaffinity-preferred created

# 查看Pod状态（运行成功）
[root@k8s-master k8s-yaml]# kubectl get pod pod-nodeaffinity-preferred -n dev
NAME                         READY   STATUS    RESTARTS   AGE
pod-nodeaffinity-preferred   1/1     Running   0          14s
```

> NodeAffinity规则设置的注意事项：
>
> 1. 如果同时定义了 nodeSelector 和 nodeAffinity，那么必须两个条件都得到满足，Pod才能运行在指定的 Node 上；
> 2. 如果 nodeAffinity 指定了多个 nodeSelectorTerms，那么只需要其中一个能够匹配成功即可；
> 3. 如果一个 nodeSelectorTerms 中有多个 matchExpressions，则一个节点必须满足所有的才能匹配成功；
> 4. 如果一个 Pod 所在的 Node 在 Pod 运行期间其标签发生了改变，不再符合该Pod的节点亲和性需求，则系统将忽略此变化。

**PodAffinity**

PodAffinity 主要实现以运行的 Pod 为参照，实现让新创建的 Pod 跟参照 Pod 在一个区域的功能。

首先来看一下 `PodAffinity` 的可配置项：

```yaml
pod.spec.affinity.podAffinity
  requiredDuringSchedulingIgnoredDuringExecution  硬限制
    namespaces       指定参照pod的namespace
    topologyKey      指定调度作用域
    labelSelector    标签选择器
      matchExpressions  按节点标签列出的节点选择器要求列表(推荐)
        key    键
        values 值
        operator 关系符 支持In, NotIn, Exists, DoesNotExist.
      matchLabels    指多个matchExpressions映射的内容
  preferredDuringSchedulingIgnoredDuringExecution 软限制
    podAffinityTerm  选项
      namespaces      
      topologyKey
      labelSelector
        matchExpressions  
          key    键
          values 值
          operator
        matchLabels 
    weight 倾向权重，在范围1-100
```

> topologyKey用于指定调度时作用域，例如：
>
> ​	如果指定为 kubernetes.io/hostname，那就是以Node节点为区分范围
>
> ​	如果指定为 beta.kubernetes.io/os，则以Node节点的操作系统类型来区分

接下来，演示一下`requiredDuringSchedulingIgnoredDuringExecution`

1. 首先创建一个参照Pod，pod-podaffinity-target.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-podaffinity-target
  namespace: dev
  labels:
    podenv: pro # 设置标签
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  nodeName: k8s-node01 # 将目标pod名确定到node1上
```

```shell
# 启动目标Pod
[root@k8s-master k8s-yaml]# kubectl create -f pod-podaffinity-target.yaml
pod/pod-podaffinity-target created

# 查看Pod状况
[root@k8s-master k8s-yaml]# kubectl get pods pod-podaffinity-target -n dev
NAME                     READY   STATUS    RESTARTS   AGE
pod-podaffinity-target   1/1     Running   0          12s
```

2. 创建pod-podaffinity-required.yaml，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-podaffinity-required
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  affinity:  #亲和性设置
    podAffinity: #设置pod亲和性
      requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
      - labelSelector:
          matchExpressions: # 匹配env的值在["xxx","yyy"]中的标签
          - key: podenv
            operator: In
            values: ["xxx","yyy"]
        topologyKey: kubernetes.io/hostname
```

上面配置表达的意思是：新 Pod 必须要与拥有标签 `nodeenv=xxx` 或者 `nodeenv=yyy` 的 Pod 在同一 Node 上，显然现在没有这样的Pod，接下来，运行测试一下。

```shell
# 启动Pod
[root@k8s-master k8s-yaml]# kubectl create -f pod-podaffinity-required.yaml
pod/pod-podaffinity-required created

# 查看Pod状态，发现未运行
[root@k8s-master k8s-yaml]# kubectl get pods pod-podaffinity-required -n dev
NAME                       READY   STATUS    RESTARTS   AGE
pod-podaffinity-required   0/1     Pending   0          12s

# 查看详细信息
[root@k8s-master k8s-yaml]# kubectl describe pods pod-podaffinity-required -n dev | tail -n    8
......
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  42s   default-scheduler  0/3 nodes are available: 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate, 2 node(s) didn't match pod affinity rules.

# 接下来修改：values:["xxx","yyy"] ————> values:["pro","yyy"]
# 意思是：新Pod必须要与拥有标签 nodeenv=xxx 或者 nodeenv=yyy 的 Pod 在同一 Node 上
[root@k8s-master k8s-yaml]# vim pod-podaffinity-required.yaml

# 然后重新创建Pod，查看效果
[root@k8s-master k8s-yaml]# kubectl delete -f  pod-podaffinity-required.yaml
pod "pod-podaffinity-required" deleted
[root@k8s-master k8s-yaml]# kubectl create -f pod-podaffinity-required.yaml
pod/pod-podaffinity-required created


# 发现此时Pod运行正常
[root@k8s-master k8s-yaml]# kubectl get pods pod-podaffinity-required -n dev
NAME                       READY   STATUS    RESTARTS   AGE
pod-podaffinity-required   1/1     Running   0          5s
```

关于`PodAffinity`的`preferredDuringSchedulingIgnoredDuringExecution`，这里不再演示。

**PodAntiAffinity**

PodAntiAffinity 主要实现以运行的 Pod 为参照，让新创建的 Pod 跟参照 Pod 不在一个区域中的功能。

它的配置方式和选项跟 PodAffinity 是一样的，这里不再做详细解释，直接以案例说明。

1. 继续使用上个案例中的目标 Pod

```shell
[root@k8s-master k8s-yaml]# kubectl get pods -n dev -o wide --show-labels
NAME                       READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES   LABELS
pod-podaffinity-required   1/1     Running   0          10m   10.244.1.45   k8s-node01   <none>           <none>            <none>
pod-podaffinity-target     1/1     Running   0          16m   10.244.1.44   k8s-node01   <none>           <none>            podenv=pro
```

2. 创建 pod-podantiaffinity-required.yaml，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-podantiaffinity-required
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  affinity: # 亲和性设置
    podAntiAffinity: # 设置Pod亲和性
      requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
      - labelSelector:
          matchExpressions: # 匹配podenv的值在["pro"]中的标签
          - key: podenv
            operator: In
            values: ["pro"]
        topologyKey: kubernetes.io/hostname
```

上面配置表达的意思是：新 Pod 必须要与拥有标签 `nodeenv=pro` 的 Pod 不在同一 Node上，运行测试一下。

```shell
# 创建Pod
[root@k8s-master k8s-yaml]# kubectl create -f pod-podantiaffinity-required.yaml
pod/pod-podantiaffinity-required created


# 查看Pod
# 发现调度到了node2上
[root@k8s-master k8s-yaml]# kubectl get pods -n dev -o wide --show-labels
NAME                           READY   STATUS    RESTARTS   AGE     IP            NODE         NOMINATED NODE   READINESS GATES   LABELS
pod-podaffinity-required       1/1     Running   0          22m     10.244.1.45   k8s-node01   <none>           <none>            <none>
pod-podaffinity-target         1/1     Running   0          28m     10.244.1.44   k8s-node01   <none>           <none>            podenv=pro
pod-podantiaffinity-required   1/1     Running   0          3m24s   10.244.2.22   k8s-node02   <none>           <none>            <none>
```



### 4.3 污点和容忍

**污点（Taints）**

前面的调度方式都是站在 Pod 的角度上，通过在 Pod 上添加属性，来确定 Pod 是否要调度到指定的 Node 上，其实我们也可以站在 Node 的角度上，通过在 Node 上添加 **污点** 属性，来决定是否允许 Pod 调度过来。

Node 被设置上污点之后就和 Pod 之间存在了一种相斥的关系，进而拒绝 Pod 调度进来，甚至可以将已经存在的 Pod 驱逐出去。

污点的格式为：`key=value:effect`，key和value是污点的标签，effect描述污点的作用，支持如果三个选项：

- PreferNoSchedule：Kubernetes 将尽量避免把 Pod 调度到具有该污点的 Node 上，除非没有其他节点可调度。
- NoScheduler：Kubernetes 将不会把 Pod 调度到具有该污点的 Node上，但不会影响当前 Node 上已存在的 Pod。
- NoExecute：Kubernetes 将不会把 Pod 调度到具有该污点的 Node 上，同时也会将 Node 上已存在的 Pod 驱离。

![image-20220325164837579](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220325164837579.png)

使用kubectl设置和去除污点的命令示例如下：

```shell
# 设置污点
kubectl taint nodes node1 key=value:effect

# 去除污点
kubectl taint nodes node1 key:effect-

# 去除所有污点
kubectl taint nodes node1 key-
```

接下来，演示下污点的效果：

1. 准备节点node1（为了演示效果更加明显，暂时停止node2节点）
2. 为node1节点设置一个污点: `tag=heima:PreferNoSchedule`；然后创建pod1( pod1 可以 )
3. 修改为node1节点设置一个污点: `tag=heima:NoSchedule`；然后创建pod2( pod1 正常 pod2 失败 )
4. 修改为node1节点设置一个污点: `tag=heima:NoExecute`；然后创建pod3 ( 3个pod都失败 )

```shell
# 为node1设置污点(PreferNoSchedule)
[root@k8s-master ~]# kubectl taint nodes node1 tag=heima:PreferNoSchedule

# 创建pod1
[root@k8s-master ~]# kubectl run taint1 --image=nginx:1.17.1 -n dev
[root@k8s-master ~]# kubectl get pods -n dev -o wide
NAME                      READY   STATUS    RESTARTS   AGE     IP           NODE   
taint1-7665f7fd85-574h4   1/1     Running   0          2m24s   10.244.1.59   node1    

# 为node1设置污点(取消PreferNoSchedule，设置NoSchedule)
[root@k8s-master ~]# kubectl taint nodes node1 tag:PreferNoSchedule-
[root@k8s-master ~]# kubectl taint nodes node1 tag=heima:NoSchedule

# 创建pod2
[root@k8s-master ~]# kubectl run taint2 --image=nginx:1.17.1 -n dev
[root@k8s-master ~]# kubectl get pods taint2 -n dev -o wide
NAME                      READY   STATUS    RESTARTS   AGE     IP            NODE
taint1-7665f7fd85-574h4   1/1     Running   0          2m24s   10.244.1.59   node1 
taint2-544694789-6zmlf    0/1     Pending   0          21s     <none>        <none>   

# 为node1设置污点(取消NoSchedule，设置NoExecute)
[root@k8s-master ~]# kubectl taint nodes node1 tag:NoSchedule-
[root@k8s-master ~]# kubectl taint nodes node1 tag=heima:NoExecute

# 创建pod3
[root@k8s-master ~]# kubectl run taint3 --image=nginx:1.17.1 -n dev
[root@k8s-master ~]# kubectl get pods -n dev -o wide
NAME                      READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED 
taint1-7665f7fd85-htkmp   0/1     Pending   0          35s   <none>   <none>   <none>    
taint2-544694789-bn7wb    0/1     Pending   0          35s   <none>   <none>   <none>     
taint3-6d78dbd749-tktkq   0/1     Pending   0          6s    <none>   <none>   <none>     
```

> 小提示：
>     使用kubeadm搭建的集群，默认就会给master节点添加一个污点标记,所以pod就不会调度到master节点上.

**容忍（Toleration）**

上面介绍了污点的作用，我们可以在 Node 上添加污点用于拒绝 Pod 调度上来，但是如果就是想将一个 Pod 调度到一个有污点的 Node 上去，这时候应该怎么做呢？这就要使用到**容忍**。

![image-20220325165202846](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220325165202846.png)

> 污点就是拒绝，容忍就是忽略，Node通过污点拒绝pod调度上去，Pod通过容忍忽略拒绝

下面先通过一个案例看下效果：

1. 上一小节，已经在 Node1 节点上打上了`NoExecute`的污点，此时 Pod 是调度不上去的
2. 本小节，可以通过给 Pod 添加容忍，然后将其调度上去

创建pod-toleration.yaml,内容如下

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-toleration
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  tolerations:      # 添加容忍
  - key: "tag"        # 要容忍的污点的key
    operator: "Equal" # 操作符
    value: "heima"    # 容忍的污点的value
    effect: "NoExecute"   # 添加容忍的规则，这里必须和标记的污点规则相同
```

```shell
# 添加容忍之前的pod
[root@k8s-master ~]# kubectl get pods -n dev -o wide
NAME             READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED 
pod-toleration   0/1     Pending   0          3s    <none>   <none>   <none>           

# 添加容忍之后的pod
[root@k8s-master ~]# kubectl get pods -n dev -o wide
NAME             READY   STATUS    RESTARTS   AGE   IP            NODE    NOMINATED
pod-toleration   1/1     Running   0          3s    10.244.1.62   node1   <none>        
```

下面看一下容忍的详细配置:

```shell
[root@k8s-master ~]# kubectl explain pod.spec.tolerations
......
FIELDS:
   key       # 对应着要容忍的污点的键，空意味着匹配所有的键
   value     # 对应着要容忍的污点的值
   operator  # key-value的运算符，支持Equal和Exists（默认）
   effect    # 对应污点的effect，空意味着匹配所有影响
   tolerationSeconds   # 容忍时间, 当effect为NoExecute时生效，表示pod在Node上的停留时间
```
