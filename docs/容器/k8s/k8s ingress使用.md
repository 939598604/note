# k8s ingress使用

## 一.ingress的安装

### 1.1 准备镜像

```
[root@k8smaster install]# cat mandatory.yaml |grep imag
image: k8s.gcr.io/defaultbackend-amd64:1.5
image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.20.0
```

需要k8s.gcr.io/defaultbackend-amd64:1.5和quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.20.0镜像，因为在linux里面访问不了`k8s.gcr.io其替换为gcr.azk8s.cn/google-containers `和`quay.io替换为 quay.azk8s.cn`

```
gcr.azk8s.cn/google-containers/defaultbackend-amd64:1.5
quay.azk8s.cn/kubernetes-ingress-controller/nginx-ingress-controller:0.20.0
```
### 1.2 获取ingress
```
获取配置文件位置: https://github.com/kubernetes/ingress-nginx/tree/nginx-0.20.0/deploy
默认下载最新的yaml：
wget  https://github.com/kubernetes/ingress-nginx/blob/nginx-0.20.0/deploy/mandatory.yaml
```


下载https://github.com/kubernetes/ingress-nginx/blob/master/deploy/static/mandatory.yaml文件

[root@k8smaster ~]# cat  /k8s/install/mandatory.yaml 

```yml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: default-http-backend
  labels:
    app.kubernetes.io/name: default-http-backend
    app.kubernetes.io/part-of: ingress-nginx
  namespace: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: default-http-backend
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: default-http-backend
        app.kubernetes.io/part-of: ingress-nginx
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: default-http-backend
          image: k8s.gcr.io/defaultbackend-amd64:1.5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 30
            timeoutSeconds: 5
          ports:
            - containerPort: 8080
          resources:
            limits:
              cpu: 10m
              memory: 20Mi
            requests:
              cpu: 10m
              memory: 20Mi

---
apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: default-http-backend
    app.kubernetes.io/part-of: ingress-nginx
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app.kubernetes.io/name: default-http-backend
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses/status
    verbs:
      - update

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      # Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<nginx>"
      # This has to be adapted if you change either parameter
      # when launching the nginx-ingress-controller.
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.20.0
          args:
            - /nginx-ingress-controller
            - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1

---
```

### 1.3 启动

```
kubectl apply -f install/mandatory.yaml
```

自动下载镜像，无法下载导入镜像即可

### 1.4 查看

pod查a看

```
[root@k8smaster ~]# kubectl get pod -n ingress-nginx
NAME                                        READY   STATUS             RESTARTS   AGE
default-http-backend-67b784f564-knn54       0/1     ImagePullBackOff   0          36m
nginx-ingress-controller-84776bcf58-4x2x5   1/1     Running            0          36m
```

svc查看

```
[root@k8smaster ~]# kubectl get svc 
NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes          ClusterIP   10.96.0.1        <none>        443/TCP        11h
nginx-ingress-svc   NodePort    10.107.188.110   <none>        80:32273/TCP   22m
```



## 二.部署nodePort

### 2.1 下载yml

下载https://github.com/kubernetes/ingress-nginx/edit/master/deploy/static/provider/baremetal/service-nodeport.yaml文件

内容如下

vim service-nodeport.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
```

### 2.2 启动nodePort

```
kubectl apply -f service-nodeport.yaml
```

##  三.部署单个域名的http ingress应用

### 3.1 创建yaml文件

vim nginx-ingress.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      name: nginx-ingress-deployment
  template:
    metadata:
      labels:
        name: nginx-ingress-deployment
    spec:
      containers:
        - name: nginx-ingress-deployment
          image: 192.168.197.224/gzstrong/nginx:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress-svc
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  selector:
    name: nginx-ingress-deployment
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  - host: www.test.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-ingress-svc
          servicePort: 80
```

设置了域名为 www.test.com

### 3.2 启动应用

```
kubectl apply -f nginx-ingress.yml
```

### 3.3 配置Hosts

在访问的机器设置hosts

```
master的ip地址  www.test.com
```

### 3.4 获取ingress的代理端口

```
[root@k8smaster yaml]#  kubectl get svc -n ingress-nginx
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
default-http-backend   ClusterIP   10.108.62.40    <none>        80/TCP                       38m
ingress-nginx          NodePort    10.102.105.33   <none>        80:32273/TCP,443:30823/TCP   25m
```

代理端口为32273

### 3.5 访问域名

```
[root@k8smaster yaml]# curl www.test.com:32273
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

总结：定义一个deployment

​           定义一个ClusterIp类型的svc

​          定义一个ingress

​          获取ingress的代理端口

访问：域名：+ingress的代理端口

## 四.部署两个域名的http ingress应用

### 4.1 创建yaml文件

vim nginx-ingress-vhost1.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-deployment-vhost1
spec:
  replicas: 3
  selector:
    matchLabels:
      name: nginx-ingress-deployment-vhost1
  template:
    metadata:
      labels:
        name: nginx-ingress-deployment-vhost1
    spec:
      containers:
        - name: nginx-ingress-deployment-vhost1
          image: 192.168.197.224/gzstrong/nginx:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress-svc-vhost1
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  selector:
    name: nginx-ingress-deployment-vhost1
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress-vhost1
spec:
  rules:
  - host: www1.test.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-ingress-svc-vhost1
          servicePort: 80
```

vim nginx-ingress-vhost2.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-deployment-vhost2
spec:
  replicas: 3
  selector:
    matchLabels:
      name: nginx-ingress-deployment-vhost2
  template:
    metadata:
      labels:
        name: nginx-ingress-deployment-vhost2
    spec:
      containers:
        - name: nginx-ingress-deployment-vhost2
          image: 192.168.197.224/gzstrong/nginx:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress-svc-vhost2
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  selector:
    name: nginx-ingress-deployment-vhost2
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress-vhost2
spec:
  rules:
  - host: www2.test.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-ingress-svc-vhost2
          servicePort: 80
```

### 4.2 启动应用

```
kubectl apply -f nginx-ingress-vhost1.yaml
kubectl apply -f nginx-ingress-vhost2.yaml
```

### 4.3 配置Hosts

在访问的机器设置hosts

```
master的ip地址  www1.test.com
master的ip地址  www2.test.com
```

### 4.4 获取ingress的代理端口

```
[root@k8smaster yaml]#  kubectl get svc -n ingress-nginx
NAME                  TYPE        CLUSTER-IP    EXTERNAL-IP PORT(S)                   AGE
default-http-backend ClusterIP   10.108.62.40    <none>   80/TCP                      48m
ingress-nginx        NodePort   10.102.105.33   <none>    80:30269/TCP,443:30823/TCP  35m
```

代理端口为30269

### 4.5 访问域名

```
curl www1.test.com:30269
curl www2.test.com:30269
```



## 五.部署单个域名的https ingress应用

### 5.1 需求

我们有这样的一个需求，就是把 Pod 集群升级为 https，目前的办法就是要么每个容器配置 https，然后前端通过 Service 进行调度，但是这样配置起来会比较麻烦，以及每个容器的建立都通过 https ，也增加了建立连接的负担。

我们需要一种这样的改造，就是客户端连接到 Service 是通过 https，而 Service 向后端 Pod 的调度通过 http，这样可以极大的优化我们的集群，这里我们就需要用到 Kubernetes 的另外一种资源 [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)。

![img](http://i2.51cto.com/images/blog/201812/19/b746214d52bae3d2f76842e987275916.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

#### 5.1.1 Ingress

Ingress 就是一个负载均衡的应用，它和 Service 的不同之处在于，Service 只可以支持 4 层的负载均衡，而 Ingress 是支持 7 层的负载均衡，支持 http 和 https，包括通过主机名的访问已经路径访问的过滤。

那为什么不直接使用 Nginx？这是因为在 K8S 集群中，如果每加入一个服务，我们都在 Nginx 中添加一个配置，其实是一个重复性的体力活，只要是重复性的体力活，我们都应该通过技术将它干掉。

Ingress就可以解决上面的问题，其包含两个组件`Ingress Controller`和`Ingress`：

**Ingress：**将Nginx的配置抽象成一个Ingress对象，每添加一个新的服务只需写一个新的Ingress的yaml文件即可；
**Ingress Controller：**将新加入的 Ingress 转化成 Nginx 的配置文件并使之生效，包含 Contour、F5、HAProxy、Istio、Kong、Nginx、Traefik，官方推荐我们使用 Nginx。

![img](http://i2.51cto.com/images/blog/201812/19/2b9b75788cd85337589bbff2c9424554.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

####  5.1.1.2 环境介绍

我们是采用了三台服务器的一个集群，部署文档请查看我之前的[博文](http://blog.51cto.com/wzlinux/2322616)。

| IP            | 角色       |
| ------------- | ---------- |
| 192.168.1.200 | k8s-master |
| 192.168.1.201 | k8s-node01 |
| 192.168.1.202 | k8s-node02 |

```
[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES    AGE    VERSION
master   Ready    master   117s   v1.13.0
node01   Ready    <none>   52s    v1.13.0
node02   Ready    <none>   42s    v1.13.0
```

### 5.2 安装部署

我们这里只针对上面架构图中的域名`www.wzlinux.com`改造成https。

我们将以官方的标准脚本为基础进行搭建，参考请戳[官方文档](https://kubernetes.github.io/ingress-nginx/deploy/)。官方文档中要求执行如下命令：

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
```

#### 5. 2.1 创建后端 Pod 应用

我们创建一个控制器`wzlinux-deploy.yaml`，内容如下：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: wzlinux-dep
spec:
  replicas: 3
  template:
    metadata:
      labels:
        run: wzlinux
    spec:
      containers:
      - name: wzlinux
        image: wangzan18/mytest:v1
        ports:
        - containerPort: 8080
```

创建好之后查看如下：

```
[root@master ingress]# kubectl get pod
NAME                           READY   STATUS    RESTARTS   AGE
wzlinux-dep-78d5d86c7c-fj8f5   1/1     Running   0          53m
wzlinux-dep-78d5d86c7c-hr6gd   1/1     Running   0          53m
wzlinux-dep-78d5d86c7c-jqf59   1/1     Running   0          53m
```

#### 5.2.2 创建后端 Pod Service

测试好 Pod 一些正常之后，我们为这一组 Pod 创建一个 Service，文件`wzlinux-svc.yaml`内容如下：

```
apiVersion: v1
kind: Service
metadata:
  name: wzlinux-svc
spec:
  selector:
    run: wzlinux
  ports:
  - port: 80
    targetPort: 8080
```

这个 Service 并不是我们用了代理访问 Pod 的，只是用来`ingress-controller`来进行选择控制使用的，所以上图描述为虚线。

```
[root@master ingress]# kubectl get svc
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP    58m
wzlinux-svc   ClusterIP   10.106.219.230   <none>        8080/TCP   50m
[root@master ingress]# curl 10.106.219.230:8080
Hello Kubernetes bootcamp! | Running on: wzlinux-dep-78d5d86c7c-fj8f5 | v=1
```

#### 5. 2.3 创建 ingress 资源

为了实现过滤以及 https 功能，我们需要创建 ingress 资源文件，ingress controller 把其中的资源加载到 nginx 里面，资源文件`wzlinux-ingress.yaml`文件内容如下：

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: wzlinux-ingress
spec:
  rules:
  - host: www.wzlinux.com
    http:
      paths:
      - path:
        backend:
          serviceName: wzlinux-svc
          servicePort: 8080
```

我们这里先不改为 https，先使用虚拟主机域名过滤模式，创建好资源之后查看

```
[root@master ingress]# kubectl get ingress
NAME              HOSTS             ADDRESS   PORTS   AGE
wzlinux-ingress   www.wzlinux.com             80      37m
```

可以看到配置了域名`www.wzlinux.com`，其他地址访问将返回404。

#### 5.2.4 为 Nginx Pod 创建 Service

我们可以查看部署的 Nginx Pod 容器，我们设定的 ingress 资源会被 controller 更新到里面，我们可以查看如下：

```
[root@master ingress]# kubectl get pod -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-766c77b7d4-dlcpf   1/1     Running   0          31m
```

为了是外网可以访问到这个 Nginx Pod，我们需要为其再创建一个 Service，文件`ingress-nginx.yaml`，文件内容如下：

```
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30080
  - name: https
    port: 443
    targetPort: 443
    nodePort: 30443
  selector:
    app.kubernetes.io/name: ingress-nginx
```

测试是否正常，记得在`/etc/hosts`中把域名执行的IP改为node节点的地址。

```
[root@master ~]# curl 192.168.1.200:30080
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.15.6</center>
</body>
</html>
[root@master ~]# curl www.wzlinux.com:30080
Hello Kubernetes bootcamp! | Running on: wzlinux-dep-78d5d86c7c-hr6gd | v=1
```

可以看到通过域名访问正常调度到后端，其他地址访问返回404，目前整个流程已经测试完成，下面我们升级为 https。

### 5.3 升级为 https

#### 5.3.1 首先我们要制作证书

关于证书大家可以使用 openssl 制作，创建私有：

```
openssl genrsa -out wzlinux.key 2048
```

制作自签证书。

```
openssl req -new -x509 -key wzlinux.key -out wzlinux.crt -subj /C=CN/ST=Shanghai/L=Shanghai/O=DevOps/CN=www.wzlinux.com
```

不过我这里使用阿里云的官方免费证书，大家可以到阿里云进行申请。
![img](http://i2.51cto.com/images/blog/201812/19/b6f388565ef5ab4ad0d593f3881ab7cb.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

![img](http://i2.51cto.com/images/blog/201812/19/6fdf91f3fdd02639b145bed1222bce3f.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)
制作好证书之后下载即可，里面包含公钥和私钥。

#### 5.3.2、创建 secret 资源

可以使用 yaml 文件创建，文件名称`wzlinux-secret.yaml`内容如下：

```
apiVersion: v1
kind: Secret
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
metadata:
  name: wzlinux-secret
  namespace: default
type: Opaque
```

因为编码的密码太长，我们这里直接使用命令行进行创建吧，操作比较简单。

```
kubectl create secret tls wzlinux-secret --cert=wzlinux.crt --key=wzlinux.key
```

查看创建好的 secret。

```
[root@master ingress]# kubectl describe secret wzlinux-secret
Name:         wzlinux-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/tls

Data
====
tls.crt:  1996 bytes
tls.key:  1675 bytes
```

#### 5.3.3 更改 ingress 资源

重新编辑`wzlinux-ingress.yaml`，增加一个 tls 字段：

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: wzlinux-ingress
spec:
  tls:
  - hosts:
    - www.wzlinux.com
    secretName: wzlinux-secret
  rules:
  - host: www.wzlinux.com
    http:
      paths:
      - path:
        backend:
          serviceName: wzlinux-svc
          servicePort: 8080
```

#### 5.3.4 浏览器访问验证

打开浏览器，记得修改好 hosts 域名解析。
![img](http://i2.51cto.com/images/blog/201812/19/5a41ceae0a746463942a825219667994.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

### 5.4 ingress 资源介绍

####  5.4.1 通过访问路径过滤

```repl
foo.bar.com -> 178.91.123.132 -> / foo    service1:4200
                                 / bar    service2:8080
```

配置文件我们设置为如下：

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: simple-fanout-example
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: service1
          servicePort: 4200
      - path: /bar
        backend:
          serviceName: service2
          servicePort: 8080
```

####  5.4.2 基于名称解析的虚拟主机

```
foo.bar.com --|                 |-> foo.bar.com s1:80
              | 178.91.123.132  |
bar.foo.com --|                 |-> bar.foo.com s2:80
```

配置文件内容格式如下：

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: first.bar.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
  - host: second.foo.com
    http:
      paths:
      - backend:
          serviceName: service2
          servicePort: 80
  - http:
      paths:
      - backend:
          serviceName: service3
          servicePort: 80
```

####  5.4.3 https

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
    - sslexample.foo.com
    secretName: testsecret-tls
  rules:
    - host: sslexample.foo.com
      http:
        paths:
        - path: /
          backend:
            serviceName: service1
            servicePort: 80
```
