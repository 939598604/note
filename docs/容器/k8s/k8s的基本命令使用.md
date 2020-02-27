# k8s的基本命令使用

## 一.命名空间

### 1.1 创建命名空间

#### 1.1.1 文件方式

创建test01-namespace

```bash
[root@master ~]# more test01-namespace.yaml 
apiVersion: v1
kind: Namespace
metadata: 
  name: test01-namespace
[root@master ~]# kubectl apply -f test01-namespace.yaml 
namespace/test01-namespace created
```

kubectl apply和kubectl create命令类似，都可以根据文件创建相关资源。

#### 1.1.2 命令方式

创建test01-namespace

```bash
[root@master ~]# kubectl create ns test01-namespace
namespace/test02-namespace created
```

### 1.2  查看命名空间列表

```
[root@localhost k8s]#  kubectl get namespaces
NAME               STATUS    AGE
default            Active    2d
kube-system        Active    2d
test02-namespace   Active    1h
```

或者简写

```
[root@localhost k8s]#  kubectl get ns
NAME               STATUS    AGE
default            Active    2d
kube-system        Active    2d
test02-namespace   Active    1h
```



### 1.3 删除命名空间

删除命名空间时，命名空间中包含的所有资源对象同时被删除。

```
 kubectl delete ns test01-namespace
```

###  1.3 获取某个命名空间下的pod

```
[root@localhost var]# kubectl get pod -n default
NAME                                READY     STATUS    RESTARTS   AGE
mysql-rc-4v95k                      1/1       Running   1          17h
nginx-deployment-3634348883-8mssd   1/1       Running   0          16h
nginx-deployment-3634348883-stctk   1/1       Running   0          16h
nginx-deployment-3634348883-z6jqs   1/1       Running   0          16h
```

### 1.4  **删除namespace中所有资源** 

```
kubectl delete all --all -n test01-namespace 
```



## 二.pod使用

### 2.1 创建pod的两种方式

#### 2.1.1  命令方式

```
kubectl run nginx --image=docker.io/nginx --replicas=3 
```

 nginx指定deployment名字 

 image=docker.io/nginx显示的是指定要运行的镜像，--replicas=3指定副本数为3 

#### 2.1.1 文件方式

```bash
[root@master ~]# more nginx-master.yaml
apiVersion: extensions/v1beta1  #描述文件遵循extensions/v1beta1版本的Kubernetes API
kind: Deployment                #创建资源类型为Deployment
metadata:                       #该资源元数据
  name: nginx-master            #Deployment名称
spec:                           #Deployment的规格说明
  replicas: 3                   #指定副本数为3
  template:                     #定义Pod的模板
    metadata:                   #定义Pod的元数据
      labels:                   #定义label（标签）
        app: nginx              #label的key和value分别为app和nginx
    spec:                       #Pod的规格说明
      containers:               
      - name: nginx             #容器的名称
        image: nginx:latest     #创建容器所使用的镜像
```

 执行创建命令

```
root@master ~]# kubectl create -f nginx-master.yaml 
deployment.extensions/nginx-master created
```

### 2.2  指定node创建pod

```
[root@master ~]# more nginx-node.yaml 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-node
spec:
  template:
    metadata:
      labels:
        env: prod
    spec:
      containers:
      - name: nginx-node
        image: nginx:latest
      nodeSelector:
        node: master
```

###  2.3  查看pod

#### 2.3.1 查看所有pods

查看所有pod

```
[root@localhost k8s]# kubectl get pods                       
NAME             READY     STATUS    RESTARTS   AGE
mysql-rc-4v95k   1/1       Running   1          20h
```

查看所有pod详细ip地址信息

```
[root@localhost k8s]# kubectl get pods -o wide
NAME                           READY     STATUS    RESTARTS   AGE       IP            NODE
nginx-master-101161715-0dd35   1/1       Running   0          32s       172.16.75.3   192.168.197.226
nginx-master-101161715-788mn   1/1       Running   0          32s       172.16.75.4   192.168.197.226
nginx-master-101161715-pctt9   1/1       Running   0          32s       172.16.66.2   192.168.197.225
```

#### 2.3.2 查看某个命名空间下的pods

```
[root@localhost k8s]# kubectl get pods --namespace default
NAME             READY     STATUS    RESTARTS   AGE
mysql-rc-4v95k   1/1       Running   1          19h
```

或者简写-n

```
[root@localhost k8s]# kubectl get pods -n default
NAME             READY     STATUS    RESTARTS   AGE
mysql-rc-4v95k   1/1       Running   1          19h
```

### 2.4 访问pod

exec直接访问pod

```
root@localhost k8s]# kubectl exec -it nginx-master-101161715-0dd35 bin/bash
root@nginx-master-101161715-0dd35:/# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

### 2.5 删除pod

#### 2.5.1 删除某个指定的pod

**命令行方式**

```
[root@localhost k8s]# kubectl delete pods mysql-rc-4v95k -n default    
pod "mysql-rc-4v95k" deleted
```

**文件方式**

```bash
[root@master ~]# more nginx-master.yaml
apiVersion: extensions/v1beta1  #描述文件遵循extensions/v1beta1版本的Kubernetes API
kind: Deployment                #创建资源类型为Deployment
metadata:                       #该资源元数据
  name: nginx                   #Deployment名称
spec:                           #Deployment的规格说明
  replicas: 3                   #指定副本数为3
  template:                     #定义Pod的模板
    metadata:                   #定义Pod的元数据
      labels:                   #定义label（标签）
        app: nginx              #label的key和value分别为app和nginx
    spec:                       #Pod的规格说明
      containers:               
      - name: nginx             #容器的名称
        image: nginx:latest     #创建容器所使用的镜像
```

 执行创建命令

```
root@master ~]# kubectl delete -f nginx-master.yaml 
deployment "nginx" deleted
```



#### 2.5.2  删除所有pods

```
root@master ~]# kubectl delete pods --all
```



##  三.标签

### 3.1 查看pod的标签列表

```
[root@localhost k8s]# kubectl get pod --show-labels  
NAME                           READY     STATUS    RESTARTS   AGE       LABELS
nginx-master-101161715-0dd35   1/1       Running   0          3m        app=nginx,pod-template-hash=101161715
nginx-master-101161715-788mn   1/1       Running   0          3m        app=nginx,pod-template-hash=101161715
nginx-master-101161715-pctt9   1/1       Running   0          3m        app=nginx,pod-template-hash=101161715
```

 通过--show-labels参数可查看pod的标签 

### 3.2  **通过标签筛选pod**

```
[root@localhost k8s]# kubectl get pod -l app --show-labels 
NAME                           READY     STATUS    RESTARTS   AGE       LABELS
nginx-master-101161715-0dd35   1/1       Running   0          4m        app=nginx,pod-template-hash=101161715
nginx-master-101161715-788mn   1/1       Running   0          4m        app=nginx,pod-template-hash=101161715
nginx-master-101161715-pctt9   1/1       Running   0          4m        app=nginx,pod-template-hash=101161715
```

 通过-l app参数可筛选所有标签为app的pod 

### 3.3 修改现有标签

```
[root@localhost k8s]# kubectl label pod nginx-master-101161715-0dd35 env=debug  --overwrite 
```

 将pod nginx-master-101161715-0dd35的标签env由prod更改为debug 

### 3.4 查看修改后的标签

```
[root@localhost k8s]# kubectl get pods --show-labels
NAME                           READY     STATUS    RESTARTS   AGE       LABELS
nginx-master-101161715-0dd35   1/1       Running   0          7m        app=nginx,env=debug,pod-template-hash=101161715
nginx-master-101161715-788mn   1/1       Running   0          7m        app=nginx,pod-template-hash=101161715
nginx-master-101161715-pctt9   1/1       Running   0          7m        app=nginx,pod-template-hash=101161715
```

### 3.5  **删除标签**

```
kubectl label pod nginx-master-101161715-0dd35 env-
```

 将pod nginx-master-101161715-0dd35的标签env删除 

### 3.6 查看修改后的标签

```
[root@localhost k8s]# kubectl get pods --show-labels
NAME                           READY     STATUS    RESTARTS   AGE       LABELS
nginx-master-101161715-0dd35   1/1       Running   0          7m        app=nginx,pod-template-hash=101161715
nginx-master-101161715-788mn   1/1       Running   0          7m        app=nginx,pod-template-hash=101161715
nginx-master-101161715-pctt9   1/1       Running   0          7m        app=nginx,pod-template-hash=101161715
```

## 四 deployment控制器

### 4.1deployment的创建

#### 4.1.1 命令行方式

```
kubectl run nginx-deployment --image=192.168.197.224/gzstrong/nginx:latest --image-pull-policy=IfNotPresent --replicas=2 
```

默认没有指定类型，其实就是deployment

nginx-deployment 是deployment名称

image 镜像是192.168.197.224/gzstrong/nginx:latest

replicas 副本数是2

#### 4.1.2 文件清单

vim nginx-deployment.yaml 

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx-deployment
    spec:
      containers:
      - name: nginx-deployment
        image: 192.168.197.224/gzstrong/nginx:latest
        ports:
        - containerPort: 80
```

创建命令

```
kubectl create -f nginx-deployment.yaml 
```

### 4.2 deployment的查看

```
kubectl get deployments
```

### 4.3 deployment的删除

带有控制器类型的Pod不能随便删除，如果必须删除，请删除控制器类型的应用名称。

如果直接删除pod，你会发现被删除pod又重新建立起来 了

#### 4.3.1 命令的方式

```
kubectl delete deployment nginxdaf
```

#### 4.3.2  文件清单方式

vim nginx-deployment.yaml 

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx-deployment
    spec:
      containers:
      - name: nginx-deployment
        image: 192.168.197.224/gzstrong/nginx:latest
        ports:
        - containerPort: 80
```



```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx
  template:
    metadata:
      labels:
        name: nginx
    spec:
      containers:
        - name: nginx
          image: 192.168.197.224/gzstrong/nginx:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-nodeport
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: NodePort
  selector:
    name: nginx
```



删除命令

```
kubectl delete -f nginx-deployment.yaml 
```



日志查看

### 4.1  查看最近的日志

```
[root@master ~]# kubectl logs kubernetes-dashboard-7b87f5bdd6-m62r6 -n kube-system --tail=20
```

 '--tail=20':查看命名空间kube-system下kubernetes-dashboard-7b87f5bdd6-m62r6最近20行的日志， 

或者不指定命名空间，直接指定pod

```
[root@master ~]# kubectl logs nginx-master-101161715-0dd35 --tail=20  
```

### 4.2 通过标签查看日志

 查看标签为'app=web'的日志 

```
[root@master ~]# kubectl logs -lapp=web
```

 '-lapp=web'该日志是标签为'app=web'的合集 



## 五.apiVersion版本问题

### 5.1 pod版本

 Pod的apiVersion为apps/v1

```
apiVersion: v1
kind: Pod
```

### 5.2 Deployment版本

 Deployment的apiVersion不能为apps/v1，修改为extensions/v1beta1

```
apiVersion: apps/v1
kind: Deployment
```

修改为

```
apiVersion: extensions/v1beta1
kind: Deployment
```

