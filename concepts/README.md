# 内容详解

本章节帮助你了解Kubernetes系统的各个部分以及Kubernetes用于表示集群的抽象，并有助于您更深入地了解Kubernetes的工作原理。


### [集群概览](overview/README.md) 

了解Kubernetes及其构建组件的高级概述。

### [集群架构](architecture/README.md)

Kubernetes背后的架构概念。

### [容器详解](containers/README.md)

用于打包应用程序及其运行时依赖项的技术。

### [工作负载](workloads/README.md)

了解Pod，这是Kubernetes中最小的可部署计算对象，以及有助于您运行它们的更高级抽象。

### [代理、负载均衡与网络](services-networking/README.md)

Kubernetes中网络背后的概念和资源。

### [后端存储](storage/README.md)

几种为你集群中的pod提供长期和临时的存储的方式

### [配置定义](configuration/README.md)

kubernetes为pod提供配置文件资源

### [安全策略](security/README.md)

保证云原声负载安全的概念

### [资源策略](policy/README.md)

您可以配置的策略适用于资源组。

### [调度驱逐](scheduling-eviction/README.md)

在Kubernetes中，调度是指确保Pod与Node匹配，以便kubelet可以运行它们。 驱逐是在资源匮乏的节点上主动使一个或多个Pod发生故障的过程。

### [集群管理](cluster-administration.md)

与创建或管理Kubernetes集群有关的低级详细信息。

### [功能扩展](extend-kubernetes.md)

多种方式改变集群行为