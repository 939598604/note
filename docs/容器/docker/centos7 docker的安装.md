## Centos7下安装Docker（详细的新手装逼教程）

我是虚拟机装的Centos7，linux 3.10 内核，docker官方说至少3.8以上，建议3.10以上（ubuntu下要linux内核3.8以上， RHEL/Centos 的内核修补过， centos6.5的版本就可以——这个可以试试）

1.root账户登录，查看内核版本如下

```
[root@localhost ~]# uname -a
Linux localhost.qgc 3.10.0-862.11.6.el7.x86_64 #1 SMP Tue Aug 14 21:49:04 UTC 2018 x86_64 x86_64 x86_64 GNU/Linuxa
```

2.把yum包更新到最新

```shell
[root@localhost ~]# yum update
已加载插件：fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: centos.ustc.edu.cn
 * extras: mirrors.aliyun.com
 * updates: centos.ustc.edu.cn
正在解决依赖关系
--> 正在检查事务
---> 软件包 bind-libs.x86_64.32.9.9.4-61.el7 将被 升级
---> 软件包 bind-libs.x86_64.32.9.9.4-61.el7_5.1 将被 更新
---> 软件包 bind-libs-lite.x86_64.32.9.9.4-61.el7 将被 升级
---> 软件包 bind-libs-lite.x86_64.32.9.9.4-61.el7_5.1 将被 更新
---> 软件包 bind-license.noarch.32.9.9.4-61.el7 将被 升级
---> 软件包 bind-license.noarch.32.9.9.4-61.el7_5.1 将被 更新......
```

3.安装需要的软件包， yum-util 提供yum-config-manager功能，另外两个是devicemapper驱动依赖的

```
[root@localhost ~]# yum install -y yum-utils device-mapper-persistent-data lvm2
已加载插件：fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: centos.ustc.edu.cn
 * extras: mirrors.aliyun.com
 * updates: centos.ustc.edu.cn
...
```

4.设置yum源

```
[root@localhost ~]# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
已加载插件：fastestmirror, langpacks
adding repo from: https://download.docker.com/linux/centos/docker-ce.repo
grabbing file https://download.docker.com/linux/centos/docker-ce.repo to /etc/yum.repos.d/docker-ce.repo
repo saved to /etc/yum.repos.d/docker-ce.repo
```

5，可以查看所有仓库中所有docker版本，并选择特定版本安装

```
[root@localhost ~]# yum list docker-ce --showduplicates | sort -r
已加载插件：fastestmirror, langpacks
可安装的软件包
 * updates: centos.ustc.edu.cn
Loading mirror speeds from cached hostfile
 * extras: mirrors.aliyun.com
docker-ce.x86_64            18.06.1.ce-3.el7                    docker-ce-stable
docker-ce.x86_64            18.06.0.ce-3.el7                    docker-ce-stable
docker-ce.x86_64            18.03.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            18.03.0.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.12.1.ce-1.el7.centos             docker-ce-stable
docker-ce.x86_64            17.12.0.ce-1.el7.centos             docker-ce-stable
...
```

6，安装Docker，命令：yum install docker-ce-版本号，我选的是17.12.1.ce，如下

```
[root@localhost ~]# yum install docker-ce-17.12.1.ce
已加载插件：fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: centos.ustc.edu.cn
 * extras: mirrors.aliyun.com
 * updates: centos.ustc.edu.cn
base                                                   | 3.6 kB     00:00     
docker-ce-stable                                       | 2.9 kB     00:00     
extras                                                 | 3.4 kB     00:00     
updates                                                | 3.4 kB     00:00     
正在解决依赖关系
--> 正在检查事务
---> 软件包 docker-ce.x86_64.0.17.12.1.ce-1.el7.centos 将被 安装
--> 正在处理依赖关系 container-selinux >= 2.9，它被软件包 docker-ce-17.12.1.ce-1.el7.centos.x86_64 需要...
```

7， 启动Docker，命令：systemctl start docker，然后加入开机启动，如下

```
[root@localhost ~]# systemctl start docker
[root@localhost ~]# systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
```

8，验证安装是否成功(有client和service两部分表示docker安装启动都成功了)

```
[root@localhost ~]# docker version 
Client:
 Version:    17.12.1-ce
 API version:    1.35
 Go version:    go1.9.4
 Git commit:    7390fc6
 Built:    Tue Feb 27 22:15:20 2018
 OS/Arch:    linux/amd64
Server:
 Engine:
  Version:    17.12.1-ce
  API version:    1.35 (minimum version 1.12)
  Go version:    go1.9.4
  Git commit:    7390fc6
  Built:    Tue Feb 27 22:17:54 2018
  OS/Arch:    linux/amd64
  Experimental:    false
```

docker ps 查看当前正在运行的容器

docker ps -a 查看所有容器的状态

docker start/stop id/name 启动/停止某个容器

docker attach id 进入某个容器(使用exit退出后容器也跟着停止运行)

docker exec -ti id 启动一个伪终端以交互式的方式进入某个容器（使用exit退出后容器不停止运行）

docker images 查看本地镜像
docker rm id/name 删除某个容器
docker rmi id/name 删除某个镜像

docker run --name test -ti ubuntu /bin/bash  复制ubuntu容器并且重命名为test且运行，然后以伪终端交互式方式进入容器，运行bash

docker build -t soar/centos:7.1 .  通过当前目录下的Dockerfile创建一个名为soar/centos:7.1的镜像

docker run -d -p 2222:22 --name test soar/centos:7.1  以镜像soar/centos:7.1创建名为test的容器，并以后台模式运行，并做端口映射到宿主机2222端口，P参数重启容器宿主机端口会发生改变