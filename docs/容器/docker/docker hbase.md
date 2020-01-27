# docker 安装单机版hbase

## 1.拉取镜像

```
docker pull harisekhon/hbase
```

## 2.启动脚本

```
docker run -d -h docker-hbase \
        -p 2181:2181 \
        -p 8080:8080 \
        -p 8085:8085 \
        -p 9090:9090 \
        -p 9000:9000 \
        -p 9095:9095 \
        -p 16000:16000 \
        -p 16010:16010 \
        -p 16201:16201 \
        -p 16301:16301 \
        -p 16020:16020 \
        -p 50070:50070 \
        --name hbase \
        harisekhon/hbase
```

-d 表示后台运行
-h 该容器的host为docker-hbase
-p 宿主机端口:容器端口
--name 该容器的名字

```
# Stargate 8080 / 8085
# Thrift 9090 / 9095
# HMaster 16000 / 16010
# RS 16201 / 16301
EXPOSE 2181 8080 8085 9090 9095 16000 16010 16201 16301
```

## 3.进入容器

执行`docker exec -ti e60a300f7749 /bin/bash`
然后再执行`hbase shell`得到如下

```
bash-4.4# hbase shell
hbase(main):001:0>

```

```
查看集群状态status
查看hbase版本version
查看所有表list
创建表 create 't1','[Column Family 1]','[Column Family 2]'
如创建个菜谱表，调味料为一个列族(类)，食材则为另一个列族(类)，脚本为create 'cookbook','seasoning','ngredients'

查看表的结构  describe 't1'
删除表 disable 't1' 之后 drop 't1'
检查表是否存在  exists 't1'
获取表中所有数据  scan 't1'
-获取表中前10行数据 scan 't1',{LIMIT=>10}
获取指定column(ad:pv)的前10行数据scan 't1',{COLUMNS=>['ad:pv'],LIMIT=>10}
加Filter：如过滤rowkey以“2015”开头的前10行数据 scan 't1',{FILTER=>"PrefixFilter('2015')",LIMIT=>10}
查询rowkey=001下所有的数据get 't1','001'
查询rowkey=001下family=ad,column=pvget 't1','001','ad:pv' 或者 get 't1','001',{COLUMN=>'ad:pv'}
添加数据put <table>,<rowkey>,<family:column>,<value>,<timestamp>
例如：向’t1’表中添加rowkey=002,family=ad,column=pv,value=1000,时间戳默认
put 't1','002','ad:pv','1000'

删除rowkey=002的所有数据deleteall 't1','002'
删除rowkey=002中ad:pv数据delete 't1','002','ad:pv'
清空表truncate 't1'
统计行数count 't1'
查询表t1中的行数，每100条显示一次count 't1',{INTERVAL=>100}
```

