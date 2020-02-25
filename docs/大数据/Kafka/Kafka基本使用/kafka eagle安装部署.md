# kafka eagle安装部署

## 1.环境准备

JDK环境

## 2.安装kafka-eagle

### 2.1.下载安装包

kafka-eagle-bin-1.4.2.tar.gz 安装包下载，官网 或者在百度网盘中下载。[点击下载](https://pan.baidu.com/s/1HqcxPKF13XhCCPPzDz86tg)

```shell
tar -zvxf /soft/kafka-eagle-bin-1.4.2.tar.gz
cd /soft/kafka-eagle-bin-1.4.2
tar -zxvf kafka-eagle-web-1.4.2-bin.tar.gz
```

### 2.2 环境配置

vim /etc/profile

```shell
export KE_HOME=/soft/kafka-eagle-bin-1.4.2/kafka-eagle-web-1.4.2-bin
export PATH=PATH:KE_HOME/bin
```

source /etc/profile

### 2.3配置修改

vim system-config.properties

```properties
#如果只有一个集群的话，就写一个cluster1就行了
kafka.eagle.zk.cluster.alias=cluster1   
#这里填上刚才上准备工作中的zookeeper.connect地址
cluster1.zk.list=localhost:2181

# zk limit -- Zookeeper cluster allows the number of clients to connect to
kafka.zk.limit.size=25

# kafka eagel webui port -- WebConsole port access address
kafka.eagle.webui.port=8048     ###web界面地址端口

# kafka offset storage -- Offset stored in a Kafka cluster, if stored in the zookeeper, you can not use this option
kafka.eagle.offset.storage=kafka

# delete kafka topic token -- Set to delete the topic token, so that administrators can have the right to delete
kafka.eagle.topic.token=keadmin

# kafka sasl authenticate, current support SASL_PLAINTEXT
#如果kafka开启了sasl认证，需要在这个地方配置sasl认证文件
kafka.eagle.sasl.enable=false
kafka.eagle.sasl.protocol=SASL_PLAINTEXT
kafka.eagle.sasl.mechanism=PLAIN
kafka.eagle.sasl.client=/data/kafka-eagle/conf/kafka_client_jaas.conf

#下面两项是配置数据库的，默认使用sqlite，如果量大，建议使用mysql，这里我使用的是sqlit
#如果想使用mysql建议在文末查看官方文档
# Default use sqlite to store data
kafka.eagle.driver=org.sqlite.JDBC
# It is important to note that the '/hadoop/kafka-eagle/db' path must exist.
kafka.eagle.url=jdbc:sqlite:/soft/kafka-eagle-bin-1.4.2/kafka-eagle-web-1.4.2-bin/db/ke.db   #这个地址，按照安装目录进行配置
kafka.eagle.username=root
kafka.eagle.password=smartloli

# <Optional> set mysql address
#kafka.eagle.driver=com.mysql.jdbc.Driver
#kafka.eagle.url=jdbc:mysql://127.0.0.1:3306/ke?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull
#kafka.eagle.username=root
#kafka.eagle.password=smartloli
```

主要修改zookeeper的地址和数据库的url

```properties
#如果只有一个集群的话，就写一个cluster1就行了
kafka.eagle.zk.cluster.alias=cluster1   
#这里填上刚才上准备工作中的zookeeper.connect地址
cluster1.zk.list=localhost:2181
kafka.eagle.url=jdbc:sqlite:/soft/kafka-eagle-bin-1.4.2/kafka-eagle-web-1.4.2-bin/db/ke.db   #这个地址，按照安装目录进行配置
```

> ```
> 如果开启了sasl认证，需要自己去修改kafka-eagle目录下的conf/kafka_client_jaas.conf
> 此处不多说
> ```

### 2.4.启动kafka-eagle

进入到kafka-eagle的安装目录

```
chmod +x /bin/ke.sh
ke.sh start
```

控制台有账号密码提示：

```
Kafka Eagle Service has started success.
Welcome, Now you can visit 'http://192.168.197.18:8048/ke'
Account:admin ,Password:123456
```

### 2.5 访问浏览器

 http://192.168.197.18:8048/ke
 默认用户名:admin     默认密码:123456

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200107161811.png)