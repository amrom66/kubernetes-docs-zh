# 名称空间

Kubernetes支持由同一物理群集支持的多个虚拟群集。 这些虚拟集群称为名称空间。

## 何时使用多个名称空间

命名空间旨在用于具有多个用户的环境，这些用户分布在多个团队或项目中。 对于拥有几到几十个用户的集群，您根本不需要创建或考虑名称空间。 当需要名称空间提供的功能时，请开始使用它们。

命名空间为名称提供了作用域。 资源名称在名称空间中必须唯一，但在名称空间之间则必须唯一。 命名空间不能彼此嵌套，并且每个Kubernetes资源只能位于一个命名空间中。

命名空间是一种在多个用户之间（通过资源配额）划分群集资源的方法。

不必仅使用多个名称空间来分隔稍有不同的资源，例如同一软件的不同版本：使用标签来区分同一名称空间中的资源。


## 开始使用名称空间

[名称空间管理指南](https://kubernetes.io/docs/tasks/administer-cluster/namespaces)中描述了名称空间的创建和删除。

注意：避免使用前缀kube-创建名称空间，因为它是为Kubernetes系统名称空间保留的。


## 查看

通过以下指令获取集群中的所有namespaces：

```code
kubectl get namespace
```

```code
NAME              STATUS   AGE
default           Active   1d
kube-node-lease   Active   1d
kube-public       Active   1d
kube-system       Active   1d
```



