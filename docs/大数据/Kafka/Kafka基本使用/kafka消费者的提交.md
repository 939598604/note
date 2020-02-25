# KafkaConsumer中的位移提交

位移是指 TopicPartition 在 Broker 端的存储偏移量。而消费者位移是指某个消费者组在不同 TopicPartition 上面的消费偏移量。下面我们介绍一下消费者位移的提交方式，其中主要包含了自动提交和手动提交。 

## 自动提交

对于启用自动提交位移，在 KafkaConsumer 实例初始化的时候，通过设置参数 enable.auto.commit 的值为 true 即可（默认为true）。同时与其相关联的参数 auto.commit.interval.ms，这个参数可以设置提交的时间间隔，这个值默认是5秒。

对于自动提交的触发条件，除了要满足时间的阈值，还需要Client端调用 KafkaConsumer.poll() 方法才能触发。每次执行都会调用 ConsumerCoordinator.poll() 执行消费者入组的流程，在方法执行的最后会执行一个异步的 offset 提交。实现代码如下：

```scala
public void maybeAutoCommitOffsetsAsync(long now) {
    if (autoCommitEnabled) {
        nextAutoCommitTimer.update(now);
        if (nextAutoCommitTimer.isExpired()) {
            nextAutoCommitTimer.reset(autoCommitIntervalMs);
            doAutoCommitOffsetsAsync();
        }
    }
}

private void doAutoCommitOffsetsAsync() {
    Map<TopicPartition, OffsetAndMetadata> allConsumedOffsets = subscriptions.allConsumed();
    log.debug("Sending asynchronous auto-commit of offsets {}", allConsumedOffsets);

    commitOffsetsAsync(allConsumedOffsets, (offsets, exception) -> {
        if (exception != null) {
            if (exception instanceof RetriableException) {
                log.debug("Asynchronous auto-commit of offsets {} failed due to retriable error: {}", offsets,
                    exception);
                nextAutoCommitTimer.updateAndReset(rebalanceConfig.retryBackoffMs);
            } else {
                log.warn("Asynchronous auto-commit of offsets {} failed: {}", offsets, exception.getMessage());
            }
        } else {
            log.debug("Completed asynchronous auto-commit of offsets {}", offsets);
        }
    });
}
```

## 手动提交

与自动提交相对应 就是手动提交了，此时在创建KafkaConsumer实例时，就需要指定 enable.auto.commit 的值为false。然后在处理完消息之后，自己手动执行 commit 操作。对于手动提交位移，又分为同步提交和异步提交。

## 同步提交

同步提交时通过 KafkaConsumer.commitSync()方法实现，其内部又调用了ConsumerCoordinator.commitOffsetsSync() 方法发送位移提交请求。同步的位移提交提供了重试的机制，其代码实现如下：

```scala
/**
 * 同步提交位移，如失败，此方法将会重试提交位移，直至超时
 */
public boolean commitOffsetsSync(Map<TopicPartition, OffsetAndMetadata> offsets, Timer timer) {
    invokeCompletedOffsetCommitCallbacks();

    if (offsets.isEmpty())
        return true;

    do {
        if (coordinatorUnknown() && !ensureCoordinatorReady(timer)) {
            return false;
        }

        RequestFuture<Void> future = sendOffsetCommitRequest(offsets);
        client.poll(future, timer);

        // 如有处理中的位移提交，则等待执行完成
        invokeCompletedOffsetCommitCallbacks();

        if (future.succeeded()) {
            if (interceptors != null)
                interceptors.onCommit(offsets);
            return true;
        }

        if (future.failed() && !future.isRetriable())
            throw future.exception();

        timer.sleep(rebalanceConfig.retryBackoffMs);
    } while (timer.notExpired());

    return false;
}
```

对于同步提交，官方文档上面提供了一次拉取，按照批次提交位移的方式，这样可以减少重复消费的批量。 

```scala
Properties props = new Properties();
props.setProperty("bootstrap.servers", "localhost:9092");
props.setProperty("group.id", "test");
props.setProperty("enable.auto.commit", "false");
props.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("foo", "bar"));
final int minBatchSize = 200;
List<ConsumerRecord<String, String>> buffer = new ArrayList<>();
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        buffer.add(record);
    }
    if (buffer.size() >= minBatchSize) {
        insertIntoDb(buffer);
        consumer.commitSync();
        buffer.clear();
    }
}
```

另外一种方式是，可以按照TopicPartition 分区的维度去提交位移。

```
try {
    while(running) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(Long.MAX_VALUE));
        for (TopicPartition partition : records.partitions()) {
            List<ConsumerRecord<String, String>> partitionRecords = records.records(partition);
            for (ConsumerRecord<String, String> record : partitionRecords) {
                System.out.println(record.offset() + ": " + record.value());
            }
            long lastOffset = partitionRecords.get(partitionRecords.size() - 1).offset();
            consumer.commitSync(Collections.singletonMap(partition, new OffsetAndMetadata(lastOffset + 1)));
        }
    }
} finally {
  consumer.close();
}
```

## 异步提交

对于同步的位移提交，通常情况下会影响系统的吞吐量。此时KafkaConsumer也提供了异步的提交方式，也就是commitAsync()。但是相对同步的位移提交，此时异步提交缺少了重试的机制，同步的重试机制可以在网络抖动的场景下，减少提交失败的场景。异步提交在这里没有重试机制，是因为重试的时候消费位移可能已经变化，此时提交已经没啥意义了。

在生产上面，还是建议使用异步的位移提交，这样也可以提升客户端的TPS。对于提交的方式，笔者从网上找到了如下的提交方式：

```
try {
    while(true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
        process(records); // 处理消息
        commitAysnc(); // 使用异步提交规避阻塞
    }
} catch(Exception e) {
    handle(e); // 处理异常
} finally {
    try {
        consumer.commitSync(); // 最后一次提交使用同步阻塞式提交
    } finally {
        consumer.close();
    }
}
```

