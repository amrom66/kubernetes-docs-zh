# 代理、负载均衡与网络

## kubernetes网络模型

每一个Pod都有自己的IP地址。这意味着你不需要明确的去创建Pod与Pod之间的链接，并且你几乎从来不需要处理容器端口与宿主机端口的路由关系。从端口分配，服务发现，负载均衡，应用配置与迁移方面来看，这会创建一个干净的，向后兼容的模型，该模型下Pod更多的被当作虚拟机或者物理机。

kubernetes将下属基础性的要求强加到任何网络实现上：

* 各节点的pod可以不通过NAT互相访问
* 节点上的代理（例如kubelet），可以与该节点的所有pod通信

**注意** 对于那些使用宿主机网络模式运行pod的平台（例如linux）：

* 以宿主机网络模式运行的pod可以不借助NAT访问所有节点的所有pod

这个模型不仅整体上减少复杂度，而且特别适合kubernetes开启从虚拟机到容器的低摩擦应用端口转发需求。如果你的任务以前工作在虚拟机上，你的虚拟机会有一个IP并且可以和项目中的其他虚拟机通信。这是一套相同的基础模型。

kubernetes的IP地址存在于pod领域-pod中的容器共享网络名称空间,包扩他们的IP地址和MAC地址。这意味着同一pod中的容器可以通过localhost访问其他容器的端口，同时意味着同一pod中的容器共享端口使用，并且于虚拟机中的进程毫无区别。这种模式被叫做`IP-per-pod`模式。

这些是如何实现的，详细在于使用的容器运行时。

可行的方案时通过申请节点自身的端口，转发到pod（叫做宿主机模式），但是这是一个非常小众的做法。转发的实现方式同样是容器运行时的细节实现。pod自身对宿主机的端口是无感知的。

kubernetes网络有4处核心点：

* 同一个pod中的容器使用回环地址通信
* 集群网络为不同pod提供通信
* 代理资源允许你暴露pod中的应用，使得可以从集群外访问到
* 你也同样可以使用代理资源仅在集群内部公开服务

[代理](service.md)
[代理拓扑](service-topology.md)
[代理和Pod的DNS](dns-pod-service.md)
[通过代理访问应用](connect-applications-service.md)
[入口](ingress.md)
[入口控制器](ingress-controllers.md)
[端点切片](endpoint-slices.md)
[内部服务流量策略](service-traffic-policy.md)
[拓扑感知提示](topology-aware-hints.md)
[网络策略](network-policies.md)
[IPv4/Ipv6双栈](dual-stack.md)
