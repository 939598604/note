# docker私服registry的安装使用

## 一、安装docker私有仓库

### 1.1 拉取私有仓库的镜像

```
docker pull registry
```

### 1.2 启动私有仓库

```
docker run -di --name=registry -p 5000:5000 registry
```

### 1.3 浏览器访问

http://192.168.25.129:5000/v2/_catalog。若浏览器显示{"repositories":[]}则表示安装成功

### 4.让docker信任私有仓库地址

1)编辑docker配置文件

```
vi /etc/docker/daemon.json
```

2)新增私有仓库地址

```
{
"registry-mirrors":["https://docker.mirrors.ustc.edu.cn"],
"insecure-registries":["192.168.25.129:5000"]
}
```

### 5.重启docker

```
systemctl restart docker
```

 

## 二、上传镜像到私有仓库

### 2.1标记此镜像为私有仓库的镜像

因为registry的镜像规则是 ip:5000/目录/镜像:版本号，且原来的jdk1.8镜像不符合规则，所以需要重新打标签

```
docker tag jdk1.8 192.168.25.129:5000/lib/jdk1.8
```

### 2.2 启动私有仓库

```
docker start registry
```

### 2.3 上传标记的镜像

```
docker push 192.168.25.129:5000/lib/jdk1.8
```

 

## 三、上传到私有仓库

使用maven插件将tenpower-config工程发布到docker，并上传到私有仓库

### 3.1 允许远程访问

修改宿主机docker配置，使其可被远程访问

```
vi /lib/systemd/system/docker.service
```

修改ExecStart=/usr/bin/dockerd这一行

```
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock
```

### 3.2 刷新配置

```
systemctl daemon-reload
systemctl restart docker
docker start registry
```