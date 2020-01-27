# centos7 yum 安装 redis

直接yum 安装的redis 不是最新版本

```undefined
yum install redis
```

```cpp
yum install -y http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
```

然后可以使用下面的命令安装最新版本的redis：

```undefined
yum --enablerepo=remi install redis
```

安装完毕后，即可使用下面的命令启动redis服务

修改配置文件/etc/redis.conf

```
bind 0.0.0.0
```

启动

```undefined
service redis start
```

或者

```undefined
systemctl start redis
```

redis安装完毕后，我们来查看下redis安装时创建的相关文件，如下：

```undefined
rpm -qa |grep redis
```