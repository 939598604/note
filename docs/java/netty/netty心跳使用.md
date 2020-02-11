# netty心跳使用

## 一.心跳机制

### 1.1 何为心跳

所谓**心跳**, 即在 `TCP` 长连接中, 客户端和服务器之间定期发送的一种特殊的数据包, 通知对方自己还在线, 以确保 `TCP` 连接的有效性.

> 注：心跳包还有另一个作用，经常被忽略，即：**一个连接如果长时间不用，防火墙或者路由器就会断开该连接**。

### 1.2 如何实现

**核心Handler —— IdleStateHandler**

在 `Netty` 中, 实现心跳机制的关键是 `IdleStateHandler`, 那么这个 `Handler` 如何使用呢? 先看下它的构造器：

```java
public IdleStateHandler(int readerIdleTimeSeconds, int writerIdleTimeSeconds, int allIdleTimeSeconds) {
    this((long)readerIdleTimeSeconds, (long)writerIdleTimeSeconds, (long)allIdleTimeSeconds, TimeUnit.SECONDS);
}
```

这里解释下三个参数的含义：

- readerIdleTimeSeconds: 读超时. 即当在指定的时间间隔内没有从 `Channel` 读取到数据时, 会触发一个 `READER_IDLE` 的 `IdleStateEvent` 事件.

- writerIdleTimeSeconds: 写超时. 即当在指定的时间间隔内没有数据写入到 `Channel` 时, 会触发一个 `WRITER_IDLE` 的 `IdleStateEvent` 事件.

- allIdleTimeSeconds: 读/写超时. 即当在指定的时间间隔内没有读或写操作时, 会触发一个 `ALL_IDLE` 的 `IdleStateEvent` 事件.

  > 注：这三个参数默认的时间单位是**秒**。若需要指定其他时间单位，可以使用另一个构造方法：``

```java
IdleStateHandler(boolean observeOutput, long readerIdleTime, long writerIdleTime, long allIdleTime, TimeUnit unit)
```

###  1.3 心跳保持实现

利用客户端的Idle事件发送心跳报文，心跳报文是由服务器端定义的。

**案例一： 服务端客户端为java项目 ** 

服务端对应着空闲事件，当为READER_IDLE时触发，将会关闭channel通道

```
ctx.channel().close();
```

所以客户端也在检查空闲的时候发送心跳包

```
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        System.out.println("当前轮询时间："+new Date());
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent event = (IdleStateEvent) evt;
            if (event.state() == IdleState.WRITER_IDLE) {
                System.out.println("HeartBeatClientHandler currentTime:"+currentTime+" 心跳次数"+ currentTime++);
                ctx.channel().writeAndFlush(HEARTBEAT_SEQUENCE.duplicate());
            }
        }
    }
```

**案例二 ： tcp报文方式案例中**

1. TCP 通信的报文格式是:

```
+--------+-----+---------------+ 
| Length |Type |   Content     |
|   17   |  1  |"HELLO, WORLD" |
+--------+-----+---------------+
```

1. 客户端每隔一个随机的时间后, 向服务器发送消息, 服务器收到消息后, 立即将收到的消息原封不动地回复给客户端.
2. 若客户端在指定的时间间隔内没有读/写操作, 则客户端会自动向服务器发送一个 PING 心跳, 服务器收到 PING 心跳消息时, 需要回复一个 PONG 消息.

```
PING_MSG = 1;  
PONG_MSG = 2; 
CUSTOM_MSG = 3;
```

公共代码块

```java
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        // IdleStateHandler 所产生的 IdleStateEvent 的处理逻辑.
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent e = (IdleStateEvent) evt;
            switch (e.state()) {
                case READER_IDLE:
                    handleReaderIdle(ctx);
                    break;
                case WRITER_IDLE:
                    handleWriterIdle(ctx);
                    break;
                case ALL_IDLE:
                    handleAllIdle(ctx);
                    break;
                default:
                    break;
            }
        }
    }
```

服务端：当检查到读写空闲时，关闭通道

```java
 @Override
    protected void handleAllIdle(ChannelHandlerContext ctx) {
        super.handleAllIdle(ctx);
        System.err.println("---client " + ctx.channel().remoteAddress().toString() + " ALL_IDLE timeout, close it---");
        ctx.close();
        System.err.println("调用关闭ctx.close()方法");
    }
```

客户端：利用读空闲发送心跳包

```java
  @Override
    protected void handleReaderIdle(ChannelHandlerContext ctx) {
        super.handleReaderIdle(ctx);
        System.out.println("调用ClientHandler的handleReaderIdle方法......");
        sendPingMsg(ctx);
    }
```



## 二.服务端客户端为java项目

### 2.1 服务端

#### 2.1.1 HeartBeatServer服务器启动

```java
package com.example.heartBeat.server;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;

public class HeartBeatServer {
    private int port;
 
    public HeartBeatServer(int port) {
        this.port = port;
    }
 
    public void start(){
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workGroup = new NioEventLoopGroup();
 
        ServerBootstrap server = new ServerBootstrap().group(bossGroup,workGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new HeartBeatServerChannelInitializer());
 
        try {
            ChannelFuture future = server.bind(port).sync();
            future.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            bossGroup.shutdownGracefully();
            workGroup.shutdownGracefully();
        }
    }
 
    public static void main(String[] args) {
        HeartBeatServer server = new HeartBeatServer(7788);
        server.start();
    }
}
```

#### 2.1.2  HeartBeatServerChannelInitializer

```java
package com.example.heartBeat.server;

import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.socket.SocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;
import io.netty.handler.timeout.IdleStateHandler;

import java.util.concurrent.TimeUnit;

public class HeartBeatServerChannelInitializer extends ChannelInitializer<SocketChannel> {
    @Override
    protected void initChannel(SocketChannel socketChannel) throws Exception {
        ChannelPipeline pipeline = socketChannel.pipeline();

        pipeline.addLast("handler",new IdleStateHandler(3, 0, 0, TimeUnit.SECONDS));
        pipeline.addLast("decoder", new StringDecoder());
        pipeline.addLast("encoder", new StringEncoder());
        pipeline.addLast(new HeartBeatServerHandler());
    }
}

```

#### 2.1.3 HeartBeatServerHandler

```java
package com.example.heartBeat.server;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.handler.timeout.IdleState;
import io.netty.handler.timeout.IdleStateEvent;


public class HeartBeatServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println(ctx.channel().remoteAddress() + "Server :" + msg.toString());
    }

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if(evt instanceof IdleStateEvent){
            //服务端对应着读事件，当为READER_IDLE时触发
            IdleStateEvent event = (IdleStateEvent)evt;
            if(event.state() == IdleState.ALL_IDLE){
                System.out.println("HeartBeatServerHandler READER_IDLE 接收消息超时");
                System.out.println("HeartBeatServerHandler READER_IDLE  关闭不活动的链接");
                ctx.channel().close();
            }else{
                super.userEventTriggered(ctx,evt);
            }
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}

```



### 2.2 客户端

####  2.2.1 客户端启动类

```
package com.example.heartBeat.client;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioSocketChannel;

public class HeartBeatsClient {
    private  int port;
    private  String address;

    public HeartBeatsClient(int port, String address) {
        this.port = port;
        this.address = address;
    }

    public void start(){
        EventLoopGroup group = new NioEventLoopGroup();

        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(group)
                .channel(NioSocketChannel.class)
                .handler(new HeartBeatsClientChannelInitializer());

        try {
            ChannelFuture future = bootstrap.connect(address,port).sync();
            future.channel().closeFuture().sync();
        } catch (Exception e) {
            e.printStackTrace();
        }finally {
            group.shutdownGracefully();
        }

    }

    public static void main(String[] args) {
        HeartBeatsClient client = new HeartBeatsClient(7788,"127.0.0.1");
        client.start();
    }
}
```

#### 2.2.2 HeartBeatsClientChannelInitializer

```
package com.example.heartBeat.client;

import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.socket.SocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;
import io.netty.handler.timeout.IdleStateHandler;

import java.util.concurrent.TimeUnit;

public class HeartBeatsClientChannelInitializer extends ChannelInitializer<SocketChannel> {

    protected void initChannel(SocketChannel socketChannel) throws Exception {
        ChannelPipeline pipeline = socketChannel.pipeline();

        pipeline.addLast("handler", new IdleStateHandler(0, 3, 0, TimeUnit.SECONDS));
        pipeline.addLast("decoder", new StringDecoder());
        pipeline.addLast("encoder", new StringEncoder());
        pipeline.addLast(new HeartBeatClientHandler());
    }
}

```

#### 2.2.3 HeartBeatClientHandler

```
package com.example.heartBeat.client;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.handler.timeout.IdleState;
import io.netty.handler.timeout.IdleStateEvent;
import io.netty.util.CharsetUtil;
import io.netty.util.ReferenceCountUtil;

import java.util.Date;

public class HeartBeatClientHandler extends ChannelInboundHandlerAdapter {

    private static final ByteBuf HEARTBEAT_SEQUENCE = Unpooled.unreleasableBuffer(Unpooled.copiedBuffer("Heartbeat",
            CharsetUtil.UTF_8));

    private int currentTime = 0;

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        System.out.println("激活时间是："+new Date());
        System.out.println("链接已经激活");
        ctx.fireChannelActive();
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        System.out.println("停止时间是："+new Date());
        System.out.println("关闭链接");
    }

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        System.out.println("当前轮询时间："+new Date());
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent event = (IdleStateEvent) evt;
            if (event.state() == IdleState.WRITER_IDLE) {
                System.out.println("HeartBeatClientHandler currentTime:"+currentTime+" 心跳次数"+ currentTime++);
                ctx.channel().writeAndFlush(HEARTBEAT_SEQUENCE.duplicate());
            }
        }
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        String message = (String) msg;
        System.out.println(message);
        if (message.equals("Heartbeat")) {
            ctx.write("has read message from server");
            ctx.flush();
        }
        ReferenceCountUtil.release(msg);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

### 2.3 启动测试

#### 2.3.1 启动服务端

```
01:04:14.314 [nioEventLoopGroup-3-1] DEBUG io.netty.util.ResourceLeakDetectorFactory - Loaded default ResourceLeakDetector: io.netty.util.ResourceLeakDetector@310f221f
/127.0.0.1:53365Server :Heartbeat
/127.0.0.1:53365Server :Heartbeat
/127.0.0.1:53365Server :Heartbeat
/127.0.0.1:53365Server :Heartbeat
/127.0.0.1:53365Server :Heartbeat
/127.0.0.1:53365Server :Heartbeat
/127.0.0.1:53365Server :Heartbeat
/127.0.0.1:53365Server :Heartbeat
/127.0.0.1:53365Server :Heartbeat
```

#### 2.3.2 启动客户端

```
激活时间是：Mon Feb 10 01:04:11 BRT 2020
链接已经激活
当前轮询时间：Mon Feb 10 01:04:14 BRT 2020
HeartBeatClientHandler currentTime:0 心跳次数0
当前轮询时间：Mon Feb 10 01:04:17 BRT 2020
HeartBeatClientHandler currentTime:1 心跳次数1
当前轮询时间：Mon Feb 10 01:04:20 BRT 2020
HeartBeatClientHandler currentTime:2 心跳次数2
当前轮询时间：Mon Feb 10 01:04:23 BRT 2020
HeartBeatClientHandler currentTime:3 心跳次数3
当前轮询时间：Mon Feb 10 01:04:26 BRT 2020
HeartBeatClientHandler currentTime:4 心跳次数4
当前轮询时间：Mon Feb 10 01:04:29 BRT 2020
HeartBeatClientHandler currentTime:5 心跳次数5
当前轮询时间：Mon Feb 10 01:04:32 BRT 2020
HeartBeatClientHandler currentTime:6 心跳次数6
当前轮询时间：Mon Feb 10 01:04:35 BRT 2020
HeartBeatClientHandler currentTime:7 心跳次数7
当前轮询时间：Mon Feb 10 01:04:38 BRT 2020
```

## 三.tcp报文方式案例

### 3.1 心跳要求

1. 在这个例子中, 客户端和服务器通过 TCP 长连接进行通信.
2. TCP 通信的报文格式是:

```
+--------+-----+---------------+ 
| Length |Type |   Content     |
|   17   |  1  |"HELLO, WORLD" |
+--------+-----+---------------+
```

1. 客户端每隔一个随机的时间后, 向服务器发送消息, 服务器收到消息后, 立即将收到的消息原封不动地回复给客户端.

2. 若客户端在指定的时间间隔内没有读/写操作, 则客户端会自动向服务器发送一个 PING 心跳, 服务器收到 PING 心跳消息时, 需要回复一个 PONG 消息.

   

### 3.2 通用的CustomHeartbeatHandler

```java
package com.example.tcpHearBeat;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.handler.timeout.IdleStateEvent;

public abstract class CustomHeartbeatHandler extends SimpleChannelInboundHandler<ByteBuf> {
    public static final byte PING_MSG = 1;
    public static final byte PONG_MSG = 2;
    public static final byte CUSTOM_MSG = 3;
    protected String name;
    private int heartbeatCount = 0;

    public CustomHeartbeatHandler(String name) {
        this.name = name;
    }

    @Override
    protected void channelRead0(ChannelHandlerContext context, ByteBuf byteBuf) throws Exception {
        if (byteBuf.getByte(4) == PING_MSG) {
            sendPongMsg(context);
        } else if (byteBuf.getByte(4) == PONG_MSG){
            System.out.println(name + " get pong msg from " + context.channel().remoteAddress());
        } else {
            handleData(context, byteBuf);
        }
    }

    protected void sendPingMsg(ChannelHandlerContext context) {
        ByteBuf buf = context.alloc().buffer(5);
        buf.writeInt(5);
        buf.writeByte(PING_MSG);
        context.writeAndFlush(buf);
        heartbeatCount++;
        System.out.println(name + " sent ping msg to " + context.channel().remoteAddress() + ", count: " + heartbeatCount);
    }

    private void sendPongMsg(ChannelHandlerContext context) {
        ByteBuf buf = context.alloc().buffer(5);
        buf.writeInt(5);
        buf.writeByte(PONG_MSG);
        context.channel().writeAndFlush(buf);
        heartbeatCount++;
        System.out.println(name + " sent pong msg to " + context.channel().remoteAddress() + ", count: " + heartbeatCount);
    }

    protected abstract void handleData(ChannelHandlerContext channelHandlerContext, ByteBuf byteBuf);

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        // IdleStateHandler 所产生的 IdleStateEvent 的处理逻辑.
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent e = (IdleStateEvent) evt;
            switch (e.state()) {
                case READER_IDLE:
                    handleReaderIdle(ctx);
                    break;
                case WRITER_IDLE:
                    handleWriterIdle(ctx);
                    break;
                case ALL_IDLE:
                    handleAllIdle(ctx);
                    break;
                default:
                    break;
            }
        }
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        System.err.println("---" + ctx.channel().remoteAddress() + " is active---");
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        System.err.println("---" + ctx.channel().remoteAddress() + " is inactive---");
    }

    protected void handleReaderIdle(ChannelHandlerContext ctx) {
        System.err.println("---READER_IDLE---");
    }

    protected void handleWriterIdle(ChannelHandlerContext ctx) {
        System.err.println("---WRITER_IDLE---");
    }

    protected void handleAllIdle(ChannelHandlerContext ctx) {
        System.err.println("---ALL_IDLE---");
    }
}
```

### 3.3 服务端

#### 3.3.1 服务端启动

```java
package com.example.tcpHearBeat.server;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.Channel;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.LengthFieldBasedFrameDecoder;
import io.netty.handler.timeout.IdleStateHandler;

public class Server {
    public static void main(String[] args) {
        NioEventLoopGroup bossGroup = new NioEventLoopGroup(1);
        NioEventLoopGroup workGroup = new NioEventLoopGroup(4);
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap
                    .group(bossGroup, workGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            ChannelPipeline p = socketChannel.pipeline();
                            p.addLast(new LengthFieldBasedFrameDecoder(1024, 0, 4, -4, 0));
                            p.addLast(new IdleStateHandler(5, 10, 15));
                            p.addLast(new ServerHandler());
                        }
                    });

            Channel ch = bootstrap.bind(12345).sync().channel();
            ch.closeFuture().sync();
        } catch (Exception e) {
            throw new RuntimeException(e);
        } finally {
            bossGroup.shutdownGracefully();
            workGroup.shutdownGracefully();
        }
    }
}
```

#### 3.3.2 ServerHandler

```java
package com.example.tcpHearBeat.server;

import com.example.tcpHearBeat.CustomHeartbeatHandler;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerContext;

public class ServerHandler extends CustomHeartbeatHandler {
    public ServerHandler() {
        super("server");
    }

    @Override
    protected void handleData(ChannelHandlerContext channelHandlerContext, ByteBuf buf) {
        byte[] data = new byte[buf.readableBytes() - 5];
        ByteBuf responseBuf = Unpooled.copiedBuffer(buf);
        buf.skipBytes(5);
        buf.readBytes(data);
        String content = new String(data);
        System.out.println(name + " get content: " + content);
        channelHandlerContext.write(responseBuf);
    }

    @Override
    protected void handleAllIdle(ChannelHandlerContext ctx) {
        super.handleAllIdle(ctx);
        System.err.println("---client " + ctx.channel().remoteAddress().toString() + " ALL_IDLE timeout, close it---");
        ctx.close();
        System.err.println("调用关闭ctx.close()方法");
    }
}
```



 ### 3.4 客户端

#### 3.4.1 客户端启动

```java
package com.example.tcpHearBeat.client;

import com.example.tcpHearBeat.CustomHeartbeatHandler;
import io.netty.bootstrap.Bootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.channel.Channel;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.LengthFieldBasedFrameDecoder;
import io.netty.handler.timeout.IdleStateHandler;

import java.util.Random;

public class Client {
    public static void main(String[] args) {
        NioEventLoopGroup workGroup = new NioEventLoopGroup(4);
        Random random = new Random(System.currentTimeMillis());
        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap
                    .group(workGroup)
                    .channel(NioSocketChannel.class)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            ChannelPipeline p = socketChannel.pipeline();
                            p.addLast(new LengthFieldBasedFrameDecoder(1024, 0, 4, -4, 0));
                            p.addLast(new IdleStateHandler(5, 10, 15));
                            p.addLast(new ClientHandler());
                        }
                    });
            ChannelFuture future = bootstrap.connect("127.0.0.1", 12345).sync();
            Channel channel = future.channel();
            for (int i = 0; i < 5; i++) {
                String content = "client msg " + i;
                ByteBuf buf = channel.alloc().buffer();
                buf.writeInt(5 + content.getBytes().length);
                buf.writeByte(CustomHeartbeatHandler.CUSTOM_MSG);
                buf.writeBytes(content.getBytes());
                channel.writeAndFlush(buf);
                Thread.sleep(random.nextInt(20000));
            }
            channel.closeFuture().sync();
        } catch (Exception e) {
            throw new RuntimeException(e);
        } finally {
            workGroup.shutdownGracefully();
        }
    }
}
```

####  3.4.2 ClientHandler

```java
package com.example.tcpHearBeat.client;

import com.example.tcpHearBeat.CustomHeartbeatHandler;
import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;

public class ClientHandler extends CustomHeartbeatHandler {
    public ClientHandler() {
        super("client");
    }

    @Override
    protected void handleData(ChannelHandlerContext channelHandlerContext, ByteBuf byteBuf) {
        byte[] data = new byte[byteBuf.readableBytes() - 5];
        byteBuf.skipBytes(5);
        byteBuf.readBytes(data);
        String content = new String(data);
        System.out.println(name + " get content: " + content);
    }

    @Override
    protected void handleReaderIdle(ChannelHandlerContext ctx) {
        super.handleReaderIdle(ctx);
        System.out.println("调用ClientHandler的handleReaderIdle方法......");
        sendPingMsg(ctx);
    }
}
```

