---
title: k8s核心概念总结
date: 2021-4-9 15:43:00
tags:
- 原创
categories:
- 运维
---

## 1. 什么是k8s？

**k8s是一个容器编排系统**

Kubernetes（k8s）是**自动化容器操作的开源平台**，**Docker是Kubernetes内部使用的低级别组件**。所以Kubernetes不仅仅支持Docker，还支持Rocket，这是另一种容器技术。

k8s配置非常简单，可以通过一个.yaml文件实现规则定义，所以可以先画部署图，然后将部署图翻译为yaml文件即可。

k8s上可以看到仪表盘，监控，日志，报警等等

k8s要求很多地方都要配置证书，很多时候k8s部署的时候是跨多云部署的，比如一个在aws，一个在google cloud上

k8s的功能如下：

- 自动化容器的部署和复制
- 很多时候对服务器或者容器的操作都不可追溯，比如通过图形界面的操作。k8s可以解决操作不可追溯这一问题。
- k8s可以动态变更队列大小，数量，当请求峰值时使队列变大，请求较少时使队列变小。
- 实现定时任务，比如每天晚上dump数据库进行备份
- 执行疑似性程序：所谓疑似性程序就是按需运行的程序，比如往etcd配置中心写数据
- 随时扩展或收缩容器规模
- 看流量追踪，比如流量先后经历了哪些节点
- 看日志
- **将容器组织成组**（组是一个新概念），并且提供容器间的负载均衡
- 很容易地升级应用程序容器的新版本
- 提供容器弹性，如果容器失效就替换它，等等...

实际上，使用Kubernetes只需一个部署文件，使用和Kubernetes API交互的命令行程序kubectl的一条命令就可以部署多层容器（前端，后台等）的完整集群：

```
kubectl create -f single-config-file.yaml
```

## 2. k8s的一些核心概念介绍

一个典型的k8s架构图如下：
我整理的图:k8s架构图
![QQ20200314-153534@2x.png](http://ww1.sinaimg.cn/large/006gLprLgy1gcthutgohsj31jq15yqec.jpg)

### 2.1 集群cluster

集群是一组节点，每个节点可以是物理服务器或者虚拟机，之上安装了Kubernetes平台。

实际生产中，可以针对不同的环境分别配置该环境对应的集群。

### 2.2 Pod

Pod（绿色方框）位于节点上，一个节点上可以有多个Pod。一个Pod包含了一组容器和卷。同一个Pod里的各个容器之间共享一个网络命名空间，可以用localhost互相通信。**Pod是短暂的，不是持续性实体。** 所以如果想要持久化容器中的数据，可以使用卷功能。而且Pod是暂时的，所以重启时Pod的IP地址可能会变化。

Pod一般会根据业务场景进行分组，是根据业务区分的。实际上与docker-compose类似，一个Pod可以可以包含多个容器，这些容器是同一个业务中的。

### 2.3 Label（标签）

一些Pod上会有Label。（图上Pod旁的深蓝色方框）一个Label是attach到Pod的一对键/值对，用来传递用户定义的属性。比如，你可能创建了一个"tier"和“app”标签，通过Label（tier=frontend, app=myapp）来标记前端Pod容器，使用Label（tier=backend, app=myapp）标记后台Pod。**然后可以使用Selectors选择带有特定Label的Pod，并且将Service或者Replication Controller应用到上面。**

### 2.4 Replication Controller（复制控制器）

Replication Controller确保任意时间都有指定数量的Pod“副本”在运行。如果为某个Pod创建了Replication Controller并且指定3个副本，它会创建3个Pod，并且持续监控它们。如果某个Pod不响应，那么Replication Controller会替换它。可以看看原文这里的动图。
![QQ20200314-145934@2x.png](http://ww1.sinaimg.cn/large/006gLprLgy1gctgtdbjvkj31au0ugtep.jpg)
由图可以看出，复制控制器保持了3个Pod副本。计算Pod是不必计算原本的那个，只需关注它的副本即可。

当创建Replication Controller时，需要指定两个东西：

- Pod模板：用来创建Pod副本的模板
- Label：Replication Controller需要监控的Pod的标签。

### 2.5 Service（服务）

因为Pods是短暂的，那么重启时IP地址可能会改变，怎么才能从前端容器正确可靠地指向后台容器呢？

Service是定义一系列Pod以及访问这些Pod的策略的一层抽象。**Service通过Label找到Pod组**。因为Service是抽象的，所以在图表里通常看不到它们的存在，这也就让这一概念更难以理解。

现在，假定有2个后台Pod，并且定义后台Service的名称为‘backend-service’，lable选择器为（tier=backend, app=myapp）。backend-service 的Service会完成如下两件重要的事情：
会为Service创建一个本地集群的DNS入口，因此前端Pod只需要DNS查找主机名为 ‘backend-service’，就能够解析出前端应用程序可用的IP地址。
现在前端已经得到了后台服务的IP地址，但是它应该访问2个后台Pod的哪一个呢？Service在这2个后台Pod之间提供透明的负载均衡，会将请求分发给其中的任意一个（如下面的动画所示）。通过每个Node上运行的代理（kube-proxy）完成。这里有更多技术细节。

有一个特别类型的Kubernetes Service，**称为'LoadBalancer'，作为外部负载均衡器使用，在一定数量的Pod之间均衡流量**。比如，对于负载均衡Web流量很有用。

### 2.6 Node（节点）

在k8s中的意思通常指**资源**。从图上可以看出，两个Node上可以分别运行一模一样的Pod副本。节点（上图橘色方框）是物理或者虚拟机器，作为Kubernetes worker，通常称为Minion。每个节点都运行如下Kubernetes关键组件：

- Kubelet：是主节点代理。会暴露API给master进行使用和调用Node，这样Node可以控制Pod和Node的相关配置。kubelet也是一个基于docker或者其他容器技术运行的容器。kubelet可以与docker引擎通信，控制该机器上的容器。
- Kube-proxy：Service使用其将链接路由到Pod，如上文所述。
- Docker或Rocket：Kubernetes使用的容器技术来创建容器。

### 2.7 Kubernetes Master

集群拥有一个Kubernetes Master（紫色方框）。

Kubernetes Master提供集群的独特视角，并且拥有一系列组件，比如Kubernetes API Server。API Server提供可以用来和集群交互的REST端点。master节点包括用来创建和复制Pod的Replication Controller。

master上运行了etcd ,etcd是一个数据中心，存储了每个Node上的每个Pod的运行情况，运行情况包括健康检查，探针（ 是由 [kubelet](https://kubernetes.io/docs/admin/kubelet/) 对容器执行的定期诊断，用于看服务是否启动成功）等。

## 3. k8s各个核心概念的关系

灰色方框表示集群，集群里有一组节点（黄色方框）。
一个节点包含多个Pod。

Pod是短暂的，重启时IP会变化。

复制控制器：确保制定Pod的指定个数的副本在运行。（如图，控制四个副本在运行）

因为Pod重启时IP会变，服务即Service是定义一系列Pod以及访问这些Pod的策略的一层抽象，Service通过label找到Pod组。Service实现了负载均衡，方便Pod之间相互访问，

**master的功能是指挥调度**:那么具体指挥调度的是什么呢？可以调度Pod，当指定某个Pod的指定个数的副本时候，master可以实现在某个Pod副本挂掉的时候，重建其他Pod，以实现保持Pod的副本个数为指定个。master可以为各个副本提供负载均衡的功能，也可以实现各个副本的滚动升级。master除了调度Pod之外，还可以控制各个Node，不过控制的是Node与k8s相关的配置。master控制Node或者控制Node上的Pod，都是通过Node上的kubelet实现的。

下面是原文的图：

![QQ20200314-144217@2x.png](http://ww1.sinaimg.cn/large/006gLprLgy1gctgbglcc2j31ei0zk49e.jpg)

参考

[入门级的原文](http://www.dockone.io/article/932)
