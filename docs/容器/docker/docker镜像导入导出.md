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

