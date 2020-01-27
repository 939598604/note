# Open-falcon Mysql监控

## 一.环境准备
需要已经安装了open-falcon。

没有golang环境的话首先在golang官网下载压缩包，解压后进入文件bin目录，把go执行文件 ln -s 绝对源文件路径/go /usr/local/bin/go 配置到系统bin执行文件

然后配置GOPATH

比如我要在/downloads/下运行这个项目，则设置gopath环境变量
export GOPATH=/downloads/

## 二.部署

### 2.1 安装golang

> Golang官网下载地址：https://golang.org/dl/

####  2.1.1 下载golang

打开官网下载地址选择对应的系统版本, 复制下载链接
这里我选择的是
`go1.10.3.linux-amd64.tar.gz`：https://dl.google.com/go/go1.10.3.linux-amd64.tar.gz

```
wget https://dl.google.com/go/go1.10.3.linux-amd64.tar.gz
```

#### 2.1.2 解压

执行`tar`解压到`/usr/loacl`目录下，得到`go`文件夹

```
tar -C /usr/local -zxvf  go1.10.3.linux-amd64.tar.gz
```

#### 2.1.3 设置环境变量

添加`/usr/loacl/go/bin`目录到PATH变量中。添加到`/etc/profile` 

```
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin
export GOPATH=/downloads/
```

source /etc/profile

### 2.2 安装mymon

#### 2.2.1 下载及编译mymon

```
mkdir -p /downloads
go get github.com/open-falcon/mymon  #成功后目录应为$GOPATH/src/github.com/open-falcon/mymon
go get -u github.com/open-falcon/mymon
cd $GOPATH/src/github.com/open-falcon/mymon
make
```

#### 2.2.2 编辑mymon配置文件

vim /downloads/src/github.com/open-falcon/mymon/etc/mymon.cfg

```
[default]
# 工作目录
basedir = .
# 日志目录，默认日志文件为myMon.log,旧版本有log_file项，如果同时设置了，会优先采用log_file
log_dir = .
# 配置报警忽略的metric项,依然会上报改metric，但原有的该metric项的报警策略将不会生效
ignore_file = ./falconignore
# 保存快照(process, innodb status)的目录
snapshot_dir = ./snapshot
# 保存快照的时间(日)
snapshot_day = 10
# 日志级别[RFC5424]
# 0 Emergency
# 1 Alert
# 2 Critical
# 3 Error
# 4 Warning
# 5 Notice
# 6 Informational
# 7 Debug
log_level  = 5
# falcon agent连接地址 安装的falcon server端地址
falcon_client=http://192.168.177.133:1988/v1/push
# 自定义endpoint
endpoint=mysql

[mysql]
# 数据库用户名
user=root
# 您的数据库密码
password=root
# 数据库连接地址
host=localhost
# 数据库端口
port=3306
```

falcon_client=http://192.168.177.133:1988/v1/push  falcon agent连接地址 安装的falcon server端地址

endpoint=mysql 端点的名称



# 自定义endpoint
endpoint=mysql

#### 2.2.3添加定时推送任务

crontab -e

```
* * * * * cd /downloads/src/github.com/open-falcon/mymon && ./mymon -c etc/myMon.cfg
```

#安装mymon
go get github.com/open-falcon/mymon  #成功后目录应为$GOPATH/src/github.com/open-falcon/mymon
cd $GOPATH/src/github.com/open-falcon/mymon

