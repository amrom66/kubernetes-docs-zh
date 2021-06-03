# 临时容器

本页提供临时容器的概况：一个特殊类型的容器，在存在的pod中临时运行，以配合用户初始化动作，例如问题解决。你可以使用临时容器来监视服务，而不是构建应用。

警告：临时容器处于早期 alpha 状态，不适合生产集群。 根据 Kubernetes 弃用政策，此 alpha 功能将来可能会发生重大变化或完全删除。

## 理解临时容器

Pod 是 Kubernetes 应用程序的基本构建块。尽管pod旨在变得一次性的和可替换的，你不可以在pod刚创建时候放入容器。作为替代，你通常应该使用deployments来删除和替换pod。

然而，有时需要检查现有Pod的状态，例如对难以重现的错误进行故障排除。 在这些情况下，您可以在现有 Pod 中运行临时容器来检查其状态并运行任意命令。

## 什么是一个临时容器

临时容器与其他容器的不同之处在于它们缺乏对资源或执行的保证，并且永远不会自动重启，因此它们不适合构建应用程序。临时容器使用与常规容器相同的 ContainerSpec 进行描述，但许多字段不兼容并且不允许用于临时容器。

* 临时容器可能没有端口，因此不允许使用端口、livenessProbe、readinessProbe 等字段
* Pod 资源分配是不可变的，因此不允许设置资源。
* 有关允许字段的完整列表，请参阅 EphemeralContainer 参考文档。

临时容器是使用 API 中的特殊 ephemeralcontainers 处理程序创建的，而不是直接将它们添加到 pod.spec，因此无法使用 kubectl edit 添加临时容器。
与常规容器一样，在将临时容器添加到 Pod 后，您不得更改或删除它。

## 使用临时容器

当 kubectl exec 因容器崩溃或容器映像不包含调试实用程序而不足时，临时容器可用于交互式故障排除。

特别是，[distroless](https://github.com/GoogleContainerTools/distroless) 镜像使您能够部署最少的容器镜像，从而减少攻击面和漏洞和漏洞的暴露。 由于 distroless 映像不包含 shell 或任何调试实用程序，因此单独使用 kubectl exec 很难对 distroless 映像进行故障排除。

使用临时容器时，启用进程命名空间共享会很有帮助，这样您就可以查看其他容器中的进程。

## 临时容器API

本节中的示例演示了临时容器在 API 中的显示方式。 您通常会使用 kubectl debug 或其他 kubectl 插件来自动化这些步骤，而不是直接调用 API。

临时容器是使用 Pod 的 ephemeralcontainers 子资源创建的，可以使用 kubectl --raw 进行演示。 首先描述要添加为 EphemeralContainers 列表的临时容器：

```json
{
    "apiVersion": "v1",
    "kind": "EphemeralContainers",
    "metadata": {
        "name": "example-pod"
    },
    "ephemeralContainers": [{
        "command": [
            "sh"
        ],
        "image": "busybox",
        "imagePullPolicy": "IfNotPresent",
        "name": "debugger",
        "stdin": true,
        "tty": true,
        "terminationMessagePolicy": "File"
    }]
}
```

要更新已经运行的 example-pod 的临时容器：

```code
kubectl replace --raw /api/v1/namespaces/default/pods/example-pod/ephemeralcontainers  -f ec.json
```

这将返回临时容器的新列表：

```json
{
   "kind":"EphemeralContainers",
   "apiVersion":"v1",
   "metadata":{
      "name":"example-pod",
      "namespace":"default",
      "selfLink":"/api/v1/namespaces/default/pods/example-pod/ephemeralcontainers",
      "uid":"a14a6d9b-62f2-4119-9d8e-e2ed6bc3a47c",
      "resourceVersion":"15886",
      "creationTimestamp":"2019-08-29T06:41:42Z"
   },
   "ephemeralContainers":[
      {
         "name":"debugger",
         "image":"busybox",
         "command":[
            "sh"
         ],
         "resources":{

         },
         "terminationMessagePolicy":"File",
         "imagePullPolicy":"IfNotPresent",
         "stdin":true,
         "tty":true
      }
   ]
}
```

您可以使用 kubectl describe 查看新创建的临时容器的状态：

```code
kubectl describe pod example-pod
```

```code
...
Ephemeral Containers:
  debugger:
    Container ID:  docker://cf81908f149e7e9213d3c3644eda55c72efaff67652a2685c1146f0ce151e80f
    Image:         busybox
    Image ID:      docker-pullable://busybox@sha256:9f1003c480699be56815db0f8146ad2e22efea85129b5b5983d0e0fb52d9ab70
    Port:          <none>
    Host Port:     <none>
    Command:
      sh
    State:          Running
      Started:      Thu, 29 Aug 2019 06:42:21 +0000
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:         <none>
...

```

您可以使用 kubectl attach、kubectl exec 和 kubectl 日志以与其他容器相同的方式与新的临时容器交互，例如：

```code
kubectl attach -it example-pod -c debugger
```
