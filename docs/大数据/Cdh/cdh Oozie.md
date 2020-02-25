

# cdh Oozie

## 1.Oozie的安装

### 1.1选择添加服务

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109154513.png)

###1. 2.选择服务

选择需要安装的服务组合 这里选择了自定义里面的Oozie

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110164544.png)



### 1.3 选择依赖

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110164643.png)

### 1.3 自定义 Oozie 的角色分配

选择一台机器，这里选择slave5

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110164842.png)

### 1.4 数据库设置

手动新建数据库，指定数据库的字符集为latin1 ，这里一定要选择字符为latin1，不能选UTF-8,否则会字符长度报错

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110165759.png)

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110165049.png)

### 1.5 指定配置目录

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110165151.png)

### 1.6 运行

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110165203.png)

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110171117.png)

### 1.6 完成安装

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110171154.png)



### 1.7 主页





## 2. oozie 控制台问题

### 1.控制台位置

![1578647595958](C:\Users\Administrator\AppData\Local\Temp\1578647595958.png)

点击web UI

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110171328.png)

报错：**Oozie web console is disabled.** 

### 2.解决方法

原因是缺失ext-2.2.zip

最后终于在Cloudera归档网页找到：<http://archive.cloudera.com/gplextras/misc/ext-2.2.zip>

将ext-2.2.zip复制到安装的oozie server(oozie安装的那台服务器)的/var/lib/oozie目录下

```
mv ext-2.2.zip /var/lib/oozie
unzip ext-2.2.zip
```

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200113095208.png)