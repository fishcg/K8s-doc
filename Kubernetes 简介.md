# Kubernetes 简介

| 版本  | 创建/更新者 | 创建/更新时间 | 版本说明                                  |
| ----- | ----------- | ------------- | ----------------------------------------- |
| 0.1.0 | 小鱼      | 2019-02-14    | 添加 Kubernetes 基本名词解释 |


导读：Kubernetes（K8s）是开源的，用于管理云平台中多个主机上的容器化的应用，应用操作包括部署，调度和节点集群间扩展。如果你曾经用过 Docker 容器技术部署容器，那么可以将 Docker 看成 Kubernetes 内部使用的低级别组件。（Kubernetes 不仅仅支持 Docker，还支持 Rocket，这是另一种容器技术）
> 小贴士：K8s 是 Kubernetes 的缩写，其中 8 是指 K 后面的 8 个字母

## Kubernetes 的作用
* 自动化容器的部署和复制
* 随时扩展或收缩容器规模
* 将容器组织成组，并且提供容器间的**负载均衡**
* 很容易地升级应用程序容器的新版本
* 提供容器弹性，如果容器失效就替换它
  ...

## K8s 基本介绍

### 架构图

![架构图](https://raw.githubusercontent.com/kubernetes/kubernetes/release-1.2/docs/design/architecture.png)

### 组件

* etcd 保存了整个集群的状态；
* apiserver 用于暴露 Kubernetes API，提供了资源操作的唯一入口。任何的资源请求/调用操作都是通过kube-apiserver提供的接口进行；
* controller manager 负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
* scheduler 负责资源的调度，按照预定的调度策略将 Pod 调度到相应的机器上；
* kubelet 负责维护容器的生命周期，同时也负责 Volume（CVI）和网络（管理 Kubernetes Master 和 Node 之间的通信）的管理；
* Container runtime 负责镜像管理以及 Pod 和容器的真正运行（CRI）；
* kube-proxy 负责为 Service 提供 cluster 内部的服务发现和负载均衡；
* kube-dns 负责为整个集群提供 DNS 服务

### 名词解释

![集群](http://dockone.io/uploads/article/20151230/d56441427680948fb56a00af57bda690.png)

#### Node

节点是物理或者虚拟机器，作为 Kubernetes worker，通常称为 Minion。每个节点都运行如下 Kubernetes 关键组件：
kubelet：是主节点代理。
kube-proxy：Service 使用其将链接路由到 Pod，如上文所述。
Docker 或 Rocket：Kubernetes 使用的容器技术来创建容器。

> 集群拥有一个 Kubernetes Master。Kubernetes Master 提供集群管理控制，并且拥有一系列组件，比如 Kubernetes API Server 与 Replication Controller。API Server 提供可以用来和集群交互的接口；Replication Controller 用于创建和复制 Pod。

#### Pod

Pod 应用于节点上，包含一组容器和卷，一个 Pod 是一个**容器环境**下的“逻辑主机”（类似模具下生产出的玩具），它可能包含一个或者多个紧密相连的应用。同一个 Pod 里的容器和应用共享同一个网络命名空间（可以使用 localhost 互相通信）。Pod 中的应用也可以共享磁盘（Volumes）。
Pod 是短暂的（无状态），不是持续性实体。
提出问题：
1. 如果 Pod 是短暂的，那么我怎么才能持久化容器数据使其能够跨重启而存在呢？
  虽然 Kubernetes 支持卷的概念，但共享磁盘目录（EmptyDir）的生命周期和所属的 Pod 是完全一致的，因此 Pod 中的数据并不能跨重启而存在。
2. 是否手动创建 Pod，如果想要创建同一个容器的多份拷贝，需要一个个分别创建出来么？
  可以手动创建单个 Pod，但是也可以使用 Replication Controller 使用 Pod 模板创建出多份拷贝，下文会详细介绍。
3. 如果 Pod 是短暂的，那么重启时 IP 地址可能会改变，那么怎么才能从前端容器正确可靠地指向后台容器呢？
  这时可以使用 Service，下文会详细介绍

#### Label

正如图所示，一些 Pod 有 Label（pod description）。一个 Label 是附加（attach）到Pod的一对键/值对，用来传递用户定义的属性。比如，你可能创建了一个“tier”和“app”标签，通过 Label（tier = backend, app = myapp）来标记后端 Pod 容器。然后可以使用 Selectors 选择带有特定 Label 的 Pod 进行相关操作，比如将 Service 或者 Replication Controller 应用到上面。

#### Service

Service 定义了访问这些 Pod 的策略的一层抽象。Service 通过 Label 找到 Pod 组。
现在，假定有 2 个后台 Pod，并且定义后台 Service 的名称为“backend-service”，Label 选择器为（tier = backend, app = myapp）。backend-service 的 Service 会完成如下两件重要的事情：
1. 服务发现
将新的 Pod 服务放入其负载均衡的体系中
2. DNS 寻址
会为 Service 创建一个本地集群的 DNS 入口，因此前端 Pod 只需要通过组件 DNS 查找主机名为“backend-service”（此过程由 kube-dns 组件实现），就能够解析出前端应用程序可用的 IP 地址。
得到了后台服务的 IP 地址后，Service 在这 2 个后台 Pod 之间提供透明的负载均衡，会将请求分发给其中的任意一个（如下面的动画所示）。通过每个 Node 上运行的代理（kube-proxy）完成。
![Service](http://dockone.io/uploads/article/20151230/125bbccce0b3bbf42abab0e520d9250b.gif)

> 有一个特别类型的 Kubernetes Service，称为“LoadBalancer”，作为外部负载均衡器使用，在一定数量的 Pod 之间均衡流量。

#### Replication Controller

Replication Controller（简单理解为上文提到的模具）确保任意时间都有指定数量的 Pod“副本”在运行。如果为某个 Pod 创建了 Replication Controller 并且指定 3 个副本，它会创建 3 个 Pod，并且**持续监控**它们。如果某个 Pod 不响应，那么 Replication Controller 会替换它，保持总数永远为 3。
如果之前不响应的 Pod 恢复了，Replication Controller 会将其中一个终止保持总数为 3。如果在运行中将副本总数改为 5，Replication Controller会立刻启动 2 个新 Pod，保证总数为 5。还可以按照这样的方式缩小 Pod。在节点挂掉重启时，可以自动创建 Pod 及相关应用服务。
创建 Replication Controller 需要的条件：
Pod 模板（模具）：用来创建 Pod 副本的模板
Label：Replication Controller 需要监控的 Pod 的标签。

#### ReplicaSet（RS)
Replication Controller（RC）的升级版本。ReplicaSet 和  Replication Controller 之间的唯一区别是对选择器的支持。ReplicaSet 支持 labels user guide 中描述的 set-based 选择器要求， 而 Replication Controller 仅支持 equality-based 的选择器要求。

#### Deployment
Deployment 为 Pod 和 Replica Set 提供声明式更新。
你只需要在 Deployment 中描述您想要的目标状态是什么，Deployment controller 就会帮您将 Pod 和 ReplicaSet 的实际状态改变到您的目标状态。您可以定义一个全新的 Deployment 来创建 ReplicaSet 或者删除已有的 Deployment 并创建一个新的来替换。
> 注意：一般情况下不该手动管理由 Deployment 创建的 Replica Set，否则就篡越了 Deployment controller 的职责！

#### kubectl
kubectl 用于运行 Kubernetes 集群命令的管理工具。
eg：
* kubectl run nginx --image=nginx
[命令参考手册](http://docs.kubernetes.org.cn/683.html)

#### ConfigMap 与 Secret

ConfigMap 与 Secret 用于保存配置数据的键值对，可以用来保存单个属性，也可以用来保存配置文件。ConfigMap 跟 Secret 很类似，但它可以更方便地处理不包含敏感信息的字符串。

## 参考文档

[中文文档](https://www.kubernetes.org.cn/docs)
[体验环境](https://kubernetes.io/docs/tutorials/hello-minikube)

