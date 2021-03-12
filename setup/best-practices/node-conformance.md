# 校验节点部署


## 节点一致性测试

节点一致性测试是一个容器化的测试框架，可为节点提供系统验证和功能测试。 该测试将验证该节点是否满足Kubernetes的最低要求； 通过测试的节点有资格加入Kubernetes集群。

## 节点先觉条件

要想运行节点一致性测试，节点必须满足于标准化k8s节点相同的先决条件。至少，该节点需要运行以下守护程序：
* Container Runtime
* Kubelet

## 运行节点一致性测试

运行节点一致性测试，请按照以下步骤操作：
1. 确认kubelet的`--kubeconfig`值；比如，指定`--kubeconfig=/var/lib/kubelet/config.yaml`。原因在于测试框架使用本地控制平面来测试kubelet，使用`http://localhost:8080`作为APIServer的地址。其他可能用到的kubelet命令行参数如下：
* `--pod-cidr`: 如果使用了`kubenet`，需要为kubelet指定一个随机的CIDR，例如：`--pod-cidr=10.180.0.0/24`。
* `--cloud-provider`：如果使用了参数`--cloud-provider=gce`，运行前需要移除该参数。
2. 运行以下命令进行测试：

```code
# $CONFIG_DIR is the pod manifest path of your Kubelet.
# $LOG_DIR is the test output path.
sudo docker run -it --rm --privileged --net=host \
  -v /:/rootfs -v $CONFIG_DIR:$CONFIG_DIR -v $LOG_DIR:/var/result \
  k8s.gcr.io/node-test:0.2
```

## 其他架构下的一致性测试

k8s为其他架构提供了一致性测试docker镜像：

arch| image
--- | ---
amd64 | node-test-amd64
arm | node-test-arm
arm64 | node-test-arm64

## 运行部分测试

要运行特定的测试，请用要运行的测试的正则表达式覆盖环境变量`FOCUS`:

```code
sudo docker run -it --rm --privileged --net=host \
  -v /:/rootfs:ro -v $CONFIG_DIR:$CONFIG_DIR -v $LOG_DIR:/var/result \
  -e FOCUS=MirrorPod \ # Only run MirrorPod test
  k8s.gcr.io/node-test:0.2
```

要跳过特定的测试，请用要跳过的测试的正则表达式覆盖环境变量`SKIP`:

```code
sudo docker run -it --rm --privileged --net=host \
  -v /:/rootfs:ro -v $CONFIG_DIR:$CONFIG_DIR -v $LOG_DIR:/var/result \
  -e SKIP=MirrorPod \ # Run all conformance tests but skip MirrorPod test
  k8s.gcr.io/node-test:0.2
```

节点一致性测试是节点e2e测试的容器化版本。默认情况下，它将运行所有一致性测试。从理论上讲，如果您配置了容器并正确安装了所需的卷，则可以运行任何节点e2e测试。 但强烈建议仅运行一致性测试，因为它需要更复杂的配置才能运行不一致性测试。

## 注意事项

* 该项测试会在节点上留下docker镜像，包含一致性测试镜像和功能测试的容器
* 测试会在节点上留下停止运行的容器，这些容器在功能测试的过程中创建。