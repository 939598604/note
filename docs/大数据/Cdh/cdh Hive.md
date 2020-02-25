# cdh Hive

## 1.hive的安装

### 1.1添加服务

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109154513.png)

### 1.2 选择hive

![1578627552342](C:\Users\Administrator\AppData\Local\Temp\1578627552342.png)

### 1.3 自定义 Hive 的角色分配

gateway是指hive客户端，可以安装到所有的机器

metastore 指定一台机器 用来连接mysql数据库

 HiveServer2 × 1 指定一台机器

 WebHCat Server 可以不安装

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110114522.png)

###  1.4 数据库设置

数据库主机名称: 要指定正确的地址

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110121642.png) 

需要手动创建数据库

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110114643.png)

### 1.5 指定Hive 仓库目录

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110121756.png)

### 1.6 运行服务

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110121830.png)

执行过程中会报错

hive报错：Failed to Create Hive Metastore Database Tables.

解决方法

在master执行以下操作

```
批量新建目录
xcall.sh mkdir -p /usr/share/java/
上传mysql驱动mysql-connector-java-5.1.46到/usr/share/java/
修改驱动名称
mv mysql-connector-java-5.1.46.jar mysql-connector-java.jar
分发到各个机器
xsync.sh /usr/share/java/
查看分发结果
xcall.sh ls -l /usr/share/java
```

### 1.6 完成安装

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110122841.png)

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110122858.png)

### 1.7 主页面

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110122947.png)

## 2.修改hive的配置

将以下配置添加到hive中

```
<property>
    <name>hive.cli.print.header</name>
    <value>true</value>
</property>
<prOperty>
    <name>hive.cli.print.current.db</name>
    <value>true</value>
</property>
```

### 2.1 选择配置

选择配置->点击Gateway

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110124007.png)

### 2.2 找到hive-site.xml

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110124036.png)

### 2.3 添加配置

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110124220.png)

### 2.4 更新配置到各个机器

刷新一下浏览器，看到有个配置图标

![1578631385508](C:\Users\Administrator\AppData\Local\Temp\1578631385508.png)

点击配置图标进行同步配置文件

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110124355.png)

## 3 .hive使用

进入到安装了gateway的机器中执行hive命令

```
[root@data-2 ~]# hive
WARNING: Hive CLI is deprecated and migration to Beeline is recommended.
```

创建表

```
hive (default)> create table person(id int);
hive (default)> show tables;
```

查询表

```
hive (default)> select * from  person;
```






