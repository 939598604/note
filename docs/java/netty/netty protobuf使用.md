# netty protobuf使用

## 一、protobuf使用

### 1.1 protobuf下载

下载地址：https://github.com/protocolbuffers/protobuf/releases

找个 [protoc-3.11.3-win64.zip](https://github.com/protocolbuffers/protobuf/releases/download/v3.11.3/protoc-3.11.3-win64.zip)进行下载，然后解压到本地

或者下载3.6.1版本的

protobuf3.6.x版本的百度网盘下载地址
由于需要更新protobuf到3.6.x版本，比较坑的官网有时候会挂，这里分享百度网盘下载地址，永久有效。

链接: https://pan.baidu.com/s/1FGCTh1VcmPV2iOyi7y-0mA   提取码: j5qs

### 1.2 配置环境变量

将解压出来的`protoc.exe`放在一全英文路径下，并把其路径名放在windows环境变量下的`path`下。

我这个里放在 E:\software\protoc-3.11.3-win64

找到环境变量path: 添加E:\software\protoc-3.11.3-win64\bin;

### 1.3 测试生效

cmd命令 ，然后输入protoc命令验证

### 1.4 编写protobuf类

新建文件名称为：stuProtobuf.proto

```protobuf
syntax = "proto3";
option java_package="com.example.protobuf";
option java_outer_classname="stuProtobuf";

message SubscribeReq{
    int32 id = 1;
    string userName = 2;
}
```

### 1.5 生成java文件

```
protoc stuProtobuf.proto --java_out=./
```



### 1.2 protobuf-java jar包的引入

引入的protobuf-java版本必须要和protoc-3.11.3-win64版本一致，否则会报错

```
<dependency>
      <groupId>com.google.protobuf</groupId>
      <artifactId>protobuf-java</artifactId>
      <version>3.11.3</version>
</dependency>
```

 

## 二、服务端代码编写

### 2.1 NettyClient.java

```java
package com.example.protobuf.client;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.LengthFieldPrepender;
import io.netty.handler.codec.protobuf.ProtobufDecoder;
import io.netty.handler.codec.protobuf.ProtobufEncoder;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class NettyClient {
	public static void main(String[] args) throws InterruptedException {
		EventLoopGroup group = new NioEventLoopGroup();
		Bootstrap bootstrap = new Bootstrap();
		bootstrap.group(group)//
				.channel(NioSocketChannel.class)//
				.option(ChannelOption.TCP_NODELAY, true)//
				.handler(new ChannelInitializer<SocketChannel>() {
					@Override
					protected void initChannel(SocketChannel ch) throws Exception {
						ch.pipeline().addLast(new LengthFieldPrepender(4));
						ch.pipeline().addLast(new ProtobufEncoder());
						ch.pipeline().addLast(new ProtoClientHandler());
					}
				});
		ChannelFuture future = bootstrap.connect("127.0.0.1", 9091).sync();
		log.info("client connect server.");
		future.channel().closeFuture().sync();
		group.shutdownGracefully();
	}
}
```

### 2.2 ProtoClientHandler.java

```java
package com.example.protobuf.client;

import com.example.protobuf.StuProtobuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

public class ProtoClientHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        StuProtobuf.SubscribeReq subscribeReq = StuProtobuf.SubscribeReq.newBuilder().setId(1).setUserName("zhangsan").build();
        ctx.writeAndFlush(subscribeReq);
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {

    }

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {

    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {

    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```



## 三、客户端代码编写

### 3.1 NettyServer.java

```java
package com.example.protobuf.server;

import com.example.protobuf.StuProtobuf;
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.LengthFieldBasedFrameDecoder;
import io.netty.handler.codec.protobuf.ProtobufDecoder;
import io.netty.handler.logging.LogLevel;
import io.netty.handler.logging.LoggingHandler;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class NettyServer {
	public static void main(String[] args) throws Exception {
		EventLoopGroup boss = new NioEventLoopGroup(1);
		EventLoopGroup worker = new NioEventLoopGroup();
		ServerBootstrap bootstrap = new ServerBootstrap();
		bootstrap.group(boss, worker)//
				.channel(NioServerSocketChannel.class)// 对应的是ServerSocketChannel类
				.option(ChannelOption.SO_BACKLOG, 128)//
				.handler(new LoggingHandler(LogLevel.TRACE))//
				.childHandler(new ChannelInitializer<SocketChannel>() {
					@Override
					protected void initChannel(SocketChannel ch) throws Exception {
						ch.pipeline().addLast(new LengthFieldBasedFrameDecoder(10240, 0, 4, 0, 4));
						ch.pipeline().addLast(new ProtobufDecoder(StuProtobuf.SubscribeReq.getDefaultInstance()));
						ch.pipeline().addLast(new ProtoServerHandler());
					}
				});
		ChannelFuture future = bootstrap.bind(9091).sync();
		log.info("server start in port:[{}]", 9091);
		future.channel().closeFuture().sync();
		boss.shutdownGracefully();
		worker.shutdownGracefully();
	}
}
```

### 3.2 ProtoServerHandler.java

```java
package com.example.protobuf.server;

import com.example.protobuf.StuProtobuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

public class ProtoServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {

    }

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {

    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        StuProtobuf.SubscribeReq subscribeReq = (StuProtobuf.SubscribeReq) msg;
        System.out.println(subscribeReq.getId()+":"+subscribeReq.getUserName());
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

## 四、启动服务端和客户端

### 4.1 启动服务端

接收到发送过来的数据：zhangsan

```
io.netty.buffer.AbstractByteBuf - -Dio.netty.buffer.checkAccessible: true
12:17:23.655 [nioEventLoopGroup-3-1] DEBUG io.netty.buffer.AbstractByteBuf - -Dio.netty.buffer.checkBounds: true
12:17:23.656 [nioEventLoopGroup-3-1] DEBUG io.netty.util.ResourceLeakDetectorFactory - Loaded default ResourceLeakDetector: io.netty.util.ResourceLeakDetector@146d996a
1:zhangsan

```

### 4.2 启动客户端

```
12:17:23.624 [nioEventLoopGroup-2-1] DEBUG io.netty.buffer.AbstractByteBuf - -Dio.netty.buffer.checkBounds: true
12:17:23.625 [nioEventLoopGroup-2-1] DEBUG io.netty.util.ResourceLeakDetectorFactory - Loaded default ResourceLeakDetector: io.netty.util.ResourceLeakDetector@4a4419b
12:17:23.630 [nioEventLoopGroup-2-1] DEBUG io.netty.util.Recycler - -Dio.netty.recycler.maxCapacityPerThread: 4096
12:17:23.630 [nioEventLoopGroup-2-1] DEBUG io.netty.util.Recycler - -Dio.netty.recycler.maxSharedCapacityFactor: 2
12:17:23.630 [nioEventLoopGroup-2-1] DEBUG io.netty.util.Recycler - -Dio.netty.recycler.linkCapacity: 16
12:17:23.630 [nioEventLoopGroup-2-1] DEBUG io.netty.util.Recycler - -Dio.netty.recycler.ratio: 8
```

