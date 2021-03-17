# k8s api

kubernetes控制平面的核心是api server。api server暴露出HTTP API，这保证了终端用户、集群间的不同部分、以及外部组件，之间互相通信。

Kubernetes API使您可以查询和操纵Kubernetes中API对象的状态（例如：Pods，命名空间，ConfigMap和Events）。

多数操作可以通过kubectl命令行界面或其他命令行工具（例如kubeadm）执行，这些工具依次使用API。 但是，您也可以使用REST调用直接访问API。

如果要使用Kubernetes API编写应用程序，请考虑使用其中一种[客户端库](https://kubernetes.io/docs/reference/using-api/client-libraries/)。


## OpenAPI 特性

使用OpenAPI记录了完整的API详细信息。

Kubernetes API服务器通过/ openapi / v2端点提供OpenAPI规范。 您可以使用请求标头来请求响应格式，如下所示：

| Header             | Possible values                                              | Notes                                          |
| ------------------ | ------------------------------------------------------------ | ---------------------------------------------- |
| `Accept-Encoding`  | `gzip`                                                       | *not supplying this header is also acceptable* |
| `Accept`           | `application/com.github.proto-openapi.spec.v2@v1.0+protobuf` | *mainly for intra-cluster use*                 |
| `application/json` | *default*                                                    |                                                |
| `*`                | *serves* `application/json`                                  |                                                |

Kubernetes实现了另一种基于Protobuf的序列化格式，该格式主要用于集群内通信。 有关此格式的更多信息，请参阅Kubernetes Protobuf序列化设计建议以及位于定义API对象的Go包中的每个架构的接口定义语言（IDL）文件。


## 持久化

Kubernetes通过将对象的序列化状态写入etcd来存储它们。


## API 分组与版本

为了更轻松地消除字段或重组资源表示形式，Kubernetes支持多个API版本，每个版本位于不同的API路径，例如/ api / v1或/apis/rbac.authorization.k8s.io/v1alpha1。

版本控制是在API级别而不是资源或字段级别完成的，以确保API呈现系统资源和行为的清晰一致的视图，并能够控制对寿命终止和/或实验性API的访问。

为了使其易于开发和扩展其API，Kubernetes实现了可以启用或禁用的API组。

API资源通过其API组，资源类型，名称空间（用于命名空间的资源）和名称来区分。 API服务器透明地处理API版本之间的转换：所有不同版本实际上是相同持久数据的表示。 API服务器可以通过多个API版本提供相同的基础数据。

例如，假设对于同一资源有两个API版本，即v1和v1beta1。 如果最初使用其API的v1beta1版本创建了对象，则以后可以使用v1beta1或v1 API版本读取，更新或删除该对象。

### API变动

任何成功的系统都需要随着新用例的出现或现有用例的变化而增长和变化。 因此，Kubernetes设计了Kubernetes API来不断变化和增长。 Kubernetes项目旨在不破坏与现有客户端的兼容性，并在一段时间内保持这种兼容性，以便其他项目有机会进行调整。

通常，可以频繁地添加新的API资源和新的资源字段。 消除资源或字段要求遵循API弃用政策。

一旦正式的Kubernetes API达到通用版本（GA）（通常在API版本v1），Kubernetes便会保持其兼容性。 此外，即使在可行的情况下，Kubernetes仍可与Beta API版本保持兼容性：如果您采用Beta API，即使功能稳定后，您仍可以继续使用该API与集群进行交互。

注意：尽管Kubernetes还旨在保持与Alpha API版本的兼容性，但是在某些情况下这是不可能的。 如果您使用任何Alpha API版本，请在升级集群时查看Kubernetes的发行说明，以防API确实发生了变化。

## API扩展

Kubernetes API可以通过以下两种方式之一进行扩展：

* 使用自定义资源，您可以声明性地定义API服务器应如何提供所选资源API。
* 您还可以通过实现聚合层来扩展Kubernetes API。
