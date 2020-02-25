

# cdh HDFS

## 1.HDFS的安装

### 1.1选择添加服务

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109154513.png)

###1. 2.选择服务

选择需要安装的服务组合 这里选择了自定义里面的HDFS

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200108142547.png)

### 1.3 自定义角色分配

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200108142432.png)



### 1.4 配置mysql账号密码

![1578470148806](C:\Users\Administrator\AppData\Local\Temp\1578470148806.png)

### 1.5 配置

继续，使用默认配置

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200108155747.png)



![1578470394615](C:\Users\Administrator\AppData\Local\Temp\1578470394615.png)



![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200108160131.png)



## 2.问题解决

### 2.1 执行上传命令的时候报错

```
hdfs dfs -mkdir -p /user/test
```

无论是用sudo hadoop dfs -mkdir 建立文件 还是 put文件，都会显示

```
Permission denied: user=root,access=WRITE, inode="/user"
```

**如何解决方法1**
/user这是文件的所有者是HDFS  权限为755  也就是只有HDFS才能对这个文件进行sudo的操作
 用指定的用户去执行命令，执行命令如下

```
 sudo -u hdfs hadoop fs -mkdir /user/test
```

**如何解决方法2**

在执行窗口设置临时变量，如果想永久设置，可以放在/etc/profile中

```
export HADOOP_USER_NAME=hdfs
```

**如何解决方法3**

修改配置文件，找到dfs.permissions选项，把勾选去掉，并保存



![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109152339.png)

改完之后要部署到所有客户端

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109152629.png)







### 2.2.浏览器打开hdfs界面告警

Path does not exist on HDFS or WebHDFS is disabled. Please check your path 

访问namenode的hdfs使用50070端口，访问datanode的webhdfs使用50075端口。访问文件、文件夹信息使用

namenode的IP和50070端口，访问文件内容或者进行打开、上传、修改、下载等操作使用datanode的IP和50075端口。

​    要想不区分端口，直接使用namenode的IP和端口进行所有的webhdfs操作，就需要在所有的datanode上都设置hefs-site.xml中的dfs.webhdfs.enabled为true。   

 ![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109145131.png)

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109145605.png)![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109145635.png)



## 3.HDFS HA

### 3.1 安装zookeeper

#### 3.1.1 添加服务

![1578559405291](C:\Users\Administrator\AppData\Local\Temp\1578559405291.png)

#### 3.1.2 选择角色

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109164423.png)

#### 3.1.3 配置修改

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109164516.png)

#### 3.1.4 部署服务

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109164654.png)

#### 3.1.5 完成

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200109164714.png)

#### 3.1.6 主页面

![1578559729916](C:\Users\Administrator\AppData\Local\Temp\1578559729916.png)

####  3.1.7 添加zookeeper示例

发现只有一台zookeeper,现在多加2台

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110110232.png)

#### 3.1.8 选择主机

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110110339.png)

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110110407.png)

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110110428.png)

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110110452.png)

然后点击完成即可

重启示例

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110110546.png)

### 3.2 HDFS HA

#### 3.2.1 启用

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110111241.png)

#### 3.2.2 填写名称

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110111310.png)![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110111715.png)

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110111805.png)



#### 3.2.3 指定JournalNode 目录

指定journalNode的目录为/dfs/jn

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110112030.png)

#### 3.2.4 运行部署

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110112205.png)

其中有个步骤报错

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110112503.png)

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110112535.png)

然后点击继续，进入完成界面

### 4.HDFS添加dataNode节点

#### 4.1添加角色示例

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110112749.png)

#### 4.2 添加角色实例到 HDFS

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110112828.png)

#### 4.3 选择DataNode的主机

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110113349.png)

#### 4.4 配置目录

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110112951.png)

#### 4.5 dataNode节点重启服务

选择刚才指定的主机节点进行服务重启

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110113055.png)

#### 4.6 重启服务

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110113155.png)

#### 4.6 完成重启

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200110113238.png)