

# k8s安装

## 一.节点规划

### 1.1 Kubernetes集群组件:

- etcd 一个高可用的K/V键值对存储和服务发现系统
- flannel 实现夸主机的容器网络的通信
- kube-apiserver 提供kubernetes集群的API调用
- kube-controller-manager 确保集群服务
- kube-scheduler 调度容器，分配到Node
- kubelet 在Node节点上按照配置文件中定义的容器规格启动容器
- kube-proxy 提供网络代理服务

| 节点   | IP地址          |
| ------ | --------------- |
| master | 192.168.197.33  |
| node1  | 192.168.197.225 |
| node2  | 192.168.197.226 |

## 二.环境准备

### 2.1 关闭selinux 

```
# 临时关闭
$ setenforce 0  
# 永久关闭
$ vim /etc/selinux/config 
SELINUX=disabled
```

### 2.2 关闭且禁用防火墙

```
systemctl stop firewalld 
systemctl disable firewalld
```

### 2.3 修改hostname

更改Hostname为 master、node1、node2

vim /ect/hostname

```
mster
node1
node2
```

### 2.4 时间安装

系统初始化安装（所有主机）-选择【最小化安装】,然后yum update,升级到最新版本

```
yum -y install epel-release
yum update
```

时间校对，所有主机

```
yum -y install ntpd
systemctl start ntpd
systemctl enable ntpd
ntpdate ntp1.aliyun.com
hwclock -w
```

### 2.5 rancherserver与agent之间ssh免密登录

```
ssh-keygen
ssh-copy-id root@192.168.197.33
ssh-copy-id root@192.168.197.225
ssh-copy-id root@192.168.197.226
```

### 2.6 修改hosts文件

```
192.168.197.33 master etcd
192.168.197.225 node1
192.168.197.226 node2
```

### 2.7 关闭Swap交换分区；

vim /etc/fstab

```
#/dev/mapper/cl-swap     swap                    swap    defaults        0 0
```

### 2.8 配置kubernetes源

vim /etc/yum.repos.d/kubernetes.repo

```
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg    https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```

## 三.yum安装

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

## 八.查看节点

```
[root@master ~]# kubectl get nodes   
NAME          STATUS    AGE
192.168.197.225   Ready     18h
192.168.197.226   Ready     13h
```

k8s支持2种方式,一种是直接通过命令参数的方式,另一种是通过配置文件的方式,配置文件的话支持json和yaml

## 九.建立pod

```
kubectl run nginx --image=nginx --port=80  --replicas=2
```

## 十 k8s之kubectl命令自动补全

```
yum install -y epel-release bash-completion
source /usr/share/bash-completion/bash_completion


source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

## 十一.遇到问题

### 11.1 创建成功但是kubectl get pods 没有结果

**提示信息：**no API token found for service account default
**解决办法：**

```
编辑/etc/kubernetes/apiserver 去除 KUBE_ADMISSION_CONTROL中的SecurityContextDeny,ServiceAccount，并重启kube-apiserver.service服务
```

### 11.2 pod-infrastructure:latest镜像下载失败

**报错信息：**image pull failed for registry.access.redhat.com/rhel7/pod-infrastructure:latest, this may be because there are no credentials on this request. 
**解决方案：**

```
yum install *rhsm* -y
```

### 11.3 容器报错

**问题1:Error from server: error dialing backend: dial tcp 192.168.197.226:10250: getsockopt: connection refused**
**解决方法：**

```
10250是kubelet的端口。 
在Node上检查/etc/kubernetes/kubelet
KUBELET_ADDRESS需要修改为node ip
```

**问题2:Error from server (BadRequest): container "nginx" in pod "nginx-deployment-1998962726-p6rv5" is waiting to start: ContainerCreating**

 **解决方法：**

执行日志查看的时候：kubectl logs nginx-deployment-1998962726-p6rv5 出现

```
   Warning FailedSync      Error syncing pod, skipping: failed to "StartContainer" for "POD" with ImagePullBackOff: "Back-off pulling image \"registry.access.redhat.com/rhel7/pod-infrastructure:latest\""

```

fa

**方法一**

1.在node节点执行 yum install *rhsm*

2.docker pull  registry.access.redhat.com/rhel7/pod-infrastructure:latest

```
[root@can225 ~]# docker pull  registry.access.redhat.com/rhel7/pod-infrastructure:latest
Trying to pull repository registry.access.redhat.com/rhel7/pod-infrastructure ... 
open /etc/docker/certs.d/registry.access.redhat.com/redhat-ca.crt: no such file or directory
```

 **方法二**

```
 wget http://mirror.centos.org/centos/7/os/x86_64/Packages/python-rhsm-certificates-1.19.10-1.el7_4.x86_64.rpm
 rpm2cpio python-rhsm-certificates-1.19.10-1.el7_4.x86_64.rpm | cpio -iv --to-stdout ./etc/rhsm/ca/redhat-uep.pem | tee /etc/rhsm/ca/redhat-uep.pem
```

 前两个命令会生成/etc/rhsm/ca/redhat-uep.pem文件

```
docker pull registry.access.redhat.com/rhel7/pod-infrastructure:latest
```

 

 