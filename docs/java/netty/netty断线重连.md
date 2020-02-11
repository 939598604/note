# netty断线重连

## 一.交通局业务

netty客户端连接交投的服务器

###  1.1 NettyClient

```
package com.gzstrong.netty;

import com.gzstrong.common.util.SpringUtil;
import com.gzstrong.util.Constants;
import io.netty.bootstrap.Bootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.timeout.IdleStateHandler;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.ApplicationListener;
import org.springframework.core.env.Environment;
import org.springframework.stereotype.Service;

import java.util.concurrent.RejectedExecutionException;
import java.util.concurrent.TimeUnit;

@Service
@Slf4j
public class NettyClient implements ApplicationListener<ApplicationReadyEvent> {

    private static Channel channel;

    private static ChannelFutureListener channelFutureListener = null;

    public static void connectServer(){
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();
            b.group(group)
                    .channel(NioSocketChannel.class)
                    .option(ChannelOption.TCP_NODELAY, true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            log.info("正在与服务器创建连接...");
                            ch.pipeline()
                            .addLast( new IdleStateHandler(120, 120, 120, TimeUnit.SECONDS))
                            .addLast(new ClientHandler());
                        }
                    });
            Environment env = SpringUtil.getBean(Environment.class);
            String nettyEnv = env.getProperty("netty.env");
            String host = env.getProperty("netty.pro.server.host");
            String port = env.getProperty("netty.pro.server.port");
            if("test".equals(nettyEnv)){
                host = env.getProperty("netty.test.server.host");
                port = env.getProperty("netty.test.server.port");
            }
            ChannelFuture f = b.connect(host, Integer.parseInt(port)).sync();
            f.addListener(new ConnectionListener());
            System.err.println("【连接服务器】 服务器连接创建完成,正在登陆..."+host+"  "+port);
            channel = f.channel();
            sendMessage(getConnectReq());
            System.err.println("【成功登陆服务器】 "+host +" "+ port+"...");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {

        }
    }

    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        connectServer();
    }


    /**
     * 向服务器发送数据
     *
     * @param bytes
     * @throws Exception
     */
    public static void sendMessage(ByteBuf bytes) {
        //重连服务器
        if (channel == null || !channel.isOpen()) {
            connectServer();
        }
        try {
            channel.writeAndFlush(bytes);
        }catch (RejectedExecutionException exception){
            log.error("RejectedExecutionException "+exception.getMessage());
        }

    }

    public static ByteBuf getConnectReq() {
        Integer msgSn = 1;  //消息序列号
        //写入消息体
        Environment env = SpringUtil.getBean(Environment.class);
        Integer userid = Integer.valueOf(env.getProperty("netty.account.userid"));
        String password = env.getProperty("netty.account.password");
        Integer centerid = Integer.valueOf(env.getProperty("netty.account.centerid"));
        System.err.println("【getConnectReq】 服务器连接用户是..."+userid+" "+password+" "+centerid);

        ByteBuf bodyByteBuf = Unpooled.buffer();
        bodyByteBuf.writeInt(userid);
        bodyByteBuf.writeBytes(password.getBytes());
        bodyByteBuf.writeInt(centerid);

        return NettyRequest.encodeReq(msgSn, Constants.CONNECT_REQ, bodyByteBuf);
    }
}
```

### 1.2 clientHandle

```java
package com.gzstrong.netty;

import com.gzstrong.client.handler.Handler;
import com.gzstrong.codec.MsgDecoder;
import com.gzstrong.util.*;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.ByteBufUtil;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.handler.timeout.IdleState;
import io.netty.handler.timeout.IdleStateEvent;
import io.netty.util.ReferenceCountUtil;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.Date;

/**
 * 读取服务器返回的响应信息
 */
@Component
@Slf4j
public class ClientHandler extends ChannelInboundHandlerAdapter {

    @Autowired
    private NettyClient nettyClient;

    /**循环次数 */
    private int fcount = 1;

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        try {
            
    }
   
    /**
     * 心跳请求处理
     * 每4秒发送一次心跳请求;
     *
     */
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object obj) throws Exception {
        if (obj instanceof IdleStateEvent) {
            IdleStateEvent event = (IdleStateEvent) obj;
            if (IdleState.WRITER_IDLE.equals(event.state())) {  //如果写通道处于空闲状态,就发送心跳命令
                System.out.println("【ClientHandler 发送心跳包】 循环请求的时间："+new Date()+"，次数"+fcount);
                ByteBuf heartbeatByteBuf = NettyRequest.encodeReq(1, Constants.LINKTEST_REQ, Unpooled.buffer());
                ctx.channel().writeAndFlush(heartbeatByteBuf);
                fcount++;
            }
        }
    }
}

```

### 1.3  ConnectionListener

```java
package com.gzstrong.netty;

import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelFutureListener;
import io.netty.channel.EventLoop;

import java.util.concurrent.TimeUnit;

class ConnectionListener implements ChannelFutureListener {
    @Override
    public void operationComplete(ChannelFuture channelFuture) throws Exception {
        if (!channelFuture.isSuccess()) {
            final EventLoop loop = channelFuture.channel().eventLoop();
            loop.schedule(() -> {
                System.err.println("【ConnectionListener】 服务端连接不上，开始重连操作...");
                try {
                    NettyClient.connectServer();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }, 3, TimeUnit.SECONDS);
        } else {
            System.err.println("【ConnectionListener】 服务端连接成功...");
        }
    }
}
```

