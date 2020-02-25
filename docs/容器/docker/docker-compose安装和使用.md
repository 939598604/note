# docker-compose安装和使用

## 一.二进制文件安装

### 1.1 安装

可以通过修改 URL 中的版本，自定义您需要的版本。

- Github源

```
curl -L https://github.com/docker/compose/releases/download/1.22.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

- Daocloud镜像

```
curl -L https://get.daocloud.io/docker/compose/releases/download/1.22.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

### 1.2 卸载

```
rm /usr/local/bin/docker-compose
```

##二.pip方式安装

### 1.1 安装pip

```
yum -y install epel-release
yum -y install python-pip
```

### 1.2 确认版本

```
pip --version
```

### 1.3 更新pip

```
pip install --upgrade pip
```

### 1.4 安装docker-compose

```
pip install docker-compose 
```

### 1.5 查看版本

```
docker-compose version
```

## 三.基础命令

需要在 docker-compose.yml 所在文件夹中执行命令

使用 docker-compose 部署项目的简单步骤

### 3.1 停止容器：

停止现有 docker-compose 中的容器

```
docker-compose down
```

### 3.2 重新拉取镜像：

```
docker-compose pull
```

### 3.3 后台启动容器

后台启动 docker-compose 中的容器

```
docker-compose up -d
```

## 四. docker-compose.yml 部署应用

###  4.1 新建 docker-compose.yml 文件

通过以下配置，在运行后可以创建两个站点(只为演示)

```
version: "3"
services:
  web1:
    image: registry.cn-hangzhou.aliyuncs.com/yimo_public/docker-nginx-test:latest
    ports:
      - "4466:80"
  web2:
    image: registry.cn-hangzhou.aliyuncs.com/yimo_public/docker-nginx-test:latest
    ports:
      - "4477:80"
```

此处只是简单演示写法，说明 docker-compose 的方便

### 4.2 构建完成，后台运行镜像

```
docker-compose up -d
```

运行后就可以使用 ip+port 访问这两个站点了

### 4.3 镜像更新重新部署

```
docker-compose down
docker-compose pull
docker-compose up -d
```