# k8s ingress使用

## 一.ingress的安装

### 1.1获取ingress

```
获取配置文件位置: https://github.com/kubernetes/ingress-nginx/tree/nginx-0.20.0/deploy
默认下载最新的yaml：
wget  https://github.com/kubernetes/ingress-nginx/blob/nginx-0.20.0/deploy/mandatory.yaml
```

### 1.2 准备镜像

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

