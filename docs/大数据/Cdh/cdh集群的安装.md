# cdh集群的安装

## 1.准备

### 1.1 机器环境

| ip地址          | 域名解析 | 安装角色 |
| --------------- | -------- | -------- |
| 192.168.197.218 | master   | master   |
| 192.168.197.219 | slave1   | slave1   |
| 192.168.197.220 | slave2   | slave2   |
| 192.168.197.224 | slave3   | slave3   |

### 1.2预先需要下载的安装包

**cdh的3个包**

```
CDH-5.16.2-1.cdh5.16.2.p0.8-el7.parcel
CDH-5.16.2-1.cdh5.16.2.p0.8-el7.parcel.sha1
manifest.json
```

以上三个包都可在该链接（https://archive.cloudera.com/cdh5/parcels/5.16.2/ ）下载

**cm管理端**

```
cloudera-manager-centos7-cm5.16.1_x86_64.tar.gz
```

cloudera-manager的安装包，可在此处（https://archive.cloudera.com/cm5/cm/5/ ）下载

**jdk安装包**

```
jdk1.8.0.1_161.tar.gz
```

## 2.关闭防火墙和SELINUX

### 2.1关闭防火墙

```
service firewalld stop
systemctl disable firewalld.service
```

### 2.2 关闭SELINUX

vim /etc/selinux/config 

```
SELINUX=disabled
```

## 3.配置本地SSH免密钥登录

### 3.1生成秘钥文件

在主从节点(集群的每一个节点节点)输入命令 ，生成 key，一律回车

```
ssh-keygen -t rsa -P ''
```

###  3.2收集公钥文件

将从节点(集群的每一个节点节点)公钥收集到一个文件中authorized_keys，并发送到各个节点

**从节点配置：**

```
在slave1的机器：scp /root/.ssh/id_rsa.pub 192.168.197.218:/root/.ssh/id_rsa.pub.s1
在slave2的机器：scp /root/.ssh/id_rsa.pub 192.168.197.218:/root/.ssh/id_rsa.pub.s2
在slave2的机器：scp /root/.ssh/id_rsa.pub 192.168.197.218:/root/.ssh/id_rsa.pub.s3
```

**主节点配置：**
将所有机器的id_rsa.pub文件收集到authorized_keys，

```
cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys
cat /root/.ssh/id_rsa.pub.s1 >> /root/.ssh/authorized_keys
cat /root/.ssh/id_rsa.pub.s2 >> /root/.ssh/authorized_keys
cat /root/.ssh/id_rsa.pub.s3 >> /root/.ssh/authorized_keys
```

主节点将authorized_keys发送到各个从节点的.ssh目录下

```
scp /root/.ssh/authorized_keys 192.168.197.219:/root/.ssh/
scp /root/.ssh/authorized_keys 192.168.197.220:/root/.ssh/
scp /root/.ssh/authorized_keys 192.168.197.224:/root/.ssh/
```

在master验证ssh免密码登录，观察是否需要输入密码

```
ssh 192.168.197.219
ssh 192.168.197.220
ssh 192.168.197.224
```

##  4.脚本编写

### 4.1 同步文件

**在master节点的/usr/local/bin/下新建文件xsync.sh**

vim xsync.sh

```
#!/bin/sh
# 获取输入参数个数，如果没有参数，直接退出
pcount=$#
if((pcount==0)); then
        echo no args...;
        exit;
fi
# 获取文件名称
p1=$1
fname=`basename $p1`
echo fname=$fname
# 获取上级目录到绝对路径
pdir=`cd -P $(dirname $p1); pwd`
echo pdir=$pdir
# 获取当前用户名称
user=`whoami`
# 循环
for((host=1; host<=3; host++)); do
        echo $pdir/$fname $user@slave$host:$pdir
        echo ==================slave$host==================
        rsync -rvl $pdir/$fname $user@slave$host:$pdir
done

```

**赋予执行权限**

```
chmod +x /usr/local/bin/xsync.sh
```

### 4.2 调用脚本

**在master节点的/usr/local/bin/下新建文件xcall.sh**

 vim xcall.sh

```
#!/bin/sh
pcount=$#
if((pcount==0));then
        echo no args...;
        exit;
fi
echo ==================master==================
$@
for((host=1; host<=3; host++)); do
        echo ==================slave$host==================
        ssh slavehost "source /etc/profile;@"
done
```

**赋予执行权限**

```
chmod +x /usr/local/bin/xcall.sh
```

## 5.配置host文件

### 5.1 修改hosts文件

在master节点的/etc/hosts文件加入以下内容

```
192.168.197.218 master
192.168.197.219 slave1
192.168.197.220 slave2
192.168.197.224 slave3
```

### 5.2 复制到各个从节点

```
xsync.sh /etc/hosts
```

## 6.安装JDK

### 6.1.下载jdk

到官网下载jdk 如:  jdk1.8.0.1_161.tar.gz

### 6.2.安装jdk

**上传到/soft/目录下，并解压jdk**

```
tar -zxvf jdk1.8.0.1_161.tar.gz
```

 **配置JDK环境变量**

vim /etc/profile

```
export JAVA_HOME=/soft/jdk1.8.0_161
export CLASSPATH=.:${JAVA_HOME}/jre/lib/rt.jar:${JAVA_HOME}/lib/dt.jar:${JAVA_HOME}/lib/tools.jar
export PATH=$PATH:${JAVA_HOME}/bin
```

source /etc/profile

**验证JDK**

```
java -version
```

## 7.安装MySQL

在master的机器上面安装mysql，在CentOS中默认安装有MariaDB，这个是MySQL的分支，但为了需要，还是要在系统中安装MySQL，而且安装完成之后可以直接覆盖掉MariaDB 

### 7.1下载并安装Repository

```
wget -i -c http://dev.mysql.com/get/mysql57-community-release-el7-10.noarch.rpm
```

### 7.2 安装mysql

使用上面的命令就直接下载了安装用的Yum Repository，大概25KB的样子，然后就可以直接yum安装了。

```
yum -y install mysql57-community-release-el7-10.noarch.rpm
yum -y install mysql-community-server
```

这步可能会花些时间，安装完成后就会覆盖掉之前的mariadb。 

###  7.3 MySQL数据库设置

**启动MySQL** 

```
systemctl start mysqld.service
```

**查看密码**

此时MySQL已经开始正常运行，不过要想进入MySQL还得先找出此时root用户的密码，通过如下命令可以在日志文件中找出密码： 

```
grep "password" /var/log/mysqld.log
```

**修改密码**

命令行进入  mysql -uroot -p   输入初始密码 

```
ALTER USER 'root'@'%' IDENTIFIED BY 'root';
## 新密码设置的时候如果设置的过于简单会报错：
SHOW VARIABLES LIKE 'validate_password%';
set global validate_password_policy=0;
set global validate_password_length=1;
```

mysql的开启启动

```
systemctl enable mysqld.service
```

## 8.安装cloudera-manager

### 8.1所有节点执行 

（1）【所有节点】 安装cdh的依赖包

```
yum -y install chkconfig python bind-utils psmisc libxslt zlib sqlite cyrus-sasl-plain cyrus-sasl-gssapi fuse portmap fuse-libs redhat-1sb
```

（2）【所有节点】 新建cloudera-manager 目录

cloudera-manager是所有节点都需要安装的，必须安装在 /opt/cloudera-manager 目录下，因为 /opt 就是它启动后的默认工作目录。

```
mkdir -p /opt/cloudera-manager
```

（3）【所有节点】  创建cloudera-scm用户

```
useradd --system --home=/opt/cloudera-manager --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm
```

--system 表示系统用户
--home=/opt/cloudera-manager/cm-5.14.2/run/cloudera-scm-server  指定该用户的主目录
--no-create-home  不再创建用户主目录
--shell=/bin/false  不作为登陆用户
--comment "Cloudera SCM User”  注释

**验证**

```
cat /etc/passwd | grep cloudera-scm
```



### 8.2 主节点执行

（1）【主节点】下载包放至/opt目录解压

在主节点上传，然后再将修改好的配置复制到其他节点

```
tar -xf cloudera-manager-centos7-cm5.16.2_x86_64.tar.gz -C /opt/cloudera-manager
```

（2）【主节点】 修改配置文件

指定cm集群的服务端（注:只能写主机名，不能写ip），指定本机，域名为master，其他的为agent节点

vim /opt/cloudera-manager/cm-5.16.2/etc/cloudera-scm-agent/config.ini

```
server_host=master
```

（3）【主节点】复制cloudera-manager到各个从节点

```
xsync /opt/cloudera-manager
```

（4）【主节点】授权mysql用户

```
grant all privileges on *.* to 'root'@'%' identified by 'root' with grant option;
FLUSH PRIVILEGES;
```

（5） 【主节点】拷贝mysql驱动

```
cp /usr/share/java/mysql-connector-java-5.1.47-bin.jar /opt/cm-5.16.2/share/cmf/lib/
```

（6）【主节点】在server节点上执行，初始化数据库脚本

```
/opt/cloudera-manager/cm-5.16.2/share/cmf/schema/scm_prepare_database.sh mysql -h 192.168.197.219 -uroot -proot --scm-host master scm scm scm
```

mysql -h 192.168.197.219指定mysql的主机地址 

-u root 以mysql的用户root访问mysql

-proot mysql的用户root的密码
--scm-host master 指定scm的服务端地址

scm  赋予mysql的用户root的角色为scm

scm 在mysql创建一个scm的数据库

scm 在scm的数据库创建一个scm的表

（7）配置cdh源

【主节点】把parcel包放到/opt/cloudera/parcel-repo下

> 只有server节点是创建目录为 /opt/cloudera/parcel-repo
>
> 其中agent节点是创建目录为 /opt/cloudera/parcels

```
mkdir -p  /opt/cloudera/parcel-repo
```

【主节点】将parcel包及校验码文件移动到server节点上的，同时需要将xxxxxxx.parcel.sha1改成xxxxxxx.parcel.sha 

```
 mv CDH-5.16.2-1.cdh5.16.2.p0.8-el7.parcel* manifest.json /opt/cloudera/parcel-repo/
 mv CDH-5.16.2-1.cdh5.16.2.p0.8-el7.parcel.sha1 CDH-5.16.2-1.cdh5.16.2.p0.8-el7.parcel.sha
```

【主节点】修改parcels目录属主权限为cloudera-scm

```
chown -R cloudera-scm:cloudera-scm /opt/cloudera/parcel-repo/
```

> 在所有agent节点（server节点除外）创建目录  /opt/cloudera/parcels及修改目录属主权限为cloudera-scm
>
> mkdir -p  /opt/cloudera/parcels



### 8.3 所有从节点执行

（1）【所有从节点】创建目录为 /opt/cloudera/parcels

```
mkdir -p /opt/cloudera/parcels
```

（2）修改parcels目录属主权限为cloudera-scm

```
chown -R cloudera-scm:cloudera-scm /opt/cloudera/parcels/
```

### 8.4 启动服务端

（1）启动服务端

```
 /opt/cloudera-manager/cm-5.16.2/etc/init.d/cloudera-scm-server start
 /opt/cloudera-manager/cm-5.16.2/etc/init.d/cloudera-scm-agent start
```

查看日志

```
tail -f /opt/cloudera-manager/cm-5.16.2/log/cloudera-scm-server/cloudera-scm-server.log
```

启动的web端口为 7180 

 http://192.168.197.218:7180  账号：admin  密码：admin（随便设置）

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200108135436.png)

### 8.5启动agent端

```
/opt/cloudera-manager/cm-5.16.2/etc/init.d/cloudera-scm-agent start
```

查看日志

```
tail -f /opt/cloudera-manager/cm-5.16.2/log/cloudera-scm-agent/cloudera-scm-agent.log
```

### 8.6 配置环境变量

```
xcall.sh echo "export PATH=$PATH:/opt/cloudera/parcels/CDH-5.16.2-1.cdh5.16.2.p0.8/bin">> /etc/profile
```

然后再各自的主机上执行

```
source /etc/profile
```

## 9.CDH集群的配置

（1）在登录了控制台之后

  **勾选：是的，我接受最终用户许可条款和条件。**

  **点击继续按钮**

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200108135545.png)



（2）选择免费版本，并继续

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200108135753.png)

点击继续

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200108135835.png)



选择要安装的集群机器

![1578463501677](C:\Users\Administrator\AppData\Local\Temp\1578463501677.png)

 选择存储库：用pracel的安装方式

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200108140711.png)

其中点击更多选项

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200108140807.png)

Parcel 目录 ： server端指定的仓库目录

本地 Parcel 存储库路径：agent端指定的目录

如果仓库目录中找不到文件，将会从远程 Parcel 存储库 URL 下载

点击继续按钮

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200108141304.png)

 各个agent节点将会从server节点下载parcel文件，并抽取各安装包到agent端的本地 Parcel 存储库路径

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200108141612.png)

自动检测主机正确性

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200108141636.png)

检测结果

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200108141725.png)

