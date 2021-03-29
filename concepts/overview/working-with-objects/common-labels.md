# 推荐的标签

您可以使用比kubectl和仪表板更多的工具来可视化和管理Kubernetes对象。 一组通用标签允许工具互操作，以所有工具都可以理解的通用方式描述对象。

除了支持工具外，推荐标签还以可查询的方式描述了应用程序。

元数据是围绕应用程序的概念组织的。 Kubernetes不是平台即服务（PaaS），并且不具有或不强制实施应用程序的正式概念。 相反，应用程序是非正式的，并使用元数据进行描述。 应用程序包含的内容的定义很松散。

注意：这些是推荐的标签。 它们使管理应用程序变得更加容易，但是任何核心工具都不是必需的。

共享的标签和注释共享一个公共前缀：app.kubernetes.io。 没有前缀的标签是用户专有的。 共享前缀可确保共享标签不会干扰自定义用户标签。


### 标签

为了充分利用这些标签，应该将它们应用于每个资源对象。

| Key                            | Description                                                  | Example        | Type   |
| ------------------------------ | ------------------------------------------------------------ | -------------- | ------ |
| `app.kubernetes.io/name`       | The name of the application                                  | `mysql`        | string |
| `app.kubernetes.io/instance`   | A unique name identifying the instance of an application     | `mysql-abcxzy` | string |
| `app.kubernetes.io/version`    | The current version of the application (e.g., a semantic version, revision hash, etc.) | `5.7.21`       | string |
| `app.kubernetes.io/component`  | The component within the architecture                        | `database`     | string |
| `app.kubernetes.io/part-of`    | The name of a higher level application this one is part of   | `wordpress`    | string |
| `app.kubernetes.io/managed-by` | The tool being used to manage the operation of an application | `helm`         | string |

为了说明这些标签的作用，请考虑以下StatefulSet对象：

```code
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxzy
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
    app.kubernetes.io/managed-by: helm
```

### 应用程序和应用程序实例

一个应用程序可以一次或多次安装到Kubernetes集群中，在某些情况下，可以安装在同一名称空间中。 例如，在不同的网站是WordPress的不同安装的地方，可以多次安装WordPress。

应用程序名称和实例名称分别记录。 例如，WordPress的app.kubernetes.io/name为wordpress，而实例名称为app.kubernetes.io/instance，其值为wordpress-abcxzy。 这使得应用程序和应用程序实例是可识别的。 应用程序的每个实例都必须具有唯一的名称。



### 示例

为了说明使用这些标签的不同方法，以下示例具有不同的复杂性。

### 一个简单的无状态服务

考虑使用Deployment和Service对象部署的简单无状态服务的情况。 以下两个片段代表如何以最简单的形式使用标签。

部署用于监视运行应用程序本身的容器。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: myservice
    app.kubernetes.io/instance: myservice-abcxzy
...
```

使用`Service`暴露应用：

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: myservice
    app.kubernetes.io/instance: myservice-abcxzy
...
```

### 带有数据库的web应用

考虑一个稍微复杂的应用程序：使用Helm安装的使用数据库（MySQL）的Web应用程序（WordPress）。 以下片段说明了用于部署此应用程序的对象的开始。

以下部署的开始用于WordPress：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: wordpress
    app.kubernetes.io/instance: wordpress-abcxzy
    app.kubernetes.io/version: "4.9.4"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: server
    app.kubernetes.io/part-of: wordpress
...
```

使用`Service`暴露应用：

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: wordpress
    app.kubernetes.io/instance: wordpress-abcxzy
    app.kubernetes.io/version: "4.9.4"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: server
    app.kubernetes.io/part-of: wordpress
...
```

MySQL作为带有状态数据的StatefulSet公开，它以及它所属的较大应用程序均带有：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxzy
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
...
```

使用`Service`将mysql暴露成wordpress的部分：

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxzy
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
...
```

使用MySQL StatefulSet和Service，您会注意到有关MySQL和WordPress（更广泛的应用程序）的信息。