# redis 持久化

## 理解Redis持久化

后端技术指南针 [后端技术指南针](javascript:void(0);) *2019-10-20*

**0.前言**

通俗讲持久化就是将内存中的数据写入非易失介质中，比如机械磁盘和SSD。

在服务器发生宕机时，作为内存数据库Redis里的所有数据将会丢失，因此Redis提供了持久化两大利器：RDB和AOF。

```
RDB:Redis Dump Binary #全量二进制导出AOF: 
Append-Only File #增量文件追加
```

- RDB 将数据库快照以二进制的方式保存到磁盘中。
- AOF 以协议文本方式，将所有对数据库进行过写入的命令和参数记录到 AOF 文件，从而记录数据库状态。



**1.RDB的配置**

- ***查看RDB配置***

```
[redis@abc]$ cat /abc/redis/conf/redis.conf   
save 900 1  
save 300 10  
save 60 10000  
dbfilename "dump.rdb" 
dir "/data/dbs/redis/rdbstro" 
```

上图为Redis配置文件里默认的RDB设置：

前三行都是对触发RDB的一个条件， 如第一行表示每900秒钟有一条数据被修改则触发RDB，依次类推；只要一条满足就会进行RDB持久化；

第四行dbfilename指定了把内存里的数据库写入本地文件的名称，该文件是进行压缩后的二进制文件；

第五行dir指定了RDB二进制文件存放目录 ；

- ***修改RDB配置***

在命令行里进行配置,服务器重启才会生效:

```
[redis@abc]$ bin/redis-cli
127.0.0.1:6379> CONFIG GET save
1) "save"
2) "900 1 300 10 60 10000"
127.0.0.1:6379> CONFIG SET save "21600 1000" 
OK
```



**2.RDB的SAVE和BGSAVE**

RDB文件是很适合数据的容灾备份与恢复，通过RDB文件恢复数据库耗时较短，可以快速恢复数据。

RDB持久化只会周期性的保存数据，在未触发下一次存储时服务宕机，就会丢失增量数据。当数据量较大的情况下，fork子进程这个操作很消耗cpu，可能会发生长达秒级别的阻塞情况。

SAVE是阻塞式持久化，执行命令时Redis主进程把内存数据写入到RDB文件中直到创建完毕，期间Redis不能处理任何命令。

　　BGSAVE属于非阻塞式持久化，创建一个子进程把内存中数据写入RDB文件里，同时主进程处理命令请求。

如图展示了bgsave的简单流程：

![img](https://mmbiz.qpic.cn/mmbiz_png/wAkAIFs11qbQwUypouw6PS6HrmAib1eBotuIRpHvZFqYdzNKojzevsrvpfqgLTuAM7pA7E6ib4hlAKKgYzCLIDUg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



**3.BGSAVE实现细节**

RDB方式的持久化是通过快照实现的，符合条件时Redis会自动将内存数据进行快照并存储在硬盘上，以BGSAVE为例，一次完整数据快照的过程：

- Redis使用fork函数创建子进程；
- 父进程继续接收并处理命令请求，子进程将内存数据写入临时文件；
- 子进程写入所有数据后会用临时文件替换旧RDB文件；

　　执行fork的时OS会使用写时拷贝策略，对子进程进行快照过程优化。

Redis在进行快照过程中不会修改RDB文件，只有快照结束后才会将旧的文件替换成新的，也就是任何时候RDB文件都是完整的。

我们可以通过定时备份RDB文件来实现Redis数据库备份，RDB文件是经过压缩的，占用的空间会小于内存中的数据大小。
　　除了自动快照还可以手动发送SAVE或BGSAVE命令让Redis执行快照。通过RDB方式实现持久化，由于RDB保存频率的限制，如果数据很重要则考虑使用AOF方式进行持久化。



**4.AOF的配置**

在使用AOF持久化方式时，Redis会将每一个收到的写命令都通过Write函数追加到文件中类似于MySQL的binlog。换言之AOF是通过保存对redis服务端的写命令来记录数据库状态的。

AOF文件有自己的存储协议格式：



```
[redis@abc]$ more appendonly.aof 
*2     # 2个参数
$6     # 第一个参数长度为 6
SELECT     # 第一个参数
$1     # 第二参数长度为 1
8     # 第二参数
*3     # 3个参数
$3     # 第一个参数长度为 4
SET     # 第一个参数
$4     # 第二参数长度为 4
name     # 第二个参数
$4     # 第三个参数长度为 4
Jhon     # 第二参数长度为 4
```

AOF配置：

```
[redis@abc]$ more ~/redis/conf/redis.conf
dir "/data/dbs/redis/abcd"           #AOF文件存放目录
appendonly yes                       #开启AOF持久化，默认关闭
appendfilename "appendonly.aof"      #AOF文件名称（默认）
appendfsync no                       #AOF持久化策略
auto-aof-rewrite-percentage 100      #触发AOF文件重写的条件（默认）
auto-aof-rewrite-min-size 64mb       #触发AOF文件重写的条件（默认）
```

　　当开启AOF后，服务端每执行一次写操作就会把该条命令追加到一个单独的AOF缓冲区的末尾，然后把AOF缓冲区的内容写入AOF文件里，由于磁盘缓冲区的存在写入AOF文件之后，并不代表数据已经落盘了，而何时进行文件同步则是根据配置的appendfsync来进行配置：

appendfsync有三个选项：always、everysec和no：

- always：服务器在每执行一个事件就把AOF缓冲区的内容强制性的写入硬盘上的AOF文件里，保证了数据持久化的完整性，效率是最慢的但最安全的；
- everysec：服务端每隔一秒才会进行一次文件同步把内存缓冲区里的AOF缓存数据真正写入AOF文件里，兼顾了效率和完整性，极端情况服务器宕机只会丢失一秒内对Redis数据库的写操作；
- no：表示默认系统的缓存区写入磁盘的机制，不做程序强制，数据安全性和完整性差一些。



**5.AOF重写**

- ***AOF重写的必要性***

AOF比RDB文件更大，并且在存储命令的过程中增长更快，为了压缩AOF的持久化文件，Redis提供了重写机制以此来实现控制AOF文件的增长。

AOF重写实现的理论基础是这样的：

- 执行set hello world 50次 
- 最后执行一次 set hello china
- 最终对于AOF文件而言前面50次都是无意义的，AOF重写就是将key只保存最后的状态。

- ***重写期间的数据一致性问题***

子进程在进行 AOF 重写期间， 主进程还需要继续处理命令， 而新的命令可能对现有的数据进行修改， 会出现数据库的数据和重写后的 AOF 文件中的数据不一致。

因此Redis 增加了一个 AOF 重写缓存， 这个缓存在 fork 出子进程之后开始启用， Redis 主进程在接到新的写命令之后， 除了会将这个写命令的协议内容追加到现有的 AOF 文件之外， 还会追加到这个缓存中。

![img](https://mmbiz.qpic.cn/mmbiz_png/wAkAIFs11qbQwUypouw6PS6HrmAib1eBod79dlrOAbFr3WHtVDfEcKPjWKP3cp3uc5umJwiawvIibT73ribhQoXibSQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- ***AOF文件覆盖***

当子进程完成 AOF 重写之后向父进程发送一个完成信号， 父进程在接到完成信号之后会调用信号处理函数，完成以下工作：

- 将 AOF 重写缓存中的内容全部写入到新 AOF 文件中
- 对新的 AOF 文件进行改名，覆盖原有的 AOF 文件

- ***AOF重写的阻塞性***

整个 AOF 后台重写过程中只有最后写入缓存和改名操作会造成主进程阻塞， 在其他时候AOF 后台重写都不会对主进程造成阻塞， 将 AOF 重写对性能造成的影响降到了最低。

- ***AOF重写的触发条件***



AOF 重写可以由用户通过调用 BGREWRITEAOF 手动触发。

服务器在 AOF 功能开启的情况下，会维持以下三个变量：

- 当前 AOF 文件大小 
- 最后一次 重写之后， AOF 文件大小的变量 
- AOF文件大小增长百分比

每次当 serverCron 函数执行时， 它都会检查以下条件是否全部满足， 如果是的话， 就会触发自动的 AOF 重写：

- 没有 BGSAVE 命令在进行 防止于RDB的冲突
- 没有 BGREWRITEAOF 在进行 防止和手动AOF冲突
- 当前 AOF 文件大小至少大于设定值 基本要求 太小没意义
- 当前 AOF 文件大小和最后一次 AOF 重写后的大小之间的比率大于等于指定的增长百分比



**6.Redis的数据恢复**

- ***Redis 恢复优先级***

- 如果只配置 AOF ，重启时加载 AOF 文件恢复数据；
- 如果同时配置了 RDB 和 AOF ，启动只加载 AOF 文件恢复数据；
- 如果只配置 RDB，启动将加载 dump 文件恢复数据。

- ***AOF恢复方式***

拷贝 AOF 文件到 Redis 的数据目录，启动 redis-server 

AOF 的数据恢复过程:Redis 虚拟一个客户端，读取AOF文件恢复 Redis 命令和参数，然后执行命令从而恢复数据。这些过程主要在loadAppendOnlyFile() 中实现。

- ***RDB恢复方式***

拷贝 RDB 文件到 Redis 的数据目录，启动 redis-server即可，因为RDB文件和重启前保存的是真实数据而不是命令状态和参数。



**7.新型的混合型持久化**

RDB和AOF都有各自的缺点：

- RDB是每隔一段时间持久化一次, 故障时就会丢失宕机时刻与上一次持久化之间的数据，无法保证数据完整性
- AOF存储的是指令序列, 恢复重放时要花费很长时间并且文件更大

Redis 4.0 提供了更好的混合持久化选项： 创建出一个同时包含 RDB 数据和 AOF 数据的 AOF 文件， 其中 RDB 数据位于 AOF 文件的开头， 它们储存了服务器开始执行重写操作时的数据库状态，至于那些在重写操作执行之后执行的 Redis 命令， 则会继续以 AOF 格式追加到 AOF 文件的末尾， 也即是 RDB 数据之后。

![img](https://mmbiz.qpic.cn/mmbiz_png/wAkAIFs11qbQwUypouw6PS6HrmAib1eBogXC1PVw6Ml8C51A7Twbawz7VMjHzElMt14Yn5rhYmXM9BTX6S1RfhQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



**8.持久化实战**

在实际使用中需要根据Redis作为主存还是缓存、数据完整性和缺失性的要求、CPU和内存情况等诸多因素来确定适合自己的持久化方案，一般来说稳妥的做法包括：

- 最安全的做法是RDB与AOF同时使用，即使AOF损坏无法修复，还可以用RDB来恢复数据，当然在持久化时对性能也会有影响。
- Redis当简单缓存，没有缓存也不会造成缓存雪崩只使用RDB即可。
- 不推荐单独使用AOF，因为AOF对于数据的恢复载入比RDB慢，所以使用AOF的时候，最好还是有RDB作为备份。
- 采用新版本Redis 4.0的持久化新方案。