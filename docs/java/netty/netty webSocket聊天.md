# netty webSocket聊天

## 1.引入maven依赖

```xml
<dependencys>
        <dependency>
			<groupId>io.netty</groupId>
			<artifactId>netty-all</artifactId>
			<version>4.1.45.Final</version>
		</dependency>
		<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>fastjson</artifactId>
			<version>1.2.55</version>
		</dependency>

		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<version>1.18.6</version>
			<scope>provided</scope>
		</dependency>
</dependencys>
```

## 2. WebSocketServer

WebSocketServer.java

```java
package com.example.netty;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.Channel;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import org.springframework.stereotype.Component;

@Component
public class WebSocketServer {
    private NioEventLoopGroup bossGroup;
    private NioEventLoopGroup workGroup;
    private ServerBootstrap server;
    private Channel channel;
    private int port=8888;

    public void start() {
        bossGroup = new NioEventLoopGroup();
        workGroup = new NioEventLoopGroup();
        try {
            server = new ServerBootstrap();
            server.group(bossGroup, workGroup);
            server.channel(NioServerSocketChannel.class);
            server.childHandler(new WebSocketChannelInitializer());
            System.out.println("服务端开启，等待客户端连接···");
            channel = server.bind(port).sync().channel();
            channel.closeFuture().sync();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            bossGroup.shutdownGracefully();
            workGroup.shutdownGracefully();
        }
    }
}
```

## 3.WebSocketChannelInitializer

WebSocketChannelInitializer.java

```java
package com.example.netty;

import io.netty.channel.ChannelInitializer;
import io.netty.channel.socket.SocketChannel;
import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.HttpServerCodec;
import io.netty.handler.stream.ChunkedWriteHandler;
import io.netty.handler.timeout.IdleStateHandler;

/**
 * 初始化连接时的各个组件
 */
public class WebSocketChannelInitializer extends ChannelInitializer<SocketChannel> {
    @Override
    protected void initChannel(SocketChannel socketChannel) throws Exception {
        //添加Http的编解码器
        socketChannel.pipeline().addLast("http-codec", new HttpServerCodec());
        //添加一个用于支持大数据流的编解码器
        socketChannel.pipeline().addLast("http-chunked", new ChunkedWriteHandler());
        //添加聚合器，主要聚合器HttpMessage和FullHttpRequest/Response
        socketChannel.pipeline().addLast("aggregator", new HttpObjectAggregator(65536));
        // 添加空闲  1.读空闲  10s
        //          2.写空闲  20s
        //          3.读写空闲 30s
        socketChannel.pipeline().addLast("idle",new IdleStateHandler(10,20,30));
        // 添加心跳
        socketChannel.pipeline().addLast("hearBeat",new HearBeatHandler());
        socketChannel.pipeline().addLast("handler", new WebSocketHandler());
    }
}
```

## 4.WebSocketHandler

WebSocketHandler.java

```java
package com.example.netty;

import com.alibaba.fastjson.JSON;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.*;
import io.netty.channel.group.ChannelGroup;
import io.netty.channel.group.DefaultChannelGroup;
import io.netty.handler.codec.http.DefaultFullHttpResponse;
import io.netty.handler.codec.http.FullHttpRequest;
import io.netty.handler.codec.http.HttpResponseStatus;
import io.netty.handler.codec.http.HttpVersion;
import io.netty.handler.codec.http.websocketx.*;
import io.netty.util.CharsetUtil;
import io.netty.util.concurrent.GlobalEventExecutor;
import lombok.extern.slf4j.Slf4j;

import java.util.Date;

/**
 * 接受/处理/响应webSocket请求的核心业务处理类
 */
@Slf4j
public class WebSocketHandler extends SimpleChannelInboundHandler<Object> {

    //用于保存所有的客户端连接
    private static ChannelGroup clients = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);
    //WebSocket的地址
    private static final String WEB_SOCKET_URL = "ws://localhost:8888/websocket";

    private WebSocketServerHandshaker handshaker;

    /**
     * 客户端与服务端创建连接时调用
     */
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        //存放所有客户端的channel
        clients.add(ctx.channel());
        System.out.println("客户端与服务端开始连接···");
    }

    /**
     * 服务端接收到新请求时自动处理
     */
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, Object msg) throws Exception {
        //处理客户端向服务端发起http握手请求业务
        if (msg instanceof FullHttpRequest){
            System.out.println("===FullHttpRequest");
            handHttpRequest(ctx, (FullHttpRequest) msg);
        }else if (msg instanceof WebSocketFrame){
            //处理webSocket连接业务
            System.out.println("===WebSocketFrame");
            handWebSocketFrame(ctx, (WebSocketFrame) msg);
        }
    }


    /**
     * 客户端与服务端断开连接时调用
     */
    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        clients.remove(ctx.channel());
        System.out.println("客户端与服务端关闭连接···");
    }
    /**
     *  服务端接受客户端发送过来的数据结束之后时调用
     */
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.flush();
    }

    /**
     * 项目工程出现异常时调用
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }

    /**
     * 处理客户端向服务端发起http请求业务
     * @param ctx
     * @param req
     * @return 
     */
    private void handHttpRequest(ChannelHandlerContext ctx, FullHttpRequest req){
        if ( !req.getDecoderResult().isSuccess() || !("websocket".equals(req.headers().get("Upgrade"))) ){
            sendHttpResponse(ctx, req, new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.BAD_REQUEST));
            return;
        }
        WebSocketServerHandshakerFactory wsFactory = new WebSocketServerHandshakerFactory(WEB_SOCKET_URL, null, false);
        handshaker = wsFactory.newHandshaker(req);
        if ( handshaker == null ){
            WebSocketServerHandshakerFactory.sendUnsupportedWebSocketVersionResponse(ctx.channel());
        }else {
            handshaker.handshake(ctx.channel(), req);
        }
    }

    /**
     * 服务端向客户端响应消息
     * @param ctx
     * @param req
     * @param res
     * @return 
     */
    private void sendHttpResponse(ChannelHandlerContext ctx, FullHttpRequest req, DefaultFullHttpResponse res){
        if ( res.getStatus().code() != 200){
            ByteBuf buf = Unpooled.copiedBuffer(res.getStatus().toString(), CharsetUtil.UTF_8);
            res.content().writeBytes(buf);
            buf.release();
        }
        //服务端向客户端发送数据
        ChannelFuture future = ctx.channel().writeAndFlush(res);
        if ( res.getStatus().code() != 200 ){
            future.addListener(ChannelFutureListener.CLOSE);
        }
    }
    /**
     * 处理客户端与服务端之前的webSocket业务
     */
    private void handWebSocketFrame(ChannelHandlerContext ctx, WebSocketFrame frame){
        //判断是否是关闭webSocket的指令
        if (frame instanceof CloseWebSocketFrame){
            handshaker.close(ctx.channel(),(CloseWebSocketFrame)frame.retain());
            UserChannelMap.removeByChannelId(ctx.channel().id().asLongText());
            UserChannelMap.print();
            return;
        }
        if ( frame instanceof PongWebSocketFrame){
            //判断是否是ping指令
            ctx.channel().write(new PongWebSocketFrame(frame.content().retain()));
            return;
        }
        //判断是否是二进制指令
        if ( !(frame instanceof TextWebSocketFrame )){
            System.out.println("目前不支持二进制消息");
            throw new RuntimeException(" [ "+  this.getClass().getName() +" ] 不支持消息 ");
        }
        //获取返回应答消息
        String text = ((TextWebSocketFrame) frame).text();
        Message message = JSON.parseObject(text, Message.class);

        switch(message.getType()){
            // 0.保存用户id与channel到UserChannelMap中
            case 0:
                UserChannelMap.put(message.getFrom(),ctx.channel());
                UserChannelMap.print();
                break;
            // 1. 心跳消息
            case 1:
                log.info("心跳消息,{}"+message);
                break;
            // 2.文本消息类型
            case 2:
                String to = message.getTo();
                Channel channel = UserChannelMap.get(to);
                TextWebSocketFrame socketFrame = new TextWebSocketFrame(message.getMsg());
                channel.writeAndFlush(socketFrame);
                System.out.println(message.getFrom()+"正发送消息给"+message.getTo()+" 消息内容："+message.getMsg());
                break;
            default:
                log.info("没有收到其他类型消息");
                break;
        }
    }

}
```

## 5.心跳

```java
package com.example.netty;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.handler.timeout.IdleState;
import io.netty.handler.timeout.IdleStateEvent;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class HearBeatHandler extends ChannelInboundHandlerAdapter {

    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if(evt instanceof IdleStateEvent){
            IdleStateEvent idleStateEvent = (IdleStateEvent) evt;
            if(idleStateEvent.state()== IdleState.READER_IDLE){
                log.info("读空闲");
            }
            if(idleStateEvent.state()== IdleState.WRITER_IDLE){
                log.info("写空闲");
            }
            if(idleStateEvent.state()== IdleState.ALL_IDLE){
                log.info("读写空闲");
                ctx.channel().close();
            }
        }
    }
}
```

## 6.UserChannelMap

```java
package com.example.netty;

import io.netty.channel.Channel;
import lombok.extern.slf4j.Slf4j;

import java.util.*;

@Slf4j
public class UserChannelMap {

    private static HashMap channelMap;

    static {
        channelMap=new HashMap<String,Channel>();
    }

    public static void put(String userId,Channel channel){
        channelMap.put(userId,channel);
        log.info("将{}添加到channelMap",userId);
    }

    public static Channel get(String userId){
        log.info("从channelMap中获取{}的Channel",userId);
        return (Channel)channelMap.get(userId);
    }

    public static void removeByUserId(String userId){
        log.info("从channelMap中移除{}的Channel",userId);
        channelMap.remove(userId);
    }

    public static void removeByChannelId(String channelLongId){
        Set set = channelMap.entrySet();
        Iterator it = set.iterator();
        while(it.hasNext()) {
            Map.Entry entry = (Map.Entry)it.next();
            Channel channel = (Channel)entry.getValue();
            if(channel.id().asLongText().equals(channelLongId)) {
                String userId = (String)entry.getKey();
                removeByUserId(userId);
            }
        }
    }

    public static void print(){
        for (Object userId : channelMap.keySet()) {
            System.out.println("userId:"+userId.toString()+"---> channel:"+channelMap.get(userId).toString());
        }
    }
}
```

## 7.Message消息

```java
package com.example.netty;

import lombok.Data;
import lombok.Getter;
import lombok.Setter;

@Data
public class Message {
    /**
     * 定义的消息类型
     * 0.保存用户id与channel到UserChannelMap中  {"type":0,"from":"11"}                            *                                        {"type":0,"from":"22"}
     * 1.心跳消息  {"type":1,"from":"11"}  {"type":1,"from":"22"}
     * 2.文本消息  {"type":2,"from":"11","msg":"来自22发往22的消息","to":"22"}                    *            {"type":2,"from":"11","msg":"来自22发往22的消息","to":"22"}
     *
     */
    private Integer type;
    private String from;
    private String to;
    private String msg;
    //扩展字段
    private String ext;
}
```

## 8 WebSocket启动监听器

WebSocketListenner.java

```java
package com.example.netty;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.ApplicationListener;
import org.springframework.stereotype.Component;

@Component
public class WebSocketListenner implements ApplicationListener<ApplicationReadyEvent> {
    @Autowired
    private WebSocketServer webSocketServer;
    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        if(event.getApplicationContext().getParent()==null){
            webSocketServer.start();
        }
    }
}
```

