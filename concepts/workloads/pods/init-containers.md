# 初始化容器

Pod中可以有多个运行应用程序的容器，但也可以有一个或多个init容器，这些容器在启动应用程序容器之前就已运行。

初始化容器与常规容器完全一样，除了以下：
* 初始化容器始终会运行到完成状态。
* 每个init容器必须成功完成才能启动下一个容器。

如果Pod的初始化容器失败，则kubelet会反复重新启动该初始化容器，直到成功为止。 但是，如果Pod的重启策略为Never，并且初始化容器在该Pod启动期间失败，则Kubernetes会将整个Pod视为失败。

要为Pod指定初始化容器，请将initContainers字段添加到Pod规范中，作为类型为Container的对象的数组，与app container数组一起。 初始化容器的状态在.status.initContainerStatuses字段中作为容器状态的数组返回（类似于.status.containerStatuses字段）。

## 与常规容器的区别

初始化容器支持应用容器的所有字段和功能，包括资源限制，数量和安全设置。 但是，如参考资料中所述，对初始化容器的资源请求和限制的处理方式有所不同。

另外，初始化容器不支持生命周期，livenessProbe，readinessProbe或startupProbe，因为它们必须运行到Pod就绪才能完成。

如果您为Pod指定了多个初始化容器，则kubelet会依次运行每个初始化容器。 每个初始化容器必须成功，然后才能运行下一个容器。 当所有初始化容器都运行完毕后，kubelet会初始化Pod的应用程序容器并像往常一样运行它们。

## 使用初始化容器

因为初始化容器的图像与应用程序容器的图像是分开的，所以它们在启动相关代码方面具有一些优势：
* 初始化容器可以包含应用程序镜像中不存在的实用程序或用于设置的自定义代码。 例如，无需在安装过程中仅使用sed，awk，python或dig之类的工具从另一张镜像制作镜像。

* 应用镜像构建器和部署者角色可以独立工作，而无需构建成一个镜像

* 初始化容器可以在与同一Pod中的应用程序容器不同的文件系统视图下运行。 因此，可以授予他们访问应用程序容器无法访问的机密的权限。

* 由于init容器在任何应用程序容器启动之前便已运行完毕，因此init容器提供了一种机制来阻止或延迟应用程序容器的启动，直到满足一组先决条件为止。 满足先决条件后，即可并行启动Pod中的所有应用程序容器。

* 初始化容器可以安全地运行实用程序或自定义代码，否则它们会使应用程序镜像的安全性降低。 通过将不必要的工具分开，您可以限制应用程序容器映像的攻击面。

### 示例

以下示例初始化容器的使用：

* 等待一个服务被创建，使用类似于以下的命令行shell：
    ```code
    for i in {1..100}; do sleep 1; if dig myservice; then exit 0; fi; done; exit 1
    ```

* pod注册到远程API上，使用如下命令：
    ```code
    curl -X POST http://$MANAGEMENT_SERVICE_HOST:$MANAGEMENT_SERVICE_PORT/register -d 'instance=$(<POD_NAME>)&ip=$(<POD_IP>)'
    ```

* 启动容器前等待一段时间，如下：
    ```code
    sleep 60
    ```
* 下载git仓库到存储卷中

* 将值放入配置文件中，然后运行模板工具为主应用程序容器动态生成配置文件。 例如，将POD_IP值放置在配置中，然后使用Jinja生成主应用程序配置文件。

### 使用初始化容器

本示例定义了一个具有两个init容器的简单Pod。 第一个等待myservice，第二个等待mydb。 一旦两个初始化容器都完成，则Pod将从其spec部分运行app容器。

```code
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
```

使用以下命令部署pod：
```code
kubectl apply -f myapp.yaml
```

输出类似于：
```code
pod/myapp-pod created
```

通过以下命令检查状态：
```code
kubectl get -f myapp.yaml
```

输出类似如下：
```code
NAME        READY     STATUS     RESTARTS   AGE
myapp-pod   0/1       Init:0/2   0          6m
```

查看更多详情：
```code
kubectl describe -f myapp.yaml
```

输出类似如下：
```code
Name:          myapp-pod
Namespace:     default
[...]
Labels:        app=myapp
Status:        Pending
[...]
Init Containers:
  init-myservice:
[...]
    State:         Running
[...]
  init-mydb:
[...]
    State:         Waiting
      Reason:      PodInitializing
    Ready:         False
[...]
Containers:
  myapp-container:
[...]
    State:         Waiting
      Reason:      PodInitializing
    Ready:         False
[...]
Events:
  FirstSeen    LastSeen    Count    From                      SubObjectPath                           Type          Reason        Message
  ---------    --------    -----    ----                      -------------                           --------      ------        -------
  16s          16s         1        {default-scheduler }                                              Normal        Scheduled     Successfully assigned myapp-pod to 172.17.4.201
  16s          16s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Pulling       pulling image "busybox"
  13s          13s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Pulled        Successfully pulled image "busybox"
  13s          13s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Created       Created container with docker id 5ced34a04634; Security:[seccomp=unconfined]
  13s          13s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Started       Started container with docker id 5ced34a04634
```

查看初始化容器的日志：
```code
kubectl logs myapp-pod -c init-myservice # Inspect the first init container
kubectl logs myapp-pod -c init-mydb      # Inspect the second init container
```

此时，初始化容器正在等待发现服务mydb和myservice。

使用以下配置使这两个服务显示出来：
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: myservice
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
---
apiVersion: v1
kind: Service
metadata:
  name: mydb
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9377
```

```code
kubectl apply -f services.yaml
```

....

## 行为的细节

在Pod启动期间，kubelet会延迟运行init容器，直到网络和存储就绪为止。 然后，kubelet按照Pod规范中出现的顺序运行Pod的init容器。

每个初始化容器必须在下一个容器启动之前成功退出。 如果容器由于运行时而无法启动或因失败而退出，则会根据Pod restartPolicy对其进行重试。 但是，如果Pod restartPolicy设置为Always，则初始化容器将使用restartPolicy OnFailure。

在所有初始化容器都成功之前，Pod不能准备就绪。 初始化容器上的端口未在服务下聚合。 正在初始化的Pod处于Pending状态，但应将条件Initialized设置为false。

如果Pod重新启动或重新启动，则所有初始化容器都必须再次执行。

对初始容器规范的更改仅限于容器映像字段。 更改初始化容器映像字段等同于重新启动Pod。

因为初始化容器可以重新启动，重试或重新执行，所以初始化容器代码应该是幂等的。 特别是，应准备在EmptyDirs上写入文件的代码，以防止输出文件已经存在。

初始化容器具有应用程序容器的所有字段。 但是，Kubernetes禁止使用readinessProbe，因为初始化容器无法定义与完成不同的就绪状态。 这是在验证期间强制执行的。

使用Pod上的activeDeadlineSeconds和容器上的livenessProbe可以防止初始化容器永远失败。 有效期限包括初始化容器。

Pod中每个应用程序和init容器的名称必须唯一； 任何与另一个共享名称的容器都会引发验证错误。

## 资源

给定初始化容器的顺序和执行，适用以下资源使用规则：

* 在所有初始化容器上定义的任何特定资源请求或限制中的最高者是有效的初始化请求/限制
* Pod对资源的有效请求/限制是以下两者中的较高者：
    * 资源的所有应用容器请求/限制的总和
    * 资源的有效初始化请求/限制
* 调度是基于有效的请求/限制完成的，这意味着init容器可以为Pod生存期内未使用的初始化保留资源。
* Pod的有效QoS层的QoS（服务质量）层是用于初始化容器和应用程序容器的QoS层。

根据有效的Pod请求和限制应用配额和限制。

Pod级别控制组（cgroups）基于有效Pod请求和限制，与调度程序相同。

### pod重启的原因

pod可能重启，带来初始化容器的重新之心，有以下原因：

* Pod基础结构容器将重新启动。 这是不常见的，必须由具有根访问节点权限的人来完成。
* 在将restartPolicy设置为Always的同时，终止Pod中的所有容器，强制重新启动，并且由于垃圾回收而丢失了初始容器完成记录。

更改初始容器映像或由于垃圾回收而丢失了初始容器完成记录时，不会重新启动Pod。 这适用于Kubernetes v1.20及更高版本。 如果您使用的是Kubernetes的早期版本，请查阅所用版本的文档。
