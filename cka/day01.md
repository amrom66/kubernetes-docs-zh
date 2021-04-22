## 集群准备

练习网站：`https://labs.play-with-k8s.com/`

1. 初始化集群
```code

## 初始化集群
kubeadm init --apiserver-advertise-address $(hostname -i) --pod-network-cidr 10.5.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubeadm join 192.168.0.8:6443 --token 2dd3q2.mdq8ao7t3s4g5i2q \
    --discovery-token-ca-cert-hash sha256:33eeb77dd23e19641a9458cadb7fd8154ebbc9243d00e38ab8b37240d47c2cc8

## 初始化网络
kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter.yaml

```

获取加入命令：`kubeadm token create --print-join-command`


2. 部署测试服务nginx

```code

kubectl apply -f  https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/application/nginx-app.yaml

```

3. 创建名称空间与应用

```code

kubectl create namespace mynamespace
kubectl run nginx --image=nginx --restart=Never -n mynamespace
kubectl run nginx --image=nginx --restart=Never --dry-run=client -n mynamespace -o yaml > pod.yaml

kubectl run nginx --image=nginx --restart=Never --dry-run=client -o yaml | kubectl create -n mynamespace -f -

```

`--dry-run`参数表示本地校验格式，[参考](https://kubernetes.io/blog/2019/01/14/apiserver-dry-run-and-kubectl-diff/#challenges)

4. 测试busybox

```code

kubectl run busybox --image=busybox --command --restart=Never -it -- env 
kubectl run busybox --image=busybox --command --restart=Never -- env
kubectl logs busybox

```

-- 空格 env，注意之间的空格

5. 生成创建ns的yaml文件

```code

kubectl create namespace myns -o yaml --dry-run=client
kubectl create quota myrq --hard=cpu=1,memory=1G,pods=2 --dry-run=client -o yaml

```

6. 获取pods

```code

kubectl get po --all-namespaces

kubectl get po -A 

```

7. 运行时修改容器的镜像名

```code

# kubectl set image POD/POD_NAME CONTAINER_NAME=IMAGE_NAME:TAG
kubectl set image pod/nginx nginx=nginx:1.7.1

```

8. 获取格式化输出

```code

kubectl get po nginx -o jsonpath='{.spec.containers[].image}{"\n"}'

```

9. 输出

```code

kubectl get po nginx -o yaml
# or
kubectl get po nginx -oyaml
# or
kubectl get po nginx --output yaml
# or
kubectl get po nginx --output=yaml

```

10. 获取容器重启前的日志

```code

kubectl logs nginx -p

```

11. 在容器中简单运行命令

```code

kubectl exec -it nginx -- /bin/sh

```

12. 操作label

```code

kubectl run nginx1 --image=nginx --restart=Never --labels=app=v1
kubectl run nginx2 --image=nginx --restart=Never --labels=app=v1
kubectl run nginx3 --image=nginx --restart=Never --labels=app=v1

kubectl get po --show-labels

kubectl label po nginx2 app=v2 --overwrite

kubectl get po -L app

kubectl get po --label-columns=app

kubectl get po -l app=v2

kubectl get po -l 'app in (v2)'

kubectl get po --selector=app=v2

kubectl label po nginx1 nginx2 nginx3 app-

kubectl label po nginx{1..3} app-

kubectl label po -l app app-

```

13. 操作annotate

```code

kubectl annotate po nginx1 nginx2 nginx3 description='my description'

#or

kubectl annotate po nginx{1..3} description='my description'

kubectl annotate po nginx{1..3} description-


```

14. 批量删除

```code

kubectl delete po nginx{1..3}

```

15. kubectl rollout

```code

kubectl rollout status deploy nginx

```

16. 修改deploy的镜像
```code

kubectl set image deploy nginx nginx=1.19.8
# or 
kubectl edit deploy nginx

```

17. 回滚操作

```code
kubectl rollout undo deploy nginx --to-revision=2
kubectl describe deploy nginx | grep Image:
kubectl rollout status deploy nginx # Everything should be OK

kubectl rollout history deploy nginx --revision=4 # You'll also see the wrong image displayed here

kubectl rollout pause deploy nginx

```

18. 扩容缩容

```code

kubectl scale deploy nginx --replicas=5
kubectl get po
kubectl describe deploy nginx

kubectl autoscale deploy nginx --min=5 --max=10 --cpu-percent=80

```
19. 任务

```code
kubectl create job pi  --image=perl -- perl -Mbignum=bpi -wle 'print bpi(2000)'

kubectl create job busybox --image=busybox -- /bin/sh -c 'echo hello;sleep 30;echo world'

```

20. ConfigMap

```code

kubectl create configmap config --from-literal=foo=lala --from-literal=foo2=lolo


echo -e "foo3=lili\nfoo4=lele" > config.txt
kubectl create cm configmap2 --from-file=config.txt

echo -e "var1=val1\n# this is a comment\n\nvar2=val2\n#anothercomment" > config.env
kubectl create cm configmap3 --from-env-file=config.env

echo -e "var3=val3\nvar4=val4" > config4.txt
kubectl create cm configmap4 --from-file=special=config4.txt

kubectl create cm options --from-literal=var5=val5
kubectl run nginx --image=nginx --restart=Never --dry-run=client -o yaml > pod.yaml
vi pod.yaml

```

pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: nginx
    resources: {}
    env:
    - name: option # name of the env variable
      valueFrom:
        configMapKeyRef:
          name: options # name of config map
          key: var5 # name of the entity in config map
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

```code

kubectl create configmap anotherone --from-literal=var6=val6 --from-literal=var7=val7
kubectl run --restart=Never nginx --image=nginx -o yaml --dry-run=client > pod.yaml

```

pod.yaml

```yaml

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: nginx
    resources: {}
    envFrom: # different than previous one, that was 'env'
    - configMapRef: # different from the previous one, was 'configMapKeyRef'
        name: anotherone # the name of the config map
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}

```

```code

kubectl create configmap cmvolume --from-literal=var8=val8 --from-literal=var9=val9
kubectl run nginx --image=nginx --restart=Never -o yaml --dry-run=client > pod.yaml

```

vi pod.yaml

```yaml

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  volumes: # add a volumes list
  - name: myvolume # just a name, you'll reference this in the pods
    configMap:
      name: cmvolume # name of your configmap
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: nginx
    resources: {}
    volumeMounts: # your volume mounts are listed here
    - name: myvolume # the name that you specified in pod.spec.volumes.name
      mountPath: /etc/lala # the path inside your container
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}

```

```code

kubectl run nginx --image=nginx --restart=Never --dry-run=client -o yaml > pod.yaml

```

vi pod.yaml

```yaml

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  securityContext: # insert this line
    runAsUser: 101 # UID for the user
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}

```

```code

 kubectl run nginx10 --image=nginx --restart=Never --requests='cpu=100m,memory=256Mi' --limits='cpu=200m,memory=512Mi' --dry-run=client  -o yaml > pod10.yaml

 ```

21. Secrets

```code

kubectl create secret generic mysecret --from-literal=password=mypass

echo -n admin > username
kubectl create secret generic mysecret2 --from-file=username

kubectl get secret mysecret2 -o yaml
echo YWRtaW4K | base64 -d # on MAC it is -D, which decodes the value and shows 'admin'

kubectl get secret mysecret2 -o jsonpath='{.data.username}{"\n"}' | base64 -d

kubectl run nginx11 --image=nginx --restart=Never -o yaml --dry-run=client > pod11.yaml 

```

pod11.yaml

```yaml

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx11
  name: nginx11
spec:
  volumes:
  - name: foo
    secret:
      secretName: mysecret2
  containers:
  - image: nginx
    name: nginx11
    resources: {}
    volumeMounts:
    - name: foo
      mountPath: /etc/foo
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}

```

pod12.yaml

```yaml 
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx12
  name: nginx12
spec:
  containers:
  - image: nginx
    name: nginx12
    resources: {}
    env:
    - name: USERNAME
      valueFrom:
        secretKeyRef:
          name: mysecret2
          key: username
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}

```

22. sa

```code

kubectl create sa myuser

kubectl run nginx --image=nginx --restart=Never --serviceaccount=myuser -o yaml --dry-run=client > pod.yaml


```