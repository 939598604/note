## Centos7下安装Docker（详细的新手装逼教程）

我是虚拟机装的Centos7，linux 3.10 内核，docker官方说至少3.8以上，建议3.10以上（ubuntu下要linux内核3.8以上， RHEL/Centos 的内核修补过， centos6.5的版本就可以——这个可以试试）

### 1.root账户登录，查看内核版本如下

```
[root@localhost ~]# uname -a
Linux localhost.qgc 3.10.0-862.11.6.el7.x86_64 #1 SMP Tue Aug 14 21:49:04 UTC 2018 x86_64 x86_64 x86_64 GNU/Linuxa
```

### 2.把yum包更新到最新

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

### 3.安装需要的软件包

 yum-util 提供yum-config-manager功能，另外两个是devicemapper驱动依赖的

```
[root@localhost ~]# yum install -y yum-utils device-mapper-persistent-data lvm2
已加载插件：fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: centos.ustc.edu.cn
 * extras: mirrors.aliyun.com
 * updates: centos.ustc.edu.cn
...
```

### 4.设置yum源

```
[root@~]# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

### 5.查看docker版本

可以查看所有仓库中所有docker版本，并选择特定版本安装

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

### 6.安装Docker

命令：yum install docker-ce-版本号，我选的是17.12.1.ce，如下

```
[root@localhost ~]# yum install docker-ce-17.12.1.ce -y
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

### 7.添加docker加速器

修改 /etc/docker/daemon.json内容为

```
{
   "registry-mirrors": ["https://fy707np5.mirror.aliyuncs.com"],
   "exec-opts":["native.cgroupdriver=systemd"],
    "insecure-registries":["192.168.197.224"]
}
```

或者

```
 {"registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]}
```



### 8. 启动Docker

命令：systemctl start docker，然后加入开机启动，如下

```
[root@localhost ~]# systemctl start docker
[root@localhost ~]# systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
```

### 9.docker的出现的问题

#### 9.1  Job for docker.service failed because the control process exited with error code. See "systemctl status docker.service" and "journalctl -xe" for details.

**问题如何出现**

```
[root@localhost ~]# systemctl start docker
Job for docker.service failed because the control process exited with error code. See "systemctl status docker.service" and "journalctl -xe" for details.
[root@localhost ~]# docker ps
Cannot connect to the Docker daemon at tcp://0.0.0.0:2375. Is the docker daemon running
```

**解决方法**

卸载Docker,对于旧版本没安装成功,卸掉。

（1）查询安装过的包

```
yum list installed | grep docker
本机安装过旧版本
docker.x86_64,docker-client.x86_64,docker-common.x86_64 
```

（2）删除安装的软件包

```
yum -y remove docker.x86_64                        
yum -y remove docker-client.x86_64                  
yum -y remove docker-common.x86_64
```

（3）确保升级之后的内核是3.1.0或3.1.0以上，查看内核

```
uname -r 
```

（4）修改 /etc/docker/daemon.json内容为：

```
{"registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]}
```

####  9.2  docker: Cannot connect to the Docker daemon at tcp://0.0.0.0:2375. Is the docker daemon running?.

**问题如何出现**

```
[root@localhost ~]# docker run -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher
 docker: Cannot connect to the Docker daemon at tcp://0.0.0.0:2375. Is the docker daemon running?.
```

**解决方法**

（1）修改/lib/systemd/system/docker.service 文件中11行的内容为

vim /lib/systemd/system/docker.service 

```
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock -H tcp://0.0.0.0:7654
```

（2）修改~/.bashrc文件，添加以下内容

 vi ~/.bashrc

```
export DOCKER_HOST=tcp://0.0.0.0:2375
```

source ~/.bashrc

（3）重新加载

```
systemctl daemon-reload
```

（4）重启docker服务

```
systemctl restart docker
```

（5）再次执行

```
docker run -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher
```

#### 9.3 oci runtime error: container_linux.go:247: starting container process caused "process_linux.go:258:

问题如何出现
docker redis或者tomcat启动是报以下错误：

```
[root@localhost ~]# docker run -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher

b43ddc48fbdfde235d5b96b6277b37165a5f987b3746fc2cf60857a6c9041846
/usr/bin/docker-current: Error response from daemon: oci runtime error: container_linux.go:235: starting container process caused "process_linux.go:258: applying cgroup configuration for process caused \"Cannot set property TasksAccounting, or unknown property.\"".
```

**解决方法**


注意是CentOS版本问题，利用yum update更新一下系统就好

### 10.验证

安装是否成功(有client和service两部分表示docker安装启动都成功了)

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

