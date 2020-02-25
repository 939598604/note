# kafka控制器

## 控制器是如何被选出来的？

​    你一定很想知道，控制器是如何被选出来的呢？我们刚刚在前面说过，每台 Broker 都能充当控制器，那么，当集群启动后，Kafka 怎么确认控制器位于哪台 Broker 呢？
实际上，Broker 在启动时，会尝试去 ZooKeeper 中创建 /controller 节点。Kafka 当前选举控制器的规则是：第一个成功创建 /controller 节点的 Broker 会被指定为控制器。

## kafka集群哪些数据是要记录到ZK中？

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/QQ截图.png)

集群中所有的代理节点，topic的所有分区，分区的副本信息（副本集，主副本，ISR副本集）。

## 控制器主要作用是什么？

1、主题管理：控制器会帮助我们完成Topic 的创建、删除以及增加分区。也就是当执行 kafka-topics.sh 脚本时，大部分的后台工作是由Kafka控制器完成的。

2、分区重分配：是指 kafka-reassign-partitions.sh 脚本对已有主题分区的细粒度的分配功能。

3、Preferred 领导者选举：主要是Kafka为了避免因部分 Broker 的负载过重，而提供的一种换 Leader 的方案。

4、集群成员管理：其中包括新增 Broker、Broker 主动或被动关闭。/brokers/ids/ 下面会存放 Broker 实例的id 临时节点，当我们看到 /brokers/ids 下面有几个节点，就表示有多少个存活的 Broker 实例。当Broker 宕机时，临时节点就会被删除，此时控制器对应的监听器就会感知到Broker 下线，进而完成对应的下线工作。

5、数据服务：控制器上面存放了最全的集群元数据信息，其他 Broker 会定期接收控制器发来的更新请求，从而定期的刷新其元数据缓存数据。

## 控制器保存了什么数据？

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/QQ截图20200102094652.png)

这里面比较重要的数据有：
    所有主题信息。包括具体的分区信息，比如领导者副本是谁，ISR 集合中有哪些副本等。
    所有 Broker 信息。包括当前都有哪些运行中的 Broker，哪些正在关闭中的 Broker 等。
    所有涉及运维任务的分区。包括当前正在进行 Preferred 领导者选举以及分区重分配的分区列表。
    值得注意的是，这些数据其实在 ZooKeeper 中也保存了一份。每当控制器初始化时，它都会从 ZooKeeper 上读取对应的元数据并填充到自己的缓存中。有了这些数据，控制器就能对外提供数据服务了。这里的对外主要是指对其他 Broker 而言，控制器通过向这些 Broker 发送请求的方式将这些数据同步到其他 Broker 上

## 控制器故障转移是怎样实现的

开始时，假如Broker 0 是控制器。当 Broker 0 宕机后，ZooKeeper 通过 Watch 机制感知到并删除了 /controller 临时节点。之后，所有存活的 Broker1，Broker2，Broker3开始竞选新的控制器身份。Broker 3 最终赢得了选举，成功地在 ZooKeeper 上重建了 /controller 节点。之后，Broker 3 会从 ZooKeeper 中读取集群元数据信息，并初始化到自己的缓存中。至此，控制器的 Failover 完成，可以行使正常的工作职责了。



