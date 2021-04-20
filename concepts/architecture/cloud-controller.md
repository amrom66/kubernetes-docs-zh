# 云控制器管理器

云基础架构基数允许您在公有云，私有云以及混合云上运行kubernetes。kubernetes坚持自动化API驱动，而不是组件间的紧耦合。云控制器管理是一个kubernetes控制平面组件，嵌入到特定云的控制逻辑。云控制器允许您链接集群到云服务提供商的API，以及分离出那些仅仅与集群交互的组件。通过解偶kubernete和底层云基础设施，云管理控制器允许云服务提供商发布不同于kubernetes项目的特性。

云控制器管理器使用插件机制构建，该机制允许不同的云提供商将其平台与Kubernetes集成。

## 设计

![设计](https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg)

云控制器管理器作为一组重复的进程（通常是Pod中的容器）在控制平面中运行。 每个云控制器管理器在单个过程中实现多个控制器。

注意：您也可以将Kubernetes插件作为云控制器管理器而不是作为控制平面的一部分来运行。


## 云控制管理器功能

云控制管理器包含以下功能：

### 节点控制器

在云基础架构中创建新服务器时，节点控制器负责创建Node对象。 节点控制器获取有关在租户内部通过云提供商运行的主机的信息。 节点控制器执行以下功能：

1. 为控制器通过云提供程序API发现的每个服务器初始化一个Node对象。
2. 使用特定于云的信息来注释和标记Node对象，例如，将节点部署到的区域以及其可用资源（CPU，内存等）。
3. 获取节点的主机名和网卡地址
4. 验证节点的运行状况。 万一节点无响应，此控制器将检查您的云提供商的API，以查看服务器是否已被停用/删除/终止。 如果该节点已从云中删除，则控制器将从您的Kubernetes集群中删除Node对象。

一些云提供商的实现将其分为节点控制器和单独的节点生命周期控制器。

### 路由控制器

路由控制器负责适当地在云中配置路由，以便Kubernetes集群中不同节点上的容器可以相互通信。

取决于云提供商，路由控制器可能还会为Pod网络分配IP地址块。

### 代理控制器

服务与云基础架构组件集成，例如托管负载平衡器，IP地址，网络数据包筛选和目标运行状况检查。 当声明需要它们的服务资源时，服务控制器会与您的云提供商的API交互以设置负载平衡器和其他基础结构组件。

## 鉴权

本节将分解云控制器管理器对各种API对象进行执行操作所需的访问权限。

### 节点控制器

Node控制器仅适用于Node对象。 它需要完全访问权限才能读取和修改Node对象。

v1/Node:

* Get
* List
* Create
* Update
* Patch
* Watch
* Delete

### 路由控制器

路由控制器侦听Node对象的创建并适当地配置路由。 它需要获得对Node对象的访问权限。

v1/Node:

* Get

### 代理控制器

服务控制器侦听服务对象的Create，Update和Delete事件，然后为这些服务适当配置端点。

要访问服务，需要“列表”和“监视”访问权限。 要更新服务，它需要Patch 和 Update访问权限。

要为服务设置端点资源，它需要访问“创建”，“列表”，“获取”，“监视”和“更新”。

v1/Service:

* List
* Get
* Watch
* Patch
* Update

### 其他

云控制器管理器核心的实现需要访问权限以创建事件对象，并且为了确保安全操作，还需要访问权限以创建ServiceAccounts。

v1/Event:

* Create
* Patch
* Update

v1/ServiceAccount:

* Create

用于云控制器管理器的RBAC ClusterRole如下所示：
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cloud-controller-manager
rules:
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
  - update
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - '*'
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - list
  - patch
  - update
  - watch
- apiGroups:
  - ""
  resources:
  - serviceaccounts
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - persistentvolumes
  verbs:
  - get
  - list
  - update
  - watch
- apiGroups:
  - ""
  resources:
  - endpoints
  verbs:
  - create
  - get
  - list
  - watch
  - update
```


## 下一步

[Cloud Controller Manager管理](https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/#cloud-controller-manager) 提供了有关运行和管理Cloud Controller Manager的说明。