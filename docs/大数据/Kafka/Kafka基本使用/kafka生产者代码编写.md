# Kafka生产者

## 1.使用脚本操作生产者

Kafka系统提供了一系列的操作脚本，这些脚本放置在$KAFKA_HOME/bin目录中。其中，kafka-console-producer.sh脚本可用来作为生产者客户端。

在安装Kafka集群后，可以执行kafka-console-producer.sh脚本快速地做一些简单的功能验证。通过调用工具类脚本（kafka-run-class.sh）并输入对应的工具类，可以实现具体的功能。

在执行kafka-console-producer.sh脚本时，可以通过设置KAFKA_HEAP_OPTS属性给客户端分配内存，具体内容为kafka-console-producer.sh源代码

```
#给客户端分配内存
if["x$KAFKA_HEAP_OPTS"="x"]；then
    export KAFKA_HEAP_OPTS="-Xmx512M"
fi
#输入工具类kafka.tools.ConsoleProducer 
exec $（dirname $0）/kafka-run-class.sh kafka.tools.ConsoleProducer"$0"
```

具体操作命令如下：

```
 kafka-console-producer. sh --broker-list dn1:9092, dn2:9092, dn3:9092 --topic test_topic
```

执行上述命令后，Linux控制台会等待用户输入信息。输入“hello kafka”后按Enter键

## 2.启动消费者程序，并查看消息

在Linux系统控制台中，通过执行kafka-console-consumer.sh脚本来启动一个消费者程序，并在Linux控制台中观察读取结果。

```
kafka-console-consumer.sh --bootstrap-server dn1:9092, dn2:9092, dn3:9092 --topic test_topic
```

##3.发送消息到Kafka主题

Kafka系统支持两种不同的发送方式——同步模式（Sync）和异步模式（ASync）。

### 3.1 了解异步模式

（1）什么场景下需要使用异步模式

假如生产者客户端与Kafka集群节点间存在网络延时（100ms），此时发送10条消息记录，则延时将达到1s。而大数据场景下有着海量的消息记录，发送的消息记录是远不止10条，延时将非常严重。大数据场景下，如果采用异步模式发送消息记录，几乎没有任何耗时，通过回调函数可以知道消息发送的结果。

（2）异步模式数据写入流程

例如，一个业务主题（ip_login）有6个分区。生产者客户端写入一条消息记录时，消息记录会先写入某个缓冲区，生产者客户端直接得到结果（这时，缓冲区里的数据并没有写到Kafka代理节点中主题的某个分区）。之后，缓冲区中的数据会通过异步模式发送到Kafka代理节点中主题的某个分区中。具体数据写入流程如图4-7所示。

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200106094201.png)

### 3.2生产者用异步模式发送消息

在生产者客户端创建一个扩展名为java的Java源代码文件，使用异步模式发送消息

```java
//实例化一个消息记录对象，用来保存主题名、分区索引、键、值和时间戳ProducerRecord<byte[]，byte[]>record=new ProducerRecord<byte[]，byte[]>（"ip_login"，key，value）；
//调用send（）方法和回调函数
producer.send（myRecord，new Callback）{
    public void onCompletion（RecordMetadata metadata，Exception e）{
        if（e！=null）{
            e.printStackTrace（）；
        }else{
            System.out.println（"The offset of the record we just sent is:"+
            metadata.offset（））；
        }
    }
}）；
```

消息记录提交给send()方法后，实际上该消息记录被放入一个缓冲区的发送队列，然后通过后台线程将其从缓冲区队列中取出并进行发送；发送成功后会触发send方法的回调函数——Callback。

### 3.3 了解同步模式

生产者客户端通过send()方法实现同步模式发送消息，并返回一个Future对象，同时调用get()方法等待Future对象，看send()方法是否发送成功。

1．什么场景下使用同步模式

如果要收集用户访问网页的数据，在写数据到Kafka集群代理节点时需要立即知道消息是否写入成功，此时应使用同步模式。

2．同步模式的数据写入流程

例如，在一个业务主题（ip_login）中有6个分区。生产者客户端写入一条消息记录到生产者服务端，生产者服务端接收到数据后会立马将其发送到主题（ip_login）的某个分区去，然后才将结果返给生产者客户端。具体流程如图所示。

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200106094950.png)

### 3.4生产者用同步模式发送消息

在生产者客户端创建一个扩展名为java的Java源代码文件，实现同步模式发送消息。

```
//将字符串转换成字节数组
byte[]key="key".getBytes（）；byte[]value="value".getBytes（）；
//实例化一个消息记录对象，用来保存主题名、分区索引、键、值和时间戳ProducerRecord<byte[]，byte[]>record=
new ProducerRecord<byte[]，byte[]>（"ip_login"，key，value）；
//调用send（）函数后，再通过get（）方法等待返回结果producer.send（record）.get（）；
```

这里通过调用Future接口中的get()方法等待Kafka集群代理节点（Broker）的状态返回。如果Producer发送消息记录成功了，则返回RecordMetadata对象，该对象可用来查看消息记录的偏移量（Offset）

> 采用同步模式发送消息记录，系统的性能会下降很多，因为需要等待写入结果。如果生产者客户端和Kafka集群间的网络出现异常，或者Kafka集群处理消息请求过慢，则消息的延时将很严重。所以，一般不建议这样使用。

## 4.生产者发送消息

### 4.1简单版

```
package com.gzstrong.kafka.producers;


import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;

import java.util.Properties;

public class SimpleProducer {
    public static void main(String[] args) {

        String topicName = "test";

        Properties props = new Properties();
        props.put("bootstrap.servers", "kafka.gzstrong.com:9092");
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        /*创建生产者*/
        Producer<String, String> producer = new KafkaProducer<>(props);

        for (int i = 0; i < 10; i++) {
            ProducerRecord<String, String> record = new ProducerRecord<>(topicName, "hello" + i, "world" + i);
            /* 发送消息*/
            producer.send(record);
        }

        /*关闭生产者*/
        producer.close();
    }
}
```

### 4.2生产者同步发送消息

```
package com.gzstrong.kafka.producers;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;

import java.util.Properties;
import java.util.concurrent.ExecutionException;

public class ProducerSyn {
    public static void main(String[] args) {

        String topicName = "test";

        Properties props = new Properties();
        props.put("bootstrap.servers", "kafka.gzstrong.com:9092");
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        /*创建生产者*/
        Producer<String, String> producer = new KafkaProducer<>(props);

        for (int i = 0; i < 10; i++) {
            try {
                ProducerRecord<String, String> record = new ProducerRecord<>(topicName, "k" + i, "world" + i);
                /*同步发送消息*/
                RecordMetadata metadata = producer.send(record).get();
                System.out.printf("topic=%s, partition=%d, offset=%s \n",
                        metadata.topic(), metadata.partition(), metadata.offset());
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }
        }

        /*关闭生产者*/
        producer.close();
    }
}
```

### 4.3 生产者异步发送消息

```
package com.gzstrong.kafka.producers;

import org.apache.kafka.clients.producer.*;

import java.util.Properties;

public class ProducerASyn {

   public static void main(String[] args){
       String topicName = "test";
       Properties props = new Properties();
       props.put("bootstrap.servers", "kafka.gzstrong.com:9092");
       props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
       props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
    /*创建生产者*/
       Producer<String, String> producer = new KafkaProducer<>(props);

       for (int i = 0; i < 10; i++) {
           ProducerRecord<String, String> record = new ProducerRecord<>(topicName, "k" + i, "world" + i);
            /*异步发送消息，并监听回调*/
           producer.send(record, new Callback() {
               @Override
               public void onCompletion(RecordMetadata metadata, Exception exception) {
                   if (exception != null) {
                       System.out.println("进行异常处理");
                   } else {
                       System.out.printf("topic=%s, partition=%d, offset=%s \n",
                               metadata.topic(), metadata.partition(), metadata.offset());
                   }
               }
           });
       }

        /*关闭生产者*/
       producer.close();
   }
}
```

### 4.4生产者分区发送消息

```
package com.gzstrong.kafka.producers;

import org.apache.kafka.clients.producer.*;

import java.util.Properties;

/*
 * Kafka生产者示例——异步发送消息
 */
public class ProducerWithPartitioner {

    public static void main(String[] args) {

        String topicName = "Kafka-Partitioner-Test";

        Properties props = new Properties();
        props.put("bootstrap.servers", "hadoop001:9092");
        props.put("key.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

        /*传递自定义分区器*/
        props.put("partitioner.class", "com.heibaiying.producers.partitioners.CustomPartitioner");
        /*传递分区器所需的参数*/
        props.put("pass.line", 6);

        Producer<Integer, String> producer = new KafkaProducer<>(props);

        for (int i = 0; i <= 10; i++) {
            String score = "score:" + i;
            ProducerRecord<Integer, String> record = new ProducerRecord<>(topicName, i, score);
            /*异步发送消息*/
            producer.send(record, (metadata, exception) ->
                    System.out.printf("%s, partition=%d, \n", score, metadata.partition()));
        }

        producer.close();
    }
}
```

**自定义分区器**

```
package com.gzstrong.kafka.producers.partitioners;

import org.apache.kafka.clients.producer.Partitioner;
import org.apache.kafka.common.Cluster;

import java.util.Map;

/**
 * 自定义分区器
 */
public class CustomPartitioner implements Partitioner {

    private int passLine;

    @Override
    public void configure(Map<String, ?> configs) {
        passLine = (Integer) configs.get("pass.line");
    }

    @Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        return (Integer) key >= passLine ? 1 : 0;
    }

    @Override
    public void close() {
        System.out.println("分区器关闭");
    }


}
```



## 5.多线程发送消息

在Kafka系统中，为了保证生产者客户端应用程序的独立运行，通常使用线程的方式发送消息。创建一个简单的生产者应用程序的步骤如下。

（1）实例化Properties类对象，配置生产者应答机制。有以下三个属性是必须设置的。其他属性一般都会有默认值，可以按需添加设置。

- bootstrap.servers：配置Kafka集群代理节点地址；
- key.serializer：序列化消息主键；
- value.serializer：序列化消息数据内容。

（2）根据属性对象实例化一个KafkaProducer。

（3）通过实例化一个ProducerRecord对象，将消息记录以“键-值”对的形式进行封装。

（4）通过调用KafkaProducer对象中带有回调函数的send()方法发送消息给Kafka集群。

（5）关闭KafkaProducer对象，释放连接资源。

###  5.1 生产者用单线程发送消息

创建一个扩展名为java的Java源代码文件，编写代码，通过异步模式发送消息，并使用单线程来执行生产者客户端程序，观察操作结果。

```java
package com.gzstrong.kafka.producers;

import com.alibaba.fastjson.JSONObject;
import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.clients.producer.*;

import java.util.Date;
import java.util.Properties;

@Slf4j
public class SingleThreadProducer extends Thread {

    String topicName = "test";

    public Properties configure() {
        Properties props = new Properties();
        //指定Kafka集群地址
        props.put("bootstrap.servers", "kafka.gzstrong.com:9092");
        props.put("acks", "1");//设置应答机制
        props.put("batch.size", 16384);//批量提交大小
        props.put("1inger.ms", 1);//延时提交
        props.put("buffer.menory", 33554432);//缓冲大小
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        return props;
    }

    /**
     * 单线程启动入口
     */
    public static void main(String[] args) {
        SingleThreadProducer producer = new SingleThreadProducer();
        producer.start();
    }

    @Override
    public void run() {
        Producer<String, String> producer = new KafkaProducer<>(configure());
        //发送100条JSON格式的数据
        for (int i = 0; i < 100; i++) {
            //封装JSON格式
            JSONObject json = new JSONObject();
            json.put("id", i);
            json.put("ip", "192.168.0." + i);
            json.put("date", new Date().toString());
            String k = "key" + i;
            //异步发送,调用回调函数
            producer.send(new ProducerRecord<String, String>(topicName, k, json.toJSONString()), new Callback() {
                @Override
                public void onCompletion(RecordMetadata metadata, Exception e) {
                    if (e != null) {
                        log.error("Send error,nsg is" + e.getMessage());
                    } else {
                        log.info("The offset of the record we just sent is:" + metadata.offset());
                    }
                }
            });
        }
        try{
            sleep(3000);//间隔3秒
        }catch(InterruptedException e) {
            log.error("Interrupted thread error，msg is" + e.getMessage());
            producer.close();//关闭生产者对象
        }
    }
}
```

这里的主题只有一个分区和一个副本，所以，发送的所有消息会写入同一个分区中。

如果希望发送完消息后获取一些返回信息（比如获取消息的偏移量、分区索引值、提交的时间戳等），则可以通过回调函数CallBack返回的RecordMetadata对象来实现。

由于Kafka系统的生产者对象是线程安全的，所以，可通过增加生产者对象的线程数来提高Kafka系统的吞吐量。

> 在实际项目中，一般会采用多个生产者对象来发送消息以提高吞吐量。线程安全是指，多线程访问时采用了加锁机制。即，当一个线程访问类的某个数据时进行保护，其他线程不能进行访问。

### 5.2 生产者用多线程发送消息

创建一个扩展名为java的Java源代码文件，编写代码，通过异步模式发送消息，并使用多线程来执行生产者客户端程序。

多线程生产者客户端的实现思路如下：

（1）定义一个多线程生产者JProducerThread类并继承Thread基类；

（2）定义一个线程池，并设置最大线程数。

```java
package com.gzstrong.kafka.producers;

import com.alibaba.fastjson.JSONObject;
import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.clients.producer.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Date;
import java.util.Properties;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

@Slf4j
public class MultiThreadProducer extends Thread{

    String topicName = "test";


    //声明最大线程数
    private final static int MAX_THREAD_SIZE=6;
    /**配置Kafka连接信息*/
    public Properties configure() {
        Properties props = new Properties();
        //指定Kafka集群地址
        props.put("bootstrap.servers", "kafka.gzstrong.com:9092");
        //设置应答机制
        props.put("acks", "1");
        //批里提交大小
        props.put("batch.size", 16384);
        //延时提交
        props.put("linger.ms", 1);
        //缓冲大小
        props.put("buffer.memory", 33554432);
        //序列化键值
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        return props;
    }

    public static void main(String[] args){
        //创建一个固定线程数量的线程池
        ExecutorService executorService= Executors.newFixedThreadPool(MAX_THREAD_SIZE);
        // 提交任务
        executorService.submit(new MultiThreadProducer());
        //关闭线程池
        executorService.shutdown();    
    }

    @Override
    public void run() {
        Producer<String,String> producer =new KafkaProducer<>(configure());
        //发送100条JSON格式的数据
        for(int i=0;i<100;i++){
            //封装JSON格式
            JSONObject json=new JSONObject();
            json.put("id",i);
            json.put("ip","192.168.0."+i);
            json.put("date",new Date());
            String k="key"+i;
            //异步发送,调用回调函数
            producer.send(new ProducerRecord<String,String>(topicName,k,json.toJSONString()), new Callback() {
                @Override
                public void onCompletion(RecordMetadata metadata, Exception e) {
                    if(e!=null){
                        log.error("Send error,msg is"+e.getMessage());
                    }else{
                        log.info("The offset of the record we just sent is:"+metadata.offset());
                    }
                }
            });
        }
        try{
            sleep(3000);//间隔s秒
        }catch( InterruptedException e){
            log.error("Interrupted thread error,msg is"+e.getMessage());
        }
    }
}
```

