## 使用 Redis 和 Spring Boot 执行异步任务
**【转载自 ImportNew 公众号】地址: [https://mp.weixin.qq.com/s/fC0rK_1jvQMHxzDHRkh6cQ](https://mp.weixin.qq.com/s/fC0rK_1jvQMHxzDHRkh6cQ)**

在本文中，将学习如何使用 Spring Boot 2.x 和 Redis 执行异步任务。文后附有演示代码。

### Spring/Spring Boot
Spring 是最流行 Java 应用程序开发框架。因此，Spring 社区也是最大的开源社区之一。除此之外，Spring 博客还提供了最新的开发文档，内容非常丰富。涵盖了框架的内部工作原理和示例项目，在StackOverflow上有10万多个问题。 

Spring 早期只支持基于XML的配置，为此饱受批评。后来 Spring 引入了基于注解的配置，情况发生了根本改变。Spring 3.0是第一个支持基于注解的配置的版本。2014年发布的 Spring Boot 1.0，彻底改变了人们对 Spring 框架生态的看法。在这里可以找到更详细的时间表。 

### Redis
Redis 是最流行的 NoSQL 内存数据库之一，支持不同类型的数据结构，包括 Set、哈希表、List、简单键值对等。Redis 调用延迟为亚毫秒级，支持 replica set 等功能。Redis 操作的延迟也是亚毫秒级，在开发者社区中更具吸引力。

### 为什么需要异步执行任务
一个典型的 API 调用包括以下五个方面：
1. 执行数据库（RDBMS 或 NoSQL）查询
2. 在某些缓存系统（内存、分布式等）上执行操作
3. 进行计算（对一些数据进行数学计算）
4. 调用其他（内部或外部）服务
5. 调度任务稍后执行或者在后台立即执行

任务可根据需要定时执行，例如在创建订单或装运单后7天后生成发票。同样，电子邮件通知无需立即发送，因此可以设为延期发送。  

考虑到这些实际场景，有时候需要异步执行任务，减少 API 响应时间。例如，如果一次在同一个 API 调用中删除一千多条记录，那么肯定会增加 API 响应时间。为了减少 API 响应时间，可以运行一个后台任务删除这些记录。 

### 延迟队列
当计划在指定时间或者按照设定间隔执行任务时，可以使用 corn job。有很多不同的工具可以执行定时任务，比如 UNIX 风格的 crontabs、Chronos。如果用 Spring 框架，那么可以用默认提供的 `Scheduled` 注解。 

大多数 cron job 会在需要执行特定操作时查找记录，例如查找所有已发货7天但未生成发票的记录。这些调度机制中大多数都会遇到扩展问题，在数据库中扫描查找相关行或者记录。多数情况会引发全表扫描，性能非常差。设想一下正在运行的应用程序与批处理系统使用相同的数据库。由于不可扩展，因此需要一些可扩展系统，可以在指定时间或按照设定时间间隔执行任务，不会出现任何性能问题。有许多扩展的方法，例如用批处理方式执行任务，或者在用户、区域子集上执行任务。另一种方法是在指定时间执行任务，任务之间没有依赖，例如 serverless 函数。定时器达到预定时间后会立即触发执行作业，这时可以使用延迟队列。有很多队列系统或软件可供使用，但很少像 SQS 这样可以设置延迟15分钟，而不是延迟7个小时或者7天。

### Rqueue
Rqueue 是针对 Spring 框架构建的消息代理，它把数据存储到 Redis 中并且提供了一种机制可以按任意延迟执行任务。Rqueue 得到了 Redis 支持。相比 Kafka、SQS 等常见队列系统，Redis 具有一些优势。在大多数 Web 应用后端程序中，Redis 用来缓存数据或者其他用途。在当今世界，8.4％的 Web 应用程序正在使用 Redis 数据库。


通常，使用 Kafka、SQS 或者其他队列系统会带来不同程度的额外开销，而 Rqueue 和 Redis 可以将费用降为零。

除了使用 Kafka 带来的开销，还需配置基础架构、进行维护等等。由于大多数程序已经使用了 Redis，因此不需要其他操作开销。实际上，可以在同一个 Redis 服务器或群集上使用 Rqueue。Rqueue支持任意长度延迟

![](./img/image-1.jpg)

### 消息传递
由于长数据不会在数据库中丢失，Rqueue 能够确保至少发送一次消息。在 Rqueue 简介中可以了解更多信息。


使用Rqueue开发库实现按任意延迟时间执行任务。Rqueue 是一个基于 Spring 的异步任务执行器，可以按照任意延迟执行任务，它基于 Spring 消息库并由 Redis 提供支持。

这里将使用 **com.github.sonus21:rqueue-spring-boot-starter:1.2-RELEASE添加 Rqueue spring boot starter** 依赖。

`rqueue` 的github[地址](https://github.com/sonus21/rqueue)

完整的maven配置接口如下:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.2.0.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
  </parent>
  <groupId>com.lx</groupId>
  <artifactId>springboot-rqueue</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <name>springboot-rqueue</name>
  <description>Springboot+Redis实现延时队列</description>

  <properties>
    <java.version>1.8</java.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <optional>true</optional>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
      <exclusions>
        <exclusion>
          <groupId>org.junit.vintage</groupId>
          <artifactId>junit-vintage-engine</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
    <dependency>
      <groupId>com.github.sonus21</groupId>
      <artifactId>rqueue-spring-boot-starter</artifactId>
      <version>1.2-RELEASE</version>
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
      <id>spring</id>
      <url>https://maven.aliyun.com/repository/spring</url>
      <releases>
        <enabled>true</enabled>
      </releases>
      <snapshots>
        <enabled>true</enabled>
      </snapshots>
    </repository>
  </repositories>

</project>
```

需要启用 Redis Spring Boot 功能。出于测试目的，这里还将启用 WEB MVC。
Application启动类 文件更新如下：
```java
@SpringBootApplication
@EnableRedisRepositories
@EnableWebMvc
public class Application {

  public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
  }

}
```

使用 Rqueue 添加任务非常简单，只要对方法添加 RqueueListener 注解。RqueuListener 注解提供了一些字段，可根据使用场景设置。对于延迟任务，需要设置 `delayedQueue="true"` 并且必须提供 `value`；否则注解将被忽略。value 是队列名称。设置 deadLetterQueue 可以将任务推送到另一个队列。否则，执行失败时任务会被丢弃。还可以使用 numRetries 字段设置任务重试次数。

创建名为 MessageListener 的 Java 文件并增加一些方法执行任务：
```java
@Component
@Slf4j
public class MessageListener {
  @RqueueListener(value = "${email.queue.name}")
  public void sendEmail(Email email) {
    log.info("Email {}", email);
  }

  @RqueueListener(delayedQueue = "true", value = "${invoice.queue.name}")
  public void generateInvoice(Invoice invoice) {
    log.info("Invoice {}", invoice);
  }
}
```

用 Email 和 Invoice 类分别存储电子邮件和发票数据。简单起见，这些类只包含少数几个字段。

`Invoice.java`：
```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Invoice {
  private String id;
  private String type;
}
```

`Email.java`:
```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Email {
  private String email;
  private String subject;
  private String content;
}
```

### 提交任务
可以使用 RqueueMessageSender  bean 提交任务。根据使用场景，可以采用多种方式设置任务，例如，对于重试可以使用方法重试计数，对于延迟任务可以设置延迟。

需要对 RqueueMessageSender 进行 autowire 或使用构造函数注入 bean。

下面展示了如何为测试创建 Controller。 

这里会在30秒内生成发票。为此，在发票队列上提交一个延迟30000（毫秒）的任务。另外，这里会尝试在后台发送一封电子邮件。为此添加两个 GET 方法，sendEmail 和  generateInvoice，当然也可以使用 POST。
```java
@RestController
@Slf4j
public class Controller {
  @Autowired
  private RqueueMessageSender rqueueMessageSender;
  @Value("${email.queue.name}")
  private String emailQueueName;
  @Value("${invoice.queue.name}")
  private String invoiceQueueName;
  @Value("${invoice.queue.delay}")
  private Long invoiceDelay;

  @GetMapping("email")
  public String sendEmail(@RequestParam String email, @RequestParam String subject,
                          @RequestParam String content) {
    log.info("Sending email");
    rqueueMessageSender.put(emailQueueName, new Email(email, subject, content));
    return "Please check your inbox!";
  }

  @GetMapping("invoice")
  public String generateInvoice(@RequestParam String id, @RequestParam String type) {
    log.info("Generate invoice");
    rqueueMessageSender.put(invoiceQueueName, new Invoice(id, type), invoiceDelay);
    return "Invoice would be generated in " + invoiceDelay + " milliseconds";
  }
}
```


在 `application.yml` 加入以下内容(**注: 此处需要自行添加redis连接配置**)：
```yaml
email:
  queue:
    name: email-queue
invoice:
  queue:
    name: invoice-queue
    # 延迟3秒执行
    delay: 3000
```

现在可以运行该程序。程序成功启动后，访问 链接。  
在日志中，可以看到电子邮件任务正在后台执行：

下面是延迟30秒生成发票：
http://localhost:8080/invoice?id=INV-1234&type=PROFORMA

### 总结

可以看到，使用 Rqueue 执行定时任务不需要冗长的模板代码！配置和使用 Rqueue 库时，进行了仔细考虑。要记住：无论是普通任务还是延迟任务，都需要尽快执行。

### 源码地址
* [官方demo](https://github.com/sonus21/rqueue-task-exector)
* [自己敲的](https://gitee.com/fengzxia/springboot_rqueue)