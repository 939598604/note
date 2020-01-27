# grafana+Prometheus监控redis

## 一. redis_exporter 安装

### 1.1 export部署

下载：https://github.com/kbudde/redis_exporter/releases

```
wget https://github.com/oliver006/redis_exporter/releases/download/v0.21.2/redis_exporter-v0.21.2.linux-amd64.tar.gz
```

### 2.解压

```
tar xf redis_exporter-v0.21.2.linux-amd64.tar.gz -C /usr/local/redis_exporter/
cd /usr/local/redis_exporter
```

### 3. 启动

```
./redis_exporter redis//127.0.0.1:6379 &
```

### 4.查看进程

```
 netstat -anop|grep 9121
 tcp6       0      0 :::9121     :::*     LISTEN      1779/./redis_exp  off (0.00/0/0)
```



## 二.prometheus配置

### 2.1 prometheus.yml 

```
  - job_name: 'redis'
    static_configs:
      - targets: ['197.168.177.133:9121']
        labels:
          instance: redis-197.168.177.133
```

## 三.在grafana中加入监控

### 3.1 查找prometheus-redis的模板

https://grafana.com/grafana/dashboards?page=1&search=redis

![1580106325721](D:\workspace\note\docs\监控\文档图片需要上传\1580106325721.png)

模板id 2751,在面板中导入模板

![1580106375911](D:\workspace\note\docs\监控\文档图片需要上传\1580106375911.png)



![1580106415166](D:\workspace\note\docs\监控\文档图片需要上传\1580106415166.png)

选择正确的数据源

![1580106437167](D:\workspace\note\docs\监控\文档图片需要上传\1580106437167.png)

主面板如下

![1580106467469](D:\workspace\note\docs\监控\文档图片需要上传\1580106467469.png)