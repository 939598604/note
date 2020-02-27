### 3.1 master节点安装

因为etcd节点和master节点是同一台。所以一起执行：

```
yum install -y etcd kubernetes-master flannel
```

### 3.2 node节点安装

```
yum install -y kubernetes-node flannel 
```

## 四.配置etcd服务

### 4.1 etcd配置文件

[root@master ~]# grep -v '^#' /etc/etcd/etcd.conf

```
ETCD_NAME=default
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_CLIENT_URLS="http://localhost:2379,http://192.168.197.33:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.197.33:2379"
```

### 4.2 启动服务

```
systemctl start etcd
systemctl enable etcd
```

### 4.3 检查etcd cluster状态 

```
[root@master ~]# etcdctl cluster-health   
member 8e9e05c52164694d is healthy: got healthy result from http://192.168.197.33:2379
cluster is healthy
```

### 4.4 检查etcd集群成员列表，这里只有一台

```
[root@master ~]# etcdctl member list
8e9e05c52164694d: name=default peerURLs=http://localhost:2380 clientURLs=http://192.168.197.33:2379 isLeader=true
```

### 4.5 配置etcd

```
[root@master ~]# etcdctl set /atomic.io/network/config '{"Network": "172.16.0.0/16"}'
{"Network": "172.16.0.0/16"}
```

### 4.6 查看验证网络信息

```
[root@master ~]# etcdctl get /atomic.io/network/config  
{ "Network": "172.16.0.0/16" }
[root@master ~]# etcdctl ls /atomic.io/network/subnets
/atomic.io/network/subnets/172.16.69.0-24
/atomic.io/network/subnets/172.16.6.0-24
[root@master ~]# etcdctl get /atomic.io/network/subnets/172.16.6.0-24
{"PublicIP":"192.168.197.225"}
[root@master ~]# etcdctl get  /atomic.io/network/subnets/172.16.69.0-24
{"PublicIP":"192.168.197.226"}
```

## 五. 配置master服务器

### 5.1 kube-apiserver配置文件

[root@master ~]# grep -v '^#' /etc/kubernetes/config

```
KUBE_LOGTOSTDERR="--logtostderr=true"
KUBE_LOG_LEVEL="--v=0"
KUBE_ALLOW_PRIV="--allow-privileged=false"
KUBE_MASTER="--master=http://192.168.197.33:8080"

```

[root@master ~]# grep -v '^#' /etc/kubernetes/apiserver

```
KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"
KUBE_ETCD_SERVERS="--etcd-servers=http://192.168.197.33:2379"
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
KUBE_ADMISSION_CONTROL="--admission-control=AlwaysAdmit"
KUBE_API_ARGS=""
```

### 5.2 kube-controller-manager配置文件

[root@master ~]# grep -v '^#' /etc/kubernetes/controller-manager

```
KUBE_CONTROLLER_MANAGER_ARGS=""
```

### 5.3 kube-scheduler配置文件

[root@master ~]# grep -v '^#' /etc/kubernetes/scheduler

```
KUBE_SCHEDULER_ARGS="--address=0.0.0.0"
```

### 5.4 启动服务

```
for i in  kube-apiserver kube-controller-manager kube-scheduler;
do 
    systemctl restart $i; 
    systemctl enable $i;
done
```

## 六.配置node1节点服务器

### 6.1 配置node1网络

本实例采用flannel方式来配置，如需其他方式，请参考Kubernetes官网。

```
[root@node1 ~]# grep -v '^#' /etc/sysconfig/flanneld 
FLANNEL_ETCD_ENDPOINTS="http://192.168.197.33:2379"
FLANNEL_ETCD_PREFIX="/atomic.io/network"
FLANNEL_OPTIONS=""        
```

### 6.2 配置node1 kube-proxy

```
[root@node1 ~]# grep -v '^#' /etc/kubernetes/config 
KUBE_LOGTOSTDERR="--logtostderr=true"
KUBE_LOG_LEVEL="--v=0"
KUBE_ALLOW_PRIV="--allow-privileged=false"
KUBE_MASTER="--master=http://192.168.197.33:8080"
```

[root@node1 ~]# grep -v '^#' /etc/kubernetes/proxy                  

```
KUBE_PROXY_ARGS="--bind=address=0.0.0.0"
```

### 6.3 配置node1 kubelet

```
[root@node1 ~]# grep -v '^#' /etc/kubernetes/kubelet 
KUBELET_ADDRESS="--address=192.168.197.225"
KUBELET_HOSTNAME="--hostname-override=192.168.197.225"
KUBELET_API_SERVER="--api-servers=http://192.168.197.33:8080"
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"
KUBELET_ARGS=""
```

### 6.4 启动node1服务

```
for i in flanneld kube-proxy kubelet docker;
do 
    systemctl restart $i;
    systemctl enable $i;
    systemctl status $i ;
done
```

## 七.配置node2节点服务器

node2与node1配置基本一致，除下面一处例外

```
[root@node2 ~]# vi /etc/kubernetes/kubelet
KUBELET_ADDRESS="--address=192.168.197.226"
KUBELET_HOSTNAME="--hostname-override=192.168.197.226"
```

