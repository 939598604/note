# k8s安装nginx

## 1.新建nginx的deployment资源清单

cat nginx-deployment.yaml 

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: 192.168.197.224/gzstrong/nginx:latest
        ports:
        - containerPort: 80
```



## 2.新建nginx的service资源清单

```
kind: Service
apiVersion: v1
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30001
```

kubectl apply -f nginx-service.yaml