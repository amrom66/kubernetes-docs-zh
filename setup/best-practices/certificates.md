# PKI证书问题

k8s需要基于TLS的PKI证书授权。如果您使用了kubeadm部署的k8s，集群会自动生成需要的证书。当然，你也可以自己手动创建证书。例如，为了保证私有证书更加安全，你可以将其不存放在API Server中。本文讲解集群需要的证书。

## 集群是如何使用证书的

k8s在以下操作的时候需要用到PKI认证：

* kubelet访问API Server需要客户端证书
* APIserver 端点需要服务端证书
* 集群管理员认证API server需要客户端证书
* API server与kubelet通信需要客户端证书
* API server与etcd通信需要客户端证书
* controller manager与API server通信时候需要客户端证书和kubeconfig
* scheduler与API server通信时候需要客户端证书和kubeconfig
* 前置代理需要客户端和服务端证书

**注意** 
前置代理需要证书的场景仅存在于运行kube-proxy支撑APIserver扩展的时候。

etcd在授权客户端和节点的时候也实现了一般的TLS认证

## 证书的存放位置

使用kubeadm安装k8s的时候，证书存放在`/etc/kubernetes/pki`，本文中提到的所有路径都是相对于该目录。

## 手动配置证书

如果你不想kubeadm自动生成证书的话，你可以使用以下几种方式创建：

### 单独根CA证书

你可以使用管理员创建一个根证书。然后，该根CA可以创建多个中间CA，并将所有进一步的创建委托给Kubernetes本身。
需要的CA：

路径 | 默认cn | 描述
--- | --- | ---
ca.crt,key | kubernetes-ca | Kubernetes general CA
etcd/ca.crt,key | etcd-ca | For all etcd-related functions
front-proxy-ca.crt,key | kubernetes-front-proxy-ca | For the front-end proxy

在上述CA之上，还需要获取用于服务帐户管理的公共/私有密钥对sa.key和sa.pub。

### 所有证书

如果你不想复制这些证书到你的集群中，你可以自己生成这些证书。

需要的证书如下：

默认CN | 父证书 | O | 类型 | hosts 
--- | --- | --- | --- | --- 
kube-etcd |	etcd-ca	|  | server, client |localhost, 127.0.0.1
kube-etcd-peer | etcd-ca |  | server, client | hostname, Host_IP, localhost, 127.0.0.1
kube-etcd-healthcheck-client | etcd-ca	| |	client	|
kube-apiserver-etcd-client | etcd-ca | system:masters | client |
kube-apiserver | kubernetes-ca |   | server | hostname, Host_IP, advertise_IP, [1]	
kube-apiserver-kubelet-client |	kubernetes-ca |	system:masters | client |
front-proxy-client | kubernetes-front-proxy-ca |   | client	|

[1]： 表示访问集群的其他任何IP或者DNS名称，其中kind映射到一种或多种x509密钥用法类型：

kind | 密钥用法
--- | ---
server | digital signature, key encipherment, server auth
client | digital signature, key encipherment, client auth

**注意**：上面列出的主机/SAN是获得工作群集的推荐主机； 如果特定设置需要，则可以在所有服务器证书上添加其他SAN。 
**注意**：对于kubeadm用户而言只需要如下：
* 在不使用私钥的情况下复制到群集CA证书的方案在kubeadm文档中称为外部CA。
* 如果将上面的列表与kubeadm生成的PKI进行比较，请注意，如果使用外部etcd，则不会生成kube-etcd，kube-etcd-peer和kube-etcd-healthcheck-client证书。

### 证书路径

证书应当存放在推荐的路径下，路径需要通过特定的参数指定而不是路径。

| Default CN                    | recommended key path         | recommended cert path        | command                 | key argument               | cert argument                                                |
| ----------------------------- | ---------------------------- | ---------------------------- | ----------------------- | -------------------------- | ------------------------------------------------------------ |
| etcd-ca                       | etcd/ca.key                  | etcd/ca.crt                  | kube-apiserver          |                            | --etcd-cafile                                                |
| kube-apiserver-etcd-client    | apiserver-etcd-client.key    | apiserver-etcd-client.crt    | kube-apiserver          | --etcd-keyfile             | --etcd-certfile                                              |
| kubernetes-ca                 | ca.key                       | ca.crt                       | kube-apiserver          |                            | --client-ca-file                                             |
| kubernetes-ca                 | ca.key                       | ca.crt                       | kube-controller-manager | --cluster-signing-key-file | --client-ca-file, --root-ca-file, --cluster-signing-cert-file |
| kube-apiserver                | apiserver.key                | apiserver.crt                | kube-apiserver          | --tls-private-key-file     | --tls-cert-file                                              |
| kube-apiserver-kubelet-client | apiserver-kubelet-client.key | apiserver-kubelet-client.crt | kube-apiserver          | --kubelet-client-key       | --kubelet-client-certificate                                 |
| front-proxy-ca                | front-proxy-ca.key           | front-proxy-ca.crt           | kube-apiserver          |                            | --requestheader-client-ca-file                               |
| front-proxy-ca                | front-proxy-ca.key           | front-proxy-ca.crt           | kube-controller-manager |                            | --requestheader-client-ca-file                               |
| front-proxy-client            | front-proxy-client.key       | front-proxy-client.crt       | kube-apiserver          | --proxy-client-key-file    | --proxy-client-cert-file                                     |
| etcd-ca                       | etcd/ca.key                  | etcd/ca.crt                  | etcd                    |                            | --trusted-ca-file, --peer-trusted-ca-file                    |
| kube-etcd                     | etcd/server.key              | etcd/server.crt              | etcd                    | --key-file                 | --cert-file                                                  |
| kube-etcd-peer                | etcd/peer.key                | etcd/peer.crt                | etcd                    | --peer-key-file            | --peer-cert-file                                             |
| etcd-ca                       |                              | etcd/ca.crt                  | etcdctl                 |                            | --cacert                                                     |
| kube-etcd-healthcheck-client  | etcd/healthcheck-client.key  | etcd/healthcheck-client.crt  | etcdctl                 | --key                      | --cert                                                       |

相同的注意事项适用于服务帐户密钥对：

| private key path | public key path | command                 | argument                           |
| ---------------- | --------------- | ----------------------- | ---------------------------------- |
| sa.key           |                 | kube-controller-manager | --service-account-private-key-file |
|                  | sa.pub          | kube-apiserver          | --service-account-key-file         |

## 用户账户的证书配置

你必须要配置这些管理员账户和服务账户：

| filename                | credential name            | Default CN                          | O (in Subject) |
| ----------------------- | -------------------------- | ----------------------------------- | -------------- |
| admin.conf              | default-admin              | kubernetes-admin                    | system:masters |
| kubelet.conf            | default-auth               | system:node:`<nodeName>` (see note) | system:nodes   |
| controller-manager.conf | default-controller-manager | system:kube-controller-manager      |                |
| scheduler.conf          | default-scheduler          | system:kube-scheduler               |                |

**注意**：kubelet.conf的<nodeName>的值必须与kubelet向apiserver注册时提供的节点名称的值完全匹配。 有关更多详细信息，请阅读节点授权。

1. 对于每个配置，请使用给定的CN和O生成一个x509证书/密钥对。

2. 对每个配置运行kubectl，如下所示：

   ```code
   KUBECONFIG=<filename> kubectl config set-cluster default-cluster --server=https://<host ip>:6443 --certificate-authority <path-to-kubernetes-ca> --embed-certs
   KUBECONFIG=<filename> kubectl config set-credentials <credential-name> --client-key <path-to-key>.pem --client-certificate <path-to-cert>.pem --embed-certs
   KUBECONFIG=<filename> kubectl config set-context default-system --cluster default-cluster --user <credential-name>
   KUBECONFIG=<filename> kubectl config use-context default-system
   ```

这些文件的用途如下：

| filename                | command                 | comment                                                      |
| ----------------------- | ----------------------- | ------------------------------------------------------------ |
| admin.conf              | kubectl                 | Configures administrator user for the cluster                |
| kubelet.conf            | kubelet                 | One required for each node in the cluster.                   |
| controller-manager.conf | kube-controller-manager | Must be added to manifest in `manifests/kube-controller-manager.yaml` |
| scheduler.conf          | kube-scheduler          | Must be added to manifest in `manifests/kube-scheduler.yaml` |