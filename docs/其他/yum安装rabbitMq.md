# CentOS yum 安装RabbitMQ

最近在做机器学习的任务系统，任务模块使用了消息对联，比较快速的搭建方法：

### 1.安装erlang

下载rpm仓库：wget http://packages.erlang-solutions.com/erlang-solutions-1.0-1.noarch.rpm

安装rpm仓库
rpm -Uvh erlang-solutions-1.0-1.noarch.rpm

安装erlang
yum -y install erlang

### 2.安装RabbitMQ

下载RabbitMQ的rpm：wget http://www.rabbitmq.com/releases/rabbitmq-server/v3.6.6/rabbitmq-server-3.6.6-1.el6.noarch.rpm

 

yum -y install rabbitmq-server-3.6.6-1.el6.noarch.rpm

 

注：

 

如果报：Requires: socat

 

更新源wget –no-cache http://www.convirture.com/repos/definitions/rhel/6.x/convirt.repo -O /etc/yum.repos.d/convirt.repo

yum install socat

启动rabbitmq服务:   

前台运行：rabbitmq-server start (用户关闭连接后,自动结束进程)  

后台运行：service rabbitmq-server start

 

### 3.安装插件

 

启动web管理界面

rabbitmq-plugins enable rabbitmq-management

 

如果提示找不到，使用查看插件名称

rabbitmq-plugins list

 

增加访问用户，默认用户guest只能本地访问。

rabbitmqctl add_user admin passwd

 

设置角色：

 

rabbitmqctl set_user_tags admin administrator

 

设置默认vhost（"/"）访问权限

rabbitmqctl set_permissions -p "/" admin "." "." ".*"

 

浏览器访问：http://IP:15672

 

用户名admin，密码passwd进行登录

 

 

最好登录console的时候，删除默认账户guest