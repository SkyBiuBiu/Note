# [云原生]Kubernetes - 资源管理（第3章）

[toc]

**参考：**

- [Kubernetes(K8S) 入门进阶实战完整教程，黑马程序员K8S全套教程（基础+高级）_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1Qv41167ck?spm_id_from=333.999.0.0)

## 一、资源管理介绍

在Kubernetes中，所有的内容都抽象为资源，用户需要通过操作资源来管理Kubernetes。

> Kubernetes的本质上就是一个集群系统，用户可以在集群中部署各种服务，所谓的部署服务，就是在Kubernetes集群中运行一个个的容器，并将指定的程序跑在容器中。
>
> Kubernetes的最小管理单元是Pod而不是容器，所以只能将容器放在`pod`中，而Kubernetes一般也不会直接管理Pod，而是通过`Pod控制器`来管理Pod。
>
> Pod可以提供服务之后，就要考虑如何访问Pod中服务，Kubernetes提供了`Service`资源实现这个功能。
>
> 当然，如果Pod中程序的数据需要持久化，Kubernetes还提供了各种`存储`系统。

![image-20220309093128697](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220309093128697.png)

> 学习Kubernetes的核心，就是学习如何对集群上的`Pod`、`Pod控制器`、`Service`、`存储`等各种资源进行操作。



## 二、YAML语言介绍

YAML是一个类似 XML、JSON 的标记性语言。它强调以**数据**为中心，并不是以标识语言为重点。因而YAML本身的定义比较简单，号称“一种人性化的数据格式语言”。

- XML

```xml
<skybiubiu>
	<age>100</age>
	<address>ShenZhen</address>
```

- YAML

```yaml
skybiubiu:
 age: 100
 address: ShenZhen
```

YAML的语法比较简单，主要有下面几个：

- 大小写敏感
- 使用缩进表示层级关系
- 缩进不允许使用tab，只允许空格（低版本限制）
- 缩进的空格数不重要，只要相同层级的元素左对齐即可
- "#" 表示注释

YAML支持一下几种数据类型：

- 纯量：单个的、不可再分的值

- 对象：键值对的集合，又称为映射（mapping）/ 哈希（hash）/ 字典（dictionary）

- 数组：一组按次序排列的值，又称为序列（sequence）/ 列表（list）

```yaml
# 纯量，就是指的一个简单的值，字符串、布尔值、整数、浮点数、Null、时间、日期

# 1. 布尔类型
c1: true #（true/True都可以）

# 2. 整型
c2: 234

# 3. 浮点型
c3: 3.14

# 4. null类型
c4: ~

# 5. 日期类型
c5: 2018-02-17

# 6. 时间类型
c6: 2018-02-17T15:02:31+08:00

# 7. 字符串类型
c7: skybiubiu
c8: line1
  : line2
```

```yaml
# 对象

# 形式一（推荐）：
skybiubiu:
 age: 100
 address: ShenZhen

# 形式二（了解）：
skybiubiu: {age: 100,address: Shenzhen}
```

```yaml
# 数组

# 形式一（推荐）：
address:
 - 南山
 - 龙岗
 
# 形式二（了解）：
address: [南山，龙岗]
```

> 小提示：
>
> 1. 书写yaml切记`:`，后面要加一个空格
>
> 2. 如果需要将多段yaml配置放在一个文件中，中间要使用`---`分割
>
> 3. 下面是一个yaml转json的网站，可以通过它验证yaml是否书写正确
>
>    https://www.json2yaml.com/convert-yaml-to-json



## 三、资源管理方式

- 命令式对象管理：直接使用命令去操作Kubernetes资源

  `kubectl run nginx-pod --image=nginx:1.17.1 --port=80`

- 命令式对象配置：通过命令配置和配置文件去操作Kubernetes资源

  `kubectl create/patch -f nginx-pod.yaml`

- 声明式对象配置：通过apply命令和配置文件去操作Kubernetes资源

  `kubectl apply -f nginx-pod.yaml`

| 类型           | 操作对象 | 适用环境 | 优点           | 缺点                           |
| -------------- | -------- | -------- | -------------- | ------------------------------ |
| 命令式对象管理 | 对象     | 测试     | 简单           | 只能操作活动对象，无法审计     |
| 命令式对象配置 | 文件     | 开发     | 可以审计、跟踪 | 项目大时，配置文件多，操作麻烦 |
| 声明式对象配置 | 目录     | 开发     | 支持目录操作   | 意外情况下难以调试             |



### 3.1 命令式对象管理

**kubectl命令**

kubectl是kubernetes集群的命令行工具，通过它能够对集群本身进行管理，并能够在集群上进行容器化应用的安装部署。kubectl命令的语法如下：

```shell
kubectl [command] [type] [name] [flags]
```

**command：**指定要对资源执行的操作，例如：create、get、delete

**type：**指定资源类型，比如deployment、pod、service

**name：**指定资源的名称、名称大小写敏感

**flags：**指定额外的可选参数

```shell
# 查看所有Pod
kubectl get pod

# 查看某个Pod
kubectl get pod pod_name

# 查看某个Pod，以yaml格式展示结果
kubectl get pod pod_name -o yaml
```



**资源类型**

kubernetes中所有的内容都抽象为资源，可以通过下面的命令进行查看：

```shell
kubectl api-resources
```

经常适用的资源有下面这些：

| 资源分类      | 资源名称                 | 缩写   | 资源作用        |
| :------------ | :----------------------- | :----- | :-------------- |
| 集群级别资源  | nodes                    | no     | 集群组成部分    |
|               | namespaces               | ns     | 隔离Pod         |
| pod资源       | pods                     | po     | 装载容器        |
| pod资源控制器 | replicationcontrollers   | rc     | 控制pod资源     |
|               | replicasets              | rs     | 控制pod资源     |
|               | deployments              | deploy | 控制pod资源     |
|               | daemonsets               | ds     | 控制pod资源     |
|               | jobs                     |        | 控制pod资源     |
|               | cronjobs                 | cj     | 控制pod资源     |
|               | horizontalpodautoscalers | hpa    | 控制pod资源     |
|               | statefulsets             | sts    | 控制pod资源     |
| 服务发现资源  | services                 | svc    | 统一pod对外接口 |
|               | ingress                  | ing    | 统一pod对外接口 |
| 存储资源      | volumeattachments        |        | 存储            |
|               | persistentvolumes        | pv     | 存储            |
|               | persistentvolumeclaims   | pvc    | 存储            |
| 配置资源      | configmaps               | cm     | 配置            |
|               | secrets                  |        | 配置            |

**操作**

Kubernetes允许对资源进行多种操作，可以通过--help查看详细的操作命令

```shell
kubectl --help
```

经常适用的操作有下面这些：

| 命令分类   | 命令         | 翻译                        | 命令作用                     |
| :--------- | :----------- | :-------------------------- | :--------------------------- |
| 基本命令   | create       | 创建                        | 创建一个资源                 |
|            | edit         | 编辑                        | 编辑一个资源                 |
|            | get          | 获取                        | 获取一个资源                 |
|            | patch        | 更新                        | 更新一个资源                 |
|            | delete       | 删除                        | 删除一个资源                 |
|            | explain      | 解释                        | 展示资源文档                 |
| 运行和调试 | run          | 运行                        | 在集群中运行一个指定的镜像   |
|            | expose       | 暴露                        | 暴露资源为Service            |
|            | describe     | 描述                        | 显示资源内部信息             |
|            | logs         | 日志输出容器在 pod 中的日志 | 输出容器在 pod 中的日志      |
|            | attach       | 缠绕进入运行中的容器        | 进入运行中的容器             |
|            | exec         | 执行容器中的一个命令        | 执行容器中的一个命令         |
|            | cp           | 复制                        | 在Pod内外复制文件            |
|            | rollout      | 首次展示                    | 管理资源的发布               |
|            | scale        | 规模                        | 扩(缩)容Pod的数量            |
|            | autoscale    | 自动调整                    | 自动调整Pod的数量            |
| 高级命令   | apply        | rc                          | 通过文件对资源进行配置       |
|            | label        | 标签                        | 更新资源上的标签             |
| 其他命令   | cluster-info | 集群信息                    | 显示集群信息                 |
|            | version      | 版本                        | 显示当前Server和Client的版本 |

下面以一个 namespace / pod 的创建和删除简单演示一下命令的使用：

```shell
# 创建一个namespace
[root@k8s-master ~]# kubectl create namespace dev
namespace/dev created

# 获取namespace
[root@k8s-master ~]# kubectl get ns
NAME              STATUS   AGE
default           Active   18h
dev               Active   6s
kube-node-lease   Active   18h
kube-public       Active   18h
kube-system       Active   18h

# 在此namespace下创建并运行一个nginx的Pod
[root@k8s-master ~]# kubectl run pod --image=nginx:latest -n dev
pod/pod created

# 查看新创建的Pod
[root@k8s-master ~]# kubectl get pods -n dev
NAME   READY   STATUS    RESTARTS   AGE
pod    1/1     Running   0          61s

# 删除指定的Pod
[root@k8s-master ~]# kubectl delete pods pod -n dev
pod "pod" deleted

# 删除指定的namespace
[root@k8s-master ~]# kubectl delete ns dev
namespace "dev" deleted
```



### 3.2 命令式对象配置

命令式对象配置就是使用命令配合配置文件一起来操作Kubernetes资源。

1. 创建一个nginxpod.yaml，内容如下：

```yaml
# 1. 创建namespace
---
apiVersion: v1
kind: Namespace
metadata: 
 name: dev
...

# 2. 创建Pod（在dev这个namespace中）
---
apiVersion: v1
kind: Pod
metadata: 
 name: nginxpod
 namespace: dev
spec:
 containers:
 - name: nginx-containers
   image: nginx:latest
...

```

2. 执行create命令，创建资源：

```shell
[root@k8s-master k8s-yaml]# kubectl create -f nginxpod.yaml
namespace/dev created
pod/nginxpod created
```

此时发现创建了两个资源对象，分别是namespace和pod

3. 执行get命令，查看资源：

```shell
[root@k8s-master k8s-yaml]# kubectl get -f nginxpod.yaml
NAME            STATUS   AGE
namespace/dev   Active   52s

NAME           READY   STATUS    RESTARTS   AGE
pod/nginxpod   1/1     Running   0          52s
```

4. 执行delete缪那个零，删除资源：

```shell
[root@k8s-master k8s-yaml]# kubectl delete -f nginxpod.yaml
namespace "dev" deleted
pod "nginxpod" deleted
```

此时发现两个资源对象被删除了

> **总结：**命令式对象配置的方式操作资源，可以简单的认为，<u>命令 + yaml 配置文件</u>（里面是命令需要的各种参数）。



### 3.3 声明式对象配置

声明式对象配置跟命令式对象配置很相似，但是它只有一个命令`apply`。

```shell
# 第一次执行时，发现创建了资源
[root@k8s-master k8s-yaml]# kubectl apply -f nginxpod.yaml
namespace/dev created
pod/nginxpod created

# 第二次执行时，发现资源没有变动
[root@k8s-master k8s-yaml]# kubectl apply -f nginxpod.yaml
namespace/dev unchanged
pod/nginxpod unchanged
```

>  **总结：**其实声明式对象配置就是使用apply描述一个资源最终的状态（在yaml中定义状态）
>
> ​      使用apply操作资源时：
>
> ​		如果资源不存在，就创建，相当于 `kubectl create`
>
> ​		如果资源已存在，就更新，相当于 `kubectl patch`



> **扩展：**kubectl 可以在node节点上运行吗？

kubectl的运行时需要进行配置的，它的配置文件是 $HOME/.kube，如果想要在node节点运行此命令，需要将master上的.kube文件复制到node节点，即在master节点上执行如下的操作。

```shell
scp -r HOME/.kube k8s-node01: HOME/
```



**使用推荐：**三种方式资源管理方式应该怎么用？

- 创建/更新资源，使用声明式对象配置 `kubectl apply -f XXX.yaml`

- 删除资源，使用命令式对象配置  `kubectl delete -f XXX.yaml`
- 查询资源，使用命令式对象管理  `kubectl get/describe 资源名称`

