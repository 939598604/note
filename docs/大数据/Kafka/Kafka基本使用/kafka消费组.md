# kafka消费组

如果 Consumer1、Consumer2 和 Consumer3 都属于 group.id 为 1 的消费组。那么 Consumer1 就会消费 p0，Consumer2 就会消费 p1，Consumer3 就会消费 p2。

可以先测试一下。创建三个 Consumer。要注意的是这里不能使用指定分区的方式：

```java
TopicPartition partition=new TopicPartition(topic,1);
kafkaConsumer.assign(Arrays.asList(partition)));
```

而且它们都是同一个消费组：

```
properties.put(ConsumerConfig.GROUP_ID_CONFIG,"KafkaConsumerDemo");
```

同时启动三个 Consumer，和 Producer：

可以看到三个 Consumer 分别消费三个 Partition，很均匀。对同一个 Group 来说，其中的 Consumer 可以消费指定分区也可以消费自动分配的分区（这里是 Consumer 数量和 partition 数量一致，均匀分配）。那么如果 Consumer 数量大于 partition 数量呢，如果 Consumer 数量小于 partition 数量呢，测试也很简单，这里就不多做测试了。

**要注意的是如果 Consumer 数量比 partition 数量多，会有的 Consumer 闲置无法消费，这样是一个浪费。如果 Consumer 数量小于 partition 数量会有一个 Consumer 消费多个 partition。Kafka 在 partition 上是不允许并发的。Consuemr 数量建议最好是 partition 的整数倍。 还有一点，如果 Consumer 从多个 partiton 上读取数据，是不保证顺序性的，Kafka 只保证一个 partition 的顺序性，跨 partition 是不保证顺序性的。增减 Consumer、broker、partition 会导致 Rebalance。**

## Kafka 分区分配策略

在 Kafka 中，同一个 Group 中的消费者对于一个 Topic 中的多个 partition 存在一定的分区分配策略。

在 Kafka 中，存在两种分区分配策略，一种是 Range（默认）、另一种是 RoundRobin（轮询）。通过partition.assignment.strategy 这个参数来设置。Range strategy（范围分区）

Range 策略是对每个主题而言的，首先对同一个主题里面的分区按照序号进行排序，并对消费者按照字母顺序进行排序。假设我们有10个分区，3个消费者，排完序的分区将会是0,1,2,3,4,5,6,7,8,9；消费者线程排完序将会是C1-0, C2-0, C3-0。然后将 partitions 的个数除于消费者线程的总数来决定每个消费者线程消费几个分区。如果除不尽，那么前面几个消费者线程将会多消费一个分区。

假如在 Topic1 中有 10 个分区，3 个消费者线程，10/3 = 3，而且除不尽，那么消费者线程 C1-0 将会多消费一个分区，所以最后分区分配的结果是这样的：

C1-0 将消费 0,1,2,3 分区
C2-0 将消费 4,5,6 分区
C3-0 将消费 7,8,9 分区

假如在 Topic1 中有 11 个分区，那么最后分区分配的结果看起来是这样的：

C1-0 将消费 0,1,2,3 分区
C2-0 将消费 4, 5, 6, 7 分区
C3-0 将消费 8,9,10 分区

假如有两个 Topic：Topic1 和 Topic2，都有 10 个分区，那么最后分区分配的结果看起来是这样的：

C1-0 将消费 Topic1 的 0,1,2,3 分区和 Topic1 的 0,1,2,3 分区
C2-0 将消费 Topic1 的 4,5,6 分区和Topic2 的 4,5,6 分区 
C3-0 将消费 Topic1 的 7,8,9 分区和Topic2 的 7,8,9 分区

其实这样就会有一个问题，C1-0 就会多消费两个分区，这就是一个很明显的弊端。

RoundRobin strategy(轮询分区)

轮询分区策略是把所有 partition 和所有 Consumer 线程都列出来，然后按照 hashcode 进行排序。最后通过轮询算法分配partition 给消费线程。如果所有 Consumer 实例的订阅是相同的，那么 partition 会均匀分布。

假如按照 hashCode 排序完的 Topic / partitions组依次为T1一5, T1一3, T1-0, T1-8, T1-2, T1-1, T1-4，T1-7,T1-6,T1-9，消费者线程排序为 C1-0，C1-1，C2-0，C2-1，最后的分区分配的结果为：

C1-0 将消费 T1-5, T1-2, T1-6分区
C1-1 将消费 T1-3, T1-1, T1-9分区
C2-0 将消费 T1-0, T1-4分区
C2-1 将消费 T1-8, T1-7分区

使用轮询分区策略必须满足两个条件
1.每个主题的消费者实例具有相同数量的流
2.每个消费者订阅的主题必须是相同的

什么时候会触发这个策略呢?

当出现以下几种情况时，Kafka 会进行一次分区分配操作，也就是 Kafka Consumer 的 Rebalance

1.同一个 Consumer group内新增了消费者
2.消费者离开当前所属的 Consumer group，比如主动停机或者宕机
3.Topic 新增了分区（也就是分区数量发生了变化）

Kafka Consuemr 的 Rebalance 机制规定了一个 Consumer group 下的所有 Consumer 如何达成一致来分配订阅 Topic 的每个分区。而具体如何执行分区策略，就是前面提到过的两种内置的分区策略。而 Kafka 对于分配策略这块，提供了可插拔的实现方式，也就是说，除了这两种之外，我们还可以创建自己的分配机制。

谁来执行 Rebalance 以及管理 Consumer 的 group 呢?

Consumer group 如何确定自己的 coordinator 是谁呢，消费者向 Kafka 集群中的任意一个 broker 发送一个 GroupCoord inatorRequest 请求，服务端会返回一个负载最小的 broker 节点的 id，并将该 broker 设置为 coordinator。

JoinGroup 的过程

在 Rebalance 之前，需要保证 coordinator 是已经确定好了的，整个 Rebalance 的过程分为两个步骤，Join和Syncjoin：表示加入到 Consumer group中，在这一步中，所有的成员都会向 coordinator 发送 joinGroup 的请求。一旦所有成员都发了 joinGroup请求，那么 coordinator 会选择一个 Consumer 担任 leader 角色，并把组成员信息和订阅信息发送给消费者：



protocol-metadata：序列化后的消费者的订阅信息
leader id：消费组中的消费者，coordinator 会选择一个作为 leader，对应的就是 member id
member metadata：对应消费者的订阅信息
members: consumer group 中全部的消费者的订阅信息
generation_id：年代信息，类似于 ZooKeeper 中的 epoch，对于每一轮 Rebalance，generation_id 都会递增。主要用来保护consumer group，隔离无效的 offset 提交。也就是上一轮的 consumer 成员无法提交 offset 到新的 Consumer group中。
