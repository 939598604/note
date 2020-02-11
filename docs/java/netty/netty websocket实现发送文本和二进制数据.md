# netty实现websocket发送文本和二进制数据



## 一.需求：

​    **1、使用 netty 实现 websocket 服务器**

​    **2、实现 文本信息 的传递**

​    **3、实现 二进制 信息的传递，如果是图片则传递到后台后在前台直接显示，非图片提示。（此处的图片和非图片是前端传递到后台的二进制数据然后后端在原封不动的直接返回到前台）**

​    **4、只需要考虑 websocket 协议，不用处理http请求**

 

## 二.实现细节：

###     1、netty中对websocket增强的处理器

​          **WebSocketServerProtocolHandler** 

​              \>> 此处理器可以处理了 **webSocket 协议的握手请求处理**，以及 **Close**、**Ping**、**Pong**控制帧的处理。对于**文本和二进制的数据帧需要我们自己处理**。

​              \>> 如果我们需要**拦截 webSocket 协议握手**完成后的处理，可以实现ChannelInboundHandler#userEventTriggered方法，并判断是否是 **HandshakeComplete** 事件。

​              \>> 参数：**websocketPath** 表示 webSocket 的路径

​              \>> 参数：**maxFrameSize** 表示最大的帧，**如果上传大文件时需要将此值调大**

###     2、文本消息的处理

​               客户端： 直接发送一个字符串即可

​               服务端： 服务端给客户端响应文本数据，需要**返回  TextWebSocketFrame 对象**，否则客户端接收不到。

###     3、二进制消息的处理

​               客户端：向后台传递一个 blob 对象即可，**如果我们需要传递额外的信息**，那么可以在 blob 对象中进行添加，此例中自定义前4个字节表示数据的类型。

​               服务端：处理 BinaryWebSocketFrame 帧，并获取前4个字节，判断是否是图片，然后返回 BinaryWebSocketFrame对象给前台。

###     4、针对二进制消息的自定义协议如下：（此处实现比较简单）

​          **前四个字节表示文件类型**，后面的字节表示具体的数据。

​          在java中一个int是4个字节，在js中使用Int32表示

​          此协议主要是判断前端是否传递的是 图片，如果是图片就直接传递到后台，然后后台在返回二进制数据到前台直接显示这个图片。非图片不用处理。

###      5、js中处理二进制数据

​           见 **webSocket.html** 文件中的处理。

  ## 三.实现步骤

### 1、主要的依赖

```
		<dependency>
			<groupId>io.netty</groupId>
			<artifactId>netty-all</artifactId>
			<version>4.1.45.Final</version>
		</dependency>
```

### 2、webSocket服务端编写

```
package com.example.websocket.picture;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.HttpServerCodec;
import io.netty.handler.codec.http.websocketx.WebSocketFrameAggregator;
import io.netty.handler.codec.http.websocketx.WebSocketServerProtocolHandler;
import io.netty.handler.codec.http.websocketx.extensions.compression.WebSocketServerCompressionHandler;
import io.netty.handler.logging.LogLevel;
import io.netty.handler.logging.LoggingHandler;
import io.netty.handler.stream.ChunkedWriteHandler;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * netty 整合 websocket
 */
public class WebSocketServer {

    private static final Logger log = LoggerFactory.getLogger(WebSocketServer.class);

    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workGroup)
                    .option(ChannelOption.SO_BACKLOG, 128)
                    .childOption(ChannelOption.TCP_NODELAY, true)
                    .childOption(ChannelOption.SO_KEEPALIVE, true)
                    .handler(new LoggingHandler(LogLevel.TRACE))
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline()
                                    .addLast(new LoggingHandler(LogLevel.TRACE))
                                    // HttpRequestDecoder和HttpResponseEncoder的一个组合，针对http协议进行编解码
                                    .addLast(new HttpServerCodec())
                                    // 分块向客户端写数据，防止发送大文件时导致内存溢出， channel.write(new ChunkedFile(new File("bigFile.mkv")))
                                    .addLast(new ChunkedWriteHandler())
                                    // 将HttpMessage和HttpContents聚合到一个完成的 FullHttpRequest或FullHttpResponse中,具体是FullHttpRequest对象还是FullHttpResponse对象取决于是请求还是响应
                                    // 需要放到HttpServerCodec这个处理器后面
                                    .addLast(new HttpObjectAggregator(10240))
                                    // webSocket 数据压缩扩展，当添加这个的时候WebSocketServerProtocolHandler的第三个参数需要设置成true
                                    .addLast(new WebSocketServerCompressionHandler())
                                    // 聚合 websocket 的数据帧，因为客户端可能分段向服务器端发送数据
                                    // https://github.com/netty/netty/issues/1112 https://github.com/netty/netty/pull/1207
                                    .addLast(new WebSocketFrameAggregator(10 * 1024 * 1024))
                                    // 服务器端向外暴露的 web socket 端点，当客户端传递比较大的对象时，maxFrameSize参数的值需要调大
                                    .addLast(new WebSocketServerProtocolHandler("/websocket", null, true, 10485760))
                                    // 自定义处理器 - 处理 web socket 文本消息
                                    .addLast(new TextWebSocketHandler())
                                    // 自定义处理器 - 处理 web socket 二进制消息
                                    .addLast(new BinaryWebSocketFrameHandler());
                        }
                    });
            ChannelFuture channelFuture = bootstrap.bind(9898).sync();
            log.info("webSocket server listen on port : [{}]", 9898);
            channelFuture.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workGroup.shutdownGracefully();
        }
    }
}
```

**注意：**

​          **1、看一下上方依次引入了哪些处理器**

​          2、对于 **webSocket 的握手、Close、Ping、Pong等的处理**，由 WebSocketServerProtocolHandler 已经处理了，我们**自己只需要处理 Text和Binary等数据帧**的处理。

​          3、对于**传递比较大的文件**，需要修改 maxFrameSize 参数。

### 3、自定义处理器握手后和文本消息

```
package com.example.websocket.picture;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.handler.codec.http.websocketx.TextWebSocketFrame;
import io.netty.handler.codec.http.websocketx.WebSocketServerProtocolHandler;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.net.InetSocketAddress;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;

/**
 * 处理 web socket 文本消息
 *
 */
public class TextWebSocketHandler extends SimpleChannelInboundHandler<TextWebSocketFrame> {

    private static final Logger log = LoggerFactory.getLogger(TextWebSocketHandler.class);

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, TextWebSocketFrame msg) {
        log.info("接收到客户端的消息:[{}]", msg.text());
        // 如果是向客户端发送文本消息，则需要发送 TextWebSocketFrame 消息
        InetSocketAddress inetSocketAddress = (InetSocketAddress) ctx.channel().remoteAddress();
        String ip = inetSocketAddress.getHostName();
        String txtMsg = "[" + ip + "][" + LocalTime.now().format(DateTimeFormatter.ofPattern("HH:mm:ss")) + "] ==> " + msg.text();
        ctx.channel().writeAndFlush(new TextWebSocketFrame(txtMsg));
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
        log.error("服务器发生了异常:", cause);
    }

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof WebSocketServerProtocolHandler.HandshakeComplete) {
            log.info("web socket 握手成功。");
            WebSocketServerProtocolHandler.HandshakeComplete handshakeComplete = (WebSocketServerProtocolHandler.HandshakeComplete) evt;
            String requestUri = handshakeComplete.requestUri();
            log.info("requestUri:[{}]", requestUri);
            String subproTocol = handshakeComplete.selectedSubprotocol();
            log.info("subproTocol:[{}]", subproTocol);
            handshakeComplete.requestHeaders().forEach(entry -> log.info("header key:[{}] value:[{}]", entry.getKey(), entry.getValue()));
        } else {
            super.userEventTriggered(ctx, evt);
        }
    }
}
```

**注意：**

​           1、**此处只处理文本消息**，因此 SimpleChannelInboundHandler 中的范型写 **TextWebSocketFrame**

​           2、发送文本消息给客户端，**需要发送 TextWebSocketFrame 对象**，否则客户端接收不到。

​           3、处理 **握手后** 的处理，判断是否是 **HandshakeComplete** 事件。

### 4、处理二进制消息

```
package com.example.websocket.picture;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.handler.codec.http.websocketx.BinaryWebSocketFrame;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * 处理二进制消息
 */
public class BinaryWebSocketFrameHandler extends SimpleChannelInboundHandler<BinaryWebSocketFrame> {
    private static final Logger log = LoggerFactory.getLogger(BinaryWebSocketFrameHandler.class);

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, BinaryWebSocketFrame msg) throws InterruptedException {
        log.info("服务器接收到二进制消息,消息长度:[{}]", msg.content().capacity());
        ByteBuf byteBuf = Unpooled.directBuffer(msg.content().capacity());
        byteBuf.writeBytes(msg.content());
        ctx.writeAndFlush(new BinaryWebSocketFrame(byteBuf));
    }
}
```

**注意：**

​        1、此处**只处理二进制消息**，因此泛型中写 **BinaryWebSocketFrame**

### 5、客户端的写法

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>web socket 测试</title>
</head>
<body>

<div style="width: 600px;height: 400px;">
    <p>服务器输出:</p>
    <div style="border: 1px solid #CCC;height: 300px;overflow: scroll" id="server-msg-container">

    </div>
    <p>
        <textarea id="inp-msg" style="height: 50px;width: 500px"></textarea><input type="button" value="发送"
                                                                                   id="send"><br/>
        选择图片： <input type="file" id="send-pic">
    </p>
</div>

<script type="application/javascript">
    var ws = new WebSocket("ws://127.0.0.1:9898/websocket");
    ws.onopen = function (ev) {

    };
    ws.onmessage = function (ev) {
        console.info("onmessage", ev);
        var inpMsg = document.getElementById("server-msg-container");
        if (typeof ev.data === "string") {
            inpMsg.innerHTML += ev.data + "<br/>";
        } else {
            var result = ev.data;
            var flagReader = new FileReader();
            flagReader.readAsArrayBuffer(result);
            flagReader.onload = function () {
                var imageReader = new FileReader();
                imageReader.readAsDataURL(result);
                console.info("服务器返回的数据大小:", result.size);
                imageReader.onload = function (img) {
                    var imgHtml = "<img src='" + img.target.result + "' style='width: 100px;height: 100px;'>";
                    inpMsg.innerHTML += imgHtml.replace("data:application/octet-stream;", "data:image/png;") + "<br />";
                    inpMsg.scroll(inpMsg.scrollWidth,inpMsg.scrollHeight);
                };
            }
        }
    };
    ws.onerror = function () {
        var inpMsg = document.getElementById("server-msg-container");
        inpMsg.innerHTML += "发生异常" + "<br/>";
    };
    ws.onclose = function () {
        var inpMsg = document.getElementById("server-msg-container");
        inpMsg.innerHTML += "webSocket 关闭" + "<br/>";
    };

    // 发送文字消息
    document.getElementById("send").addEventListener("click", function () {
        ws.send(document.getElementById("inp-msg").value);
    }, false);

    // 发送图片
    document.querySelector('#send-pic').addEventListener('change', function () {
        var files = this.files;
        if (files && files.length) {
            var file = files[0];
            var fileReader = new FileReader();
            fileReader.readAsArrayBuffer(file);
            fileReader.onload = function (e) {
                // 获取到文件对象
                var result = e.target.result;
                // 发送数据到服务器端
                ws.send(result)
            }
        }
    }, false);
</script>
</body>
</html>
```

