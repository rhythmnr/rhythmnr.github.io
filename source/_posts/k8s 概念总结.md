---
title: k8s 概念总结
date: 2021-4-9 15:41:00
tags:
- 原创
categories:
- 运维
---

K8s 的核心功能：自动化运维管理多个容器化程序

![Xnip2021-06-27_10-01-10.jpg](http://ww1.sinaimg.cn/large/006gLprLgy1grwlfqg8l6j61640kijye02.jpg)

K8S 中，主节点一般被称为**Master Node** 或者 **Head Node**，而从节点则被称为**Worker Node** 或者 **Node**。同一个集群可能存在多个 Master Node 和 Worker Node。

**Master Node** 的组件有：**API Server**，**Scheduler**，**Controller Manager**，**etcd**

**Worker Node** 的组件有：**Kubelet**，**Kube-Proxy**，**Container Runtime**，**Logging Layer(K8S** 的监控状态收集器**)**，**Add-Ons(K8S** 管理运维 **Worker Node** 的插件组件**)**

————————————— 概念

————————————— 概念：组件（至此只与 Node 有关）

- **API Server**

**K8S** 的请求入口服务。API Server 负责接收 K8S 所有请求（来自 UI 界面或者 CLI 命令行工具），然后，API Server 根据用户的具体请求，去通知其他组件干活。

- **Scheduler**

**K8S** 所有 **Worker Node** 的调度器。当用户要部署服务时，Scheduler 会选择最合适的 Worker Node（服务器）来部署。

- **Controller Manager**

**K8S** 所有 **Worker Node** 的监控器，会指挥 **Scheduler** 干活。

Controller 负责监控和自动调节。负责监控和调整在 Worker Node 上部署的服务的状态，比如用户要求 A 服务部署 2 个副本，那么当其中一个服务挂了的时候，Controller 会马上调整，让 Scheduler 再选择一个 Worker Node 重新部署服务。

- Etcd

有 Secret 对象，存放了容器需要使用的敏感信息。k8s 会在指定 Pod 启动时，自动把 Secret 数据挂载到容器中。

K8S 中仅 API Server 才具备读写权限

- **Kubelet**

**Worker Node** 的监视器，以及与 **Master Node** 的通讯器。Kubelet 是 Master Node 安插在 Worker Node 上的“眼线”，它会定期向 Worker Node 汇报自己 Node 上运行的服务的状态，并接受来自 Master Node 的指示采取调整措施。

- **Kube-Proxy**

**K8S** 的网络代理。负责网络和负载均衡。

- **Container Runtime**

**Worker Node** 的运行环境。即安装了容器化所需的软件环境确保容器化程序能够跑起来，比如 Docker Engine。

- **Logging Layer**

**K8S** 的监控状态收集器。Logging Layer 负责采集 Node 上所有服务的 CPU、内存、磁盘、网络等监控项信息。

- **Add-Ons**

**K8S** 管理运维 **Worker Node** 的插件组件。让用户可以扩展更多定制化功能。

————————————— 概念：其他

- Pod（没有什么对应的英文翻译）

**Pod**是可以在 Kubernetes 中创建和管理的、最小的可部署的计算单元。

一个Pod总是运行在一个**Node**上。

一个 **Pod** 内可以有多个容器 **container**。

**Pod** 可以被理解成一群可以共享网络、存储和计算资源的容器化服务的集合。

内含多个容器。Pod 内的容器在同一个 Pod 里的几个 Docker 服务/程序，好像被部署在同一台机器上，可以通过 localhost 互相访问，且共享信息（包括配置）。不同的 **Pod** 之间的 **Container** 不能用 **localhost** 访问，也不能挂载其他 **Pod** 的数据卷。

可以在 yaml 中声明 kind 为 Pod 来创建 Pod。例如：

apiVersion: v1 # K8S 的 API Server 版本

kind: Pod

metadata: #  Pod 自身的元数据

 name: memory-demo # Pod 的名字

 namespace: mem-example # 这个 Pod 属于哪个 namespace

spec: # Pod 内部所有的资源的详细信息

 containers:

 \- name: memory-demo-ctr # 容器名

  image: polinux/stress

  resources: # 容器需要的 CPU、内存、GPU 等资源

   limits:

​    memory: "200Mi"

   requests:

​    memory: "100Mi"

  command: ["stress"] # 容器的入口命令

  args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"] # 容器的入口参数

  volumeMounts: # 容器要挂载的 Pod 数据卷等

  \- name: redis-storage

   mountPath: /data/redis

 volumes: # Pod 内的数据卷信息

 \- name: redis-storage

  emptyDir: {}

- Node 节点

一个 Node 对应了一台实体服务器。每个 Node 上可有多个 Pod。

- Volumn 数据卷

数据卷 **volume** 是 **Pod** 内部的磁盘资源。

- **Container** 容器

部署的大多是标准容器（ **Application Container**）

- **Deployment**

**Deployment** 的作用是管理和控制 **Pod** 和 **ReplicaSet**，管控它们运行在用户期望的状态中。打个形象的比喻，**Deployment** 就是包工头，主要负责监督底下的工人 Pod 干活，确保每时每刻有用户要求数量的 Pod 在工作。如果一旦发现某个工人 Pod 不行了，就赶紧新拉一个 Pod 过来替换它。

Deployment并不是直接控制着Pod的，中间实际上还有一个ReplicaSet

根据我们的需求（比如通过标签）将Pod调度到目标机器上，调度完成之后，它还会继续帮我们继续监控容器是否在正确运行，一旦出现问题，会立刻告诉我们Pod的运行不正常以及寻找可能的解决方案，比如目标节点不可用的时候它可以快速地调度到别的机器上去。另外，如果需要对应用扩容提升响应能力的时候，通过Deployment可以快速地进行扩展。

将Pod调度到目标机器上是什么意思？？？？？？？这是问题，知道的话解释下：：：应该是在哪个机器上产生 Pod 的意思吧

![Xnip2021-06-27_10-01-42.jpg](http://ww1.sinaimg.cn/large/006gLprLgy1grwlg709tdj61ds0sygqu02.jpg)

下面是一个 **Deployment** 的配置文件：

apiVersion: extensions/v1beta1

 kind: Deployment

 metadata:

  name: rss-site

  namespace: mem-example # 如果没有指明 namespace，那么就是用 kubectl 默认的 namespace 即 default

 spec:

  replicas: 2 # 副本个数。也就是该 Deployment 需要起多少个相同的 Pod

  template:

   metadata:

​    labels:

​     app: web

   spec: # 写明了 Deployment 下属管理的每个 Pod 的具体内容。

   containers:

​    \- name: memory-demo-ctr

​     image: polinux/stress

​     resources:

​     limits:

​      emory: "200Mi"

​     requests:

​      memory: "100Mi"

​     command: ["stress"]

​     args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]

​     volumeMounts:

​     \- name: redis-storage

​      mountPath: /data/redis

   volumes:

   \- name: redis-storage

​    emptyDir: {}

- ReplicaSet

简称 RS。ReplicaSet 的目的是维护一组在任何时候都处于运行状态的 Pod 副本的稳定集合。 因此，它通常用来保证给定数量的、完全相同的 Pod 的可用性。

ReplicaSet 的作用就是管理和控制 Pod，管控他们好好干活。但是，ReplicaSet 受控于 Deployment。

用户会直接操作 Deployment 部署服务，而当 Deployment 被部署的时候，K8S 会自动生成要求的 ReplicaSet 和 Pod。在[**K8S** 官方文档](https://link.zhihu.com/?target=https%3A//www.kubernetes.org.cn/replicasets)中也指出用户只需要关心 Deployment 而不操心 ReplicaSet：

![Xnip2021-06-27_10-02-17.jpg](http://ww1.sinaimg.cn/large/006gLprLgy1grwlgsddpjj61cq0kowh402.jpg)

- **Service**

**Service** 不一定有个固定的 **IP**。

**Service** 負責管控 **Pod** 網絡服務。是將運行在一組 **Pods** 上的應用程序公開為網絡服務的抽象方法。

使用 Kubernetes，您无需修改应用程序即可使用不熟悉的服务发现机制。 Kubernetes 为 Pods 提供自己的 IP 地址，并为一组 Pod 提供相同的 DNS 名， 并且可以在它们之间进行负载均衡。

Service是一个完全虚拟的网络层，并不会存在于任何网络设备上。它通过修改集群内部的路由规则，仅对集群内部有效。

Deploment创建好应用之后，再为它生成一个Service对象。接下来就可以通过Service的域名访问到服务，形式是<Service Name>.<NameSpace>，比如你有为Deployment的应用创建了一个名为portal的Service在默认的命名空间，那么集群内想要通过Http访问这个应用，就可以使用[http://portal.default](https://link.zhihu.com/?target=http%3A//portal.default)。这个域名仅在集群内有效，因为是内部的一个DNS负责解析。

**Service** 是 **K8S** 服务的核心，屏蔽了服务细节，统一对外暴露服务接口，真正做到了**“**微服务**”**。举个例子，我们的一个服务 A，部署了 3 个备份，也就是 3 个 Pod；对于用户来说，只需要关注一个 Service 的入口就可以，而不需要操心究竟应该请求哪一个 Pod。优势非常明显：一方面外部用户不需要感知因为 **Pod** 上服务的意外崩溃、**K8S** 重新拉起 **Pod** 而造成的 **IP** 变更，外部用户也不需要感知因升级、变更服务带来的 **Pod** 替换而造成的 **IP** 变化，另一方面，**Service** 还可以做流量负载均衡。

![Xnip2021-06-27_10-02-44.jpg](http://ww1.sinaimg.cn/large/006gLprLgy1grwlhci9k2j61ek0skq8b02.jpg)

- Ingress

**Ingress** 负责管控 **Pod** 网络服务。

Service 主要负责 K8S 集群内部的网络拓扑。Ingress 用于对外提供服务。它是集群的入口。比如我们的集群Web应用想要让用户能够访问，那必然要在Ingress入口上增加一条解析记录。这一点，熟悉像Nginx的朋友应该比较容易理解，事实上Nginx Ingress也是K8s生态中的一个成员。

![Xnip2021-06-27_10-03-18.jpg](http://ww1.sinaimg.cn/large/006gLprLgy1grwlhv0o6oj61cy0ps43r02.jpg)

![Xnip2021-06-27_10-03-44.jpg](http://ww1.sinaimg.cn/large/006gLprLgy1grwliarw7pj61eu0skadw02.jpg)

- namespace 命名空间

namespace 跟 Pod 没有直接关系，而是 K8S 另一个维度的对象。或者说，前文提到的概念都是为了服务 Pod 的，而 namespace 则是为了服务整个 K8S 集群的。

Kubernetes 支持多个虚拟集群，它们底层依赖于同一个物理集群。 这些虚拟集群被称为名字空间。**namespace** 是为了把一个 **K8S** 集群划分为若干个资源不可共享的虚拟集群而诞生的。

可以通过在 **K8S** 集群内创建 **namespace** 来分隔资源和对象。比如我有 2 个业务 A 和 B，那么我可以创建 ns-a 和 ns-b 分别部署业务 A 和 B 的服务，如在 ns-a 中部署了一个 deployment，名字是 hello，返回用户的是“hello a”；在 ns-b 中也部署了一个 deployment，名字恰巧也是 hello，返回用户的是“hello b”（要知道，在同一个 namespace 下 deployment 不能同名；但是不同 namespace 之间没有影响）。前文提到的所有对象，都是在 namespace 下的；当然，也有一些对象是不隶属于 namespace 的，而是在 K8S 集群内全局可见的

- **DaemonSet** 守护进程

保证在所有的目标节点上运行一个Pod的副本。在这期间，如果有新的Node加入到K8s集群中的话，它也会自动完成调度，在新的机器上运行一个Pod副本。守护进程的使用场景比如日志收集等。

- Job和CronJob对象

用于定时任务

![Xnip2021-06-27_10-04-07.jpg](http://ww1.sinaimg.cn/large/006gLprLgy1grwlirakv5j613i0n077v02.jpg)

- **kubectl**

Kubectl 是一个命令行接口，用于对 Kubernetes 集群运行命令。
