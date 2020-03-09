# docker镜像导入导出

## 一.镜像导出

### 1.1 查看镜像

```
[root@can225 ~]# docker images 
REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
192.168.197.224/gzstrong/nginx        latest              2073e0bcb60e        3 weeks ago         127 MB
docker.io/nginx                       latest              2073e0bcb60e        3 weeks ago         127 MB
docker.io/rancher/rancher-agent       v2.3.5              f29ece87a195        4 weeks ago         282 MB
docker.io/rancher/rke-tools           v0.1.52             6e421b8753a2        2 months ago        132 MB
```

### 1.2 镜像打包

```
[root@can225 ~]# docker save -o nginx.tar docker.io/nginx:latest
```

### 1.3 查看当前目录

```
[root@can225 ~]# ll
-rw-------  1 root root 130548224 Feb 26 09:26 nginx.tarll 
```

## 二.镜像导入

### 2.1 导入.镜像

```
docker load -i nginx.tar 
docker images
```

## 三.docker批量操作

### 3.1 docker批量启动关闭删除

**docker中 启动所有的容器命令**

```dart
docker start $(docker ps -a | awk '{ print $1}' | tail -n +2)
```

**docker中 关闭所有的容器命令**

```dart
docker stop $(docker ps -a | awk '{ print $1}' | tail -n +2)
```

**docker中 删除所有的容器命令**

```dart
docker rm -f $(docker ps -a | awk '{ print $1}' | tail -n +2)
```

**docker中 删除所有的镜像命令**

```
docker rmi -f $(docker images | awk '{ print $3}')
```



### 3.2 docker 批量列表下载

```
#!/bin/bash
img_list='
registry.cn-shenzhen.aliyuncs.com/rancher_cn/flannel
registry.cn-shenzhen.aliyuncs.com/rancher_cn/flannel-cni
registry.cn-shenzhen.aliyuncs.com/rancher_cn/etcd:latest
rancher/k8s:v1.8.3-rancher2
rancher/k8s:v1.8.3-rancher2
rancher/k8s:v1.8.3-rancher2
rancher/k8s:v1.8.3-rancher2
registry.cn-shenzhen.aliyuncs.com/rancher_cn/pause-amd64:3.0
rancher/k8s:v1.8.3-rancher2
registry.cn-shenzhen.aliyuncs.com/rancher_cn/k8s-dns-kube-dns-amd64:1.14.5
registry.cn-shenzhen.aliyuncs.com/rancher_cn/k8s-dns-dnsmasq-nanny-amd64:1.14.5
registry.cn-shenzhen.aliyuncs.com/rancher_cn/k8s-dns-sidecar-amd64:1.14.5
registry.cn-shenzhen.aliyuncs.com/rancher_cn/cluster-proportional-autoscaler-amd64:1.0.0
nginx'

for img in $img_list
do
   docker pull $img
done
```

### 3.3  docker镜像批量打包

将镜像打包成保存为一个文件

```
docker save $(docker images | grep -v REPOSITORY | awk 'BEGIN{OFS=":";ORS=" "}{print $1,$2}') -o  dockerImages.tar
```

将镜像一个一个分别保存到当前目录

```
#!/bin/bash
img_list=`docker images | grep -v REPOSITORY | awk 'BEGIN{OFS=":";ORS=" \n"}{print $1,$2}'`
for img in $img_list
do
   img2="${img//\//_}_`date +%Y%m%d`.tar"
   img2=${img2/:/_}
   echo ${img2}
   docker save $img -o $img2
done
```





### 3.4 镜像批量还原

单独导入一个镜像文件(任选一种)

```
docker load --input haha.tar
docker load < haha.tar
docker load -i haha.tar
```

导入档期路径下的所有tar文件

```
#!/bin/bash
files=$(ls $PWD/*.tar)
for filename in $files
do
 echo $filename 
 docker load -i $filename
done
```

