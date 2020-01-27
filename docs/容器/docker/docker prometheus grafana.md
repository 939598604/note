# docker prometheus grafana

## 一.docker安装node_exporter

### 1.1 下载镜像

```
docker pull prom/node-exporter
```

### 1.2 启动node-exporter

```
docker run -d -p 9100:9100 \
  -v "/proc:/host/proc:ro" \
  -v "/sys:/host/sys:ro" \
  -v "/:/rootfs:ro" \
  --net="host" \
  prom/node-exporter
```

等待几秒钟，查看端口是否起来了

```
[root@flume3 conf]# netstat -anpt |grep 9100
tcp6       0      0 :::9100         :::*             LISTEN      45666/node_exporter 
```

### 1.3 放开防火墙

```
firewall-cmd --permanent --add-port=9100/tcp
firewall-cmd --reload
```

### 1.4 访问url

```
http://宿主机器:9100/metrics
```

## 二.docker安装prometheus

### 2.1 下载镜像

```
docker pull prom/prometheus
```

### 2.2 启动prometheus

新建目录/docker/prometheus/conf，编辑配置文件prometheus.yml

```
mkdir /docker/prometheus/conf
vim /docker/prometheus/conf/prometheus.yml
```

内容如下：

```
global:
  scrape_interval:     60s
  evaluation_interval: 60s
 
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']
        labels:
          instance: prometheus
 
  - job_name: linux
    static_configs:
      - targets: ['宿主机的ip地址:9100']
        labels:
          instance: localhost
```

注意：修改IP地址，这里的宿主机的ip地址

```
docker run  -d -p 9090:9090 -v /docker/prometheus/conf/prometheus.yml:/etc/prometheus/prometheus.yml   prom/prometheus
```

等待几秒钟，查看端口状态

```
netstat -anpt |grep 9090
tcp6       0      0 :::9090     :::*         LISTEN      3336/docker-proxy
```

## 2.3 放开防火墙

```
firewall-cmd --permanent --add-port=9090/tcp
firewall-cmd --reload
```

### 2.4  访问url

http://宿主机ip:9090/graph

## 三.docker安装grafana

### 3.1 下载镜像

```
docker pull grafana/grafana
```

### 3.2 存储grafana数据

新建空文件夹/docker/grafana，用来存储数据

```
mkdir /docker/grafana
```

设置权限

```
chmod 777 -R /docker/grafana
```

因为grafana用户会在这个目录写入文件，直接设置777，比较简单粗暴！

### 3.3 启动grafana

```
docker run -d \
  -p 3000:3000 \
  --name=grafana \
  -v /docker/grafana:/var/lib/grafana \
  grafana/grafana
```

### 3.4 检查端口

```
netstat -anpt |grep 3000
tcp6       0      0 :::3000        :::*      LISTEN      3494/docker-proxy
```

## 3.5 放开防火墙

```
firewall-cmd --permanent --add-port=3000/tcp
firewall-cmd --reload
```

### 3.6  访问url

http://宿主机ip:3000/graph

默认会先跳转到登录页面，默认的用户名和密码都是admin

登录之后，它会要求你重置密码。你还可以再输次admin密码！