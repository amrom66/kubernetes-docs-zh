# 镜像

容器镜像表示封装应用程序及其所有软件依赖项的二进制数据。容器镜像是可执行的软件包，可以独立运行，并对其运行环境做出非常明确的假设。
您通常会创建应用程序的容器映像并将其推送到仓库，然后再在 Pod 中引用它。本节提供容器镜像概念的概要。

## 镜像名称

容器镜像通常有一个名称，例如 pause、example/mycontainer 或 kube-apiserver。 图像还可以包含仓库名； 例如：fictional.registry.example/imagename，也可能是端口号； 例如：fictional.registry.example:10443/imagename。
如果您未指定仓库名，Kubernetes 会假定您指的是 Docker 公共仓库。在镜像名称部分之后，您可以添加一个标签（也可以与 docker 和 podman 等命令一起使用）。标签可让您识别同一系列镜像的不同版本。镜像标签由小写和大写字母、数字、下划线 (_)、句点 (.) 和破折号 (-) 组成。关于在镜像标签中放置分隔符（_、- 和 .）的位置还有其他规则。如果您未指定标签，kubernetes会假定您指定的是latest标签。

注意：
在生产环境中部署容器时，您应该避免使用 latest 标签，因为跟踪正在运行的镜像版本更加困难，并且更难以回滚到工作版本。作为替代，请指定一个有意义的标签，例如 v1.42.0。

## 更新镜像

当您第一次创建 Deployment、StatefulSet、Pod 或其他包含 Pod 模板的对象时，如果未明确指定，则默认情况下该 Pod 中所有容器的拉取策略将设置为 IfNotPresent。 如果镜像已经存在，这个策略会导致 kubelet 跳过拉取镜像。如果您想要强制拉取镜像，您可以采取以下措施之一：
* 设置容器的`imagePullPolicy`为`Always`
* 忽略`imagePullPolicy`，并且设置`:latest`作为使用的镜像，kubernetes将会将策略设置为`Always`
* 开启[AlwaysPullImage](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#alwayspullimages)管理控制器

提示：
容器的 imagePullPolicy 的值总是在对象第一次创建时设置，如果以后镜像的标签发生变化，则不会更新。例如，如果您使用标签不是`:latest`的映像创建部署，然后将该部署的镜像更新为`:latest`标签，则`imagePullPolicy`字段不会更改为`Always`。 您必须在初始创建后手动更改任何对象的拉取策略。
当 imagePullPolicy 定义为没有特定值时，它也设置为 Always。

## 具有镜像索引的多架构镜像

除了提供二进制镜像，容器仓库还可以提供容器镜像索引。一个镜像索引可以指向容器的架构特定版本的多个镜像清单。这个想法是你可以为一个镜像命名（例如：pause、example/mycontainer、kube-apiserver）并允许不同的系统为他们正在使用的机器架构获取正确的二进制镜像。
Kubernetes 本身通常使用后缀 -$(ARCH) 命名容器镜像。 为了向后兼容，请生成带有后缀的旧镜像。这个想法是生成`pause`镜像，它具有所有架构的清单，并说`pause-amd64`向后兼容旧配置或 YAML 文件，这些文件可能对图像进行了后缀硬编码。

## 使用私有仓库

私有仓库可能需要密钥才能从中读取镜像。认证可以通过数种方式提供：

* 配置节点到私有仓库的认证
  * 所有pods都可以任意配置的私有仓库
  * 需要集群管理员配置节点
* 提前拉取镜像
  * 所有pod可以使用任意节点已经缓存的镜像
  * 需要正对所有节点的root授权来设置
* 在pod种指定镜像拉取密钥
  * 只有提供了自身密钥的pods才可以访问私有仓库
* 供应商特定或者本地插件
  * 如果你使用自定义的节点配置文件，你或者云服务提供商，可以实现自己的认证节点到仓库的机制

这些方式将在下面描述。

### 配置节点到仓库的认证

如果您在节点上运行 Docker，则可以配置 Docker 容器运行时以对私有容器仓库进行身份验证。

如果您可以控制节点配置，则此方法适用。

注意：默认 Kubernetes 仅支持 Docker 配置中的 auths 和 HttpHeaders 部分。 不支持 Docker 凭证助手（credHelpers 或 credsStore）。

Docker 将私有仓库的密钥存储在 $HOME/.dockercfg 或 $HOME/.docker/config.json 文件中。 如果您将相同的文件放在下面的搜索路径列表中，kubelet 会在拉取镜像时将其用作凭证提供程序。

* {--root-dir:-/var/lib/kubelet}/config.json
* {cwd of kubelet}/config.json
* ${HOME}/.docker/config.json
* /.docker/config.json
* {--root-dir:-/var/lib/kubelet}/.dockercfg
* {cwd of kubelet}/.dockercfg
* ${HOME}/.dockercfg
* /.dockercfg

注意：您可能需要在 kubelet 进程的环境中显式设置 HOME=/root。

以下是配置节点以使用私有仓库的推荐步骤。 在本例中，在您的台式机/笔记本电脑上运行这些：
1. 未每个你想使用的认证运行`docker login [server]`。这会更新你电脑上的`$HOME/.docker/config.json`文件。
2. 在记事本中查看`$HOME/.docker/config.json`，确保只包含你想使用的证书。
3. 获取你的节点的列表，例如：
  * 如果你想要名称 nodes=$( kubectl get nodes -o jsonpath='{range.items[*].metadata}{.name} {end}' )
  * 如果你想要IP地址 nodes=$( kubectl get nodes -o jsonpath='{range .items[*].status.addresses[?(@.type=="ExternalIP")]}{.address} {end}' )
4. 复制本地的`.docker/config.json`到以上列出的路径之一
  * 例如，for n in $nodes; do scp ~/.docker/config.json root@"$n":/var/lib/kubelet/config.json; done

通过创建一个pod来验证使用私有镜像，例如：
```shell
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: private-image-test-1
spec:
  containers:
    - name: uses-private-image
      image: $PRIVATE_IMAGE_NAME
      imagePullPolicy: Always
      command: [ "echo", "SUCCESS" ]
EOF
```
如果一切正常，一段时间后，你可运行：`kubectl logs private-image-test-1`，然后看到命令行输出`SUCCESS`，如果你认为命令失败，你可以运行：`kubectl describe pods/private-image-test-1 | grep 'Failed'`，如果出现了错误，那么输出类似于：
```code
Fri, 26 Jun 2015 15:36:13 -0700    Fri, 26 Jun 2015 15:39:13 -0700 19 {kubelet node-i2hq}    spec.containers{uses-private-image}    failed     Failed to pull image "user/privaterepo:v1": Error: image user/privaterepo:v1 not found
```

您必须确保集群中的所有节点都具有相同的 .docker/config.json。 否则，Pod 将在某些节点上运行而无法在其他节点上运行。 例如，如果您使用节点自动缩放，则每个实例模板都需要包含 .docker/config.json 或挂载包含它的驱动器。将私有仓库添加到 .docker/config.json 后，所有 pod 都可以读取任何私有仓库中的图像。

### 提前拉取镜像

注意：如果您可以控制节点配置，则此方法适用。 如果您的云提供商管理节点并自动替换它们，它将无法可靠地工作。
默认情况下，kubelet 会尝试从指定的仓库中拉取每个镜像。 但是，如果容器的 imagePullPolicy 属性设置为 IfNotPresent 或 Never，则使用本地镜像（分别优先或专门）。如果您想依靠预拉镜像来代替私有仓库身份验证，您必须确保集群中的所有节点都具有相同的预拉镜像。这可用于预加载某些镜像以提高速度或作为对私有仓库进行身份验证的替代方法。所有 pod 都可以读取任何预先提取的镜像。

### 在 Pod 上指定 imagePullSecrets

注意：这是基于私有仓库的映像运行容器的推荐方法。
kubernetes支持在pod中指定容器镜像仓库密钥。

#### 使用 Docker 配置创建 Secret

运行以下命令，替换大写字母：
```code
kubectl create secret docker-registry <name> --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
```

如果您已经有一个 Docker 凭证文件，那么您可以将凭证文件作为 Kubernetes Secrets 导入，而不是使用上述命令。

如果您使用多个私有仓库，这尤其有用，因为 kubectl create secret docker-registry 创建一个仅适用于单个仓库的 Secret。

注意：Pod 只能在自己的命名空间中引用镜像 pull secret，所以这个过程需要每个命名空间做一次。

#### 在Pod上引用imagePullSecrets

现在，您可以通过将 imagePullSecrets 部分添加到 Pod 定义来创建引用该密钥的 Pod。例如：
```shell
cat <<EOF > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: foo
  namespace: awesomeapps
spec:
  containers:
    - name: foo
      image: janedoe/awesomeapp:v1
  imagePullSecrets:
    - name: myregistrykey
EOF

cat <<EOF >> ./kustomization.yaml
resources:
- pod.yaml
EOF
```
这需要为使用私有仓库的每个 pod 完成。但是，可以通过在 ServiceAccount 资源中设置 imagePullSecrets 来自动设置此字段。

您可以将它与每个节点的 .docker/config.json 结合使用。 凭据将被合并。

## 使用案例

有许多用于配置私有仓库的解决方案。 以下是一些常见用例和建议的解决方案。

1. 集群仅运行在非私有镜像上，无需隐藏镜像
  * 使用Docker hub的公有镜像
    * 无需配置
    * 一些云服务提供商自动缓存或者镜像了公有镜像，这提升了可用性，以及减少了拉取镜像的时间
2. 集群运行一些私有镜像，这些镜像应该对公司外部人员隐藏，但是对集群用户可见。
  * 使用托管的Docker仓库
    * 可能被托管在Docker hub，或者其他地方
    * 如上文描述的配置每个节点的`.docker/config.json`文件
  * 或者，运行一个防火墙后的私有仓库，开放所有读取权限
    * 无需配置kubernetes
  * 使用托管的镜像仓库服务，并开启权限控制
    * 配合集群自动扩容会比手动配置节点更好
  * 或者，在一个不方便修改节点配置的集群中，使用`imagePullSecrets`
3. 集群使用私有仓库，一部分需要更严格的访问控制
  * 确保AlwaysPullImage管理控制类激活。否则所有的Pods可能拥有针对所有镜像的所有权限。
  * 敏感信息迁移到Secret资源中，而不是打包在镜像中
4. 一个多租户集群，其中每个租户都需要自己的私有仓库
  * 确保AlwaysPullImage管理控制类开启，否则所有的Pods可能拥有针对所有镜像的所有权限
  * 运行一个需要认证的私有仓库
  * 为每个租户生成私有密钥，分发secret到每个租户的namespace
  * 租户添加secret到每个namespace的imagePullSecrets

如果你需要访问多个仓库，你可以为每个仓库创建一个secret，kubelet会合并任意imagePullSecrets到一个单独的虚拟`.docker/config.json`

## 接下来

* 阅读[OCI镜像标记文档](https://github.com/opencontainers/image-spec/blob/master/manifest.md)


