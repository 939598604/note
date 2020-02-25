

# cdh Hbase

## 1.Hbase的安装

### 1.1选择添加服务

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109154513.png)

###1. 2.选择服务

选择需要安装的服务组合 这里选择了自定义里面的Hbase

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110163026.png)

### 1.3 自定义角色分配

 Master  : hbase主节点 

 HBase REST Server ：运行客户端以rest的方式访问hbase

 HBase Thrift Server :hue 通过这个服务去访问hbase

 RegionServer  : hbase数据节点

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110162907.png)

### 1.4 设置HDFS 根目录

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110163119.png)

### 1.5 运行

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110163209.png)

### 1.6 重启服务

因为hbase安装，需要依赖hdfs，因此需要重启HDFS服务，如果重启HDFS服务的话，所有的服务都需要重启

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110163612.png)

**重启过时服务**

****![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110163634.png)

**点击立即重启**

****![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110163653.png)

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110163758.png)

完成重启

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110164009.png)



### 1.7 主页

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110164229.png)