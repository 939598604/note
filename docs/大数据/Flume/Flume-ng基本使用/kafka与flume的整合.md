# kafka与flume的整合

环境准备:

安装好flume，kafka的机器

## 1.kafka source

### 1.1flume的kafka source配置文件

kafka_source.conf

```properties
a1.sources = r1
a1.channels = c1
a1.sinks = k1

a1.sources.r1.type = org.apache.flume.source.kafka.KafkaSource
a1.sources.r1.channels = channel1
a1.sources.r1.batchSize = 5000
a1.sources.r1.batchDurationMillis = 2000
a1.sources.r1.kafka.bootstrap.servers = kafka.gzstrong.com:9092
a1.sources.r1.kafka.topics = test
a1.sources.r1.kafka.consumer.group.id = custom.gid

# channel
a1.channels.c1.type = memory
a1.channels.c1.capacity = 10000
a1.channels.c1.transactionCapacity = 5000

# bind
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1

a1.sinks.k1.type = logger
```

### 1.2.启动flume

```shell
/soft/flume-1.9.0/bin/flume-ng agent -c /soft/flume-1.9.0/conf -f /soft/flume-1.9.0/conf/kafka_source.conf -n a1 -Dflume.root.logger=INFO,console
```

### 1.3 启动kafka生产者数据源

```shell
bin/kafka-console-producer.sh --broker-list kafka.gzstrong.com:9092 --topic testbin/kafka-console-producer.sh --broker-list kafka.gzstrong.com:9092 --topic test
```

在生产者输入消息，然后观察flume的控制台有接收到消息



## 2.kafka sink

### 2.1flume的kafka sink 配置文件

kafka_sink.conf

```properties
a1.sources=r1
a1.channels=c1
a1.sinks=k1

a1.sources.r1.type = netcat
a1.sources.r1.bind = localhost
a1.sources.r1.port = 44444

# sink
a1.sinks.k1.type = org.apache.flume.sink.kafka.KafkaSink
a1.sinks.k1.kafka.bootstrap.servers = kafka.gzstrong.com:9092
a1.sinks.k1.kafka.topic = test
a1.sinks.k1.kafka.flumeBatchSize = 20 
a1.sinks.k1.kafka.producer.acks = 1
a1.sinks.k1.kafka.producer.linger.ms = 1

# channel
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# bind
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

### 2.2.启动flume

```shell
/soft/flume-1.9.0/bin/flume-ng agent -c /soft/flume-1.9.0/conf -f /soft/flume-1.9.0/conf/kafka_sink.conf -n a1 -Dflume.root.logger=INFO,console
```

### 2.3 启动一个telnet数据源

```shell
[root@reg ~]# telnet localhost 44444
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
dfad
OK
faddfadfadfadf
OK
adfadfadsf
OK
nimabi
OK
```

###  2.4 启动kafka消费者

```shell
bin/kafka-console-consumer.sh --bootstrap-server kafka.gzstrong.com:9092 --topic test --from-beginning
```

可以查看到消费者控制台有输出消息