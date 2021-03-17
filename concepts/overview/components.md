# k8s组件

当你部署kubernetes的时候，你得到的是一个集群。

一个kubernetes集群由一系列叫做工作机器的节点组成，在这些节点上运行容器化的应用。每个集群拥有至少一个工作节点。

工作节点托管pod，这些pod是应用程序工作负载的组成部分。控制平面负责管理工作节点，以及集群中的pod。生产环境下，控制平面经常运行在多个机器上，同时一个集群通常运行多个节点，这提供了容错性与高可用。

本文档概述了拥有完整且有效的Kubernetes集群所需的各种组件。
以下是Kubernetes集群的示意图，其中所有组件都捆绑在一起。
![架构图](https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg)

## 控制平面组件

控制平面的组件对集群作出全局性的决策，例如调度，同时可以监测并相应集群事件；例如，当deployment的副本数量不满足的时候，启动一个新的pod。

控制平面的组件可以运行在集群中的任一台机器上。但是，为简单起见，设置脚本通常在同一台计算机上启动所有控制平面组件，并且不在该计算机上运行用户容器。有关多主虚拟机设置示例，请参阅[构建高可用性群集](https://kubernetes.io/docs/admin/high-availability/)。

### apiserver

api server是Kubernetes控制平面的组件，该组件公开Kubernetes API。 api server是Kubernetes控制平面的前端。

Kubernetes API server的主要实现是kube-apiserver。 kube-apiserver旨在水平扩展-即，它通过部署更多实例进行扩展。 您可以运行kube-apiserver的多个实例，并平衡这些实例之间的流量。

### etcd

一致且高度可用的键值存储用作所有集群数据的Kubernetes的后备存储。

如果您的Kubernetes集群使用etcd作为其后备存储，请确保您有针对这些数据的备份计划。

您可以在官方文档中找到有关etcd的详细信息。

### kube-scheduler

控制平面组件，该组件监视没有分配节点的新创建的Pod，并选择一个节点以在其上运行。

计划决策要考虑的因素包括：个体和集体资源需求，硬件/软件/策略约束，亲和力和反亲和力规范，数据局部性，工作负载之间的干扰以及期限。

### kube-controller-manager 

运行控制器进程的控制平面组件。

从逻辑上讲，每个控制器是一个单独的进程，但是为了降低复杂性，它们都被编译为单个二进制文件并在单个进程中运行。

这些控制器的类型包含：

节点控制器：负责在节点出现故障时进行通知和响应。
作业控制器：监视代表一次性任务的作业对象，然后创建Pod以运行这些任务以完成任务。
端点控制器：填充“端点”对象（即，加入“服务和窗格”）。
服务帐户和令牌控制器：为新的名称空间创建默认帐户和API访问令牌。

### cloud-controller-manager

嵌入了特定于云的控制逻辑的Kubernetes控制平面组件。 云控制器管理器使您可以将集群链接到云提供商的API，并将与该云平台交互的组件与仅与集群交互的组件分开。
cloud-controller-manager仅运行特定于您的云提供商的控制器。 如果您是在自己的场所或PC内的学习环境中运行Kubernetes，则该群集没有云控制器管理器。

与kube-controller-manager一样，cloud-controller-manager将多个逻辑上独立的控制循环组合为一个二进制文件，您可以将其作为单个进程运行。 您可以水平缩放（运行多个副本）以提高性能或帮助容忍故障。

以下控制器可以具有云提供程序依赖性：

节点控制器：用于检查云提供程序以确定节点停止响应后是否已在云中删除该节点
路由控制器：用于在基础云基础架构中设置路由
服务控制器：用于创建，更新和删除云提供商负载平衡器


## 工作节点组件

节点组件在每个节点上运行，维护运行中的Pod，并提供Kubernetes运行时环境。

### kubelet

在集群中每个节点上运行的代理。 确保容器在Pod中运行。

kubelet包含通过各种机制提供的一组PodSpec，并确保这些PodSpec中描述的容器正在运行且状况良好。 Kubelet不管理不是Kubernetes创建的容器。

### kube-proxy

kube-proxy是一个网络代理，它在集群中的每个节点上运行，实现了Kubernetes Service概念的一部分。

kube-proxy维护节点上的网络规则。 这些网络规则允许从群集内部或外部的网络会话与Pod进行网络通信。

如果有kube-proxy可用，它将使用操作系统数据包过滤层。 否则，kube-proxy会转发流量本身。


### Container runtime 

`Container runtime`是负责运行容器的软件。

Kubernetes支持几种容器运行时：Docker，容器化，CRI-O以及Kubernetes CRI（容器运行时接口）的任何实现。

## 插件

插件使用Kubernetes资源（DaemonSet，Deployment等）来实现集群功能。 由于这些功能提供集群级功能，因此插件的命名空间资源属于kube-system命名空间。

所选的插件说明如下： 有关可用插件的扩展列表，请参阅[插件](https://kubernetes.io/docs/concepts/cluster-administration/addons/)。


## DNS

尽管并非严格要求其他附加组件，但由于许多示例都依赖于此，因此所有Kubernetes群集都应具有群集DNS。

除了您环境中的其他DNS服务器之外，群集DNS还是一个DNS服务器，它为Kubernetes服务提供DNS记录。

由Kubernetes启动的容器会在其DNS搜索中自动包括此DNS服务器。

## 仪表盘

仪表板是Kubernetes集群的基于Web的通用UI。 它允许用户管理集群中运行的应用程序以及集群本身并进行故障排除。

## 容器资源监控

容器资源监视在中央数据库中记录有关容器的一般时间序列指标，并提供用于浏览该数据的UI。

## 集群级别的日志

集群级别的日志记录机制负责通过搜索/浏览界面将容器日志保存到中央日志存储中。