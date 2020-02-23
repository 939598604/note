# rancher的安装

先在服务器安装好docker服务

## 一.环境

### 1.1 主机角色

| ip地址          | rancher角色    | 安装节点角色 |      |
| --------------- | -------------- | ------------ | ---- |
| 192.168.177.220 | rancher server |              |      |
| 192.168.177.221 | rancher agent1 |              |      |
| 192.168.177.222 | rancher agent2 |              |      |

### 1.2 docker开启启动

```
systemctl start docker
systemctl enable docker
```

### 1.3 修改主机名称

**k8s集群连接如果主机名称相同，会造成连接不上，所以需要修改主机名称**

使用hostnamectl来重新设置主机名是永久生效的即使是服务器重启也生效。

语法： hostnamectl set-hostname  主机名;

> k8s的主机名称不可以加_
>
> 如 rancher_server 是注册不成功的
>
> 如 rancher_agent1是注册不成功的
>
> 如 rancher_agent2是注册不成功的
>
> 

```
hostnamectl set-hostname rancherserver
hostnamectl set-hostname rancheragent1
hostnamectl set-hostname rancheagent2
```

### 1.4 时间同步

**k8s集群如果服务器时间不同步会造成连接不上**

#### 1.4.1 时间服务器的搭建

（1）三台主机依次执行

```
yum install -y ntp ntpdate
```

（2）选择rancher server作为时间服务器，修改ntp配置文件

修改/etc/ntp.conf 文件，注释掉一下4个选项，添加一个阿里云的时间服务器

```
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst

server ntp3.aliyun.com iburst
```

（3）同时修改配置文件将restrict 127.0.0.1修改为主机网段的网络，内容如下：

```
restrict 192.168.177.0 mask 255.255.255.0 nomodify notrap
```

（4）启动ntpd服务

```
systemctl start ntpd.service
ntpq -p  #查看上级服务
```



#### 1.4.2 时间客户端

其他两台rancher agent是客户端

（1）修改/etc/ntp.conf 文件，注释掉一下4个选项，添加一个rancher server作为时间服务器

```
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst

server 192.168.177.220 prefer
```

（2）启动ntpd服务

```
systemctl start ntpd.service
ntpq -p  #查看上级服务
```



### 1.5 rancher的版本

rancher 有1.x和2.x版本

```
rancher/server为1.x版本
rancher/rancher 为2.x版本
```



## 二.Rancher server安装

### 2.1 配置阿里云镜像加速地址

安装racher server

vim /etc/docker/daemon.json

```
{
	  "registry-mirrors": ["https://fy707np5.mirror.aliyuncs.com"]
}
```

### 2.2 获取rancher镜像

```
docker pull rancher/rancher
```

### 2.3 创建rancher挂载目录

rancher是容器，随着进程挂掉数据会丢失

```
mkdir -p /docker/rancher/rancher_home
mkdir -p /docker/rancher/auditlog
```

### 2.4 启动rancher

```
docker run -d --restart=unless-stopped -p 80:80 -p 443:443 \
-v /docker/rancher/rancher_home:/var/lib/rancher \
-v /docker/rancher/auditlog:/var/log/auditlog \
--name rancher rancher/rancher
```

如果不需要映射

```
docker run -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher
```



### 2.3 登录Rancher

访问rancher的web服务 [https://192.168.177.220](https://192.168.177.220/)

浏览器访问racher 第一次访问需要设置登录密码：

![](https://img-blog.csdnimg.cn/2019022810210696.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjA4MjYzNA==,size_16,color_FFFFFF,t_70)
因为Rancher是自动使用的自签名证书，在第一次登录会提示安全授信问题，信任即可

### 2.4 确认密码和url

设置密码

admin  确认密码:admin

然后设置rancher server的url，我直接使用了虚拟机的内网ip了：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190228102525929.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjA4MjYzNA==,size_16,color_FFFFFF,t_70)

### 2.5 设置语言

完成以上设置后，即可进入到rancher server的主页，在这里我们可以设置语言为中文：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190228102600463.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjA4MjYzNA==,size_16,color_FFFFFF,t_70)

在右下角，可以设置语言为中文：

### 2.6 添加一个集群

选择“集群”->"添加集群"

选择CUSTOM自定义集群，集群的名称可自定义：

配置主机的及角色地址，勾选3个主机角色，然后复制命令到rancer server上执行
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190228102754278.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjA4MjYzNA==,size_16,color_FFFFFF,t_70)

### 2.7 复制命令到rancher server执行

```
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.3.5 --server https://192.168.177.220 --token sbmnphbrzcwc4bxgqrqvh8nbzbp6zsz5m5mpxpj6429rzfqwggkjfl --ca-checksum 215cb9ace095a6fb093540fbae49ab502bf051afcb866d8ee6912cbcb6da5794 --etcd --controlplane --worker
```

与此同时，先不要点击集群的完成按钮。。。。执行勾选controller和worker主机角色到其他agent机器执行完成，并有注册成功的提示再点击完成按钮。否则会注册不到集群中。

## 三.Rancher agent安装

### 3.1添加rancher agent

勾选 control和worker主机角色

复制生成的命令到rancher agent机器上执行。执行过程如下：

```
[root@reg mysql]# sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.3.5 --server https://192.168.177.220 --token 2pms4wqs49nss886w6jt8qtcbzkglnxnp7jcjfqq9fwhkk4z44blmx --ca-checksum b57792efd733e533747964d88da7b24516f1b92d56ebe5ef2c0689a54f631ed2 --controlplane --worker
```

执行结果如下

```
Unable to find image 'rancher/rancher-agent:v2.3.5' locally
v2.3.5: Pulling from rancher/rancher-agent
5c939e3a4d10: Downloading [===============>                                   ]  8.289MB/26.69MB
c63719cdbe7a: Download complete 
19a861ea6baf: Download complete 
651c9d2d6c4f: Download complete 
6d7564564095: Download complete 
2ac07de5e7c6: Downloading [=============>                                     ]  8.884MB/32.88MB
fb3555e7bfa2: Download complete 
f1f0c5164382: Download complete 
46dcc1618a26: Downloading [=========>                                         ]  4.814MB/25.24MB
450e4e38bf51: Waiting 
```

**注册成功**

执行成功后，页面下方会显示新主机注册成功!!

此时我们的集群处于等待注册的状态，点击主机下的数字可以查看主机信息：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190228102946487.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjA4MjYzNA==,size_16,color_FFFFFF,t_70)

**拉取镜像**

等待了一个多小时，一直在提示这个

>当前集群**Provisioning**中...，在API准备就绪之前，直接与API交互的功能将不可用。
>
>Pre-pulling kubernetes images
>
>且主机列表中集群各个主机的状态为“Registering”

![](https://raw.githubusercontent.com/939598604/blog/static/_dataQQ%E6%88%AA%E5%9B%BE20200222221453.png)

主机信息如下，这会主机处于注册中的状态：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190228103005931.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjA4MjYzNA==,size_16,color_FFFFFF,t_70)
经过一段等待后，主机注册成功，转换为可用状态：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190228103026701.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjA4MjYzNA==,size_16,color_FFFFFF,t_70)
点击集群也能够看到集群的仪表盘信息了：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190228103046622.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjA4MjYzNA==,size_16,color_FFFFFF,t_70)
集群创建完成后，默认会生成Default项目，点击Default可以切换到项目视图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019022810311134.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjA4MjYzNA==,size_16,color_FFFFFF,t_70)

## 四.部署工作负载

在rancher里工作负载是一个对象，包括pod以及部署应用程序所需的其他文件和信息。我们这里以nginx作为demo示例，在Default视图下，点击工作负载—部署服务

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190228103214968.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjA4MjYzNA==,size_16,color_FFFFFF,t_70)
在部署工作负载页面，设置工作负载名称、副本数量、镜像名称、命名空间、端口映射，其他参数保持默认，不设置端口映射的话，默认是随机映射端口，我这里选择随机，最后点击启动：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190228103245659.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjA4MjYzNA==,size_16,color_FFFFFF,t_70)
部署完成：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190228103309808.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjA4MjYzNA==,size_16,color_FFFFFF,t_70)
此时我们可以通过随机映射的31671端口去访问nginx服务，能访问成功代表我们部署的没有问题：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190228103341827.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjA4MjYzNA==,size_16,color_FFFFFF,t_70)



```
192.168.197.224.225.226.228 (228用于交投，其他用于测试)
root：Can#129!
```