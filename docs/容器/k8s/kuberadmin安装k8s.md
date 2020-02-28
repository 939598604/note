

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
# 检查防火墙是否还在运行
firewall-cmd  --state
```

### 2.3 修改hostname

更改Hostname为 master、node1、node2

```
#设置主机名称
[root@reg ~]# hostnamectl set-hostname master
#查看设置是否成功
[root@reg ~]# hostname
master
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

### 2.8 添加网桥过滤及地址转发

vim /etc/sysctl.d/k8s.conf

```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
```

**加载br_netfilter模块**

```
[root@localhost k8s]#  modprobe br_netfilter
```

**查看是否加载**

```
[root@localhost k8s]# lsmod |grep br_netfilter
br_netfilter           22256  0 
bridge                151336  1 br_netfilter
```

**加载网桥过滤配置文件**

```
[root@localhost k8s]# sysctl -p /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
```



### 2.9开启ipvs功能

安装ipset及ipvsadm

```
[root@localhost k8s]# yum -y install ipset ipvsadm
```

添加需要加载的模块

```
cat > /etc/sysconfig/modules/ipvs.modules << EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
```

授权、运行文件、检查是否加载

```
chmod 755 /etc/sysconfig/modules/ipvs.modules
sh /etc/sysconfig/modules/ipvs.modules 
lsmod |grep -e ip_vs -e nf_conntrack_ipv4
```

### 2.10 docker的安装

如果还没有安装docker，需要安装docker



### 2.11 配置kubernetes源

所有k8s集群节点均需安装，默认YUM源是谷歌，可以使用阿里云YUM

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

### 3.1 安装kubeadm kubelet kubectl

所有节点安装

```
yum -y install kubeadm-1.17.3 kubectl-1.17.3 kubelet-1.17.3
```

删除

```
 yum -y remove kubeadm kubectl kubelet 
```



### 3.2 修改cgroup的一致性

主要配置kubelet，如果不配置可能会导致k8s集群无法启动。
为了实现docker使用的cgroupdriver与kubelet使用的cgroup的一致性，建议修改如下文件内容。

vim /etc/sysconfig/kubelet

```
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"
```

### 3.3  kubelet 开机自启动

设置为开机自启动即可，由于没有生成配置文件，集群初始化后自动启动

```
systemctl enable kubelet
# 查看状态
systemctl status kubelet
# 查看到变成了enabled
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
```

### 3.4 k8s 镜像获取

#### 替换镜像域名下载

（1）查看集群使用的容器镜像

```
[root@localhost ~]# kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.17.3
k8s.gcr.io/kube-controller-manager:v1.17.3
k8s.gcr.io/kube-scheduler:v1.17.3
k8s.gcr.io/kube-proxy:v1.17.3
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.4.3-0
k8s.gcr.io/coredns:1.6.5
```

（2）列出镜像列表到文件中，便于下载使用。

```
[root@localhost ~]# kubeadm config images list >> image.list
```

（3）查看已列出镜像文件列表

```
[root@localhost ~]# cat image.list
k8s.gcr.io/kube-apiserver:v1.17.3
k8s.gcr.io/kube-controller-manager:v1.17.3
k8s.gcr.io/kube-scheduler:v1.17.3
k8s.gcr.io/kube-proxy:v1.17.3
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.4.3-0
k8s.gcr.io/coredns:1.6.5
```

众所周知,国内是不太容易下载`k8s.gcr.io`站点的镜像的

参考：https://www.cnblogs.com/liyongjian5179/p/11318318.html

| Global                | Proxy in China (Azure中国镜像)   |
| --------------------- | -------------------------------- |
| dockerhub (docker.io) | `dockerhub.azk8s.cn`             |
| gcr.io                | `gcr.azk8s.cn`                   |
| k8s.gcr.io            | `gcr.azk8s.cn/google-containers` |
| quay.io               | `quay.azk8s.cn`                  |

```
#这两条语句是等效的
docker pull  k8s.gcr.io/kube-apiserver:v1.15.2
docker pull  gcr.azk8s.cn/google-containers/kube-apiserver:v1.15.2

#这两条也是等效的
docker pull quay.io/xxx/yyy:zzz
docker pull quay.azk8s.cn/xxx/yyy:zzz
```

（4）替换镜像地址

编写镜像下载脚本

```
[root@localhost ~]# vim image.list 
#!/bin/bash
img_list='
gcr.azk8s.cn/google-containers/kube-apiserver:v1.17.3
gcr.azk8s.cn/google-containers/kube-apiserver:v1.17.3
gcr.azk8s.cn/google-containers/kube-controller-manager:v1.17.3
gcr.azk8s.cn/google-containers/kube-scheduler:v1.17.3
gcr.azk8s.cn/google-containers/kube-proxy:v1.17.3
gcr.azk8s.cn/google-containers/pause:3.1
gcr.azk8s.cn/google-containers/etcd:3.4.3-0
gcr.azk8s.cn/google-containers/coredns:1.6.5'

for img in $img_list
do
   docker pull $img
done
```

（5）开始下载镜像

```
sh image.list
```

（6）查看是否下载完成

```
docker images
```

（7）修改tag标签：

```
docker tag  gcr.azk8s.cn/google-containers/kube-apiserver:v1.17.3 k8s.gcr.io/kube-apiserver:v1.17.3
docker tag  gcr.azk8s.cn/google-containers/kube-controller-manager:v1.17.3 k8s.gcr.io/kube-controller-manager:v1.17.3
docker tag  gcr.azk8s.cn/google-containers/kube-scheduler:v1.17.3 k8s.gcr.io/kube-scheduler:v1.17.3
docker tag  gcr.azk8s.cn/google-containers/kube-proxy:v1.17.3 k8s.gcr.io/kube-proxy:v1.17.3
docker tag  gcr.azk8s.cn/google-containers/pause:3.1 k8s.gcr.io/pause:3.1
docker tag  gcr.azk8s.cn/google-containers/etcd:3.4.3-0 k8s.gcr.io/etcd:3.4.3-0
docker tag  gcr.azk8s.cn/google-containers/coredns:1.6.5 k8s.gcr.io/coredns:1.6.5
```

#### 坚果云下载镜像image.tar.gz

或者直接到坚果云https://www.jianguoyun.com/p/Da4u4T0Q3suRCBjwq-UC 下载镜像image.tar.gz

```
docker load -i coredns_1_6_5.tar
docker load -i etcd_3_4_3-0.tar
docker load -i flannel_v0_11_0-amd64.tar
docker load -i kube-apiserver_v_1_17.3.tar
docker load -i kube-controller-manager_v_1_17.3.tar
docker load -i kube-proxy_v_1_17.3.tar
docker load -i kube-scheduler_v_1_17.3.tar
docker load -i pause_3_1.tar
```



### 3.5 kubeadm初始化

**在master节点操作**

192.168.197.33为主节点的ip地址，网络是自定义的，如果是flannel的网络的话是 --pod-network-cidr=10.244.0.0/16 

```
kubeadm init --kubernetes-version=v1.17.3 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.197.33
```

也可以指定service网络 

```
kubeadm init --kubernetes-version=v1.17.3 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --apiserver-advertise-address=192.168.197.33
```

初始化遇到的问题

```
[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
```

解决方法：

```
vim /etc/docker/daemon.json
添加如下内容：
{
"exec-opts":["native.cgroupdriver=systemd"]
}
```

执行后出现结果如下

```
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.197.33:6443 --token ux1o0b.8wy6odha36z4icvv \
    --discovery-token-ca-cert-hash sha256:8295aa1f424abab2eca2bd104b1526d023cdfb2541e60885a1c7c4ea97232135 
```

执行

```
rm -rf $HOME/.kube/config
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```



### 3.6  安装flannel网络

（1）下载yml文件

```
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

（2）如果需要修改网络 这个网段即可 172.16.0.0

 "Network": "172.16.0.0/16",

```
 net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```

（3）因为被墙，将quay.io改为quay.azk8s.cn

**其他：**

如果Node有多个网卡的话，目前需要在kube-flannel.yml中使用--iface参数指定集群主机内网网卡的名称，否则可能会出现dns无法解析。容器无法通信的情况，需要将kube-flannel.ym下载到本地，flanneld启动参数加上-iface=<iface-name>

```
      containers:
      - name: kube-flannel
        image: quay.azk8s.cn/coreos/flannel:v0.11.0-ppc64le
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=ens33
        - --iface=eth0
```

（4）修改之后结果如下

```
---
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.flannel.unprivileged
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: docker/default
    seccomp.security.alpha.kubernetes.io/defaultProfileName: docker/default
    apparmor.security.beta.kubernetes.io/allowedProfileNames: runtime/default
    apparmor.security.beta.kubernetes.io/defaultProfileName: runtime/default
spec:
  privileged: false
  volumes:
    - configMap
    - secret
    - emptyDir
    - hostPath
  allowedHostPaths:
    - pathPrefix: "/etc/cni/net.d"
    - pathPrefix: "/etc/kube-flannel"
    - pathPrefix: "/run/flannel"
  readOnlyRootFilesystem: false
  # Users and groups
  runAsUser:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  # Privilege Escalation
  allowPrivilegeEscalation: false
  defaultAllowPrivilegeEscalation: false
  # Capabilities
  allowedCapabilities: ['NET_ADMIN']
  defaultAddCapabilities: []
  requiredDropCapabilities: []
  # Host namespaces
  hostPID: false
  hostIPC: false
  hostNetwork: true
  hostPorts:
  - min: 0
    max: 65535
  # SELinux
  seLinux:
    # SELinux is unused in CaaSP
    rule: 'RunAsAny'
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
rules:
  - apiGroups: ['extensions']
    resources: ['podsecuritypolicies']
    verbs: ['use']
    resourceNames: ['psp.flannel.unprivileged']
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes/status
    verbs:
      - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-amd64
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: beta.kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: beta.kubernetes.io/arch
                    operator: In
                    values:
                      - amd64
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.azk8s.cn/coreos/flannel:v0.11.0-amd64
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.azk8s.cn/coreos/flannel:v0.11.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run/flannel
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-arm64
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: beta.kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: beta.kubernetes.io/arch
                    operator: In
                    values:
                      - arm64
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.azk8s.cn/coreos/flannel:v0.11.0-arm64
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.azk8s.cn/coreos/flannel:v0.11.0-arm64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
             add: ["NET_ADMIN"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run/flannel
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-arm
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: beta.kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: beta.kubernetes.io/arch
                    operator: In
                    values:
                      - arm
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.azk8s.cn/coreos/flannel:v0.11.0-arm
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.azk8s.cn/coreos/flannel:v0.11.0-arm
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
             add: ["NET_ADMIN"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run/flannel
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-ppc64le
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: beta.kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: beta.kubernetes.io/arch
                    operator: In
                    values:
                      - ppc64le
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.azk8s.cn/coreos/flannel:v0.11.0-ppc64le
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.azk8s.cn/coreos/flannel:v0.11.0-ppc64le
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
             add: ["NET_ADMIN"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run/flannel
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-s390x
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: beta.kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: beta.kubernetes.io/arch
                    operator: In
                    values:
                      - s390x
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.azk8s.cn/coreos/flannel:v0.11.0-s390x
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.azk8s.cn/coreos/flannel:v0.11.0-s390x
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
             add: ["NET_ADMIN"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run/flannel
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
```

（5）启动flannel

```
[root@master kubernetes]# kubectl apply -f kube-flannel.yaml 
serviceaccount "flannel" created
configmap "kube-flannel-cfg" created
daemonset "kube-flannel-ds" created
```

（6）查看

```
[root@localhost install]# kubectl get pods -n kube-system
NAME                                            READY   STATUS    RESTARTS   AGE
coredns-6955765f44-84n5l                        1/1     Running   0          16h
coredns-6955765f44-zwcx2                        1/1     Running   0          16h
etcd-localhost.localdomain                      1/1     Running   0          16h
kube-apiserver-localhost.localdomain            1/1     Running   0          16h
kube-controller-manager-localhost.localdomain   1/1     Running   0          16h
kube-flannel-ds-amd64-4gjx8                     1/1     Running   0          51s
kube-proxy-hw6xl                                1/1     Running   0          16h
kube-scheduler-localhost.localdomain            1/1     Running   0          16h
```

kube-flannel-ds-amd64-4gjx8的pod状态是Running 

### 3.7 加入node节点

（1）加入node节点的命令如下

```
kubeadm join 192.168.197.33:6443 --token ux1o0b.8wy6odha36z4icvv \
    --discovery-token-ca-cert-hash sha256:8295aa1f424abab2eca2bd104b1526d023cdfb2541e60885a1c7c4ea97232135 
```

**重新生成token**

因为token是半小时失效，如果需要重新生成，执行以下命令

```
[root@localhost install]# kubeadm token create 
W0227 10:14:26.672199  115832 validation.go:28] Cannot validate kube-proxy config - no validator is available
W0227 10:14:26.672308  115832 validation.go:28] Cannot validate kubelet config - no validator is available
39kka2.3vvd520sk1y3m664
```

**查看token列表**

```
[root@localhost install]# kubeadm token list
TOKEN                     TTL         EXPIRES                     USAGES                   DESCRIPTION                                                EXTRA GROUPS
39kka2.3vvd520sk1y3m664   23h         2020-02-28T10:14:26+08:00   authentication,signing   <none>                                                     system:bootstrappers:kubeadm:default-node-token
ux1o0b.8wy6odha36z4icvv   7h          2020-02-27T17:42:42+08:00   authentication,signing   The default bootstrap token generated by 'kubeadm init'.   system:bootstrappers:kubeadm:default-node-token
```

**获取ca证书sha256编码hash值**

```
[root@localhost install]# openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^ .* //'
(stdin)= 8295aa1f424abab2eca2bd104b1526d023cdfb2541e60885a1c7c4ea97232135
```

 如果忘记了执行后的 join 命令，也可以使用命令 `kubeadm token create --print-join-command`重新获取。 



**从节点加入集群**

kubeadm join 192.168.40.8:6443 --token token填这里 --discovery-token-ca-cert-hash sha256:哈希值填这里

```
kubeadm join 192.168.197.33:6443 --token 39kka2.3vvd520sk1y3m664 \
    --discovery-token-ca-cert-hash sha256:8295aa1f424abab2eca2bd104b1526d023cdfb2541e60885a1c7c4ea97232135
```

**在node1和node2节点上执行加入集群**

```
kubeadm join 192.168.197.33:6443 --token 39kka2.3vvd520sk1y3m664  --discovery-token-ca-cert-hash sha256:8295aa1f424abab2eca2bd104b1526d023cdfb2541e60885a1c7c4ea97232135

[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.17" ConfigMap in the kube-system namespace
```

#### 问题1

执行kubectl -n kube-system get cm kubeadm-config -oyaml

```
[root@node1 ~]# kubectl -n kube-system get cm kubeadm-config -oyaml
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

配置kubenetes的flannel网络的时候，出现以下报错

The connection to the server localhost:8080 was refused - did you specify the right host or port?

原因：kubenetes master没有与本机绑定，集群初始化的时候没有设置



#### 问题2

节点加入集群中报如下的错误

```
[root@ken2 ~]# kubeadm join 192.168.232.199:6443 --token 5fsbj7.3iobmze5ow8jcz4w     --discovery-token-ca-cert-hash sha256:c17bcdcf9b65fef975527d3b02d9cc22621b77916a974a35b112fc5639cb4e77
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
        [WARNING SystemVerification]: this Docker version is not on the list of validated versions: 19.03.1. Latest validated version: 18.09
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.15" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
[kubelet-check] Initial timeout of 40s passed.
error execution phase kubelet-start: error uploading crisocket: timed out waiting for the condition

解决办法
在从节点执行如下的命令

[root@ken2 ~]# kubeadm reset
[reset] WARNING: Changes made to this host by 'kubeadm init' or 'kubeadm join' will be reverted.
[reset] Are you sure you want to proceed? [y/N]: y
[preflight] Running pre-flight checks
W0820 16:03:33.400935   56927 removeetcdmember.go:79] [reset] No kubeadm config, using etcd pod spec to get data directory
[reset] No etcd config found. Assuming external etcd
[reset] Please, manually reset etcd to prevent further issues
[reset] Stopping the kubelet service
[reset] Unmounting mounted directories in "/var/lib/kubelet"
[reset] Deleting contents of config directories: [/etc/kubernetes/manifests /etc/kubernetes/pki]
[reset] Deleting files: [/etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf /etc/kubernetes/bootstrap-kubelet.conf /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf]
[reset] Deleting contents of stateful directories: [/var/lib/kubelet /etc/cni/net.d /var/lib/dockershim /var/run/kubernetes]
The reset process does not reset or clean up iptables rules or IPVS tables.
If you wish to reset iptables, you must do so manually.
For example:
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X

If your cluster was setup to utilize IPVS, run ipvsadm --clear (or similar)
to reset your system's IPVS tables.

The reset process does not clean your kubeconfig files and you must remove them manually.
Please, check the contents of the $HOME/.kube/config file.
 

 

然后根据提示执行下面的命令

[root@ken2 ~]# iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
 

再次加入即可成功
```

如果主机状态还是NotReady，可使用 kubectl get pods -n kube-system -o wide

```
[root@k8smaster install]# kubectl get nodes
NAME        STATUS     ROLES    AGE     VERSION
can225      NotReady   <none>   4m6s    v1.17.3
can226      Ready      <none>   4m10s   v1.17.3
k8smaster   Ready      master   5m18s   v1.17.3
```

**使用以下命令在master节点上查看pod的状态**

```
[root@master ~]# kubectl get pods -n kube-system -o wide
NAME                                            READY   STATUS              RESTARTS   AGE   IP                NODE                    NOMINATED NODE   READINESS GATES
coredns-6955765f44-84n5l                        1/1     Running             0          17h   172.16.0.4        localhost.localdomain   <none>           <none>
coredns-6955765f44-zwcx2                        1/1     Running             0          17h   172.16.0.3        localhost.localdomain   <none>           <none>
etcd-localhost.localdomain                      1/1     Running             0          17h   192.168.197.33    localhost.localdomain   <none>           <none>
kube-apiserver-localhost.localdomain            1/1     Running             0          17h   192.168.197.33    localhost.localdomain   <none>           <none>
kube-controller-manager-localhost.localdomain   1/1     Running             0          17h   192.168.197.33    localhost.localdomain   <none>           <none>
kube-flannel-ds-amd64-4gjx8                     1/1     Running             0          49m   192.168.197.33    localhost.localdomain   <none>           <none>
kube-flannel-ds-amd64-l4qt8                     0/1     Init:0/1            0          23m   192.168.197.225   can225                  <none>           <none>
kube-proxy-hw6xl                                1/1     Running             0          17h   192.168.197.33    localhost.localdomain   <none>           <none>
kube-proxy-jbv2p                                0/1     ContainerCreating   0          23m   192.168.197.225   can225                  <none>           <none>
kube-scheduler-localhost.localdomain            1/1     Running             0          17h   192.168.197.33    localhost.localdomain   <none>           <none>
```

kube-proxy-jbv2p  的pod状态为ContainerCreating ,同时kube-flannel-ds-amd64-l4qt8 也为Init:0/1 

可以看到两个node节点上的kube-flannel以及kube-proxy都没有启动起来，那是因为两个node节点上都还没有这两个pod的相关镜像，当然起不起来了，所以接下来需要将master节点上的这两个镜像copy到node节点上

**接着查看pod的日志信息**

```
[root@k8smaster install]# kubectl logs kube-flannel-ds-amd64-l4qt8   
Error from server (NotFound): pods "kube-flannel-ds-amd64-l4qt8" not found
```

接着查看到can225 连镜像都没有

```
[root@can225 .ssh]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
```

后面就是导入镜像了

**对于Error from server (NotFound): pods "kube-proxy-jbv2p " not found的镜像镜像删除时，会提示“*error:* *the* *server* *doesn't* *have* *a* *resource* *type* "pods" ”**

正确的处理方法：原因是之前加入节点时失败，导致存在了pod,应在主节点上正确的移除节点,然后进行重启

```
[root@k8smaster install]# kubectl delete node can225
[root@k8smaster install]# reboot
```

重启之后发现之前的kube-proxy-jbv2p没有了

## 七.重新清除主节点的kubeadm生成的文件

1.清除所有下载安装包

```
 yum -y remove kubeadm kubectl kubelet 
```



```
kubeadm config print init-defaults > kubeadm-config.yaml
```

2.修改kubeadm-config.yaml文件

advertiseAddress为master的ip地址 192.168.197.33

```
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.197.33
  bindPort: 6443
nodeRegistration:
```

kubernetesVersion的版本为v1.17.3

```
kubernetesVersion: v1.17.3
```

添加pod网络

```
networking:
  dnsDomain: cluster.local
  podSubnet: 172.16.0.0/16
  serviceSubnet: 10.96.0.0/12
```

添加ipvs

```
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
featureGates:
  SupportIPVSProxyMode: true
mode: ipvs
```

执行初始化脚本

```
kubeadm init --config=kubeadm-config.yaml | tee kubeadm-init.log
```

如果之前执行过会报错

```
[root@localhost images]# kubeadm init --config=kubeadm-config.yaml | tee kubeadm-init.log
W0227 11:55:46.245391   33864 validation.go:28] Cannot validate kube-proxy config - no validator is available
W0227 11:55:46.245478   33864 validation.go:28] Cannot validate kubelet config - no validator is available
[init] Using Kubernetes version: v1.17.3
[preflight] Running pre-flight checks
error execution phase preflight: [preflight] Some fatal errors occurred:
        [ERROR Port-6443]: Port 6443 is in use
        [ERROR Port-10259]: Port 10259 is in use
        [ERROR Port-10257]: Port 10257 is in use
        [ERROR FileAvailable--etc-kubernetes-manifests-kube-apiserver.yaml]: /etc/kubernetes/manifests/kube-apiserver.yaml already exists
        [ERROR FileAvailable--etc-kubernetes-manifests-kube-controller-manager.yaml]: /etc/kubernetes/manifests/kube-controller-manager.yaml already exists
        [ERROR FileAvailable--etc-kubernetes-manifests-kube-scheduler.yaml]: /etc/kubernetes/manifests/kube-scheduler.yaml already exists
        [ERROR FileAvailable--etc-kubernetes-manifests-etcd.yaml]: /etc/kubernetes/manifests/etcd.yaml already exists
        [ERROR Port-10250]: Port 10250 is in use
        [ERROR Port-2379]: Port 2379 is in use
        [ERROR Port-2380]: Port 2380 is in use
        [ERROR DirAvailable--var-lib-etcd]: /var/lib/etcd is not empty
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```

需要进行处理，reboot

```
netstat -anpt |grep 6443
netstat -anpt |grep 10259
netstat -anpt |grep 10257
netstat -anpt |grep 10250
netstat -anpt |grep 2379
netstat -anpt |grep 2380
以上的pid分别杀掉

rm -rf /etc/kubernetes/manifests/kube-apiserver.yaml
rm -rf /etc/kubernetes/manifests/kube-controller-manager.yaml
rm -rf /etc/kubernetes/manifests/kube-scheduler.yaml
rm -rf /etc/kubernetes/manifests/etcd.yaml
rm -rf /var/lib/etcd
```

重新执行初始化脚本

```
[root@k8smaster images]# kubeadm init --config=kubeadm-config.yaml | tee kubeadm-init.log
W0227 12:03:29.420393   10822 validation.go:28] Cannot validate kube-proxy config - no validator is available
W0227 12:03:29.420498   10822 validation.go:28] Cannot validate kubelet config - no validator is available
[init] Using Kubernetes version: v1.17.3
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Using existing ca certificate authority
[certs] Using existing apiserver certificate and key on disk
[certs] Using existing apiserver-kubelet-client certificate and key on disk
[certs] Using existing front-proxy-ca certificate authority
[certs] Using existing front-proxy-client certificate and key on disk
[certs] Using existing etcd/ca certificate authority
[certs] Using existing etcd/server certificate and key on disk
[certs] Using existing etcd/peer certificate and key on disk
[certs] Using existing etcd/healthcheck-client certificate and key on disk
[certs] Using existing apiserver-etcd-client certificate and key on disk
[certs] Using the existing "sa" key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Using existing kubeconfig file: "/etc/kubernetes/admin.conf"
[kubeconfig] Using existing kubeconfig file: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Using existing kubeconfig file: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Using existing kubeconfig file: "/etc/kubernetes/scheduler.conf"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
W0227 12:03:32.922700   10822 manifests.go:214] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
W0227 12:03:32.923951   10822 manifests.go:214] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s

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

 

 