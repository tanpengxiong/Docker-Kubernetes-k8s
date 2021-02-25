# k8s入门系列之集群安装
## 1. Kubernetes集群组件:

    - etcd 一个高可用的K/V键值对存储和服务发现系统
　　- flannel 实现夸主机的容器网络的通信
　　- kube-apiserver 提供kubernetes集群的API调用
　　- kube-controller-manager 确保集群服务
　　- kube-scheduler 调度容器，分配到Node
　　- kubelet 在Node节点上按照配置文件中定义的容器规格启动容器
　　- kube-proxy 提供网络代理服务




## 2. 集群示意图
Kubernetes工作模式server-client，Kubenetes Master提供集中化管理Minions。部署1台Kubernetes Master节点和4台Minion节点

```


## 3. 操作
#### 3.1 如下操作在所有机器执行
```bash
# 1、确保系统已经安装epel-release源
 yum -y install epel-release
 
# 2、关闭防火墙服务和selinx，避免与docker容器的防火墙规则冲突。
 systemctl stop firewalld
 systemctl disable firewalld
 setenforce 0
```

#### 3.2 安装配置Kubernetes Master， 如下操作在master上执行
```bash
# 1.使用yum安装etcd和kubernetes-master
yum -y install etcd kubernetes-master
# 2.编辑/etc/etcd/etcd.conf文件
ETCD_NAME=default
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://localhost:2379"
# 3.编辑/etc/kubernetes/apiserver文件

KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"
KUBE_API_PORT="--port=8080"
KUBELET_PORT="--kubelet-port=10250"
KUBE_ETCD_SERVERS="--etcd-servers=http://127.0.0.1:2379"
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota"
KUBE_API_ARGS=""

# 4.启动etcd、kube-apiserver、kube-controller-manager、kube-scheduler等服务，并设置开机启动
for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler; 
do 
  systemctl restart $SERVICES;
  systemctl enable $SERVICES;
  systemctl status $SERVICES ; 
done

# 5.在etcd中定义flannel网络
etcdctl mk /atomic.io/network/config '{"Network":"172.17.0.0/16"}'

```


#### 3.3 安装配置Kubernetes Node ，如下操作在node1、node2、node3、node4上执行
```bash
# 1.使用yum安装flannel和kubernetes-node
yum -y install flannel kubernetes-node
# 2.为flannel网络指定etcd服务，修改/etc/sysconfig/flanneld文件
FLANNEL_ETCD="http://192.168.30.20:2379"
FLANNEL_ETCD_KEY="/atomic.io/network"
# 3.修改/etc/kubernetes/config文件
KUBE_LOGTOSTDERR="--logtostderr=true"
KUBE_LOG_LEVEL="--v=0"
KUBE_ALLOW_PRIV="--allow-privileged=false"
KUBE_MASTER="--master=http://192.168.30.20:8080"
# 4.按照如下内容修改对应node的配置文件/etc/kubernetes/kubelet
node1:

KUBELET_ADDRESS="--address=0.0.0.0"
KUBELET_PORT="--port=10250"
KUBELET_HOSTNAME="--hostname-override=192.168.30.21" #修改成对应Node的IP
KUBELET_API_SERVER="--api-servers=http://192.168.30.20:8080" #指定Master节点的API Server
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"
KUBELET_ARGS=""
 
node2:
node3:
node4: 

# 5.在所有Node节点上启动kube-proxy,kubelet,docker,flanneld等服务，并设置开机启动。
for SERVICES in kube-proxy kubelet docker flanneld;
do 
  systemctl restart $SERVICES;
  systemctl enable $SERVICES;
  systemctl status $SERVICES;

done

```
#### 3.4 验证集群是否安装成功
```bash
# 1.在master上执行如下命令
[root@master ~]# kubectl get node
NAME            STATUS    AGE
192.168.30.21   Ready     1m
192.168.30.22   Ready     1m
192.168.30.23   Ready     1m
192.168.30.24   Ready     1m

```




