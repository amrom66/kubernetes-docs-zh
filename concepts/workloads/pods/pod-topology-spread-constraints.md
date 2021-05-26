# Pod拓扑传播限制

您可以使用拓扑扩展约束来控制Pod如何在故障域（例如区域，区域，节点和其他用户定义的拓扑域）之间跨群集分布。 这可以帮助实现高可用性以及有效的资源利用。

注意：在v1.18之前的Kubernetes版本中，必须使用API服务器和调度程序启用EvenPodsSpread功能门，才能使用Pod拓扑扩展约束。

## 先决条件

### 节点标签

拓扑扩展约束条件依赖于节点标签来标识每个节点所在的拓扑域。例如，一个节点可能具有以下标签：node = node1，zone = us-east-1a，region = us-east-1

假设，你有一个4节点的集群，附带以下标签：

```code
NAME    STATUS   ROLES    AGE     VERSION   LABELS
node1   Ready    <none>   4m26s   v1.16.0   node=node1,zone=zoneA
node2   Ready    <none>   3m58s   v1.16.0   node=node2,zone=zoneA
node3   Ready    <none>   3m17s   v1.16.0   node=node3,zone=zoneB
node4   Ready    <none>   2m43s   v1.16.0   node=node4,zone=zoneB
```

集群的逻辑视图如下：

...

除了可以手动设置标签，你也可以复用在大多数集群中流行的[知名标签](https://kubernetes.io/docs/reference/labels-annotations-taints/)

## pods的分发限制

### API

API字段pod.spec.topologySpreadConstraints定义如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  topologySpreadConstraints:
    - maxSkew: <integer>
      topologyKey: <string>
      whenUnsatisfiable: <string>
      labelSelector: <object>
```

你可以定义一个或者多个`topologySpreadConstraint`来告诉kube-scheduler如何将每个传入的pod相对于整个集群的现有pod进行放置。这些字段如下：

* maxSkew 描述Pod可能不均匀分布的程度。 它是给定拓扑类型的任何两个拓扑域中的匹配Pod数量之间的最大允许差值。 它必须大于零。 根据whenUnsatisfiable的值，其语义也有所不同： 
    * 当whenUnsatisfiable的值等于DoNotSchedule的时候，maxSkew是目标拓扑中的匹配吊舱数与全局最小值之间的最大允许差值。
    * 当不满意的时间等于“ ScheduleAnyway”时，调度程序将较高的优先级给予拓扑结构，这将有助于减少时滞。
* topologyKey 是节点标签的key。如果两个节点都用此关键字标记并且具有相同的值，则调度程序会将两个节点都视为处于同一拓扑中。 调度程序尝试将平衡数量的Pod放入每个拓扑域中。
* whenUnsatisfiable表示如果Pod不满足传播约束，则如何处理：
    * DoNotSchedule（默认）告诉调度程序不要调度它
    * ScheduleAnyway告诉调度程序在对节点进行优先级划分以最大程度地减少偏斜的同时仍要对其进行调度。
* labelSelector用于查找匹配的Pod。 计算与该标签选择器匹配的Pod，以确定其相应拓扑域中的Pod数。 有关更多详细信息，请参见标签选择器。
您可以通过运行kubectl explain Pod.spec.topologySpreadConstraints 来阅读有关此字段的更多信息。

### 实例： 一种拓扑扩展约束

假设您有一个4节点群集，其中3个标记为foo：bar的Pod分别位于node1，node2和node3中：
。。。

如果我们希望传入的Pod与现有Pod均匀分布在各个区域，则规格可以指定为：

pods/topology-spread-constraints/one-constraint.yaml

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod
  labels:
    foo: bar
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        foo: bar
  containers:
  - name: pause
    image: k8s.gcr.io/pause:3.1
```

topologyKey：zone表示均匀分布将仅应用于标签对为“ zone：<any value>”的节点。 whenUnsatisfiable：如果传入Pod无法满足约束条件，则DoNotSchedule通知调度程序使其处于待处理状态。

如果调度程序将此传入Pod放入“ zoneA”，则Pods分布将变为[3，1]，因此实际时滞为2（3-1）-这违反了maxSkew：1.在此示例中，传入Pod仅能 放在“ zoneB”上：

您可以调整pod规范，以满足各种要求：
* 将maxSkew更改为更大的值，例如“ 2”，以便将传入的Pod也可以放置在“ zoneA”上。
* 将topologyKey更改为“ node”，以便在节点而不是区域之间均匀分布Pod。 在上面的示例中，如果maxSkew保持为“ 1”，则传入的Pod只能放置在“ node4”上
* 将whenUnsatisfiable：DoNotSchedule更改为whenUnsatisfiable：ScheduleAnyway，以确保传入的Pod始终是可调度的（假设满足其他调度API）。 但是，最好将其放置在匹配Pod较少的拓扑域中。 （请注意，这种可取性已与其他内部调度优先级（如资源使用率等）共同标准化。）

示例：多个拓扑扩展约束

这是建立在前面的例子的基础上的。 假设您有一个4节点群集，其中3个标记为foo：bar的Pod分别位于node1，node2和node3中：
您可以使用2个TopologySpreadConstraints来控制Pod在区域和节点上的扩散：

```yaml 
kind: Pod
apiVersion: v1
metadata:
  name: mypod
  labels:
    foo: bar
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        foo: bar
  - maxSkew: 1
    topologyKey: node
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        foo: bar
  containers:
  - name: pause
    image: k8s.gcr.io/pause:3.1
```

在这种情况下，为了匹配第一个约束，只能将传入的Pod放置在“ zoneB”上； 而就第二个约束而言，传入的Pod只能放置在“ node4”上。 然后将2个约束的结果进行“与”运算，因此唯一可行的选择是将其放置在“ node4”上。


