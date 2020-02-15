# redis延时对列

## 一.延时队列作用

```
场景：订单超时未支付，取消订单，恢复库存
可以将创建的订单加入redis延时队列，开一个单独的线程轮循处理过期订单
注意，如果订单已经支付，需要在延时队列中删除该订单记录
```

## 二.几种延时队列

*延时队列就是一种带有延迟功能的消息队列。下面会介绍几种目前已有的延时队列：*

- **Java中java.util.concurrent.DelayQueue**

​       优点：JDK自身实现，使用方便，量小适用*

​       缺点：队列消息处于jvm内存，不支持分布式运行和消息持久化*

- **Rocketmq延时队列**

  优点：消息持久化，分布式

  缺点：不支持任意时间精度，只支持特定level的延时消息

- **Rabbitmq延时队列（TTL+DLX实现）**

  优点：消息持久化，分布式

  缺点：延时相同的消息必须扔在同一个队列

- **redis延时队列**

- 基于zset实现方式

  ```
  通过 Redis 的 zset （有序列表）来实现。将消息序列化成一个字符串作为 zset 的 value，
  这个消息的到期处理时间作为 score，然后用多个线程轮询zset 获取到期的任务进行处理 。
  多个线程是为了保障可用性，万一挂了一个线程还有其他线程可以继续处理。
  因为有多个线程，所以需要考虑并发争抢任务，确保任务不会被多次执行。
  ```

- 基于key的过期订阅方式实现

  ```
  
  ```

  
##  三.基于Redis实现

### 3.1 引入依赖

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
<modelVersion>4.0.0</modelVersion>
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.2.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
<groupId>com.example</groupId>
<artifactId>springboot_test</artifactId>
<version>0.0.1-SNAPSHOT</version>
<name>springboot_test</name>
<description>Demo project for Spring Boot</description>

<properties>
    <java.version>1.8</java.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>

<repositories>
    <repository>
        <id>spring-milestones</id>
        <name>Spring Milestones</name>
        <url>https://repo.spring.io/milestone</url>
    </repository>
</repositories>
<pluginRepositories>
    <pluginRepository>
        <id>spring-milestones</id>
        <name>Spring Milestones</name>
        <url>https://repo.spring.io/milestone</url>
    </pluginRepository>
</pluginRepositories>

</project>
```

### 3.2 application.yml文件

```
server:
  port: ${random.int(10000,12000)}
```

### 3.3 TimeDelayConsumer 

```
package com.example.delay;

import java.util.Collection;

public interface TimeDelayConsumer {
    /***
     * 是否支持
     * @param type
     * @return
     */
    public boolean support(String type);

    /***
     * 消费延迟任务
     * @param id
     * @param type
     * @return
     */
    public void consume(String id, String type);

}
```

### 3.4 TimeDelayManager

```
package com.example.delay;

import java.util.concurrent.TimeUnit;

public interface TimeDelayManager {
    /****
     * 添加延时任务
     * @param type 类型，如:"order-delete" "order-confirm"
     * @param id id,唯一标示，如订单id
     * @param timeDelay 延迟时间
     * @param timeUnit 时间单位
     * @return
     */
    public boolean addTask(String type, String id, Long timeDelay, TimeUnit timeUnit);

    /***
     * 注册消费者
     * @return
     */
    public void addConsumer(TimeDelayConsumer timeDelayConsumer);

    /***
     * 消费
     */
    public void consume();

}
```

### 3.5  RedisTimeDelayManager

```
package com.example.delay;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.ZSetOperations;
import org.springframework.util.Assert;

import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class RedisTimeDelayManager implements TimeDelayManager, Runnable {

    private static final Logger log = LoggerFactory.getLogger(RedisTimeDelayManager.class);

    protected RedisTemplate redisTemplate;

    /***
     * 键值前缀
     */
    protected String keyPrefix = "timedelay";

    /***
     * 消费者List
     */
    protected List<TimeDelayConsumer> consumerList = new ArrayList<>(10);

    /***
     * 在redis中保存现在注册的type，对应的键值
     */
    private String typesKey = "";


    public RedisTimeDelayManager() {
        typesKey = keyPrefix + ":types";
    }

    public RedisTimeDelayManager(RedisTemplate redisTemplate) {
        this();
        Assert.notNull(redisTemplate, "redisTemplate不能为空");
        this.redisTemplate = redisTemplate;
    }

    public RedisTimeDelayManager(RedisTemplate redisTemplate, String keyPrefix) {
        this(redisTemplate);
        Assert.hasText(keyPrefix, "keyPrefix不能为空");
        this.keyPrefix = keyPrefix;
    }

    @Override
    public boolean addTask(String type, String id, Long timeDelay, TimeUnit timeUnit) {
        try {
            if (timeDelay <= 0) {
                //如果延迟时间小于0，则直接返回false
                return false;
            }
            timeDelay = timeUnit.toMillis(timeDelay);
            Long currentTime = System.currentTimeMillis();
            String key = keyPrefix + ":" + type;
            redisTemplate.opsForZSet().add(key, id, currentTime + timeDelay);
            redisTemplate.opsForSet().add(typesKey, type);
            return true;
        } catch (Exception ex) {
            ex.printStackTrace();
            return false;
        }
    }

    @Override
    public void addConsumer(TimeDelayConsumer timeDelayConsumer) {
        consumerList.add(timeDelayConsumer);
    }

    @Override
    public void consume() {
        ExecutorService executors = Executors.newCachedThreadPool();
        for (int i = 0; i < 10; i++) {
            executors.execute(() -> {
                Thread consumeThread = new Thread(this);
                consumeThread.start();
            });
        }
    }


    @Override
    public void run() {
        while (true) {
            try {
                Set<String> types = redisTemplate.opsForSet().members(typesKey);
                if (types == null || types.isEmpty()) {
                    Thread.sleep(500);
                    continue;
                }
                Long currentTime = System.currentTimeMillis();
                for (String type : types) {
                    String key = keyPrefix + ":" + type;
                    Set<ZSetOperations.TypedTuple<String>> items = redisTemplate.opsForZSet().rangeByScoreWithScores(key, 0, currentTime);
                    if (items == null || items.isEmpty()) {
                        continue;
                    }
                    for (TimeDelayConsumer consumer : consumerList) {
                        if (!consumer.support(type)) {
                            continue;
                        }
                        for (ZSetOperations.TypedTuple<String> item : items) {
                            boolean removeSuccess = redisTemplate.opsForZSet().remove(key, item.getValue()) == 1;
                            if (!removeSuccess) {
                                continue;
                            }
                            try {
                                consumer.consume(item.getValue(), type);
                            } catch (Exception e) {
                                //消费延时任务失败，重新投递
                                redisTemplate.opsForZSet().add(key, item.getValue(), item.getScore());
                                log.error("消费延时任务异常", e);
                                e.printStackTrace();
                            }

                        }
                        break;
                    }
                }
                Thread.sleep(500);
            } catch (InterruptedException e) {
                log.error("消费延时任务异常", e);
                e.printStackTrace();
            }
        }
    }
}
```

### 3.6  OrderConsumer

```
package com.example.delay;

public class OrderConsumer implements TimeDelayConsumer {
    @Override
    public boolean support(String type) {
        return true;
    }

    @Override
    public void consume(String id, String type) {
        System.out.println(Thread.currentThread().getName() + ":消费了订单:" + id);
    }
}
```

### 3.7  DelayController

```
package com.example.delay;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.web.bind.annotation.RestController;

import javax.annotation.PostConstruct;
import java.util.concurrent.TimeUnit;

@RestController
public class DelayController {

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @PostConstruct
    public void init (){
        TimeDelayManager timeDelayCenter = new RedisTimeDelayManager(stringRedisTemplate);
        Thread orderProducter = new Thread(() -> {
            for (int i = 0; i < 100; i++) {
                timeDelayCenter.addTask("order", i + "", 5L, TimeUnit.SECONDS);
                System.out.println("生成了订单:" + i);
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        orderProducter.start();
        TimeDelayConsumer timeDelayConsumer = new OrderConsumer();
        timeDelayCenter.addConsumer(timeDelayConsumer);
        timeDelayCenter.consume();
    }
}
```

## 四 代码

https://github.com/939598604/java_learning.git