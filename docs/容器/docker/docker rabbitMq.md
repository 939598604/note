# docker RabbitMQ

## 1.RabbitMQ镜像

```
docker pull rabbitmq  (镜像未配有控制台)
docker pull rabbitmq:management (镜像配有控制台)
```

## 2.拉取镜像

```
docker pull rabbitmq:management
```

## 3.查看镜像列表

```
[root@host~]# docker images |grep rabbitmq
rabbitmq          management              9b188f5fb1e6        3 weeks ago         98.2MB
```

## 4.运行容器

我们可以将主机目录挂载到容器内的指定目录，这样就可以在主机目录下存放我们的配置文件，让容器加载；

新建目录

```
mkdir -p /docker/rabbitMq/data
```

运行

```
docker run -d --name rabbitmq3.7.7 -p 5672:5672 -p 15672:15672 -v /docker/rabbitMq/data:/var/lib/rabbitmq --hostname myRabbit -e RABBITMQ_DEFAULT_VHOST=my_vhost  -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin rabbitmq:management
```

/var/lib/rabbitmq挂载到/docker/rabbitMq/data

-d 后台运行容器；

--name 指定容器名；

-p 指定服务运行的端口（5672：应用访问端口；15672：控制台Web端口号）；

-v 映射目录或文件；

--hostname  主机名（RabbitMQ的一个重要注意事项是它根据所谓的 “节点名称” 存储数据，默认为主机名）；

-e 指定环境变量；（RABBITMQ_DEFAULT_VHOST：默认虚拟机名；RABBITMQ_DEFAULT_USER：默认的用户名；RABBITMQ_DEFAULT_PASS：默认用户名的密码）

5、使用命令：docker ps 查看正在运行容器

## 5.进入容器的rabbitMq

docker ps 查看正在运行容器

```
docker exec -it myrabbitMq bash
```

## 6.放开防火墙

```
firewall-cmd --permanent --add-port=5672/tcp
firewall-cmd --permanent --add-port=15672/tcp
firewall-cmd --reload
```

## 7.外网的rabbitMq客户端访问

可以使用浏览器打开web管理端：http://Server-IP:15672

## 8.docker端口映射或启动容器时报错

Error response from daemon: driver failed programming external connectivity on endpoint quirky_allen

 **原因:**

docker服务启动时定义的自定义链DOCKER由于某种原因被清掉
重启docker服务及可重新生成自定义链DOCKER  

 **解决:**

重启docker服务后再启动容器

```
systemctl restart docker
docker start mysql01
```

