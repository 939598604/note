# docker 安装单机hadoop

1.下载hadoop镜像

```
docker pull sequenceiq/hadoop-docker:2.7.0
```

2.查看镜像

```
docker images
```

3.运行hadoop

```
docker run -i -t -p 50070:50070 -p 9000:9000 -p 8088:8088 -p 8040:8040 -p 8042:8042  -p 49707:49707  -p 50010:50010  -p 50075:50075  -p 50090:50090 --name hadoop  sequenceiq/hadoop-docker:2.7.0 /etc/bootstrap.sh -bash
```

4.测试是否安装成功
先进入hadoop容器

```
docker exec -it hadoop /bin/bash
```

5.执行完成docker run 也就是上一步,该步骤可以省略

```
cd /usr/local/hadoop-2.7.0
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.0.jar grep input output 'dfs[a-z.]+'
```

6.如果执行mapreduce程序说明安装成功，可以打开浏览器查看
http://宿主机IP:50070
7.为了正常使用还需安装以下东西

vi /etc/profile

```
export HADOOP_HOME="/usr/local/hadoop-2.7.0"
export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
```

使得配置生效  source /etc/profile
8.查看命令行是否能用

```
hadoop version
```

9.测试jar文件在hadoop中启动
上传一个jar到宿主机
我用hadoop-mapreduce-examples-2.7.0.jar(这个可以自己在网上下一个)
拷贝jar文件到容器
docker cp 要拷贝的文件路径 容器名：要拷贝到容器里面对应的路径

```
docker cp /root/hadoop-mapreduce-examples-2.7.0.jar b7d7f88574fb:/usr/local/hadoop-2.7.0
```

10.查看是否成功

```
docker exec -it b7d7f88574fb /bin/bash
cd /usr/local/hadoop-2.7.0
ls
```

上传一个 a.txt文件到hdfs

内容如下

```
canglaoshi is mylove
xiaoze is mylove
wutenglan is mylove
```

hadoop创建文件夹

```
hadoop fs -mkdir -p /wordcount/input
hadoop fs -put a.txt /wordcount/input
hadoop jar hadoop-mapreduce-examples-2.7.0.jar wordcount /wordcount/input /wordcount/output
```

查看输出内容

```
hadoop fs -cat /wordcount/output/part-r-00000
```

