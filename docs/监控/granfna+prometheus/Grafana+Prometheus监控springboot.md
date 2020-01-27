---
typora-root-url: ..\文档图片需要上传
---

# Grafana+Prometheus监控springboot

## 一.springboot2与prometheus的集成

### 1.1 pom文件依赖

在项目pom.xml中添加如下依赖

```xml
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>

		<dependency>
			<groupId>io.micrometer</groupId>
			<artifactId>micrometer-registry-prometheus</artifactId>
			<version>1.0.6</version>
		</dependency>
```

### 1.2 application.yml配置

在 application.yml中添加如下配置（因为是测试，所以我把所有端点都暴露了，生产环境自行选择打开端点）

```
management:
    security:
# 仅限于 开发环境可对security进行关闭。
        enabled: false
    metrics:
        export:
            prometheus:
                enabled: true
                step: 1m
                descriptions: true
    web:
        server:
            auto-time-requests: true
    endpoints:
        prometheus:
            id: springmetrics
        web:
            exposure:
                include:         health,info,env,prometheus,metrics,httptrace,threaddump,heapdump,springmetrics
```

### 1.3 prometheus 地址

启动项目，在idea中可以看到接口 /prometheus 

```
2020-01-26 04:50:56.502  INFO 12644 --- [           main] s.b.a.e.w.s.WebMvcEndpointHandlerMapping : Mapped "{[/actuator/prometheus],methods=[GET],produces=[text/plain;version=0.0.4;charset=utf-8]}" onto public java.lang.Object org.springframework.boot.actuate.endpoint.web.servlet.AbstractWebMvcEndpointHandlerMapping$OperationHandler.handle(javax.servlet.http.HttpServletRequest,java.util.Map<java.lang.String, java.lang.String>)
```

### 1.4 通过浏览器查看prometheus.json如下

http://192.168.177.1:8080/actuator/prometheus

```
# HELP process_start_time_seconds The start time of the Java virtual machine
# TYPE process_start_time_seconds gauge
process_start_time_seconds 1.580025051489E9
# HELP tomcat_sessions_alive_max_seconds  
# TYPE tomcat_sessions_alive_max_seconds gauge
tomcat_sessions_alive_max_seconds 0.0
# HELP jvm_memory_used_bytes The amount of used memory
# TYPE jvm_memory_used_bytes gauge
jvm_memory_used_bytes{area="nonheap",id="Code Cache",} 8816832.0
jvm_memory_used_bytes{area="nonheap",id="Metaspace",} 3.6227984E7
jvm_memory_used_bytes{area="nonheap",id="Compressed Class Space",} 5017352.0
jvm_memory_used_bytes{area="heap",id="PS Eden Space",} 4.3151648E7
jvm_memory_used_bytes{area="heap",id="PS Survivor Space",} 0.0
jvm_memory_used_bytes{area="heap",id="PS Old Gen",} 2.5784008E7
# HELP system_cpu_count The number of processors available to the Java virtual machine
# TYPE system_cpu_count gauge
system_cpu_count 8.0
# HELP tomcat_cache_hit_total  
# TYPE tomcat_cache_hit_total counter
tomcat_cache_hit_total 0.0
# HELP jvm_memory_max_bytes The maximum amount of memory in bytes that can be used for memory management
# TYPE jvm_memory_max_bytes gauge
jvm_memory_max_bytes{area="nonheap",id="Code Cache",} 2.5165824E8
jvm_memory_max_bytes{area="nonheap",id="Metaspace",} -1.0
jvm_memory_max_bytes{area="nonheap",id="Compressed Class Space",} 1.073741824E9
jvm_memory_max_bytes{area="heap",id="PS Eden Space",} 1.369964544E9
jvm_memory_max_bytes{area="heap",id="PS Survivor Space",} 1.7825792E7
```

## 二.部署prometheus

### 1.下载prometheus

下载你想安装的prometheus版本，地址为[download prometheus](https://prometheus.io/download)

我下载的是[prometheus-2.6.0.linux-amd64.tar.gz](https://github.com/prometheus/prometheus/releases/download/v2.6.0/prometheus-2.6.0.linux-amd64.tar.gz) 

解压

```
tar xvfz prometheus-*.tar.gz
cd prometheus-*
```

### 2.配置prometheus.yml文件

在某目录创建 prometheus.yml文件内容如下

```
scrape_configs:
  - job_name: 'springboot'
    metrics_path: /actuator/prometheus #直接输入IP + 端口号/actuator/prometheus访问到
    static_configs:
    - targets: ['192.168.177.133:8080'] #此处填写 Spring Boot 应用的 IP + 端口号
```

注意：metrics_path:和targets的配置。

### 3.启动Prometheus

```
./prometheus --config.file="prometheus.yml" 
```

在Status->Targets页面下，我们可以看到我们配置的Target，它们的State为UP 

![1580026014536](/1580026014536.png)

至此，prometheus和springboot已经连接成功。

## 三.部署Grafana

### 3.1 安装Grafana

使用的是ubuntu 16.04TLS，所以找到官网相对应的Ubuntu方式，这是官网的链接地址：https://grafana.com/grafana/download?platform=linux

```
wget https://dl.grafana.com/oss/release/grafana_5.4.2_amd64.deb 
sudo dpkg -i grafana_5.4.2_amd64.deb 
```

启动grafana
方式一、Start Grafana by running:

```
sudo service grafana-server start
sudo update-rc.d grafana-server defaults //设置开机启动（可选）
```

方式二、To start the service using systemd:

```
systemctl daemon-reload
systemctl start grafana-server
systemctl status grafana-server
sudo systemctl enable grafana-server.service //设置开机启动
```

### 3.2 grafana添加数据源

![1580023177791](D:\workspace\note\docs\监控\文档图片需要上传\1580023177791.png)



### 3.3 创建看板

grafana支持很多种看板，你可以根据不同的指标生成图表

https://grafana.com/dashboards 在这里可以搜索不同的模板

https://grafana.com/grafana/dashboards/6756 这个是springboot的看板

![285763-20181226161310635-107066985](/285763-20181226161310635-107066985.png)

选择一个你想要的点击去，然后复制id

![285763-20181226161334353-567349179](/285763-20181226161334353-567349179.png)

打开下图页面，并将id粘贴进去，光标离开输入框，会自动加载模板配置

![285763-20181226162147485-187669112](/285763-20181226162147485-187669112.png)

接着选datasource：

![285763-20181226162049615-164331456](/285763-20181226162049615-164331456.png)

然后选取数据源，点击import按钮，完成模板添加

![285763-20181226162252012-1305751070](/285763-20181226162252012-1305751070.png)

看数据都是空的，因为这个是springboot2.x的。要换成springboot1.x的，如下图：

![285763-20181226163322737-1399110366](/285763-20181226163322737-1399110366.png)

结果：

![285763-20181226163342009-1148902621](/285763-20181226163342009-1148902621.png)

完成。

### 3.4 多个应用的配置

如下配置两个微服务应用：

```
scrape_configs:
  - job_name: 'service-productor'
    metrics_path: /actuator/prometheus #直接输入IP + 端口号/actuator/prometheus访问到
    static_configs:
    - targets: ['10.200.110.100:8082']
    
  - job_name: 'service-consumer'
    metrics_path: /actuator/prometheus #直接输入IP + 端口号/actuator/prometheus访问到
    static_configs:
    - targets: ['10.200.110.100:8081']
    
```

重启prometheus后，再刷新target页面

![285763-20181227143148591-2024134499](/285763-20181227143148591-2024134499.png)

回到Grafana的页面：

![285763-20181227143111644-235367074](/285763-20181227143111644-235367074.png)