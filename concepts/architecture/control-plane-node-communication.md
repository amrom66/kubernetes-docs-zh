# 控制平面与节点通信

本文档对控制平面（apiserver）与Kubernetes集群之间的通信路径进行了分类。 目的是允许用户自定义其安装以强化网络配置，以便可以在不受信任的网络上（或在云提供商的完全公共IP上）运行集群。

## 节点到控制平面

Kubernetes具有“中心辐射型” API模式。 来自节点（或它们运行的Pod）的所有API使用都在apiserver处终止。 其他控制平面组件均未设计为公开远程服务。 apiserver配置为侦听启用了一种或多种形式的客户端身份验证的安全HTTPS端口（通常为443）上的远程连接。 应该启用一种或多种形式的授权，尤其是在允许匿名请求或服务帐户令牌的情况下。

应该为节点配置群集的公共根证书，以便它们可以与有效的客户端凭据一起安全地连接到apiserver。 一个好的方法是提供给kubelet的客户端凭据采用客户端证书的形式。 有关自动配置kubelet客户端证书的信息，请参阅kubelet TLS引导。

希望连接到apiserver的Pod可以通过利用服务帐户来安全地这样做，以便Kubernetes在实例化Pod时会自动将公共根证书和有效的承载令牌注入Pod。 kubernetes服务（默认名称空间）配置有虚拟IP地址，该地址被重定向（通过kube-proxy）到apiserver上的HTTPS端点。

控制平面组件还通过安全端口与群集apiserver通信。

结果，从节点和在节点上运行的Pod到控制平面的连接的默认操作模式在默认情况下是安全的，并且可以在不受信任和/或公共网络上运行。

## 控制平面到节点

从控制平面（apiserver）到节点有两条主要的通信路径。 第一个是从apiserver到在集群中每个节点上运行的kubelet进程。 第二个是通过apiserver的代理功能从apiserver到任何节点，pod或服务。

### apiserver到kubelet

从apiserver到kubelet的连接用于：

* 正在获取pod的日志。
* 附加到运行中的pod
* 提供kubelet的端口转发功能。

这些连接在kubelet的HTTPS端点处终止。 默认情况下，apiserver不会验证kubelet的服务证书，这会使该连接遭受中间人攻击，并且无法在不受信任和/或公共网络上运行。

要验证此连接，请使用--kubelet-certificate-authority标志为apiserver提供一个根证书捆绑包，以用于验证kubelet的服务证书。

如果无法做到这一点，请在apiserver和kubelet之间使用SSH隧道（如果需要），以避免通过不可信或公共网络进行连接。

最后，应该启用Kubelet身份验证和/或授权以保护kubelet API。

### apiserver 到节点, pods, 以及services

从apiserver到节点，pod或服务的连接默认为纯HTTP连接，因此未经身份验证或加密。 可以通过在API URL中的节点，pod或服务名称前添加https：来在安全的HTTPS连接上运行它们，但是它们不会验证HTTPS终结点提供的证书，也不会提供客户端凭据。 因此，尽管连接将被加密，但它不会提供任何完整性保证。 这些连接当前在通过不可信或公共网络运行时并不安全。

## SSH 隧道

Kubernetes支持SSH隧道以保护控制平面到节点的通信路径。 在此配置中，apiserver启动到群集中每个节点的SSH隧道（连接到侦听端口22的ssh服务器），并通过隧道传递发往kubelet，节点，pod或服务的所有流量。 该隧道确保流量不会暴露在运行节点的网络外部。

SSH隧道目前已被弃用，因此除非您知道自己在做什么，否则不应该选择使用它们。 Konnectivity服务是此通信渠道的替代产品。

## Konnectivity 服务

作为SSH隧道的替代，Konnectivity服务为控制平面提供群集通信的TCP级别代理。 Konnectivity服务由两部分组成：控制平面网络中的Konnectivity服务器和节点网络中的Konnectivity代理。 Konnectivity代理启动与Konnectivity服务器的连接并维护网络连接。 启用Konnectivity服务后，所有控制平面到节点的流量都将通过这些连接进行。

访问Konnectivity服务任务以在您的集群中设置Konnectivity服务。

