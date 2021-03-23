# 标签和标签选择器

标签是附加到对象（例如pods）的键/值对。 标签旨在用于指定有意义且与用户相关的对象的标识属性，但并不直接暗示核心系统的语义。 标签可用于组织和选择对象的子集。 标签可以在创建时附加到对象，然后可以随时添加和修改。 每个对象可以定义一组键/值标签。 每个键对于给定的对象必须是唯一的。

```code
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

标签允许进行有效的查询和监视，并且非常适合在UI和CLI中使用。 非识别信息应使用[注释](annotations.md)记录。


## 动机

标签使用户能够以松散耦合的方式将自己的组织结构映射到系统对象上，而无需客户端存储这些映射。

服务部署和批处理管道通常是多维实体（例如，多个分区或部署，多个发布轨道，多个层，每个层多个微服务）。 管理通常需要横切操作，这破坏了严格层次表示形式的封装，尤其是由基础结构而不是用户确定的刚性层次结构。

示例标签：
* "release" : "stable", "release" : "canary"
* "environment" : "dev", "environment" : "qa", "environment" : "production"
* "tier" : "frontend", "tier" : "backend", "tier" : "cache"
* "partition" : "customerA", "partition" : "customerB"
* "track" : "daily", "track" : "weekly"

这些是常用标签的示例； 您可以自由制定自己的约定。 请记住，标签Key对于给定的对象必须是唯一的。


## 语法和字符集

标签是键/值对。 有效的标签键分为两部分：可选的前缀和名称，用斜杠（/）分隔。 名称段是必需的，并且必须为63个字符或更少，以字母数字字符（[a-z0-9A-Z]）开头和结尾，并带有破折号（-），下划线（_），点（。），以及之间的字母数字 。 前缀是可选的。 如果指定，则前缀必须是DNS子域：一系列由点（。）分隔的DNS标签，总计不超过253个字符，后跟斜杠（/）。

如果省略前缀，则假定标签Key对用户是私有的。 向最终用户对象添加标签的自动化系统组件（例如kube-scheduler，kube-controller-manager，kube-apiserver，kubectl或其他第三方自动化）必须指定前缀。

The kubernetes.io/ and k8s.io/ prefixes are reserved for Kubernetes core components.

合法的标签值：
* 必须小于等于63个字符，且不能为空
* 以字母数字开头，[a-z0-9A-Z]
* 可以包含横线，下划线以及英文点，中间夹杂字母数字


例如，以下是一个pod的配置文件，包含2个标签：`environment: production` 和 `app: nginx`

```yaml 

apiVersion: v1
kind: Pod
metadata:
  name: label-demo
  labels:
    environment: production
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```


## 标签选择器

与名称和UID不同，标签不提供唯一性。 通常，我们希望许多对象带有相同的标签。

通过标签选择器，客户端/用户可以识别一组对象。 标签选择器是Kubernetes中的核心分组原语。

该API当前支持两种选择器：基于等式和基于集合。 标签选择器可以由逗号分隔的多个要求组成。 在有多个要求的情况下，必须满足所有要求，以便逗号分隔符充当逻辑AND（&&）运算符。

空的或未指定的选择器的语义取决于上下文，并且使用选择器的API类型应记录它们的有效性和含义。

注意：对于某些API类型（例如ReplicaSets），两个实例的标签选择器在名称空间内一定不能重叠，否则控制器会将其视为冲突的指令，而无法确定应该存在多少个副本。

警告：对于基于相等的条件和基于集合的条件，都没有逻辑OR（||）运算符。 确保您的过滤器语句具有相应的结构。


### 基于平等的要求

基于等式或不等式的要求允许按标签键和值进行过滤。 匹配对象必须满足所有指定的标签约束，尽管它们也可能具有其他标签。 允许使用三种运算符=，==，！=。 前两个代表平等（并且是同义词），而第二个代表不平等。 例如：

```code
environment = production
tier != frontend
```

前者选择等于环境且值等于生产的所有资源。 后者选择键等于层且值与前端不同的所有资源，并选择不带有层键的标签的所有资源。 一个人可以使用逗号运算符过滤生产中不包括前端的资源：environment = production，tier！= frontend

基于相等性的标签要求的一种使用场景是Pod指定节点选择标准。 例如，下面的示例Pod选择标签为“ accelerator = nvidia-tesla-p100”的节点。


```code
apiVersion: v1
kind: Pod
metadata:
  name: cuda-test
spec:
  containers:
    - name: cuda-test
      image: "k8s.gcr.io/cuda-vector-add:v0.1"
      resources:
        limits:
          nvidia.com/gpu: 1
  nodeSelector:
    accelerator: nvidia-tesla-p100
```

### 基于集合的需求

基于集合的标签要求允许根据一组值过滤关键字。 支持三种运算符：in，notin和存在（仅键标识符）。 例如：

```code
environment in (production, qa)
tier notin (frontend, backend)
partition
!partition
```

* 第一个示例选择键等于环境且值等于生产或qa的所有资源。
* 第二个示例选择键等于tier的所有资源以及前端和后端以外的其他值，以及没有标签的所有带有tier key的资源。
* 第三个示例选择所有资源，包括带有键分区的标签； 不检查任何值。
* 第四个示例选择不带键分区标签的所有资源。 不检查任何值。

同样，逗号分隔符充当AND运算符。 因此，可以使用partition，environment notin（qa）来实现使用分区键（无论其值）和环境是否不同于qa来过滤资源。 基于集合的标签选择器是一般的相等形式，因为环境=生产等同于（生产）中的环境。 类似地用于！=和notin。

基于集合的需求可以与基于相等的需求混合。 例如：（客户，客户），环境中的分区！= qa。


## API

### 列表和手表过滤

LIST和WATCH操作可以指定标签选择器以过滤使用查询参数返回的对象集。 这两个要求都是允许的（在此处显示，因为它们将出现在URL查询字符串中）：

基于等式的要求：？labelSelector = environment％3Dproduction，tier％3Dfrontend

基于集合的要求：？labelSelector =环境+ in +％28production％2Cqa％29％2Ctier + in +％28frontend％29

两种标签选择器样式均可用于通过REST客户端列出或监视资源。 例如，使用kubectl定位apiserver并使用基于等式的apiserver可能会这样写：

`kubectl get pods -l environment=production,tier=frontend`

或使用基于集合的要求：

`kubectl get pods -l 'environment in (production),tier in (frontend)'`

如前所述，基于集合的需求更具表现力。 例如，他们可以在值上实现OR运算符：

`kubectl get pods -l 'environment in (production, qa)'`

or restricting negative matching via exists operator:

或通过存在运算符限制否定匹配：

`kubectl get pods -l 'environment,environment notin (frontend)'`

### 在API对象中设置引用

一些Kubernetes对象（例如服务和复制控制器）也使用标签选择器来指定其他资源集（例如Pod）。

### 服务和副本控制器

服务目标的Pod集合是使用标签选择器定义的。 同样，还使用标签选择器定义了复制控制器应管理的Pod数量。

使用地图在json或yaml文件中定义了两个对象的标签选择器，并且仅支持基于相等性的需求选择器：

```code
"selector": {
    "component" : "redis",
}
```

或者：
```code
selector:
    component: redis
```

此选择器（分别为json或yaml格式）等效于component = redis或（redis）中的component。

### 支持基于集合的需求的资源

较新的资源（例如Job，Deployment，ReplicaSet和DaemonSet）也支持基于集合的要求。

```code
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
```

matchLabels是{key，value}对的映射。 matchLabels映射中的单个{key，value}等同于matchExpressions的元素，其key字段为“ key”，运算符为“ In”，而values数组仅包含“ value”。 matchExpressions是Pod选择器要求的列表。 有效的运算符包括In，NotIn，Exists和DidNotExist。 对于In和NotIn，设置的值必须为非空。 matchLabel和matchExpressions中的所有要求都进行了AND运算-必须全部满足才能匹配。

### 选择节点集

用于选择标签的一个用例是约束Pod可以调度到的节点集。 有关更多信息，请参见有关节点选择的文档。






