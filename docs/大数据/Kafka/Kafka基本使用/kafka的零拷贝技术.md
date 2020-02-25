# Kafka零拷贝

## 为什么Kafka那么快

Kafka速度的秘诀在于，它把所有的消息都变成一个的文件。通过mmap提高I/O速度，写入数据的时候它是末尾添加所以速度最优；读取数据的时候配合sendfile直接暴力输出

## 传统IO|缓存IO

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200102102552.png)

传统IO也就是缓存IO。数据先从磁盘复制到内核空间缓冲区，然后从内核空间缓冲区复制到应用程序的地址空间。这里的内核缓冲区也就是页缓存-**PageCache**，是虚拟内存空间

读操作：操作系统检查内核的缓冲区有没有需要的数据，如果已经缓存了，那么就直接从缓存中返回；否则从磁盘中读取，然后缓存在操作系统的缓存中

写操作：将数据从用户空间复制到内核空间的缓存中。这时对用户程序来说写操作就已经完成，至于什么时候再写到磁盘中由操作系统决定，除非显示地调用了sync同步命令：[sync、fsync与fdatasync](http://blog.csdn.net/younger_china/article/details/51127127)

优点：分离了内核空间和用户空间，保护系统本身的运行安全；减少读盘次数，提升性能

缺点：在缓存I/O机制中，DMA方式可以将数据直接从磁盘读到页缓存中，或者将数据从页缓存直接写回到磁盘上，而不能直接在应用程序地址空间和磁盘之间进行数据传输。这样，数据在传输过程中需要在应用程序地址空间（用户空间）和缓存（内核空间）之间进行多次数据拷贝操作，这些数据拷贝操作所带来的CPU以及内存开销是非常大的

## 零拷贝

Linux零拷贝分为：直接io、mmap()、sendfile()、splice()

### 直接IO

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200102102633.png)

直接IO：应用程序直接访问磁盘数据，而不经过内核缓冲区。这样做的目的是减少一次从内核缓冲区到用户程序缓存的数据复制。比如说数据库管理系统这类应用，它们更倾向于选择它们自己的缓存机制，因为数据库管理系统往往比操作系统更了解数据库中存放的数据，数据库管理系统可以提供一种更加有效的缓存机制来提高数据库中数据的存取性能

优点：通过减少操作系统内核缓冲区和应用程序地址空间的数据拷贝次数，降低了对文件读取和写入时所带来的CPU的使用以及内存带宽的占用

缺点：直接I/O的读写数据操作会造成磁盘的同步读写，导致进程执行缓慢。所以，应用程序使用直接I/O进行数据传输的时候通常会和使用异步I/O结合使用(异步IO：当访问数据的线程发出请求之后，线程会接着去处理其他事，而不是阻塞等待)

### MMAP

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200102102715.png)

应用程序调用了mmap()之后，数据会先通过DMA拷贝到操作系统内核的缓冲区。接着，应用程序跟操作系统共享这个缓冲区。这样，操作系统内核和应用程序存储空间就不需要再进行任何的数据拷贝操作。

也就是说内存映射文件MMAP只有一次页缓存的复制，读时从磁盘文件复制到页缓存，写时从页缓存flush到磁盘文件，默认30s。MMAP与操作系统的Pagecache打交道

普通文件IO需要复制两次，内存映射文件mmap复制一次，普通文件IO是堆内操作，内存映射文件是堆外操作

### SendFile

![](https://gitee.com/chenjinhua_939598604/resources/raw/img/static/20200102102736.png)

sendfile()系统调用利用DMA引擎将文件中的数据拷贝到操作系统内核缓冲区中，然后数据被拷贝到与socket相关的内核缓冲区。接下来，DMA引擎将数据从内核socket缓冲区中拷贝到协议引擎

sendfile()系统调用不需要将数据拷贝或映射到应用程序地址空间，所以sendfile()只适用于应用程序地址空间不需要对所访问数据进行处理的情况。比如apache、nginx等web服务器使用sendfile传输静态文件

```
servletOutputStream = response.getOutputStream();
FileChannel channel = new FileInputStream(imgPath).getChannel();
channel.transferTo(0, channel.size(), Channels.newChannel(servletOutputStream));
SocketAddress sad = new InetSocketAddress(host, port);
SocketChannel sc = SocketChannel.open();
FileChannel fc = new FileInputStream(fname).getChannel();
fc.transferTo(0, fc.size(), sc);
```

### 总结

常用的读文件方式read的过程是：
磁盘->文件缓冲区->用户空间

mmap是：
磁盘->用户空间

可以看到mmap少了一次内存copy。另外mmap可以通过多个用户进程映射到一个相同的文件（也可以是虚拟文件）达到共享内存，进程通信的效用。实现方法就是为两个进程的分配相同的页

再说sendfile
nginx，apache都有开启sendfile的配置项，sendfile相对于传统的read+write+buffer的socket通信方式会高效一些

传统过程：
磁盘->文件缓冲区->用户空间->socket缓冲区->协议引擎

sendfile：
磁盘->文件缓冲区->socket缓冲区->协议引擎

相对于mmap，sendfile少了内存映射的环节，如果传输很大的文件，内存映射的损耗可以忽略不计，但是如果传输的文件比较小，内存映射的损耗占比就会扩大

splice：和sendfile()非常类似，用户应用程序必须拥有两个已经打开的文件描述符，一个用于表示输入设备，一个用于表示输出设备。与sendfile()不同的是，splice()允许任意两个文件之间互相连接，而并不只是文件到socket进行数据传输

对于从一个文件描述符发送数据到socket这种特例来说，一直都是使用sendfile()这个系统调用，而splice一直以来就只是一种机制，它并不仅限于sendfile()的功能。也就是说，sendfile()只是splice()的一个子集，在Linux 2.6.23中，sendfile()这种机制的实现已经没有了，但是这个API以及相应的功能还存在，只不过API以及相应的功能是利用了splice()这种机制来实现的

硬件设备跟内存通讯通过DMA完成，内存之间的copy是由cpu完成，减少内存copy可以解放cpu，提高系统负载

## Kafka机制

### partition 顺序写入

每一个Partition其实都是一个文件，收到消息后Kafka会把数据插入到文件末尾。消费者对每个Topic都有一个offset(存放ZK中)用来表示读取到了第几条数据

```
FileChannel fc = new RandomAccessFile("data.txt" , "rw").getChannel();  
long length = fc.size();  // 设置映射区域的开始位置，这是末尾写入的重点
MappedByteBuffer mbb = fc.map(FileChannel.MapMode.READ_WRITE, length, 20);  
//由于要写入的字符串"Write in the end"占16字节，所以长度设置为20就
```

直接利用操作系统的Page来实现文件到物理内存的直接映射。完成映射之后你对物理内存的操作会被同步到硬盘上（操作系统在适当的时候）。也就是MappedByteBuffer类

mmap的文件映射在full gc时才会进行释放。当close时，需要手动清除内存映射文件，反射调用sun.misc.Cleaner方法：[MappedByteBuffer](http://langgufu.iteye.com/blog/2107023)

[MappedByteBuffer未关闭导致慢磁盘访问](http://www.cnblogs.com/huxi2b/p/6637425.html)

阿里的RocketMQ实现就是Kafka的java版本：[MappedFile](https://github.com/alibaba/RocketMQ/blob/master/rocketmq-store/src/main/java/com/alibaba/rocketmq/store/MapedFile.java)

### sendfile读取

Kafka把所有的消息都存放在一个个文件中，当消费者需要数据的时候Kafka直接把“文件”发送给消费者

Kafka是用mmap作为文件读写方式的，它就是一个文件句柄，所以直接把它传给sendfile；偏移也好解决，用户会自己保持这个offset，每次请求都会发送这个offset

### 总结

Kafka速度的秘诀在于，它把所有的消息都变成一个的文件。通过mmap提高I/O速度，写入数据的时候它是末尾添加所以速度最优；读取数据的时候配合sendfile直接暴力输出