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

kubernetes有4个初始化名称空间：
* `default` 没有其他名称空间的对象的默认名称空间
* `kube-system` Kubernetes系统创建的对象的名称空间
* `kube-public` 该名称空间是自动创建的，并且对所有用户（包括未经身份验证的用户）可读。 此名称空间主要保留给集群使用，以防某些资源在整个集群中公开可见。 此名称空间的公共方面仅是约定，不是要求。
* `kube-node-release` 与每个节点关联的租用对象的此名称空间，可在群集扩展时提高节点心跳的性能。

### 设置请求的名称空间

为现有的请求设置名称空间， 请使用`--namespace`标签。

例如：
```code
kubectl run nginx --image=nginx --namespace=<insert-namespace-name-here>
kubectl get pods --namespace=<insert-namespace-name-here>
```

### 设置默认名称空间

您可以为该上下文中的所有后续kubectl命令永久保存名称空间：
```code
kubectl config set-context --current --namespace=<insert-namespace-name-here>
# Validate it
kubectl config view --minify | grep namespace:
```


## 名称空间与DNS

创建服务时，它会创建一个相应的DNS条目。 此项的格式为<service-name>。<namespace-name> .svc.cluster.local，这意味着如果容器仅使用<service-name>，它将解析为名称空间本地的服务 。 这对于在多个名称空间（例如开发，登台和生产）中使用相同的配置很有用。 如果要跨命名空间访问，则需要使用完全限定的域名（FQDN）。


## 并不是所有的对象都属于名称空间

大多数Kubernetes资源（例如Pod，服务，复制控制器和其他资源）都位于某些命名空间中。 但是，名称空间资源本身并不在名称空间中。 而且低级资源（例如节点和persistentVolumes）不在任何命名空间中。

要查看哪些Kubernetes资源在命名空间中以及不在命名空间中：

```code
# In a namespace
kubectl api-resources --namespaced=true

# Not in a namespace
kubectl api-resources --namespaced=false
```




