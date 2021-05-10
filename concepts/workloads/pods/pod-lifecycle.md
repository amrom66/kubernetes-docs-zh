# 容器组声明周期

本页介绍pod的生命周期。pods遵循一个定义好的生命周期，一开始是`Pending`,如果至少一个主容器shunli启动，则转为`Running`,然后进入`Succeeded`或者`Succeeded`状态，这取决于pods中是否有容器处于失败状态。

在运行Pod的同时，kubelet能够重新启动容器以处理某些类型的错误。 在Pod中，Kubernetes跟踪不同的容器状态，并确定采取何种措施使Pod再次健康。

在Kubernetes API中，Pod具有规范和实际状态。 Pod对象的状态由一组Pod条件组成。 您也可以将自定义准备信息插入Pod的条件数据中，如果这对您的应用程序有用的话。

pods在其生命周期内仅调度一次。 将Pod调度（分配）到某个节点后，该Pod将在该Node上运行，直到其停止或终止。

## pod生命周期

类似独立的应用容器，Pods被认为是相对短暂的（而不是持久的）实体。 创建Pod，为其分配唯一ID（UID），并将其调度到保留它们的节点，直到终止（根据重新启动策略）或删除为止。 如果某个节点死亡，则安排在该超时时间段后删除该节点的Pod。

pods本身无法自我修复。 如果将Pod调度到随后发生故障的节点，则Pod将被删除；否则，该Pod将被删除。 同样，由于缺乏资源或Node维护，Pod无法幸免。 Kubernetes使用称为控制器的更高级别的抽象，它处理管理相对易用的Pod实例的工作。

给定的Pod（由UID定义）永远不会“重新安排”到其他节点； 取而代之的是，该Pod可以替换为一个新的，几乎相同的Pod，如果需要，甚至可以使用相同的名称，但使用不同的UID。

如果说某个事物与Pod具有相同的生存期（例如体积），则表示事物存在的时间与该特定Pod（具有确切的UID）存在的时间一样长。 如果出于任何原因删除了该Pod，即使创建了相同的替换，也将销毁相关对象（在本示例中为卷）并重新创建。

![示意图](https://d33wubrfki0l68.cloudfront.net/aecab1f649bc640ebef1f05581bfcc91a48038c4/728d6/images/docs/pod.svg)

一个多容器Pod，其中包含一个文件提取器和一个Web服务器，该Web服务器使用持久卷进行容器之间的共享存储。

## pod状态

Pod的状态字段是PodStatus对象，该对象具有一个状态字段。

Pod的阶段是Pod在其生命周期中所处位置的简单概括。该阶段既不打算是对容器状态或Pod状态的观察的全面汇总，也不是要成为全面的状态机。

Pod状态的数量和含义受到严格保护。 除了此处记录的内容外，对于具有给定状态的Pod，不应做任何假设。

以下是状态的一些可选值：
值|描述
---|---
Pending|kubernetes集群接受该pod，但是一个或者多个容器并未准备好。这包括Pod等待排定所花费的时间，以及通过网络下载容器映像所花费的时间。
Running|pod已经绑定到节点。并且所有容器都已经启动，至少一个容器处在运行中，或者处在启动中和重启中。
Succeeded|pod中的所有容器进入销毁，并且不会在此启动。
Failed|Pod中的所有容器均已终止，并且至少一个容器因故障而终止。 也就是说，容器要么以非零状态退出，要么被系统终止。
Unknown|由于某种原因，无法获得Pod的状态。 此阶段通常是由于与Pod应该在其中运行的节点通信时发生错误而发生的。

注意：删除Pod时，某些kubectl命令将其显示为“正在终止”。 此终止状态不是Pod阶段之一。 授予Pod适当终止的期限，默认为30秒。 您可以使用标志--force强制终止Pod。

如果某个节点死亡或与集群的其余部分断开连接，Kubernetes将应用一项策略，将丢失的节点上所有Pod的状态设置为失败。


## 容器状态

除了Pod的总体阶段之外，Kubernetes还会跟踪Pod内每个容器的状态。 您可以使用容器生命周期挂钩来触发事件，以在容器生命周期中的某些时间点运行。

调度程序将Pod分配给节点后，kubelet将开始使用容器运行时为该Pod创建容器。 有三种可能的容器状态：等待，运行和终止。

要检查Pod容器的状态，可以使用kubectl describe pod <name-of-pod>。 输出显示该Pod中每个容器的状态。

每个状态都有特定的含义：
* Waiting

如果容器未处于“正在运行”或“已终止”状态，则为“正在等待”。 处于等待状态的容器仍在运行，以完成启动所需的操作：例如，从容器映像注册表中提取容器映像，或应用密钥数据。 当您使用kubectl来查询带有Waiting容器的Pod时，您还会看到一个Reason字段来总结为什么容器处于该状态。

* Running

正在运行状态表示容器正在执行，没有任何问题。 如果配置了postStart挂钩，则它已经执行并完成。 当您使用kubectl查询带有正在运行的容器的Pod时，您还将看到有关容器何时进入运行状态的信息。

* Terminated

处于“已终止”状态的容器开始执行，然后运行完成或由于某种原因而失败。 当您使用kubectl查询带有Terminated容器的Pod时，您会看到一个原因，一个退出代码以及该容器执行期间的开始和结束时间。

如果容器配置了preStop挂钩，则该挂钩将在容器进入“已终止”状态之前运行。

## 容器重启策略

Pod的规范包含一个restartPolicy字段，其可能值为Always，OnFailure和Never。 默认值为始终。

restartPolicy适用于Pod中的所有容器。 restartPolicy仅指通过kubelet在同一节点上重新启动容器。 在Pod出口中的容器退出后，kubelet会以指数级的退避延迟（10s，20s，40s等）重新启动它们，上限为5分钟。 一旦容器执行了10分钟而没有任何问题，kubelet将重置该容器的重启退避计时器。

## pod状况

一个Pod具有PodStatus，该PodStatus具有PodConditions数组，该Pod通过或未通过PodConditions：

* PodScheduled：已将Pod调度到一个节点。
* ContainersReady：Pod中的所有容器均已准备就绪。
* Initialized：所有初始化容器已成功启动。
* Ready：该Pod能够处理请求，应将其添加到所有匹配服务的负载平衡池中。

字段名称|描述
---|---
type|pod状况的名称
status|指示该条件是否适用，可能的值为“ True”，“ False”或“ Unknown”。
lastProbeTime|上一次探测Pod条件的时间戳。
lastTransitionTime|Pod上一次从一种状态转换为另一种状态的时间戳。
reason|机器语言、大写驼峰字母文本，包含上次状态转换的原因
message|人类可读的消息，指示有关最后状态转换的详细信息。

## pod准备就绪

您的应用程序可以向PodStatus：Pod准备就绪中注入额外的反馈或信号。 要使用此功能，请在Pod的规范中设置readinessGates，以指定kubelet为Pod准备状态评估的其他条件列表。

准备就绪取决于Pod的status.condition字段的当前状态。 如果Kubernetes在Pod的status.conditions字段中找不到这样的条件，则该条件的状态默认为“ False”。

以下示例：
```yaml
kind: Pod
...
spec:
  readinessGates:
    - conditionType: "www.example.com/feature-1"
status:
  conditions:
    - type: Ready                              # a built in PodCondition
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
    - type: "www.example.com/feature-1"        # an extra PodCondition
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
  containerStatuses:
    - containerID: docker://abcd...
      ready: true
...
```
您添加的Pod条件必须具有符合Kubernetes标签密钥格式的名称。

### pod就绪的状态

kubectl patch命令不支持修补对象状态。 要为Pod设置这些状态条件，应用程序和操作员应使用PATCH操作。 您可以使用Kubernetes客户端库编写为Pod准备状态设置自定义Pod条件的代码。

对于使用自定义条件的Pod，仅当以下两个状态均适用时，该Pod才被评估为就绪：
* pod中的所有容器准备就绪
* 所有定义了readinessGates状态的都为true

当Pod的容器准备就绪，但至少缺少一个自定义条件或False时，kubelet将Pod的条件设置为ContainersReady。

## 容器探针

探针是由容器上的kubelet定期执行的诊断。 为了执行诊断，kubelet调用由容器实现的Handler。 有三种类型的处理程序：
* ExecAction：在容器内执行指定的命令。 如果命令以状态码0退出，则认为诊断成功。
* TCPSocketAction：对指定端口上Pod的IP地址执行TCP检查。 如果端口打开，则认为诊断成功。
* HTTPGetAction：对指定端口和路径上Pod的IP地址执行HTTP GET请求。 如果响应的状态码大于或等于200且小于400，则认为诊断成功。

每个探针有3种结果：
* Success：容器通过了诊断。
* Failure：容器诊断失败
* Unknown：诊断失败，因此不应采取任何措施。

Kubelet可以选择对正在运行的容器执行三种探测并对其做出反应：
* livenessProbe：指示容器是否正在运行。 如果活动性探针失败，则kubelet将杀死该容器，并且该容器将接受其重新启动策略。 如果容器不提供活动性探针，则默认状态为“成功”。
* readinessProbe：指示容器是否准备好响应请求。 如果就绪探针失败，则端点控制器将从与Pod匹配的所有服务的端点中删除Pod的IP地址。 初始延迟之前的默认就绪状态为“失败”。 如果容器未提供就绪探测器，则默认状态为“成功”。
* startupProbe：指示是否启动容器中的应用程序。 如果提供了启动探针，则将禁用所有其他探针，直到成功为止。 如果启动探针失败，则kubelet将杀死该容器，并且该容器将接受其重新启动策略。 如果容器未提供启动探针，则默认状态为“成功”。

有关如何设置活动性，就绪性或启动探针的更多信息，请参阅配置活动性，就绪性和启动探针。

## 何时适用存活探针

如果您的容器中的过程在遇到问题或变得不正常时能够自行崩溃，则您不一定需要使用活动性探针；否则，无需进行任何操作。 kubelet将根据Pod的restartPolicy自动执行正确的操作。

如果您希望您的容器在探测失败时被杀死并重新启动，请指定一个活动性探测，并指定“ Always”或“ OnFailure”的restartPolicy。


## 何时适用就绪探针

如果您仅想在探测成功后才开始向Pod发送流量，请指定就绪探测器。 在这种情况下，就绪探针可能与活动探针相同，但是规范中存在就绪探针意味着Pod将在不接收任何流量的情况下启动，并且仅在探针开始成功之后才开始接收流量。 如果您的容器需要在启动过程中加载大型数据，配置文件或迁移，请指定准备情况探针。

如果希望容器能够拆卸下来进行维护，则可以指定一个就绪探针，以检查特定于与活跃探针不同的就绪端点。

注意：如果希望在删除Pod时能够清空请求，则不一定需要准备就绪探针；否则，无需进行任何准备。 删除后，无论是否准备就绪探针，Pod都会自动将自己置于未就绪状态。 在等待Pod中的容器停止时，Pod仍处于未就绪状态。

## 何时适用启动探针

对于具有需要很长时间才能投入使用的容器的Pod，启动探针非常有用。 您可以配置一个单独的配置来探测容器启动时的情况，而不是设置较长的活动时间间隔，从而允许它花费比活动时间间隔所允许的时间更长的时间。

如果您的容器通常在超过initialDelaySeconds + failureThreshold×periodSeconds的时间内启动，则应指定一个启动探针，该探针检查与活动探针相同的终结点。 periodSeconds的默认值为10s。 然后，应将其failureThreshold设置得足够高，以允许容器启动，而不更改活动性探针的默认值。 这有助于防止死锁。

## pod的销毁

因为Pod表示正在集群中的节点上运行的进程，所以重要的是，当不再需要它们时，请允许这些进程正常终止（而不是通过KILL信号突然停止并且没有机会进行清理）。

设计目标是使您能够请求删除并知道进程何时终止，而且还可以确保删除最终完成。 当您请求删除Pod时，集群会记录并跟踪允许的Pod被强制杀死之前的预定宽限期。 有了适当的强制关闭跟踪，kubelet就会尝试正常关闭。

通常，容器运行时将TERM信号发送到每个容器中的主进程。 许多容器运行时都遵循容器映像中定义的STOPSIGNAL值，并发送该值而不是TERM。 一旦宽限期到期，就会将KILL信号发送到所有剩余的进程，然后从API服务器中删除Pod。 如果在等待进程终止时重新启动了kubelet或容器运行时的管理服务，则群集将从一开始重试，包括完整的原始宽限期。

以下是一个流程：

1. 您可以使用kubectl工具以默认的宽限期（30秒）手动删除特定的Pod。
2. API服务器中的Pod将使用宽限期来更新，超过该时间Pod被视为“死亡”。 如果您使用kubectl describe检查要删除的Pod，则该Pod会显示为“正在终止”。 在Pod运行的节点上：kubelet看到Pod已被标记为终止（设置了正常关闭持续时间）后，kubelet便开始本地Pod关闭过程。
    1. 如果Pod的一个容器定义了preStop挂钩，则kubelet会在容器内部运行该挂钩。 如果宽限期到期后preStop挂钩仍在运行，则kubelet要求一次性将宽限期延长2秒。
    2. kubelet触发容器运行时以向每个容器内的进程1发送TERM信号。
3. 在kubelet开始正常关闭的同时，控制平面将从终结点（以及启用了EndpointSlice）的对象中删除该关闭Pod，在这些对象中，这些对象表示已配置选择器的服务。 副本集和其他工作负载资源不再将关闭的Pod视为有效的服务中副本。 终止宽限期一开始，缓慢关闭的Pod就无法继续为流量提供服务，因为负载平衡器（如服务代理）会从端点列表中删除Pod。
4. 当宽限期到期时，kubelet会触发强制关闭。 容器运行时将SIGKILL发送到仍在Pod中任何容器中运行的任何进程。 如果该容器运行时使用一个隐藏的暂停容器，则kubelet还会清除该容器。
5. 通过将宽限期设置为0（立即删除），kubelet触发了从API服务器强制删除Pod对象的操作。
6. API服务器删除Pod的API对象，然后该对象不再从任何客户端可见。

## 强制删除pod

注意：强制删除可能会对某些工作负载及其Pod造成破坏。

默认情况下，所有删除均会在30秒内正常显示。 kubectl delete命令支持--grace-period = <seconds>选项，该选项可让您覆盖默认值并指定您自己的值。

将宽限期强行设置为0，并立即从API服务器中删除Pod。 如果pod仍在节点上运行，则强制删除会触发kubelet开始立即清除。

注意：必须指定附加标志--force和--grace-period = 0才能执行强制删除。

当执行强制删除时，API服务器不会等待来自kubelet的确认，即Pod已在其运行的节点上终止。 它会立即删除API中的Pod，以便可以使用相同的名称创建一个新的Pod。 在该节点上，被设置为立即终止的Pod在被强制杀死之前，仍将给予一个小的宽限期。

如果需要强制删除StatefulSet中的Pod，请参阅任务文档以从StatefulSet中删除Pod。

## 失败pod的垃圾回收

对于失败的Pod，API对象将保留在集群的API中，直到人工或控制器进程将其删除为止。

当Pod的数量超过配置的阈值（由kube-controller-manager中的Terminate-pod-gc-threshold确定）时，控制平面将清理终止的Pod（阶段为成功或失败）。 这样可以避免资源泄漏，因为Pod会随着时间的推移创建和终止。

