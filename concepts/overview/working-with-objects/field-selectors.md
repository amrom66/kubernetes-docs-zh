# 字段选择器

字段选择器是你能够根据一个或者多个资源字段筛选kubernetes资源。以下是一些实例：
* metadata.name=my-service
* metadata.namespace!=default
* status.phase=Pending

使用`kubectl`命令筛选`status.phase`字段值是`Running`的pod：

```code
kubectl get pods --field-selector status.phase=Running
```

注意：字段选择器本质上是资源过滤器。 默认情况下，不应用选择器/过滤器，这意味着将选择指定类型的所有资源。 这使得kubectl查询kubectl获取pod和kubectl获取pod --field-selector“”等效。


### 支持的字段

支持的字段选择器因Kubernetes资源类型而异。 所有资源类型都支持metadata.name和metadata.namespace字段。 使用不受支持的字段选择器会产生错误。 例如：

```code
kubectl get ingress --field-selector foo.bar=baz
```

Error from server (BadRequest): Unable to find "ingresses" that match label selector "", field selector "foo.bar=baz": "foo.bar" is not a known field selector: only "metadata.name", "metadata.namespace"


### 支持的操作符

您可以将=，==和！=运算符与字段选择器一起使用（=和==表示同一意思）。 例如，以下kubectl命令选择不在默认名称空间中的所有Kubernetes服务：

```code
kubectl get services  --all-namespaces --field-selector metadata.namespace!=default
```

### 链式选择器

与标签选择器和其他选择器一样，字段选择器可以链接在一起，以逗号分隔的列表形式。 此kubectl命令选择status.phase不等于Running且spec.restartPolicy字段等于Always的所有Pod：

```code
kubectl get pods --field-selector=status.phase!=Running,spec.restartPolicy=Always
```

### 多种资源类型

您可以在多种资源类型之间使用字段选择器。 此kubectl命令选择不在默认名称空间中的所有Statefulsets和Services：

```code
kubectl get statefulsets,services --all-namespaces --field-selector metadata.namespace!=default
```





