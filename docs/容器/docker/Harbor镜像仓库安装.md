# Harbor镜像仓库安装

## 一.Harbor的安装

### 1.1 下载离线安装软件

```
wget http://harbor.orientsoft.cn/harbor-v1.3.0-rc4/harbor-offline-installer-v1.3.0-rc4.tgz
```

### 1.2 解压文件

解压后的文件夹是harbor

```
tar -zxf harbor-offline-installer-v1.3.0-rc4.tgz
```

### 1.3 修改配置文件

vim harbor.cfg 

```
#主机地址，不可以设置为127或者localhost，要设置为ip地址
hostname = 192.168.197.224
# 访问协议，默认是http，也可以设置https，如果设置https，则nginx ssl需要设置on
ui_url_protocol = http
# mysql数据库root用户默认密码root123，实际使用时修改下
db_password = root123
#这里是web登录页面的密码，可以更改
harbor_admin_password = admin
```

### 1.4 执行准备脚本

```
[root@~localhost ]# ./prepare
```

###  1.5 启动harbor

修改完配置文件就可以执行该目录下的install.sh文件即可，程序会自动启动相关镜像，因为harbor是用你镜像进行安装的。 

```
[root@~localhost ]# ./install.sh 
```

### 1.6 登陆harbor UI

直接访问：http://192.168.197.224

账号密码：admin  admin

## 二.docker机器使用镜像仓库

### 2.1 添加镜像地址

**方式一:**

(1) 修改文件/lib/systemd/system/docker.service

在里面修改ExecStart那一行，增加 --insecure-registry=192.168.197.224，然后重启docker 

```
ExecStart=/usr/bin/dockerd-current \
          --add-runtime docker-runc=/usr/libexec/docker/docker-runc-current \
          --default-runtime=docker-runc \
          --exec-opt native.cgroupdriver=systemd \
          --userland-proxy-path=/usr/libexec/docker/docker-proxy-current \
          --init-path=/usr/libexec/docker/docker-init-current \
          --seccomp-profile=/etc/docker/seccomp.json \
          --insecure-registry=192.168.197.224 \
          $OPTIONS \
          $DOCKER_STORAGE_OPTIONS \
          $DOCKER_NETWORK_OPTIONS \
          $ADD_REGISTRY \
          $BLOCK_REGISTRY \
          $INSECURE_REGISTRY \
          $REGISTRIES
```

**方式二：**

（1）直接修改 /etc/docker/daemon.json

```
{
  "registry-mirrors": ["https://xh2kgvix.mirror.aliyuncs.com"],
  "insecure-registries":["192.168.197.224"]
}
```

###  2.2 重启docker

```
systemctl daemon-reload
systemctl restart docker
```

## 三.客户端上传镜像到仓库

### 3.1 打标签

```
docker tag rancher/rancher-agent:v2.3.5 192.168.197.224/gzstrong/gzstrong-rancher:1.0.0
```

### 3.2 登录Harbor

```
docker login 192.168.197.224
username: admin
password: admin
```

### 3.3 推送镜像到Harbor

```
docker push 192.168.197.224/gzstrong/gzstrong-rancher:1.0.0
```

