# Prometheus 安装

## 一.windows 版本

### 1.1 下载prometheus

https://prometheus.io/download/

本地是64位系统，所以选择了[prometheus-2.15.2.windows-amd64.tar.gz](https://github.com/prometheus/prometheus/releases/download/v2.15.2/prometheus-2.15.2.windows-amd64.tar.gz)。下载完毕将压缩包解压，执行prometheus.exe，然后prometheus服务就跑起来了



## 二.linux 版本

### 2.1 下载

```
wget https://github.com/prometheus/prometheus/releases/download/v2.12.0/prometheus-2.12.0.linux-amd64.tar.gz
```

### 2.2 解压

```
tar -zxvf prometheus-2.12.0.linux-amd64.tar.gz
```

### 2.3 配置环境变量

设置prometheus的环境变量

vi /etc/profile

```
PATH=$PATH
PATH=$PATH:/root/prometheus/server/prometheus-2.12.0.linux-amd64
export PATH
```

source /etc/profile

### 2.4 启动promethus

```
./promethus 
```

