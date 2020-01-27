# grafana 安装

## 一.windows 版本

###  1.1下载

https://grafana.com/grafana/download?platform=windows

下载的.zip文件解压到您的想运行Grafana的任何地方，然后进入`conf`目录复制一份`sample.ini`并重命名为`custom.ini`。以后所有的配置应该编辑`custom.ini`，永远不要去修改`defaults.ini`。



如果不想用localhost，可以**自定义ip**，如图：

![img](https://img-blog.csdnimg.cn/2019080214223772.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDYwNjIxNw==,size_16,color_FFFFFF,t_70)



### 1.2 运行Grafana

执行`bin`目录下的`grafana-server.exe`程序，然后打开浏览器访问

### 1.3 访问Grafana

http://localhost:3000

Grafana的默认登录名和密码`admin`/`admin`，第一次登录会提示修改密码。


## 二.linux 版本

### 2.1 官网安装文档

https://grafana.com/docs/grafana/latest/installation/rpm/

### 2.2 新建仓库

vim /etc/yum.repos.d/grafana.repo

```
[grafana]
name=grafana
baseurl=https://packages.grafana.com/enterprise/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
```

### 2.3 执行安装

```
 yum clean all
 yum install grafana-enterprise -y
```

### 2.4 启动grafana

```
systemctl start grafana-server
systemctl status grafana-server
systemctl stop grafana-server
systemctl enable grafana-server.service
```

### 2.5 登陆

http://localhost:3000

