# docker基本命令

查看当前正在运行的容器

```
docker ps 
```

查看所有容器的状态

```
docker ps -a 
```

启动/停止某个容器

```
docker start/stop id/name 
```

进入某个容器(使用exit退出后容器也跟着停止运行)

```
docker attach id 
```

 启动一个伪终端以交互式的方式进入某个容器（使用exit退出后容器不停止运行）

```
docker exec -ti id
```

查看本地镜像

```
docker images 
```

删除某个容器

```
docker rm id/name 
```

 删除某个镜像

```
docker rmi id/name
```

复制ubuntu容器并且重命名为test且运行，然后以伪终端交互式方式进入容器，运行bash

```
docker run --name test -ti ubuntu /bin/bash  
```

 通过当前目录下的Dockerfile创建一个名为soar/centos:7.1的镜像

```
docker build -t soar/centos:7.1 . 
```

 以镜像soar/centos:7.1创建名为test的容器，并以后台模式运行，并做端口映射到宿主机2222端口，P参数重启容器宿主机端口会发生改变

```
docker run -d -p 2222:22 --name test soar/centos:7.1 
```

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

