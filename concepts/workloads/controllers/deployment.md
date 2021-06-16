# 部署器

您在 Deployment 中描述了所需的状态，并且 Deployment Controller 以受控的速率将实际状态更改为所需状态。 您可以定义 Deployment 来创建新的 ReplicaSet，或者删除现有的 Deployment 并通过新的 Deployment 来采用它们的所有资源。
注意：不要管理 Deployment 拥有的 ReplicaSet。 如果下面未涵盖您的用例，请考虑在主 Kubernetes仓库中提问。

## 使用案例

以下是一个使用deployment的典型案例：
* 创建部署以推出副本集。 ReplicaSet 在后台创建 Pod。 检查 rollout 的状态以查看它是否成功。
* 通过更新部署的 PodTemplateSpec 来声明 Pod 的新状态。 一个新的 ReplicaSet 被创建，Deployment 管理以受控速率将 Pod 从旧 ReplicaSet 移动到新 ReplicaSet。 每个新的 ReplicaSet 都会更新 Deployment 的修订版。
* 如果 Deployment 的当前状态不稳定，则回滚到较早的 Deployment 版本。 每次回滚都会更新部署的修订版。
* 扩大部署以促进更多负载。
* 暂停部署以对其 PodTemplateSpec 应用多个修复程序，然后恢复它以开始新的部署。
* 使用部署的状态作为部署已卡住的指示器。
* 清理不再需要的旧 ReplicaSet。

## 创建一个deployment

以下是一个部署示例。 它创建了一个 ReplicaSet 来启动三个 nginx Pod：
```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
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

在这个示例中：
* 一个叫做`nginx-deployment`的Deployment创建，通过`.metadata.name`字段指定
* Deployment创建了3个副本Pods，通过`.spec.replicas`字段声明
* 字段`.spec.selector`字段声明了Deployment如何查找管理哪个Pods。在此例中，你可以选择一个定义得Pod模版中的标签，`app: nginx`。但是，更复杂的选择规则也是可能的，只要Pod模版自身满足规则。
    注意： .spec.selector.matchLabels 字段是 {key,value} 对的映射。 matchLabels 映射中的单个 {key,value} 相当于 matchExpressions 的一个元素，其键字段为“key”，运算符为“In”，values 数组仅包含“value”。 必须满足 matchLabels 和 matchExpressions 的所有要求才能匹配。
* `template`字段包含以下子字段：
    * Pods标签`app: nginx`使用`.metadata.labels`字段
    * pod模板定义，或者`.template.spec`字段，定义了pod运行一个容器，这就是运行了Docker Hub 镜像nginx，版本号1.14.2的容器
    * 创建一个容器，并命名为nginx，使用`.spec.template.spec.containers[0].name`字段

在开始之前，确认集群启动并正常运行。以下步骤给出了创建以上Deployment：
1. 通过运行一些命令创建Deployment
```shell

kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml

```
注意：您可以指定 --record 标志写入资源注解 kubernetes.io/change-cause 中执行的命令。 记录的更改对于将来的自省很有用。 例如，查看在每个部署版本中执行的命令。

2. 运行`kubectl get deployments` 检车Deployment被创建







