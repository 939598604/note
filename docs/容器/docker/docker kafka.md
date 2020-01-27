# docker kafka

## 一.docker-compose的安装

### 1.1 pip安装

```
wget https://bootstrap.pypa.io/get-pip.py   
python get-pip.py  
```

### 1.2 安装docker-compose

```
pip install --upgrade pip
pip install --upgrade docker-compose==1.24.1
```

##  二.kafka镜像的构建
### 2.1 docker-compose.yml

创建一个kafka的文件目录，后续所有操作都在该目录，然后创建docker-compose.yml文件

```
mkdir /kafka
cd /kafka
vim docker-compose.yml
```

docker-compose.yml文件内容如下

```
version: '2'
services:
  zookeeper:
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
  kafka:
    image: wurstmeister/kafka:2.11-0.11.0.3
    ports:
      - "9092"
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://:9092
      KAFKA_LISTENERS: PLAINTEXT://:9092
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

### 2.2 拉取镜像

```
docker search kafka
docker pull wurstmeister/kafka
```

### 2.3 构建

切换到刚才创建docker-compsoe.yml文件的地，执行命令

```
cd /kafka 
docker-compose up -d
```

查看创建容器，可以看到创建了一个kafka-zookeeper容器

```
docker ps 
```

查看kafka版本号

```
docker exec kafka_kafka_1 find / -name \*kafka_\* | head -1 | grep -o '\kafka[^\n]*'
```

查看zookeeper版本号

```
docker exec （zookeeper容器名称或者容器id） pwd
```

### 2.4 扩展broker

```
docker-compose scale kafka=4
```

查看创建的多个broker，可以看到已经创建了4个

```
docker ps
```

创建topic

```
docker exec kafka_kafka_1 \
kafka-topics.sh \
--create --topic topic001 \
--partitions 4 \
--zookeeper zookeeper:2181 \
--replication-factor 2
```

查看创建的topic(每个broker的容器id或者名称都可以)kafka_kafka_1即第一个容器名称

```
docker exec kafka_kafka_1 \
kafka-topics.sh --list \
--zookeeper zookeeper:2181 \
topic001
```

查看刚刚创建的topic的情况，borker和副本情况一目了然，代码一行一行输入

```
docker exec kafka_kafka_1 \     
kafka-topics.sh \
--describe \
--topic topic001 \
--zookeeper zookeeper:2181
```

创建消费者开始生产消息消费消息演示，此处还没有生产消息，所有没有任何显示

```
docker exec kafka_kafka_1 \
kafka-console-consumer.sh \
--topic topic001 \
--bootstrap-server kafka_kafka_1:9092,kafka_kafka_2:9092,kafka_kafka_3:9092,kafka_kafka_4:9092
```

生产者生产消息（新开一个窗口）

```
docker exec -it kafka_kafka_1 \
kafka-console-producer.sh \
--topic topic001 \
--broker-list kafka_kafka_1:9092,kafka_kafka_2:9092，kafka_kafka_3:9092,kafka_kafka_4:9092
```

在生产者窗口运行15的命令之后，随意输入几个字符串，即可在消费者端看到显示，到此，docker安装kafka就结束了。

docker-compose 开启关闭命令

```
#开启
docker-compose up -d
#关闭
docker-compose stop
```