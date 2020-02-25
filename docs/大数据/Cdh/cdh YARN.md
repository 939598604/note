# cdh YARN

## 1.YARN安装

### 1.1添加服务

****![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109154513.png)

###  1.2 找到yarn服务

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109154635.png)

###  1.3 选择角色

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109154922.png)

一般情况下，只要有dataNode节点的都部署一个NodeManager

###  1.4 设置目录

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109155052.png)



###  1.5 部署服务

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109155156.png)

###  1.6 完成安装

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109155307.png)



##2.YARN的使用

###  2.1.cm安装的软件目录位置

 cd /opt/cloudera/parcels/CDH-5.16.2-1.cdh5.16.2.p0.8/

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109155556.png)



### 2.2 使用Yarn执行单词计数

**（1）新建文件/root/wc.txt**

```
spark hadoop storm
spark sqoop yarn
hadoop storm
spark hadoop flink
storm hbase test
hive spark hadoop storm
```

**（2）上传至hdfs**

```
hdfs dfs -put /root/wc.txt /user
```

**（3）使用yarn计数**

```
yarn jar /opt/cloudera/parcels/CDH/jars/hadoop-examples-2.6.0-mr1-cdh5.16.2.jar wordcount /user/wc.txt /user/output1
```

**（4）执行控制台可以看到reduce task的数量**

    Job Counters 
           Launched map tasks=1
           Launched reduce tasks=16
从上面看到reduce tasks=16，需要优化

**（5）yarn控制台可以看到正在运行的任务**

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109160427.png)

**（6）从控制台看到yarn的1988端口，查看reduce tasks任务数量**

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109162144.png)



![1578558191773](C:\Users\Administrator\AppData\Local\Temp\1578558191773.png)



![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109162350.png)



## 3 修改YARN的reduce tasks数量

cdh 在使用时如果修改了配置文件,需要重启过时服务,而不是重启,重启过时服务才会修改配置文件

修改步骤如下：（直接搜索配置 hue_safety_valve.ini）

进入CDH Manager的“YARN” -> “配置”页面，查询内容：mapreduce.job.reduces 



![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109163033.png)

将默认数量修改为1

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109163139.png)

然后刷新一下浏览器，会出现同步配置的图标，点击图标

![1578558729152](C:\Users\Administrator\AppData\Local\Temp\1578558729152.png)

可以看到数量有16变为1

![1578558782456](C:\Users\Administrator\AppData\Local\Temp\1578558782456.png)

点击部署客户端配置按钮