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

2. 运行`kubectl get deployments` 检查Deployment被创建

如果，Deployment仍然在创建中，则会输出如下：
```code
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   0/3     0            0           1s
```
当你检查集群中的Deployments的时候，以下的字段会被显示：
* `NAME`列出了名称空间中的Deployments
* `READY`显示了用户可用的应用副本数量。它使用以下表达式：`ready/desired`
* `UP-TO-DATE`展示为了达到需求的状态，有多少副本发生了更新
* `AVAILABLE`展示了当前用户可用的应用副本数量
* `AGE`表示应用运行的时间数值
注意需求的副本数量为3，是根绝`.spec.replicas`字段确定的。

3. 为了查看Deploymen推出状态，运行`kubectl rollout status deployment/nginx-deployment`.
输出如下：
```code
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment "nginx-deployment" successfully rolled out
```

4. 再次运行`kubectl get deployments`，输出如下：
```code
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           18s
```
注意到Deployment创建了3个副本，并且3个副本都处于最新状态，并且可用。

5. 想要查看Deployment创建的ReplicaSet(rs)，可以运行`kubectl get rs`。输出如下：
```code
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-75675f5897   3         3         3       18s
```

ReplicaSet的输出显示了以下字段：
* `NAME`列出名称空间中的所有ReplicaSet名称
* `DESIRED`显示了应用的副本数需求的数值，这个值在你创建Deployment的时候定义。这就是需求状态
* `CURRENT`显示当前多少副本运行中
* `READY`显示了当前用户可用的应用副本数
* `AGW`显示了当前应用运行的时长

请注意，ReplicaSet 的名称始终格式为 [DEPLOYMENT-NAME]-[RANDOM-STRING]。 随机字符串是随机生成的，并使用 pod-template-hash 作为种子。

6. 要查看为每个 Pod 自动生成的标签，请运行`kubectl get pods --show-labels`。 输出类似于：

```code
NAME                                READY     STATUS    RESTARTS   AGE       LABELS
nginx-deployment-75675f5897-7ci7o   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
nginx-deployment-75675f5897-kzszj   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
nginx-deployment-75675f5897-qqcnn   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
```

注意：
您必须在部署中指定适当的选择器和 Pod 模板标签（在本例中为 app: nginx）。
不要将标签或选择器与其他控制器（包括其他部署和 StatefulSet）重叠。 Kubernetes 不会阻止您重叠，如果多个控制器具有重叠的选择器，这些控制器可能会发生冲突并出现意外行为。

### Pod模板哈希标签

**注意**：不要改变该标签

`pod-template-hash`标签由`Deployment`控制器添加到`Deployment`创建或采用的每个`ReplicaSet`中。
此标签可确保 Deployment 的子 ReplicaSet 不会重叠。 它是通过对 ReplicaSet 的 PodTemplate 进行散列并将结果散列用作添加到 ReplicaSet 选择器、Pod 模板标签以及 ReplicaSet 可能具有的任何现有 Pod 中的标签值来生成的。

## 更新一个Deployment

注意：当且仅当 Deployment 的 Pod 模板（即 .spec.template）发生更改时，才会触发 Deployment 的 rollout，例如，如果模板的标签或容器镜像被更新。 其他更新，例如扩展部署，不会触发推出。

按照下面给出的步骤更新您的部署：

1. 让我们更新nginx的pods以使用`nginx:1.16.1`镜像来代替`nginx:1.14.2`

```shell
kubectl --record deployment.apps/nginx-deployment set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1
```

或者运行以下命令：

```shell
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1 --record
```

输出类似于：
```shell
deployment.apps/nginx-deployment image updated
```

同样，你也可以编辑Deployment并且修改`.spec.template.spec.containers[0].image`，从`nginx:1.14.2`到`nginx:1.16.1`:

```code
kubectl edit deployment.v1.apps/nginx-deployment
```

输出类似于：
```code
deployment.apps/nginx-deployment edited
```

2. 查看rollout状态，运行：

```code
kubectl rollout status deployment/nginx-deployment
```

输出类似：
```code
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
```
或者
```code
deployment "nginx-deployment" successfully rolled out
```

获取你更新的Deployment的更多细节：
* rollout成功后，你可以使用`kubectl get deployments`查看Deployment，输出如下：
```code
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           36s
```
* 运行`kubectl get rs`查看Deployment通过创建一个新的ReplicaSet并且扩展到3个副本更新的Pods，同时还有缩小旧的ReplicaSet到0个副本的过程。
```code
kubectl get rs
```
输出类似如下：
```code
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1564180365   3         3         3       6s
nginx-deployment-2035384211   0         0         0       36s
```
* 运行`get pods`现在应该能够展示新的pods
```code
kubectl get pods
```
输出类似如下：
```code
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-1564180365-khku8   1/1       Running   0          14s
nginx-deployment-1564180365-nacti   1/1       Running   0          14s
nginx-deployment-1564180365-z9gth   1/1       Running   0          14s
```

下次想要更新这些pods，你只需要更新Deployment的Pod模版即可。
Deployment保证了更新的过程中，只有一定数量的Pods处于下线状态。默认场景下，它确保至少75%的需求pod数量正常运行。





