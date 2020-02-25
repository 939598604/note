#  cdh Flume

## 1.Flume的安装

### 1.1添加服务

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109154513.png)

### 1.2 选择Flume

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110133727.png)

### 1.3 自定义 Flume 的角色分配

gateway是指Flume客户端，可以安装到所有的机器

![1578634703999](C:\Users\Administrator\AppData\Local\Temp\1578634703999.png)

###  1.4 运行服务

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110133844.png)

### 1.6 完成安装

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110134008.png)

### 1.7 主页面

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110134540.png)

### 2 Flume 使用

```
Flume list-databases --connect jdbc:mysql://192.168.197.219:3306 --username root --password root 
```

问题1：执行sgoop命令的服务器上缺少驱动包

```
scp mysql-connector-java.jar 所有的机器:/usr/share/java
```

问题2：连接数据库时所使用的用户是否有访问权限

```
grant all privileges on.to'root'@'%' identified by'root'；
flush privileges；
```

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110134503.png)