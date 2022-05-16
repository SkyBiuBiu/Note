# [Linux]iptables防火墙

[toc]

<img src="https://www.zsythink.net/wp-content/uploads/ueditor/php/upload/image/20170212/1486863972980583.png" alt="iptables.png" style="zoom:50%;" />

## 一、iptables介绍

iptables是一个针对IPv4/IPv6数据包的管理工具，能够实现包过滤和NAT功能。

它作为一个管理工具，可以去设置、维护和检查表，每个表中包含了许多的内置链以及用户自定义链，然后链中包含许多的规则，用于匹配一组数据包。每条规则制定如何处理匹配的数据包，这个被称为目标（Target），这条规则可能会使得数据包跳转到同一个表中的其他用户自定义链中。

iptables只是Linux防火墙的管理工具而已，其实真正实现防火墙功能的是netfilter，它是Linux内核中实现包过滤的内部结构。

![img](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/1124877-20170313192058276-752439770.png)

这里我们需要明确几个iptables中的重要概念概念：

- 表（Table）
- 链（Chain）
- 规则（Rule）

在后面会详细的介绍这几个概念！



## 二、表（Table）

![img](https://www.cnbugs.com/wp-content/uploads/2020/04/image-78.png)

iptables中有四个类型的表：

- filter表：用于过滤数据包，不指定的情况下，iptables命令默认操作的就是filter表
- nat表：用于网络地址转换（IP、端口）
- mangle表：修改数据包的服务类型、TTL并且可以配置路由实现QoS的功能
- raw表：决定数据包是否被状态跟踪机制处理



## 三、链（Chain）

表中包含了许多的内置链和用户自定义链，它有以下的类型：

- INPUT：进来的数据包应用此规则链中的策略
- OUTPUT：外出的数据包应用此规则链中的策略
- FORWARD：转发数据包时应用此规则链中的策略
- PREROUTING：对数据包作路由选择前应用此链中的规则（所有的数据包进来的时侯都先由这个链处理）
- POSTROUTING：对数据包作路由选择后应用此链中的规则（所有的数据包出来的时侯都先由这个链处理）



![img](https://www.cnbugs.com/wp-content/uploads/2020/04/image-79.png)

iptables的数据包转发流程：

- 目的地址是本地，则发送到 INPUT，让 INPUT 决定是否接收下来送到用户空间，流程为①--->②；
- 若满足 PREROUTING 的 nat表 上的转发规则，则发送给 FORWARD ，然后再经过 POSTROUTING 发送出去，流程为： ①--->③--->④--->⑥；
- 主机发送数据包时，流程则是⑤--->⑥；
- 其中 PREROUTING 和 POSTROUTING 指的是数据包的流向，如上图所示 POSTROUTING 指的是发往公网的数据包，而PREROUTING 指的是来自公网的数据包。



## 四、规则（Rule）

![image-20220409002849720](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220409002849720.png)

规则（rules）其实就是网络管理员预定义的条件，规则一般的定义为“如果数据包头符合这样的条件，就这样处理这个数据包”。规则存储在内核空间的信息包过滤表中，这些规则分别指定了源地址、目的地址、传输协议（如TCP、UDP、ICMP）和服务类型（如HTTP、FTP和SMTP）等。当数据包与规则匹配时，iptables就根据规则所定义的方法来处理这些数据包，如放行（ACCEPT）、拒绝（REJECT）和丢弃（DROP）等。配置防火墙的主要工作就是添加、修改和删除这些规则。



## 五、iptables规则的增删改查

![image-20220409003444651](https://gitee.com/skybiubiu/mdpic/raw/master/imgs/image-20220409003444651.png)

1. 查看filter表中规则（如果不加-t 指定表类型，默认就是filter表）

```shell
[root@linux ~]# iptables -L -n --line-number |grep 21 //--line-number可以显示规则序号，在删除的时候比较方便
5    ACCEPT     tcp  --  192.168.1.0/24       0.0.0.0/0           tcp dpt:21
```

2. 修改规则

```shell
[root@linux ~]# iptables -R INPUT 3 -j DROP    //将规则3改成DROP，这里的R代表replace，也就是替换的意思
```

3. 删除iptables规则

```shell
[root@linux ~]# iptables -D INPUT 3  //删除input的第3条规则
[root@linux ~]# iptables -t nat -D POSTROUTING 1  //删除nat表中postrouting的第一条规则
[root@linux ~]# iptables -F INPUT   //清空 filter表INPUT所有规则
[root@linux ~]# iptables -F    //清空所有规则
[root@linux ~]# iptables -t nat -F POSTROUTING   //清空nat表POSTROUTING所有规则
```

4. 设置默认规则

```shell
[root@linux ~]# iptables -P INPUT DROP  //设置filter表INPUT默认规则是 DROP
```

