# 集群概览

[什么时k8s](concepts/overview/what-is-kubernetes.md)

k8s是一个便携的，可扩展的，以及开源的平台，用来管理容器化的工作负载和服务，该基础设施兼具声明式配置和自动化特性。k8s是一个大型的，快速成长的生态系统。使用k8s服务，支持以及工具，具备广泛的可行性。

[k8s组件](concepts/overview/components.md)

k8s集群由代表控制平面的组件以及机器集合组成的node节点组件组成。

[k8s api](concepts/overview/kubernetes-api.md)

k8s api允许你查询和管理集群对象的状态。k8s核心的控制平面包含由api server以及http api暴露出来的接口。用户之间，集群中的不同部分，以及外部组件之间的所有通信都是通过api server完成。

[理解k8s对象](concepts/overview/working-with-objects/README.md)

k8s对象是k8s系统中的持久化内容。k8s使用这些内容代表集群。学习k8s对象模型，以及如何使用这些对象。