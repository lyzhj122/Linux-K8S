# K8S Installation -- Ubuntu

------

- Ubuntu version: 16.04.1
- Calico: 3.4.1
- Kubernetes:  1.18.3

------

## 配置基础环境（在所有节点上执行）

在Ubuntu上启动SSH服务：

```shell
sudo apt update
sudo apt install openssh-server
```



Setup IP address for each node:

- master: 192.168.1.201
- node01: 192.168.1.202
- node02: 192.168.1.203

```shell
vim /etc/network/interfaces
```

change IP address of each node

```shell
#primay network interface
auto ens33
iface ens33 inet static
address 192.168.1.201
netmask 255.255.255.0
gateway 192.168.1.1
dns-nameservers 192.168.1.1 8.8.8.8
```

更改主机名

```shell
sudo hostnamectl set-hostname master
```

设置Host文件

```shell
sudo vim /etc/hosts
#添加以下内容
192.168.1.201 master
192.168.1.202 node01
192.168.1.203 node02
```

关闭swap memory

```shell
sudo swapoff -a
```

永久关闭swap

```shell
#注释掉swap行
sudo vim /etc/fstab

#或者直接执行
#查找 swap 那一行并在行首添加#
swapoff -a && sed  -i '/ swap / s/^\(.*\)$/#\1/g'  /etc/fstab    
```

```shell
#关闭SELinux  （CentOS）
sed -i 's/^SELINUX=.*/SELINUX=disabled/'  /etc/selinux/config
```



### 安装 Docker

安装需要的包，并添加Docker Repo

```shell
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common -y
```

下载添加Docker的GPG key

```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
```

添加Docker repo

```shell
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

更新repo并安装Docker

```shell
sudo apt-get update -y
sudo apt-get install docker-ce -y
```



### 安装Kubernetes

下载GPG key

```shell
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
```

添加Kubernetes Repo

```shell
echo 'deb http://apt.kubernetes.io/ kubernetes-xenial main' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

更新repo并安装kubelet, kubeadm, kubectl 

```shell
sudo apt-get update -y
sudo apt-get install kubelet kubeadm kubectl -y
```



## 配置Master节点 （在Master节点上执行）

使用私有IP地址来初始化节点

```shell
kubeadm init --pod-network-cidr=10.60.0.0/16 --apiserver-advertise-address=192.168.1.201
# 10.60.0.0/16 是POD分配的IP地址
# 192.168.1.201 是api server的IP地址
```

配置kuberctl

```shell
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

查看Master节点状态

```shell
kubectl get nodes
-------------------------
NAME     STATUS     ROLES    AGE     VERSION
master   NotReady   master   2m36s   v1.18.3
# Status处于NotReady状态，因为网络服务（CNI）还没有安装
```

为Master节点部署Calico CNI （Container Networking Interface ）

```shell
curl https://docs.projectcalico.org/manifests/calico.yaml -O
#根据情况修改yaml文件，这里采用默认
kubectl apply -f calico.yaml
```

验证Calico正确配置

```shell
kubectl get pods --all-namespaces
```

验证Master节点状态

```shell
kubectl get nodes

root@master:/home/user1# kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   19m   v1.18.3
# Status处于Ready状态 / 如果还是现实NotReady需要多等待一些时间
```



## 添加Slave节点到集群中（在Slave节点上执行）

登录到Slave节点，使用master节点初始化时最后的提示，来将slave节点加入到集群中

```shell
kubeadm join 192.168.1.201:6443 --token txopft.vn10io67ymv22cp2 \
    --discovery-token-ca-cert-hash sha256:8865fa4c6552e7fd5bc54542dbbfed3f79b314d39873b46dc8f6aae95e78c9bd
    
#看到以下信息说明成功加入
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

在master节点查看node状态

```shell
kubectl get nodes
```

**至此，K8S集群安装完毕。**



------

## Deploying the Dashboard UI

```shell
# 参考：https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
kubectl proxy

# 本地访问地址
 http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

远程访问

```shell
#更改连接类型
kubectl -n kubernetes-dashboard edit service kubernetes-dashboard
----
......
spec:
 clusterIP: 10.102.76.186
 externalTrafficPolicy: Cluster
 ports:
 - nodePort: 31969
   port: 443
   protocol: TCP
   targetPort: 8443
 selector:
   k8s-app: kubernetes-dashboard
 sessionAffinity: None
 type: NodePort     # 将类型从 clusterIP 更改为 NodePort 
 status:
  loadBalancer: {}
----------------------
# 获取端口信息
kubectl get service -n kubernetes-dashboard 
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.102.184.116   <none>        8000/TCP        31h
kubernetes-dashboard        NodePort    10.102.76.186    <none>        443:31969/TCP   31h 
# 31969 即为远程访问的端口，访问地址为 https://master node IP:31969

# 获取 token
kubectl -n kube-system describe $(kubectl -n kube-system get secret -n kube-system -o name | grep namespace) | grep token
# 将获取的token填入dashboard 起始页面后登陆 Dashboard
```



------



## 添加worker node到集群

Creating a New Token

```shell
#list the current tokens, If your cluster was initialized over 24-hour ago, the list will likely be empty, since a token’s lifespan is only 24-hours. 
kubeadm tokens list
#Create a new token using kubeadm. By using the –print-join-command argument kubeadm will output the token and SHA hash required to securely communicate with the master.
kubeadm token create --print-join-command

```

Joining the New Worker to the Cluster

```shell
#Use the kubeadm join command with our new token to join the node to our cluster
kubeadm join 192.168.1.130:6443 --token qt57zu.wuvqh64un13trr7x --discovery-token-ca-cert-hash sha256:5ad014cad868fdfe9388d5b33796cf40fc1e8c2b3dccaebff0b066a0532e8723
```

List your cluster’s nodes to verify your new worker has successfully joined the cluster

```shell
kubectl get nodes
```

Verify that the worker’s status to ensure no problems were encountered

```shell
kubectl get nodes my-worker-name
```



------



## 添加Apache Container到集群中 (Master节点)

注册docker hub

```shell
docker login
```

在master节点上运行

```shell
kubectl create deployment httpd --image=httpd
```

查看创建结果

```shell
kubectl get deployments
----
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
httpd   1/1     1            1           6m53s
```

使容器可用

```shell
kubectl create service nodeport httpd --tcp=80:80
```

列出当前运行的service

```shell
kubectl get svc
-------------------
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
httpd        NodePort    10.103.208.15   <none>        80:31996/TCP   7m51s
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        12h
```

列出当前的pod状态

```shell
kubectl get pod -o wide
-------------------
NAME                     READY   STATUS    RESTARTS   AGE     IP              NODE     NOMINATED NODE   READINESS GATES
httpd-5b9c4df55f-jm747   1/1     Running   0          8m27s   10.60.196.130   node01   <none>           <none>
```

