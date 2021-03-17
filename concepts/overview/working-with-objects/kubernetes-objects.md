# 理解k8s对象

本文介绍了如何在Kubernetes API中表示Kubernetes对象，以及如何以.yaml格式表示它们。

## 理解k8s对象

Kubernetes对象是Kubernetes系统中的持久实体。 Kubernetes使用这些实体来表示集群的状态。 具体来说，它们可以描述：
* 哪些容器化的应用程序正在运行（以及在哪些节点上）
* 这些应用程序可用的资源
* 有关这些应用程序的行为的策略，例如重新启动策略，升级和容错

Kubernetes对象是“意图记录”-创建对象后，Kubernetes系统将不断工作以确保该对象存在。 通过创建一个对象，您可以有效地告诉Kubernetes系统您希望集群的工作负荷是什么样子。 这是您的集群的期望状态。

要使用Kubernetes对象-无论是创建，修改还是删除它们-您都需要使用Kubernetes API。 例如，当您使用kubectl命令行界面时，CLI会为您进行必要的Kubernetes API调用。 您还可以使用客户端库之一在自己的程序中直接使用Kubernetes API。

## 对象特性和状态

几乎每个Kubernetes对象都包含两个嵌套的对象字段，这些字段控制对象的配置：对象规范和对象状态。 对于具有规格的对象，必须在创建对象时进行设置，并提供所需资源的特征描述：所需状态。

状态描述了对象的当前状态，该状态由Kubernetes系统及其组件提供和更新。 Kubernetes控制平面连续不断地主动管理每个对象的实际状态，以匹配您提供的所需状态。

例如：在Kubernetes中，Deployment是一个对象，可以表示您在集群上运行的应用程序。 创建展开时，可以将展开规范设置为指定要运行该应用程序的三个副本。 Kubernetes系统读取Deployment规范并启动所需应用程序的三个实例-更新状态以符合您的规范。 如果这些实例中的任何一个都应该失败（状态更改），则Kubernetes系统将通过进行更正来响应规范和状态之间的差异-在这种情况下，将启动替换实例。

有关对象规范，状态和元数据的更多信息，请参见Kubernetes API约定。

## 描述kubernetes对象

在Kubernetes中创建对象时，必须提供描述其所需状态的对象规范以及有关该对象的一些基本信息（例如名称）。 当您使用Kubernetes API创建对象（直接或通过kubectl）时，该API请求必须在请求正文中包含该信息作为JSON。 通常，您会在.yaml文件中将信息提供给kubectl。 发出API请求时，kubectl会将信息转换为JSON。

这是一个示例.yaml文件，显示了Kubernetes部署的必填字段和对象规范：

```code
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

使用.yaml文件创建展开的一种方法是使用上述方法，是在kubectl命令行界面中使用kubectl apply命令，并将.yaml文件作为参数传递。 这是一个例子：

```code
kubectl apply -f https://k8s.io/examples/application/deployment.yaml --record
```

输出类似如下：
```code
deployment.apps/nginx-deployment created
```

## 必备字段

在您要创建的Kubernetes对象的.yaml文件中，您需要为以下字段设置值：

* `apiVersion`：您正在使用哪个版本的Kubernetes API创建该对象
* `kind`：你想要创建那种类型的对象
* `metadata`：有助于唯一表示对象的数据，例如`name`,`UID`,`namespace`
* `spec`：你定义的对象期望状态

每个Kubernetes对象的对象规范的精确格式都不同，并且包含特定于该对象的嵌套字段。 Kubernetes API参考可以帮助您找到可以使用Kubernetes创建的所有对象的规范格式。 例如，可以在PodSpec v1核心中找到Pod的规范格式，而可以在DeploymentSpec v1应用中找到Deployment的规范格式。

