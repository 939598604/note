# k8s内外网访问

## 一.内网访问



## 二.外网访问

### 2.1.nodePort方式

#### 2.1.1 新建Deployment

vim nginx-deployment.yaml

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx-deployment
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx-deployment
    spec:
      containers:
      - name: nginx-deployment
        image: 192.168.197.224/gzstrong/nginx:latest
```

2.1.2 新建servcie

vim nginx-svc.yaml

```bash
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  type: NodePort
  selector:
    app: nginx-deployment
  ports:
  - protocol: TCP
    nodePort: 30002
    port: 8080
    targetPort: 80
```

新增'type: NodePort'和'nodePort: 30002'，通过NodePort方式外网访问pod，映射端口为30002，重新kubectl apply

 



```bash
[root@master ~]# for i in {1..10};do sleep 1;curl 172.27.9.131:30002;done
web-svc-58956c55fc-7vnw5
web-svc-58956c55fc-8wbst
 web-svc-58956c55fc-nxt4r
web-svc-58956c55fc-7vnw5
web-svc-58956c55fc-7vnw5
 web-svc-58956c55fc-nxt4r
web-svc-58956c55fc-8wbst
web-svc-58956c55fc-7vnw5
 web-svc-58956c55fc-nxt4r
web-svc-58956c55fc-7vnw5
```

通过node ip + nodePort方式外网访问pod

![k8s实践(三)：pod常用操作](https://s1.51cto.com/images/blog/201908/22/352a776aa1bd54a30bf4cc734f8f8501.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)