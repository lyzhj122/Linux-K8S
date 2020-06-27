# K8S Components 

- **Proxy**

  ```
  The Kube proxy does low-level, network housekeeping on each node. It reflects the Kubernetes services locally and can do TCP and UDP forwarding. It finds cluster IPs through environment variables or DNS.  
  ```

  


- **Kubelet**
  The kubelet is the Kubernetes representative on the node. It oversees communicating with the master components and manages the running pods. This includes the following actions:

  ```
  Downloading pod secrets from the API server
  Mounting volumes
  Running the pod's container (through the CRI or rkt)
  Reporting the status of the node and each pod
  Running container liveness probes  
  ```

  




# K8s 命令

查询组件状态

```shell
kubectl get componentstatus
#简写：
kubectl get cs
--------
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
```

获取pod资源信息

```shell
#获取pod资源信息。 什么也不加，即从默认的namespace中获取pod资源信息
kubectl get pod       # 相当于 kubectl get pod -n default
#从kube-system这个名称空间下获取pod资源信息
kubectl get pod -n kube-system   
```

获取名称空间信息

```shell
kubectl get ns
```





## 资源

### K8S里面的资源

k8s里面所有的内容都抽象为资源，资源实例化以后叫做对象。

工作负载型的资源（workload）： Pod，  ReplicaSet，  Deployment，  StatefulSet，  DaemonSet，  Job，  CronJob

服务发现及负载均衡类型资源（Service Discovery， LoadBalance）： Service， Ingress ......

配置与存储型资源：Volume，CSI （容器存储接口，可以扩展各种各样的第三方存储）

特殊类型的存储卷：ConfigMap，Secret（保存敏感数据），DownardAPI（把外部环境中的信息输出给容器）

以上资源都是配置在名称空间级别。

![](C:\D\学习内容\Linux&K8S summary\CRI.jpg)

### 资源清单

什么是资源清单

```yaml
在k8s中，一般使用yaml格式的文件来创建符合我们预期期望的pod，这样的yaml文件我们一般称为资源清单
```

资源清单的格式

```yaml
piVersion: group/apiversion  # 如果没有给定 group 名称，那么默认为 core，可以使用 kubectl api-versions # 获取当前 k8s 版本上所有的 apiVersion 版本信息( 每个版本可能不同 )
kind:       #资源类别
metadata：  #资源元数据  
  name   namespace   
  lables   annotations   # 主要目的是方便用户阅读查找
  annotation   #主要方便用户阅读查找
spec: # 期望的状态（disired state）
status：# 当前状态，本字段由 Kubernetes 自身维护，用户不能去定义
```

格式里面字段的含义

```shell
#获取每个资源的解释信息
kubectl explain pod
-------
root@master:/etc/docker# kubectl explain pod
KIND:     Pod
VERSION:  v1

DESCRIPTION:
     Pod is a collection of containers that can run on a host. This resource is
     created by clients and scheduled onto hosts.

FIELDS:
   apiVersion   <string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources

   kind <string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds
     ... 
  #查看更详细的信息
  kubectl explain pod.apiVersion
  
    
```

查看pod信息

```shell
# 查看mypod的描述信息
kubectl describe pod mypod

# 查看pod的详细信息
kubectl get pod -o wide
```

查看log信息

```shell
# 查看容器的log信息, log 后面跟 pod名称，-c 指定pod中的 container 名字 (如果有多个 container 的话)
kubectl log myapp-pod -c test
# 查看log信息， -n 执行名称空间
kubectl logs kube-proxy-5495p  -n kube-system
```

创建pod

```shell
# 以abc.yml文件来创建pod
kubectl create -f abc.yml
```

删除pod

```shell
# 删除myapp-pod
kubectl delete pod myapp-pod
# 删除所有 pod
kubectl delete pod --all
```



## 创建Pod

### 通过deployment创建

创建一个ReplicaSet 为 3的nginx Pods, Yaml文件如下

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

创建Pod

```shell
kubectl apply -f nginx-deployment.yml
```

查看Deployment是否已经创建

```shell
kubectl get deployments
-----
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           4m54s
```

查看ReplicaSet

```shell
kubectl get rs
-----
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-6b474476c4   3         3         3       10m
```

查看每个Pod自动生成的label信息

```shell
kubectl get pods --show-labels
------
NAME                                READY   STATUS    RESTARTS   AGE   LABELS
nginx-deployment-6b474476c4-gfwf5   1/1     Running   0          12m   app=nginx,pod-template-hash=6b474476c4
nginx-deployment-6b474476c4-ngz7k   1/1     Running   0          12m   app=nginx,pod-template-hash=6b474476c4
nginx-deployment-6b474476c4-tqkwt   1/1     Running   0          12m   app=nginx,pod-template-hash=6b474476c4
```

查看Deployment的详细信息

```shell
kubectl describe deployments
```



### Scaling a Deployment

```shell
# scale replicas 到 10
kubectl scale deployment.v1.apps/nginx-deployment --replicas=10
```

自动扩容

```shell
kubectl autoscale deployment.v1.apps/nginx-deployment --min=10 --max=15 --cpu-percent=80
```

### 删除Deployment

```shell
kubectl delete deployment nginx-deployment
```



## Controllers

Yaml里面的kind说明 Controllers 的类型，可以是：

```
ReplicaSet
ReplicationController
Deployment
StatefulSet
DaemonSet
Job
CronJob
```

jobs example yaml file 

It computes π to 2000 places and prints it out. It takes around 10s to complete.

```yaml
apiVersion: batch/v1
kind: Job   # 表明类型
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```



## Service

创建的Container是会消亡的，如果有由Deployment来创建的话，那么它会自动创建和销毁Pod.

每个Pod都有分配的自己的IP地址，而且每个Pod销毁后重新创建的Pod的地址都是不同的，那么如何让有业务联系的Pod能够互相找到对方？  使用 Services

Service是定义了逻辑上一组Pod和Policy的抽象。 （In Kubernetes, a Service is an abstraction which defines a logical set of Pods and a policy by which to access them (sometimes this pattern is called a micro-service)). 



