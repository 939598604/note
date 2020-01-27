# Open-falcon 服务端部署

## 一.安装mysql

### 1.1 搜索镜像

```
[root@host~]## docker search mysql
NAME         DESCRIPTION                                    STARS      OFFICIAL AUTOMATED
mysql        MySQL is a widely used, open-source relation…   9063        [OK]        
mariadb      MariaDB is a community-developed fork of MyS…   3202        [OK]             
mysql/mysql-server   Optimized MySQL Server Docker images. Create…   673 [OK]
percona      Percona Server is a fork of the MySQL relati…   467         [OK]
```

选择第一个START数量最多的那个9063的mysql官方镜像

### 1.2 拉取镜像

```
docker pull mysql:5.6
```

### 1.3 查看镜像列表

```
docker images |grep mysql
```

### 1.4 运行容器

#### 1.4.1  my.cnf配置文件挂载

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

#### 1.4.2 数据配置挂载目录

```
mkdir -p /docker/mysql/data
```

#### 1.4.3 日志配置挂载目录

```
mkdir -p /docker/mysql/logs
```

#### 1.4.4  启动mysql

```
docker run --name mysqld -v /docker/mysql/conf:/etc/mysql/conf.d  -v /docker/mysql/data:/var/lib/mysql -v /docker/mysql/logs:/logs -e MYSQL_ROOT_PASSWORD=root -p 3306:3306 -d mysql:5.6
```

参数说明：

`-v  `就是把主机目录挂载到容器目录

`/docker/mysql/conf`配置目录挂载到容器内部的`/etc/mysql/conf.d`

`/docker/mysql/data`数据目录挂载到容器内部的`/var/lib/mysql`

`/docker/mysql/logs`日志目录挂载到容器内部的`/logs`

MYSQL_ROOT_PASSWORD=root  设置root账号密码为root

### 1.5 连接容器

```
docker exec -it mymysql bash
```

### 1.6 设置远程登陆

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

### 1.7 宿主机器远程连接mysql

```
[root@host~]## mysql --host=127.0.0.1 --port=3309 --user=root --password=root
[root@host~]## mysql> show databases;
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

### 1.8 外网的mysql客户端访问

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
firewall-cmd --permanent --add-port=8080/tcp
```

移除端口

```
firewall-cmd --permanent --remove-port=8080/tcp
```

重启防火墙(修改配置后要重启防火墙)

```
firewall-cmd --reload
```

## 二.安装redis

### 2.1 搜索镜像

```
[root@host~]## docker search redis
NAME          DESCRIPTION                                    STARS     OFFICIAL AUTOMATED
redis         Redis is an open source key-value store that…   7752      [OK]             
bitnami/redis Bitnami Redis Docker Image                      137       [OK]
sameersbn/redis                                               79        [OK]
```

选择第一个START数量最多的那个7752的redis官方镜像

### 2.2 拉取镜像

```
docker pull docker.io/redis
```

### 2.3 查看镜像列表

```
[root@host~]## docker images |grep redis
redis               latest              9b188f5fb1e6        3 weeks ago         98.2MB
```

### 2.4  redis.conf配置文件挂载

新建配置目录

```
mkdir -p /docker/redis/conf
```

首先，新建一个需要被挂载的主机目录，在/docker/redis/conf下新建redis.conf文件，

**我的配置是直接注释掉bind**

​    **protected-mode yes**

内容是

```ini
daemonize no
pidfile /var/run/redis.pid
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 0
loglevel notice
logfile ""
databases 16
 
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir ./
slave-serve-stale-data yes
slave-read-only yes
repl-disable-tcp-nodelay no
slave-priority 100
appendonly no
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
lua-time-limit 5000 
slowlog-log-slower-than 10000
slowlog-max-len 128
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-entries 512
list-max-ziplist-value 64
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64

hll-sparse-max-bytes 3000
 
activerehashing yes
 
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60

hz 10
aof-rewrite-incremental-fsync yes
```

##### 2.4.1 数据配置挂载目录

```
mkdir -p /docker/redis/data
```

##### 2.4.2 启动redis

```
docker run -p 6379:6379 --name myredis -v /docker/redis/conf/redis.conf:/etc/redis/redis.conf  -v /docker/redis/data:/data -d redis redis-server /etc/redis/redis.conf --appendonly yes
```

参数说明：

`-v  `就是把主机目录挂载到容器目录

`/docker/redis/conf`把宿主机配置好的redis.conf放到容器内的这个位置`/etc/redis/conf.d`中

`/docker/redis/data` 把redis持久化的数据在宿主机内显示，做数据备份 `/var/lib/redis`

-p 6379:6379:把容器内的6379端口映射到宿主机6379端口
redis-server /etc/redis/redis.conf：这个是关键配置，让redis不是无配置启动，而是按照这个redis.conf的配置启动
-appendonly yes：redis启动后数据持久化

### 2.5 进入容器的redis

```
docker exec -ti myredis redis-cli -h localhost -p 6379
或者
docker exec -it myredis bash
或者
docker exec -it redis_s redis-cli -h localhost -p 6379 -a password //密码，使用-a参数
```

### 2.6 外网的redis客户端访问

使用redis desktop Manager连接工具测试连接

### 2.7 给redis设置密码

需要永久配置密码的话就去redis.conf的配置文件中找到requirepass这个参数，如下配置：

修改docker/redis/conf/redis.conf配置文件

```
requirepass 123   #指定密码123
```

保存后重启redis就可以了

```
docker run -p 6379:6379 --name myredis -v /docker/redis/conf/redis.conf:/etc/redis/redis.conf  -v /docker/redis/data:/data -d redis redis-server /etc/redis/redis.conf --appendonly yes
```

## 三.安装后端

### 3.1 拉取镜像

```
docker pull openfalcon/falcon-plus:v0.3
```

### 3.2 启动falcon-plus

```
docker run -itd --name falcon-plus \
     --link=mysql01:db.falcon \
     --link=myredis:redis.falcon \
     -p 8433:8433 \
     -p 1988:1988 \
     -p 8080:8080 \
     -p 6030:6030 \
     -e MYSQL_PORT=root:123456@tcp\(db.falcon:3306\) \
     -e REDIS_PORT=redis.falcon:6379  \
     -v /docker/open-falcon/data:/open-falcon/data \
     -v /docker/open-falcon/logs:/open-falcon/logs \
     openfalcon/falcon-plus:v0.3
```

### 3.3 启动后端其他模块

```
docker exec falcon-plus sh ctrl.sh start \
            graph hbs judge transfer nodata aggregator agent gateway api alarm
```

## 四 安装前端

### 4.1 拉取镜像

```
docker pull openfalcon/falcon-dashboard:v0.2.1
```

### 4.2 启动前端

```
docker run -itd --name falcon-dashboard \
    -p 8081:8081 \
    --link=mysql01:db.falcon \
    --link=falcon-plus:api.falcon \
    -e API_ADDR=http://api.falcon:8080/api/v1 \
    -e PORTAL_DB_HOST=db.falcon \
    -e PORTAL_DB_PORT=3306 \
    -e PORTAL_DB_USER=root \
    -e PORTAL_DB_PASS=123456 \
    -e PORTAL_DB_NAME=falcon_portal \
    -e ALARM_DB_HOST=db.falcon \
    -e ALARM_DB_PORT=3306 \
    -e ALARM_DB_USER=root \
    -e ALARM_DB_PASS=123456 \
    -e ALARM_DB_NAME=alarms \
    -w /open-falcon/dashboard openfalcon/falcon-dashboard:v0.2.1  \
   './control startfg'
```



## 五.前端界面

打开页面，在windows系统浏览器中，可以输入http://宿主机器ip:8081

在windows系统浏览器中，可以输入http://宿主机器ip:8081

**在windows系统浏览器中，可以输入centos主机ip:8081进入**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190809133851355.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0MTY5MDQz,size_16,color_FFFFFF,t_70)

第一次登录需要注册

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190809133912205.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0MTY5MDQz,size_16,color_FFFFFF,t_70)

**如果打不开页面可能是防火墙没有放开，Centos7默认的是firewall作为防火墙**

