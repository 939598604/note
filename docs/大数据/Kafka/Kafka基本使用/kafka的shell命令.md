#  kafka的shell命令

1.启动 KafkaServer 

执行以下命令启动 KafkaServer: 

```
 bin/kafka-server-start.sh -daemon config/server.properties
```

## 主题

1.创建主题

```
kafka-topics.sh --create --zookeeper server-1:2181, server-2:2181,server-3:2181 --replication-factor 2 --partitions 3 --topic test
```

2.查看所有主题 

执行以下命令 

```
kafka-topics.sh --list --zookeeper localhost:2181
```

3.删除主题

```
kafka-topics --delete --zookeeper localhost:2181 --topic test
```

4.查看某个特定题信息 

```
bin/kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic test
```

5.添加分区

```
kafka-topics.sh --alter --zookeeper localhost:2181 --topic config-test --partitions 5
```

## 生产者

1.启动一个生产者

```
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
```



## 消费者

1.启动一个消费者

```
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
```

