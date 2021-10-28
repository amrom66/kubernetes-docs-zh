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
Deployment同时保证了只会创建出来比需求的数量高出一定数量的pods。默认场景下，它确保最多pod数量为需求的125%。
举例来说，如果你仔细观察以上的Deployment，你会发现他会先创建一个新的Pod，然后删除旧的Pod，再创建新的Pod。直到充足的新pod启动，它才会杀死旧的pod；直到充足的旧pod被杀死，它才会创建新的pod。它保证了至少有2个pod可用，并且最多有4个pod可用。
* 查看Deployment的详细
```code
kubectl describe deployments
```
输出如下：
```code
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Thu, 30 Nov 2017 10:56:25 +0000
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision=2
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
   Containers:
    nginx:
      Image:        nginx:1.16.1
      Port:         80/TCP
      Environment:  <none>
      Mounts:       <none>
    Volumes:        <none>
  Conditions:
    Type           Status  Reason
    ----           ------  ------
    Available      True    MinimumReplicasAvailable
    Progressing    True    NewReplicaSetAvailable
  OldReplicaSets:  <none>
  NewReplicaSet:   nginx-deployment-1564180365 (3/3 replicas created)
  Events:
    Type    Reason             Age   From                   Message
    ----    ------             ----  ----                   -------
    Normal  ScalingReplicaSet  2m    deployment-controller  Scaled up replica set nginx-deployment-2035384211 to 3
    Normal  ScalingReplicaSet  24s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 1
    Normal  ScalingReplicaSet  22s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 2
    Normal  ScalingReplicaSet  22s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 2
    Normal  ScalingReplicaSet  19s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 1
    Normal  ScalingReplicaSet  19s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 3
    Normal  ScalingReplicaSet  14s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 0
```
此时，你可以看出：当你首次创建Deployment的时候，它会创建一个ReplicaSet（nginx-deployment-2035384211）,并且直接将副本数量扩展到3个。当你更新Deployment的时候，它会创建一个新的ReplicaSet（nginx-deployment-1564180365），并且数量扩展到1，然后缩小旧的ReplicaSet到2，所以整个过程最少有2个pod，最多有4个pods。随后，它会持续的根据滚动更新策略扩大新的ReplicaSet数量，以及缩小旧的ReplicaSet数量。最终，你会发现2个可用的副本在新的ReplicaSet中，并且旧的ReplicaSet缩小为0.

回滚（又名飞行中的多次更新）

每次 Deployment 控制器观察到一个新的 Deployment 时，都会创建一个 ReplicaSet 来启动所需的 Pod。 如果 Deployment 被更新，控制标签匹配 .spec.selector 但模板不匹配 .spec.template 的现有 ReplicaSet 将被缩减。 最终，新的 ReplicaSet 被缩放到 .spec.replicas，所有旧的 ReplicaSet 被缩放到 0。

如果在现有部署正在进行时更新部署，部署会根据更新创建一个新的 ReplicaSet 并开始向上扩展，并滚动之前扩展的 ReplicaSet——它将添加到它的列表中 旧的 ReplicaSets 并开始缩小它。

例如，假设您创建了一个部署来创建 nginx:1.14.2 的 5 个副本，但随后更新部署以创建 nginx:1.16.1 的 5 个副本，而此时仅创建了 nginx:1.14.2 的 3 个副本。 在这种情况下，Deployment 立即开始杀死它创建的 3 个 nginx:1.14.2 Pod，并开始创建 nginx:1.16.1 Pod。 在更改任务之前，它不会等待 nginx:1.14.2 的 5 个副本被创建。

标签选择器更新

通常不鼓励更新标签选择器，建议预先计划您的选择器。 在任何情况下，如果您需要执行标签选择器更新，请务必小心并确保您已掌握所有含义。
注意：在 API 版本 apps/v1 中，Deployment 的标签选择器在创建后是不可变的。
* 选择器添加要求部署规范中的 Pod 模板标签也更新为新标签，否则将返回验证错误。 这种变化是非重叠的，意味着新的选择器不会选择使用旧选择器创建的 ReplicaSet 和 Pod，导致孤立所有旧的 ReplicaSet 并创建一个新的 ReplicaSet。
* 选择器更新会更改选择器键中的现有值——导致与添加相同的行为。
* 选择器删除从部署选择器中删除现有的键——不需要对 Pod 模板标签进行任何更改。 现有的 ReplicaSet 不是孤立的，也不会创建新的 ReplicaSet，但请注意，删除的标签仍然存在于任何现有的 Pod 和 ReplicaSet 中。

### 回滚一个Deployment

有时，您可能想要回滚部署； 例如，当 Deployment 不稳定时，例如崩溃循环。 默认情况下，所有 Deployment 的 rollout 历史都保存在系统中，以便您可以随时回滚（您可以通过修改修订历史限制来更改它）。

注意：当一个部署的推出被触发时，一个部署的修订被创建。 这意味着当且仅当 Deployment 的 Pod 模板 (.spec.template) 发生更改时，才会创建新修订，例如，如果您更新模板的标签或容器映像。 其他更新，例如扩展部署，不会创建部署修订，因此您可以促进同时手动或自动扩展。 这意味着当您回滚到较早的版本时，只会回滚部署的 Pod 模板部分。

* 假设您在更新 Deployment 时犯了一个错字，将镜像名称设为 nginx:1.161 而不是 nginx:1.16.1：
```code
kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.161 --record=true
```
输出如下：
```code
deployment.apps/nginx-deployment image updated
```

* 发布卡住了。 您可以通过检查推出状态来验证它：
```code
kubectl rollout status deployment/nginx-deployment
```
输出类似：
```code
Waiting for rollout to finish: 1 out of 3 new replicas have been updated...
```

* 按 Ctrl-C 停止上述 rollout 状态监视。 有关卡住部署的更多信息，请在此处阅读更多信息。

* 你看到旧副本的数量（nginx-deployment-1564180365和nginx-deployment-2035384211）是2，新副本（nginx-deployment-3066724191）是1。
```code
kubectl get rs
```
输出如下：
```code
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1564180365   3         3         3       25s
nginx-deployment-2035384211   0         0         0       36s
nginx-deployment-3066724191   1         1         0       6s
```
* 查看创建的 Pod，您会看到新 ReplicaSet 创建的 1 个 Pod 卡在镜像拉取循环中
```code
kubectl get pods
```
输出如下：
```code
NAME                                READY     STATUS             RESTARTS   AGE
nginx-deployment-1564180365-70iae   1/1       Running            0          25s
nginx-deployment-1564180365-jbqqo   1/1       Running            0          25s
nginx-deployment-1564180365-hysrc   1/1       Running            0          25s
nginx-deployment-3066724191-08mng   0/1       ImagePullBackOff   0          6s
```
注意：部署控制器会自动停止错误的部署，并停止扩展新的 ReplicaSet。 这取决于您指定的滚动更新参数（特别是 maxUnavailable）。 默认情况下，Kubernetes 将该值设置为 25%。

* 获取Deployment的描述
```code
kubectl describe deployment
```
输出如下：
```code
Name:           nginx-deployment
Namespace:      default
CreationTimestamp:  Tue, 15 Mar 2016 14:48:04 -0700
Labels:         app=nginx
Selector:       app=nginx
Replicas:       3 desired | 1 updated | 4 total | 3 available | 1 unavailable
StrategyType:       RollingUpdate
MinReadySeconds:    0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.161
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    ReplicaSetUpdated
OldReplicaSets:     nginx-deployment-1564180365 (3/3 replicas created)
NewReplicaSet:      nginx-deployment-3066724191 (1/1 replicas created)
Events:
  FirstSeen LastSeen    Count   From                    SubObjectPath   Type        Reason              Message
  --------- --------    -----   ----                    -------------   --------    ------              -------
  1m        1m          1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-2035384211 to 3
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 1
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 2
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 2
  21s       21s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 1
  21s       21s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 3
  13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 0
  13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-3066724191 to 1
  ```
要解决此问题，您需要回滚到稳定的部署的先前版本。

### 检查Deployment的发布历史

按照下面给出的步骤检查部署历史：

1. 首先，检查Deployment的版本：
```code
kubectl rollout history deployment nginx-deployment
```
输出类似：
```code
deployments "nginx-deployment"
REVISION    CHANGE-CAUSE
1           kubectl apply --filename=https://k8s.io/examples/controllers/nginx-deployment.yaml --record=true
2           kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1 --record=true
3           kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.161 --record=true
```
CHANGE-CAUSE 在创建时从部署注释 kubernetes.io/change-cause 复制到其修订版。 您可以通过以下方式指定 CHANGE-CAUSE 消息：
* 使用 kubectl annotate deployment.v1.apps/nginx-deployment kubernetes.io/change-cause="image updated to 1.16.1" 注释部署
* 附加 --record 标志以保存对资源进行更改的 kubectl 命令。
* 手动编辑资源的清单。

2. 查看每个修订的细节，运行：
```code
kubectl rollout history deployment.v1.apps/nginx-deployment --revision=2
```
输出类似：
```code
deployments "nginx-deployment" revision 2
  Labels:       app=nginx
          pod-template-hash=1159050644
  Annotations:  kubernetes.io/change-cause=kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1 --record=true
  Containers:
   nginx:
    Image:      nginx:1.16.1
    Port:       80/TCP
     QoS Tier:
        cpu:      BestEffort
        memory:   BestEffort
    Environment Variables:      <none>
  No volumes.
```

### 回滚到之前的修订

按照下面给出的步骤将部署从当前版本回滚到以前的版本，即版本 2。
1. 现在您已决定撤消当前的部署并回滚到以前的版本：
```code
kubectl rollout undo deployment.v1.apps/nginx-deployment
```
输出如下：
```code
deployment.apps/nginx-deployment rolled back
```
或者，您可以通过使用 --to-revision 指定它来回滚到特定版本：
```code
kubectl rollout undo deployment.v1.apps/nginx-deployment --to-revision=2
```
输出类似：
```code
deployment.apps/nginx-deployment rolled back
```
部署现在回滚到以前的稳定版本。 如您所见，用于回滚到修订版 2 的 DeploymentRollback 事件是从部署控制器生成的。

2. 检查回滚是否成功并且部署是否按预期运行，运行：

```code
kubectl get deployment nginx-deployment
```

输出如下：
```code
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           30m
```

3. 获取Deployment的描述

```code
kubectl describe deployment nginx-deployment
```
输出如下：
```code
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Sun, 02 Sep 2018 18:17:55 -0500
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision=4
                        kubernetes.io/change-cause=kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1 --record=true
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.16.1
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-c4747d96c (3/3 replicas created)
Events:
  Type    Reason              Age   From                   Message
  ----    ------              ----  ----                   -------
  Normal  ScalingReplicaSet   12m   deployment-controller  Scaled up replica set nginx-deployment-75675f5897 to 3
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-c4747d96c to 1
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled down replica set nginx-deployment-75675f5897 to 2
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-c4747d96c to 2
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled down replica set nginx-deployment-75675f5897 to 1
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-c4747d96c to 3
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled down replica set nginx-deployment-75675f5897 to 0
  Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-595696685f to 1
  Normal  DeploymentRollback  15s   deployment-controller  Rolled back deployment "nginx-deployment" to revision 2
  Normal  ScalingReplicaSet   15s   deployment-controller  Scaled down replica set nginx-deployment-595696685f to 0
```

## 扩容一个Deployment

你可以使用以下的命令扩容一个Deployment：
```code
kubectl scale deployment nginx-deployment --replica=10
```

输出类似如下：
```code
deployment.apps/nginx-deployment scaled
```

假设集群中启用了水平 Pod 自动缩放，您可以为部署设置自动缩放器，并根据现有 Pod 的 CPU 利用率选择要运行的最小和最大 Pod 数量。

```code
kubectl autoscale deployment nginx-deployment --min=10 --max=15 --cpu-percent=80 
```

输出如下：

```code
deployment.apps/nginx-deployment scaled
```

### 比例缩放

RollingUpdate 部署支持同时运行多个版本的应用程序。 当您或自动缩放器扩展处于推出过程中（正在进行或暂停）的 RollingUpdate 部署时，部署控制器会平衡现有活动 ReplicaSet（带有 Pod 的 ReplicaSet）中的额外副本以降低风险。 这称为比例缩放。

例如，您正在运行具有 10 个副本、maxSurge=3 和 maxUnavailable=2 的部署。

* 确保Deployment有10个副本正在运行
```code
kubectl get deploy
```
输出如下：
```code
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment     10        10        10           10          50s
```
* 您更新到一个新镜像，该映像恰好无法从集群内部解析。
```code
kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:sometag
```
输出如下：
```code
deployment.apps/nginx-deployment image updated
```

* 镜像更新开始使用 ReplicaSet nginx-deployment-1989198191 进行新的部署，但由于您上面提到的 maxUnavailable 要求，它被阻止了。 查看发布状态：

```code
kubectl get rs
```
输出如下：
```code
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-1989198191   5         5         0         9s
nginx-deployment-618515232    8         8         8         1m
```
* 然后出现了一个新的 Deployment 扩展请求。 自动缩放器将部署副本增加到 15。部署控制器需要决定在哪里添加这 5 个新副本。 如果您不使用比例缩放，则所有 5 个都将添加到新的 ReplicaSet 中。 通过按比例缩放，您可以将额外的副本分布在所有 ReplicaSet 中。 较大的比例分配给具有最多副本的 ReplicaSet，而较低的比例分配给具有较少副本的 ReplicaSet。 任何剩余的都被添加到具有最多副本的 ReplicaSet 中。 具有零副本的 ReplicaSet 不会扩展。

在我们上面的例子中，3 个副本被添加到旧的 ReplicaSet，2 个副本被添加到新的 ReplicaSet。 推出过程最终应该将所有副本移动到新的 ReplicaSet，假设新副本变得健康。 要确认这一点，请运行：

```code
kubectl get deploy
```

输出如下：
```code
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment     15        18        7            8           7m
```

发布状态确认副本如何添加到每个 ReplicaSet。
```code
kubectl get rs
```

输出类似如下：
```code
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-1989198191   7         7         0         7m
nginx-deployment-618515232    11        11        11        7m
```

### 暂停和恢复部署

您可以在触发一个或多个更新之前暂停部署，然后再恢复它。 这允许您在暂停和恢复之间应用多个修复程序，而不会触发不必要的发布。

* 例如，使用已创建的部署：获取部署详细信息：
```code
kubectl get deploy
```
输出如下：
```code
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx     3         3         3            3           1m
```
获取发布状态：
```code
kubectl get rs
```
输出如下：
```code
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   3         3         3         1m
```
* 通过以下命令暂停发布
```code
kubectl rollout pause deployment nginx-deployment
```
输出如下：
```code
deployment.apps/nginx-deployment paused
```
* 然后更新Deployment的镜像
```code
kubectl set image deployment nginx-deployment nginx:nginx:1.16.1
```
输出如下：
```code
deployment.apps/nginx-deployment image updated
```

* 注意新的发布开始了:
```code
kubectl rollout history deployment nginx-deployment
```
输出如下：
```code
deployments "nginx"
REVISION  CHANGE-CAUSE
1   <none>
```

* 获取发布状态，以确保Deployment成功更新
```code 
kubectl get rs 
```
输出如下：
```code
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   3         3         3         2m
```
* 您可以根据需要进行任意数量的更新，例如，更新将要使用的资源：
```code
kubectl set resources deployment.v1.apps/nginx-deployment -c=nginx --limits=cpu=200m,memory=512Mi
```
输出如下：
```code
deployment.apps/nginx-deployment resource requirements updated
```
暂停之前 Deployment 的初始状态将继续其功能，但只要 Deployment 暂停，对 Deployment 的新更新就不会产生任何影响。
* 最后，恢复Deployment并且观察新的额ReplicaSet以新的更新启动
```code
kubectl rollout resume deployment nginx-deployment
```
输出如下：
```code
deployment.apps/nginx-deployment resumed
```
* 观察滚动输出，知道结束
```code
kubectl get rs -w 
```
输出类似如下：
```code
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   2         2         2         2m
nginx-3926361531   2         2         0         6s
nginx-3926361531   2         2         1         18s
nginx-2142116321   1         2         2         2m
nginx-2142116321   1         2         2         2m
nginx-3926361531   3         2         1         18s
nginx-3926361531   3         2         1         18s
nginx-2142116321   1         1         1         2m
nginx-3926361531   3         3         1         18s
nginx-3926361531   3         3         2         19s
nginx-2142116321   0         1         1         2m
nginx-2142116321   0         1         1         2m
nginx-2142116321   0         0         0         2m
nginx-3926361531   3         3         3         20s
```
* 获取最新的滚动日志状态
```code
kubectl get rs
```
输出如下：
```code
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   0         0         0         2m
nginx-3926361531   3         3         3         28s
```

**小贴士:** 暂停状态的deployment无法回滚，除非你恢复它

### Deployment状态

在一个Deployment的生命周期中，会遇到各种状态。当推出新的ReplicaSet的时候，它是处于进行中，它也可以处于完成或者进行失败状态。

#### 进行中的Deployment

当一下的任务执行的时候，kubernetes会标记Deployment为进行中的状态
* Deployment创建了一个新的ReplicaSet
* Deployment正在扩充其最新的ReplicaSet
* Deployment正在缩减旧版本的ReplicaSet
* 新的Pod变成就绪或者可用状态

您可以检测Deployment的过程，通过使用`kubectl rollout status`

#### 完成的Deployment

当有一下特征的时候，kubernetes标记Deployment为完成状态：
* 与Deployment相关的所有副本数更新至您指定的最新版本，意味着你的任何更新请求都已经完成
* 与Deployment相关的所有副本都变成可用
* 没有该Deployment的旧副本在运行中

您可以使用命令`kubectl rollout status`来检查Deployment是否完成。如果滚出成功完成，`kubectl rollout status`返回一个退出代码0
```shell
kubectl rollout status deployment/nginx-deployment
```
输出如下：
```code
Waiting for rollout to finish: 2 of 3 updated replicas are available...
deployment "nginx-deployment" successfully rolled out
```
并且`kubectl rollout`的推出状态是0(success)
```code
echo $?
```
```code
0
```

#### 失败的Deployment

您的`Deployment`可能会在尝试部署其最新的`ReplicaSet`而从未完成时卡住。这可能是由于以下一些因素造成的：
* 不足的配额
* 就绪探针故障
* 镜像拉取错误
* 不足的权限
* 限制范围
* 应用程序运行时配置错误

发现这种状态的一种可行方案是：在Deployment的spec中定义一个截止日期字段:`.spec.progressDeadlineSeconds`。.spec.progressDeadlineSeconds字段表示Deployment控制器在部署进度已经停止之前等待的秒数。
以下 kubectl 命令使用 progressDeadlineSeconds 设置规范，以使控制器在 10 分钟后报告部署缺乏进度：
```shell
kubectl patch deployment/nginx-deployment -p '{"spec":{"progressDeadlineSeconds":600}}'
```
输出如下：
```code
deployment.apps/nginx-deployment patched
```
一旦超过最后期限，Deployment 控制器就会向 Deployment 的 .status.conditions 添加一个具有以下属性的 DeploymentCondition：
* Type=Progressing
* Status=False
* Reason=ProgressDeadlineExceeded

有关状态条件的更多信息，请参阅 Kubernetes API 约定。

**注意**：除了报告 Reason=ProgressDeadlineExceeded 的状态条件外，Kubernetes 不会对停滞的 Deployment 采取任何措施。 更高级别的编排器可以利用它并采取相应的行动，例如，将部署回滚到其以前的版本。

**注意**：如果您暂停部署，Kubernetes 不会根据您指定的截止日期检查进度。 您可以在部署过程中安全地暂停部署并恢复，而不会触发超过截止日期的条件。

您的部署可能会遇到暂时性错误，原因可能是您设置的超时时间过短，或者是任何其他类型的可被视为暂时性的错误。 例如，假设您的配额不足。 如果您描述部署，您会注意到以下部分：

```shell
kubectl describe deployment nginx-deployment
```

输出如下：

```code
<...>
Conditions:
  Type            Status  Reason
  ----            ------  ------
  Available       True    MinimumReplicasAvailable
  Progressing     True    ReplicaSetUpdated
  ReplicaFailure  True    FailedCreate
<...>
```

如果运行 `kubectl get deployment nginx-deployment -o yaml`，Deployment 状态类似于：

```code
status:
  availableReplicas: 2
  conditions:
  - lastTransitionTime: 2016-10-04T12:25:39Z
    lastUpdateTime: 2016-10-04T12:25:39Z
    message: Replica set "nginx-deployment-4262182780" is progressing.
    reason: ReplicaSetUpdated
    status: "True"
    type: Progressing
  - lastTransitionTime: 2016-10-04T12:25:42Z
    lastUpdateTime: 2016-10-04T12:25:42Z
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: 2016-10-04T12:25:39Z
    lastUpdateTime: 2016-10-04T12:25:39Z
    message: 'Error creating: pods "nginx-deployment-4262182780-" is forbidden: exceeded quota:
      object-counts, requested: pods=1, used: pods=3, limited: pods=2'
    reason: FailedCreate
    status: "True"
    type: ReplicaFailure
  observedGeneration: 3
  replicas: 2
  unavailableReplicas: 2
```

最终，一旦超过部署进度截止日期，Kubernetes 会更新状态和 Progressing 条件的原因：

```code
Conditions:
Type            Status  Reason
----            ------  ------
Available       True    MinimumReplicasAvailable
Progressing     False   ProgressDeadlineExceeded
ReplicaFailure  True    FailedCreate
```

您可以通过缩减部署、缩减可能正在运行的其他控制器或增加命名空间中的配额来解决配额不足的问题。 如果您满足配额条件并且部署控制器随后完成部署部署，您将看到部署的状态更新为成功条件（Status=True 和 Reason=NewReplicaSetAvailable）。

```code
Conditions:
Type          Status  Reason
----          ------  ------
Available     True    MinimumReplicasAvailable
Progressing   True    NewReplicaSetAvailable
```

Type=Available 和 Status=True 意味着您的 Deployment 具有最低可用性。 最低可用性由部署策略中指定的参数决定。 Type=Progressing with Status=True 意味着您的 Deployment 要么处于推出的中间并且正在进行中，要么已成功完成其进度并且所需的最少新副本可用（有关详细信息，请参阅条件原因 - 在我们的例子中 Reason=NewReplicaSetAvailable 意味着部署已经完成）。

您可以使用 `kubectl rollout status` 检查部署是否未能进行。 如果部署已超过进度截止日期，则 `kubectl rollout status` 将返回非零退出代码。

```shell
kubectl rollout status deployment/nginx-deployment
```

输出如下：
```code
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
error: deployment "nginx" exceeded its progress deadline
```

并且 `kubectl rollout` 的退出状态为 1（表示错误）：

```code
echo $?
```

#### 操作失败的Deployment

适用于完整部署的所有操作也适用于失败的部署。 如果您需要在 `Deployment Pod` 模板中应用多个调整，您可以放大/缩小它，回滚到以前的版本，甚至可以暂停它。

### 回收策略

您可以在 Deployment 中设置 `.spec.revisionHistoryLimit` 字段，以指定要保留此 Deployment 的旧 ReplicaSet 数量。 其余的将在后台进行垃圾收集。 默认情况下，它是 10。

**注意**：将此字段显式设置为 0，将导致清理您的 Deployment 的所有历史记录，因此 Deployment 将无法回滚。

## 金丝雀部署

如果您想使用 Deployment 向一部分用户或服务器推出版本，您可以创建多个 Deployment，每个版本一个，遵循管理资源中描述的金丝雀模式。

## 编辑一个Deployment的spec

与所有其他 Kubernetes 配置一样，部署需要 `.apiVersion`、`.kind` 和 `.metadata` 字段。 有关使用配置文件的一般信息，请参阅部署应用程序、配置容器和使用 kubectl 管理资源文档。 Deployment 对象的名称必须是有效的 DNS 子域名。Deployment 还需要一个 `.spec` 部分。

### Pod模版

`.spec.template` 和 `.spec.selector` 是 `.spec` 的唯一必填字段。

`.spec.template` 是一个 Pod 模板。 它具有与 Pod 完全相同的架构，除了它是嵌套的并且没有 apiVersion 或种类。

除了 Pod 的必填字段外，Deployment 中的 Pod 模板必须指定适当的标签和适当的重启策略。 对于标签，请确保不要与其他控制器重叠。 见选择器。

仅允许 `.spec.template.spec.restartPolicy` 等于 Always ，如果未指定，则为默认值。

### 副本数量

`.spec.replicas` 是一个可选字段，用于指定所需 Pod 的数量。 它默认为 1。

### 选择器

`.spec.selector` 是必填字段，用于指定此 Deployment 所针对的 Pod 的标签选择器。

`.spec.selector` 必须匹配 `.spec.template.metadata.labels`，否则会被 API 拒绝。

在 API 版本 apps/v1 中，如果未设置，`.spec.selector` 和 `.metadata.labels` 不会默认为 `.spec.template.metadata.labels`。 因此必须明确设置它们。 另请注意，在 apps/v1 中创建部署后 `.spec.selector` 是不可变的。

如果模板与 `.spec.template` 不同，或者此类 Pod 的总数超过 `.spec.replicas`，Deployment 可能会终止标签与选择器匹配的 Pod。 如果 Pod 的数量少于所需的数量，它会使用 `.spec.template` 调出新的 Pod。

**注意**：您不应直接通过创建另一个 Deployment 或创建另一个控制器（例如 ReplicaSet 或 ReplicationController）来创建标签与此选择器匹配的其他 Pod。 如果这样做，第一个 Deployment 会认为它创建了这些其他 Pod。 Kubernetes 不会阻止您这样做。

如果您有多个具有重叠选择器的控制器，则这些控制器将相互冲突并且无法正常运行。

### 策略

`.spec.strategy` 指定用于用新 Pod 替换旧 Pod 的策略。 `.spec.strategy.type` 可以是“Recreate”或“RollingUpdate”。 “RollingUpdate”是默认值。

### 重新创建Deployment

当 `.spec.strategy.type==Recreate` 时，所有现有 Pod 在创建新 Pod 之前都会被杀死。

**注意**：这只会保证在创建升级之前终止 Pod。 如果升级 Deployment，旧版本的所有 Pod 将立即终止。 在创建新修订的任何 Pod 之前，等待成功删除。 如果手动删除一个 Pod，生命周期由 ReplicaSet 控制，并且会立即创建替换（即使旧 Pod 仍处于 Termination 状态）。 如果您的 Pod 需要“最多”保证，您应该考虑使用 StatefulSet。

### 滚动升级Deployment

当 `.spec.strategy.type==RollingUpdate` 时，部署以滚动更新方式更新 Pod。 您可以指定 maxUnavailable 和 maxSurge 来控制滚动更新过程。

#### 最大不可用

`.spec.strategy.rollingUpdate.maxUnavailable` 是一个可选字段，用于指定在更新过程中不可用的最大 Pod 数。 该值可以是绝对数（例如 5）或所需 Pod 的百分比（例如 10%）。 绝对数是通过四舍五入的百分比计算的。 如果 `.spec.strategy.rollingUpdate.maxSurge` 为 0，则该值不能为 0。默认值为 25%。

例如，当这个值设置为 30% 时，旧的 ReplicaSet 可以在滚动更新开始时立即缩小到所需 Pod 的 70%。 一旦新的 Pod 准备好，旧的 ReplicaSet 可以进一步缩小，然后扩大新的 ReplicaSet，确保更新期间始终可用的 Pod 总数至少是所需 Pod 的 70%。

#### 最大数量

`.spec.strategy.rollingUpdate.maxSurge` 是一个可选字段，用于指定可以在所需 Pod 数量上创建的最大 Pod 数量。 该值可以是绝对数（例如 5）或所需 Pod 的百分比（例如 10%）。 如果 MaxUnavailable 为 0，则该值不能为 0。绝对数由百分比四舍五入计算得出。 默认值为 25%。

例如，当这个值设置为 30% 时，可以在滚动更新开始时立即扩容新的 ReplicaSet，使得新旧 Pod 的总数不超过所需 Pod 的 130%。 一旦旧的 Pod 被杀死，新的 ReplicaSet 可以进一步扩展，确保更新期间任何时间运行的 Pod 总数最多为所需 Pod 的 130%。

### 过程中止秒数

`.spec.progressDeadlineSeconds` 是一个可选字段，用于指定在系统报告部署已失败进展之前您希望等待部署进展的秒数 - 作为 Type=Progressing, Status=False 的条件浮出水面。 和 Reason=ProgressDeadlineExceeded 在资源的状态中。 部署控制器将不断重试部署。 默认为 600。以后，一旦实现自动回滚，Deployment 控制器会在观察到这种情况后立即回滚一个 Deployment。如果指定，此字段需要大于 `.spec.minReadySeconds`。

### 最小可用秒数

`.spec.minReadySeconds` 是一个可选字段，用于指定新创建的 Pod 应准备就绪且其任何容器不会崩溃的最小秒数，以便将其视为可用。 这默认为 0（Pod 一准备好就被认为是可用的）。 要了解有关 Pod 何时被视为就绪的更多信息，请参阅容器探测。

### 历史版本限制

Deployment 的修订历史存储在它控制的 ReplicaSet 中。

`.spec.revisionHistoryLimit` 是一个可选字段，指定要保留以允许回滚的旧 ReplicaSet 的数量。 这些旧的 ReplicaSet 消耗了 etcd 中的资源并挤占了 kubectl get rs 的输出。 每个 Deployment 版本的配置都存储在它的 ReplicaSet 中； 因此，一旦旧的 ReplicaSet 被删除，您将无法回滚到该版本的 Deployment。 默认情况下，将保留 10 个旧 ReplicaSet，但其理想值取决于新部署的频率和稳定性。更具体地说，将此字段设置为零意味着将清除所有具有 0 个副本的旧 ReplicaSet。 在这种情况下，新的部署部署无法撤消，因为它的修订历史已被清除。

### 暂停

`.spec.paused` 是一个可选的布尔字段，用于暂停和恢复部署。 暂停部署和未暂停部署之间的唯一区别是，只要暂停部署，对暂停部署的 PodTemplateSpec 的任何更改都不会触发新的部署。 部署在创建时默认不会暂停。

## 接下来阅读

* [深入了解Pods](https://kubernetes.io/docs/concepts/workloads/pods)
* [使用Deployment运行一个无状态应用](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/)
* Deployment是一个kubernetes restful api的一个顶级资源。阅读[Deployment](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/deployment-v1/)对象的定义并理解api
* 阅读有关 [PodDisruptionBudget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/) 以及如何在中断期间使用它来管理应用程序可用性的信息。