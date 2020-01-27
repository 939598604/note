# grafana+Prometheus监控rabbitmq

## 一. rabbitMq 监控端

### 1.1 export部署

下载：https://github.com/kbudde/rabbitmq_exporter/releases

```
wget https://github.com/kbudde/rabbitmq_exporter/releases/download/v0.29.0/rabbitmq_exporter-0.29.0.linux-amd64.tar.gz
```

### 2.解压

```
tar -xvf rabbitmq_exporter-0.29.0.linux-amd64.tar.gz
cd rabbitmq_exporter-0.29.0.linux-amd64

```

### 3. 启动

```
RABBIT_USER=guest RABBIT_PASSWORD=guest OUTPUT_FORMAT=JSON PUBLISH_PORT=9099 RABBIT_URL=http://localhost:15672 nohup ./rabbitmq_exporter &
```

### 4.查看进程

```
 netstat -anop|grep 9099
 tcp6       0      0 :::9099     :::*     LISTEN      1779/./rabbitmq_exp  off (0.00/0/0)
```

9099

## 二.prometheus配置

### 2.1 prometheus.yml 

```
  - job_name: 'RabbitMQ'
    static_configs:
      - targets: ['197.168.177.133:9099']
        labels:
          instance: RabbitMQ-197.168.177.133
```

