# docker mysql

## 1.搜索镜像
```
[root@host~]# docker search mysql
NAME         DESCRIPTION                                    STARS      OFFICIAL AUTOMATED
mysql        MySQL is a widely used, open-source relation…   9063        [OK]        
mariadb      MariaDB is a community-developed fork of MyS…   3202        [OK]             
mysql/mysql-server   Optimized MySQL Server Docker images. Create…   673 [OK]
percona      Percona Server is a fork of the MySQL relati…   467         [OK]
```

选择第一个START数量最多的那个9063的mysql官方镜像

## 2.拉取镜像

```
docker pull mysql:5.6
```

## 3.查看镜像列表

```
docker images |grep mysql
```

## 4.运行容器

### 4.1 没有配置文件`cnf`的方式启动

```
docker run -p 3306:3306 --name mymysql -v ~/mysql/conf:/etc/mysql/conf.d -v ~/mysql/logs:/logs -v ~/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.6
```

```
docker run \
--name MYSQL5.6 \
-e MYSQL_ROOT_PASSWORD=123456 \
-p 3306:3306 \
-d mysql:5.6 \
--character-set-server=utf8 \
--collation-server=utf8_unicode_ci
```

参数说明（每一行后面的\是Linux命令换行）：

-e设置容器相关参数，这里是设置root密码为123456（其他设置，可以参照官方文档：mysql-docker）

-p做端口映射，将主机的3306端口映射到容器的3306端口

-d后台启动，参数可以是镜像的IMAGE_ID，也可以是name:TAG

最后两行是对这个容器的字符编码，和排序规则的设置

### 4.2 使用挂载方式

我们可以将主机目录挂载到容器内的指定目录，这样就可以在主机目录下存放我们的配置文件，让容器加载；

#### 4.2.1 my.cnf配置文件挂载

新建配置目录

```
mkdir -p /docker/mysql/conf
```

首先，新建一个需要被挂载的主机目录，在/docker/mysql/conf下新建my.cnf文件，内容是

```
[client]
default-character-set=utf8
[mysqld]
character-set-server=utf8
default-storage-engine=INNODB
collation-server=utf8_general_ci
[mysql]
default-character-set=utf8
```

#### 4.2.2 数据配置挂载目录

```
mkdir -p /docker/mysql/data
```

#### 4.2.3  日志配置挂载目录

```
mkdir -p /docker/mysql/logs
```

#### 4.2.4  启动mysql

```
docker run --name mysqld -v /docker/mysql/conf:/etc/mysql/conf.d  -v /docker/mysql/data:/var/lib/mysql -v /docker/mysql/logs:/logs -e MYSQL_ROOT_PASSWORD=root -p 3306:3306 -d mysql:5.6
```

参数说明：

`-v  `就是把主机目录挂载到容器目录

`/docker/mysql/conf`配置目录挂载到容器内部的`/etc/mysql/conf.d`

`/docker/mysql/data`数据目录挂载到容器内部的`/var/lib/mysql`

`/docker/mysql/logs`日志目录挂载到容器内部的`/logs`

MYSQL_ROOT_PASSWORD=root  设置root账号密码为root

## 5.连接容器

```
docker exec -it mymysql bash
```

## 6.设置远程登陆

Mysql 8 以下版本

```
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root' WITH GRANT OPTION;
```

Mysql 8 以上版本

```
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'root';
```

刷新

```
flush privileges;
```

## 7.宿主机器远程连接mysql

```
[root@host~]# mysql --host=127.0.0.1 --port=3309 --user=root --password=root
[root@host~]# mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
3 rows in set (0.40 sec)

mysql> 
```

## 8.外网的mysql客户端访问

**docker 暴露端口无法访问**

解决方法是打开防火墙：利用防火墙进行转发

启动firewalld

```
systemctl start firewalld.service
```

把firewalld加入到系统服务

```
systemctl enable firewalld.service
```

查询端口是否开放

```
firewall-cmd --query-port=8080/tcp
```

开放80端口

```
firewall-cmd --permanent --add-port=80/tcp
```

移除端口

```
firewall-cmd --permanent --remove-port=8080/tcp
```

重启防火墙(修改配置后要重启防火墙)

```
firewall-cmd --reload
```

## 9.docker端口映射或启动容器时报错

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

