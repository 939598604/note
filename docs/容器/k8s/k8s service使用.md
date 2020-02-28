# k8s service使用

Service在K85中有以下四种类型

- Clusterlp：默认类型，自动分配一个仅Cluster内部可以访问的虚拟lP
- NodePort：在ClusterIP基础上为Service在每台机器上绑定一个端口，这样就可以通过<NodelP>：
-  NodePort 来访问该服务
-  LoadBalancer：在NodePort的基础上，借助 cloud provider 创建一个外部负载均衡器，并将请求转发到
-  ExternalName：把集群外部的服务引入到集群内部来，班集群内部直接使用。没有任何类型代理被创建，这只有kubernetes 1.7或更高版本的kube-dns才支持

## 一. Clusterlp

默认类型，自动分配一个仅Cluster内部可以访问的虚拟lP

### 1.1 Clusterlp原理

clusterlP主要在每个node节点使用iptables，将发向clusterlP对应端口的数据，转发到kube-proxy中，然后kube-proxy 自己内部实现有负载均衡的方法，并可以查询到这个service下对应pod的地址和端口。进而把数据转发给对应的pod的地址和端口



![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200228140731.png)



为了实现图上的功能，主要需要以下几个组件的协同工作：
- apiserver 用户通过kubectl命令向apiserver发送创建service的命令，apiserver接收到请求后将数据存储到etcd中
- kube-proxy kubernetes的每个节点中都有一个叫做kube-porxy的进程，这个进程负责感知service，pod的变化，并将变化的信息写入本地的iptables规则中
- iptables 使用NAT等技术格virtuallP的流量转至endpoint中

### 1.2 配置

vim  nginx-clusterlp-svc.yml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-clusterlp-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      name: nginx-clusterlp
  template:
    metadata:
      labels:
        name: nginx-clusterlp
    spec:
      containers:
        - name: nginx-clusterlp
          image: 192.168.197.224/gzstrong/nginx:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterlp-svc
spec:
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
  selector:
    name: nginx-clusterlp
```

启动服务

```
[root@k8smaster k8s]# kubectl apply -f nginx-clusterlp-svc.yml
deployment.apps/nginx-clusterlp-deployment created
```

###1.3 查看pod

```
[root@k8smaster k8s]# kubectl  get pods -o wide
NAME                                          READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
nginx-clusterlp-deployment-7d5d9f9cb7-rhf5r   1/1     Running   0          12s   10.244.3.5   can224   <none>           <none>
nginx-clusterlp-deployment-7d5d9f9cb7-sjpld   1/1     Running   0          12s   10.244.2.5   can225   <none>           <none>
nginx-clusterlp-deployment-7d5d9f9cb7-wfbsq   1/1     Running   0          12s   10.244.1.5   can226   <none>           <none>
```

### 1.4 查看service

```
[root@k8smaster k8s]# kubectl  get svc -o wide
NAME                  TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE    SELECTOR
kubernetes            ClusterIP   10.96.0.1    <none>        443/TCP        3h9m   <none>
nginx-clusterlp-svc   NodePort    10.100.2.1   <none>        80:30015/TCP   20s    name=nginx-clusterlp
```

### 1.5 访问

在k8s的node节点，直接访问 nginx-clusterlp-svc 的ip地址：10.100.2.1

```
[root@can224 ~]# curl 10.102.117.41
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```



## 二.Headless Service

### 1.1 无头服务

有时不需要或不想要负载均衡，以及单独的Service IP。遇到这种情况，可以通过指定 Cluster lP（spec.clusterIP）的值为“None”来创建 Headless Service.这类Service 并不会分配Cluster IP，kube-proxy不会处理它们，而且平台也不会为它们进行负载均衡和路由

### 1.2 配置

vim  nginx-headless-svc.yml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-headless-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      name: nginx-headless
  template:
    metadata:
      labels:
        name: nginx-headless
    spec:
      containers:
        - name: nginx-headless
          image: 192.168.197.224/gzstrong/nginx:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless-svc
spec:
  ports:
    - port: 80
      targetPort: 80
  clusterIP: None
  selector:
    name: nginx-headless
```

启动服务

```
[root@k8smaster k8s]# kubectl apply -f nginx-headless-svc.yml
deployment.apps/nginx-headless-deployment created
```



### 1.3 查看pod

```
[root@k8smaster k8s]# kubectl  get pods -o wide                   
NAME                                         READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
nginx-headless-deployment-56f8c75d8f-4jwss   1/1     Running   0          23s   10.244.1.9   can226   <none>           <none>
nginx-headless-deployment-56f8c75d8f-lvhkk   1/1     Running   0          23s   10.244.3.9   can224   <none>           <none>
nginx-headless-deployment-56f8c75d8f-nk7v9   1/1     Running   0          23s   10.244.2.9   can225   <none>           <none>
```

### 1.4 查看service

```
[root@k8smaster k8s]# kubectl  get svc -o wide                   
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE     SELECTOR
kubernetes           ClusterIP   10.96.0.1    <none>        443/TCP   3h40m   <none>
nginx-headless-svc   ClusterIP   None         <none>        80/TCP    29s     name=nginx-headless
```

### 1.5 域名解析

在上面的CLUSTER-IP地址是为空的，当创建一个无头服务会往kubernetes的coreDns里面写入一个域名，域名格式是：svc名字+.命名空间+.集群的域名

因没有改集群名字，默认是 .default.svc.cluster.local  

```
nginx-headless.default.svc.cluster.local
```

获取一个coredns的ip地址

```
[root@k8smaster k8s]# kubectl get pods -o wide -n kube-system 
NAME                                READY   STATUS    RESTARTS   AGE    IP                NODE        NOMINATED NODE   READINESS GATES
coredns-6955765f44-29tzf            1/1     Running   0          4h3m   10.244.0.4        k8smaster   <none>           <none>
coredns-6955765f44-8w4x9            1/1     Running   0          4h3m   10.244.0.3        k8smaster   <none>           <none>
```

通过coredns的地址10.244.0.3或者10.244.0.4 解析

```
[root@k8smaster k8s]# dig -t A nginx-headless-svc.default.svc.cluster.local. @10.244.0.3

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-9.P2.el7 <<>> -t A nginx-headless-svc.default.svc.cluster.local. @10.244.0.3
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41902
;; flags: qr aa rd; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;nginx-headless-svc.default.svc.cluster.local. IN A

;; ANSWER SECTION:
nginx-headless-svc.default.svc.cluster.local. 30 IN A 10.244.3.9
nginx-headless-svc.default.svc.cluster.local. 30 IN A 10.244.1.9
nginx-headless-svc.default.svc.cluster.local. 30 IN A 10.244.2.9

;; Query time: 0 msec
;; SERVER: 10.244.0.3#53(10.244.0.3)
;; WHEN: 五 2月 28 15:10:53 CST 2020
;; MSG SIZE  rcvd: 253
```

上面解析出来的地址是

```
nginx-headless-svc.default.svc.cluster.local. 30 IN A 10.244.3.9
nginx-headless-svc.default.svc.cluster.local. 30 IN A 10.244.1.9
nginx-headless-svc.default.svc.cluster.local. 30 IN A 10.244.2.9
```



## 三.NodePort

### 3.1 NodePort原理

在ClusterIP基础上为Service在每台机器上绑定一个端口，这样就可以通过NodePort 来访问该服务

nodePort的原理在于在node上开了一个端口，将向该端口的流量导入到kube-proxy，然后由kube-proxy 进一步到给对应的pod

### 3.2 配置

vIm nginx-nodeport-svc.yml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-nodeport-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      name: nginx-nodeport-deployment
  template:
    metadata:
      labels:
        name: nginx-nodeport-deployment
    spec:
      containers:
        - name: nginx-nodeport-deployment
          image: 192.168.197.224/gzstrong/nginx:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport-svc
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: NodePort
  selector:
    name: nginx-nodeport-deployment
```

启动服务

```
[root@k8smaster k8s]# kubectl apply -f nginx-nodeport-svc.yml
deployment.apps/nginx-nodeport-deployment created
```

###3.3 查看pod

```
[root@k8smaster k8s]# kubectl  get pods -o wide
NAME                                        READY   STATUS    RESTARTS   AGE     IP            NODE     NOMINATED NODE   READINESS GATES
nginx-nodeport-deployment-66c5854cc-596zt   1/1     Running   0          2m29s   10.244.2.10   can225   <none>           <none>
nginx-nodeport-deployment-66c5854cc-5jmj5   1/1     Running   0          2m29s   10.244.3.10   can224   <none>           <none>
nginx-nodeport-deployment-66c5854cc-flcrp   1/1     Running   0          2m29s   10.244.1.10   can226   <none>           <none>
```

### 3.4 查看service

```
[root@k8smaster k8s]# kubectl  get svc -o wide
NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE     SELECTOR
nginx-nodeport-svc   NodePort    10.97.9.219   <none>        80:32197/TCP   2m43s   name=nginx-nodeport-deployment
```

### 3.5 访问

在浏览器上输入http://node节点ip地址:32197，3个node节点都可以访问到

```
http://192.168.197.224:32197/
http://192.168.197.225:32197/
http://192.168.197.226:32197/
```



## 四. LoadBalancer

### 4.1 LoadBalancer原理

在NodePort的基础上，借助 cloud provider 创建一个外部负载均衡器，并将请求转发到NodePort

loadBalancer和nodePort其实是同一种方式。区别在于loadBalancer比nodePort多了一步，就是可以调用cloud provider 去创建LB来向节点导流

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200228152823.png)



## 五.ExternalName

### 5.1ExternalName原理

把集群外部的服务引入到集群内部来，班集群内部直接使用。没有任何类型代理被创建，这只有kubernetes 1.7或更高版本的kube-dns才支持

这种类型的Service 通过返回CNAME和它的值，可以将服务映射到externalName字段的内容（例如：hub.atguigu.com）。ExternalName Service 是Service的特例，它没有selector，也没有定义任何的端口和Endpoint。相反的，对于运行在集群外部的服务，它通过返回该外部服务的别名这种方式来提供服务



当查询主机 my-servide.defalut.svc.cluster.local（SVC_NAME.NAMESPACE.sVC.cluster.local）时，集群的DNS 服务将返回一个值my.database.example.com的CNAME记录。访问这个服务的工作方式和其他的相同，唯一不同的是重定向发生在DNS层，而且不会进行代理或转发

### 5.2 配置

vIm nginx-externalname-svc.yml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-externalname-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      name: nginx-externalname-deployment
  template:
    metadata:
      labels:
        name: nginx-externalname-deployment
    spec:
      containers:
        - name: nginx-externalname-deployment
          image: 192.168.197.224/gzstrong/nginx:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-externalname-svc
spec:
  type: ExternalName
  selector:
  externalName: www1.gzstrong.com
```

启动服务

```
[root@k8smaster k8s]# kubectl apply -f nginx-externalname-svc.yml
deployment.apps/nginx-externalname-deployment created
```

###5.3 查看pod

```
[root@k8smaster k8s]# kubectl  get pods -o wide
NAME                                             READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
nginx-externalname-deployment-6d545b5587-6s8js   1/1     Running   0          41s   10.244.2.11   can225   <none>           <none>
nginx-externalname-deployment-6d545b5587-hmlsj   1/1     Running   0          41s   10.244.3.11   can224   <none>           <none>
nginx-externalname-deployment-6d545b5587-s48j6   1/1     Running   0          41s   10.244.1.11   can226   <none>           <none>
```

### 5.4 查看service

```
[root@k8smaster k8s]# kubectl  get svc -o wide
NAME                     TYPE           CLUSTER-IP   EXTERNAL-IP         PORT(S)   AGE     SELECTOR
nginx-externalname-svc   ExternalName   <none>       www1.gzstrong.com   <none>    53s     <none>
```

### 5.5 解析

dig -t A nginx-externalname-svc.default.svc.cluster.local. @10.244.0.3

```
[root@k8smaster k8s]# dig -t A nginx-externalname-svc.default.svc.cluster.local. @10.244.0.3

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-9.P2.el7 <<>> -t A nginx-externalname-svc.default.svc.cluster.local. @10.244.0.3
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41083
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;nginx-externalname-svc.default.svc.cluster.local. IN A

;; ANSWER SECTION:
nginx-externalname-svc.default.svc.cluster.local. 26 IN CNAME www1.gzstrong.com.

;; Query time: 0 msec
;; SERVER: 10.244.0.3#53(10.244.0.3)
;; WHEN: 五 2月 28 15:37:25 CST 2020
;; MSG SIZE  rcvd: 156
```

解析出是CNAME记录，其域名是 www1.gzstrong.com

