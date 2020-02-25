#kafka消费者代码编写

## 6.多线程消费消息

### 6.1 消费者用单线程消费消息

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

### 6.2消费用多线程消费消息

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

