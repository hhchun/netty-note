# TCP粘包和拆包基本介绍

1. TCP 是面向连接的，面向流的，提供高可靠性服务。收发两端（客户端和服务器端）都要有成对的 socket。因此，发送端为了将多个数据包更有效的发给接收端，使用优化方法（Nagle 算法），将多次间隔较小且数据量小的数据，合并成一个大的数据块，然后进行封包。这样做虽然提高了效率，但是接收端就难于分辨出完整的数据包，因为面向流的通信是无消息保护边界的。

2. 由于 TCP 无消息保护边界，需要在接收端处理消息边界问题，也就是粘包、拆包问题。

   ![TCP粘包、拆包3](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/TCP%E7%B2%98%E5%8C%85%E3%80%81%E6%8B%86%E5%8C%853.svg)

   > 假设客户端分别发送两个数据包 D1 和 D2 给服务端，由于服务端一次读取到字节数是不确定的，故可能存在以下4种情况：
   >
   > 1. 服务端分两次读取到两个独立的数据包，分别是 D1 和 D2，没有出现粘包或拆包问题。
   > 2. 服务端一次接受到两个数据包，D1 和 D2 粘合在一起，这就出现了粘包问题。
   > 3. 服务端分两次读取到数据包，第一次读取到完整的 D1 包和 D2 包的部分内容D2_1，第二次读取到 D2 包的剩余内容，这就出现了拆包问题。
   > 4. 服务端分两次读取到数据包，第一次读取到 D1 包的部分内容 D1_1，第二次读取到 D1 包的剩余部分内容 D1_2 和完整的 D2 包，这也是出现了拆包问题。

# TCP粘包和拆包现象案例

## 案例代码

```java
// NettyStickyUnpackPackageServer.java

package com.qqs.netty.server;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;

public class NettyStickyUnpackPackageServer {
    public static void main(String[] args) {
        NioEventLoopGroup bossGroup = new NioEventLoopGroup(1);
        NioEventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 128)
                    .childOption(ChannelOption.SO_KEEPALIVE, true)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast(new NettyStickyUnpackPackageServerHandler());
                        }
                    });
            ChannelFuture cf = bootstrap.bind(3208).sync();
            cf.channel().closeFuture().sync();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

```java
// NettyStickyUnpackPackageServerHandler.java

package com.qqs.netty.server;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

import java.nio.charset.StandardCharsets;
import java.util.UUID;

public class NettyStickyUnpackPackageServerHandler extends ChannelInboundHandlerAdapter {

    private int count = 0;

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        for (int i = 0; i < 10; i++) {
            ByteBuf request = Unpooled.copiedBuffer("hello,client " + i, StandardCharsets.UTF_8);
            ctx.writeAndFlush(request);
        }
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("server receive message: " + ((ByteBuf) msg).toString(StandardCharsets.UTF_8));
        System.out.println("server receive count: " + (++count));

        // 回复给客户端一个随机ID
        ByteBuf response = Unpooled.copiedBuffer(UUID.randomUUID().toString(), StandardCharsets.UTF_8);
        ctx.writeAndFlush(response);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        System.out.println("server read complete");
        ByteBuf response = Unpooled.copiedBuffer("server read complete", StandardCharsets.UTF_8);
        ctx.writeAndFlush(response);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

```java
// NettyStickyUnpackPackageClient.java

package com.qqs.netty.client;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;

public class NettyStickyUnpackPackageClient {
    public static void main(String[] args) {
        NioEventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(group)
                    .channel(NioSocketChannel.class)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            ChannelPipeline pipeline = socketChannel.pipeline();
                            pipeline.addLast(new NettyStickyUnpackPackageClientHandler());
                        }
                    });
            ChannelFuture channelFuture = bootstrap.connect("127.0.0.1", 3208).sync();
            channelFuture.channel().closeFuture().sync();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            group.shutdownGracefully();
        }
    }
}
```

```java
// NettyStickyUnpackPackageClientHandler.java

package com.qqs.netty.client;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

import java.nio.charset.StandardCharsets;

public class NettyStickyUnpackPackageClientHandler extends ChannelInboundHandlerAdapter {

    private int count = 0;

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        for (int i = 0; i < 10; i++) {
            ByteBuf request = Unpooled.copiedBuffer("hello,server " + i, StandardCharsets.UTF_8);
            ctx.writeAndFlush(request);
        }
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("client receive message: " + ((ByteBuf) msg).toString(StandardCharsets.UTF_8));
        System.out.println("client receive count: " + (++count));
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

## 运行结果

![image-20230806041659677](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230806041659677.png)

![image-20230806041748638](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230806041748638.png)

# TCP粘包和拆包解决方案

## 解决方案说明

1. 使用自定义协议和解码器来解决。
2. 关键就是要解决接收端每次读取数据长度的问题，解决这个问题就能解决接收端多读或少读的问题，从而避免TCP粘包、拆包问题。

## 案例说明

```java
// MessageProtocol.java

package com.qqs.netty.common;

public class MessageProtocol {
    private int len;
    private byte[] content;

    public int getLen() {
        return len;
    }

    public void setLen(int len) {
        this.len = len;
    }

    public byte[] getContent() {
        return content;
    }

    public void setContent(byte[] content) {
        this.content = content;
    }
}
```

```java
// ByteToMessageProtocolDecoder.java

package com.qqs.netty.common;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.ByteToMessageDecoder;

import java.util.List;

public class ByteToMessageProtocolDecoder extends ByteToMessageDecoder {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        System.out.println("ByteToMessageProtocolDecoder decode call");
        int len = in.readInt();
        byte[] content = new byte[len];
        in.readBytes(content);
        MessageProtocol messageProtocol = new MessageProtocol();
        messageProtocol.setLen(len);
        messageProtocol.setContent(content);
        out.add(messageProtocol);
    }
}
```

```java
// MessageProtocolToByteEncoder.java

package com.qqs.netty.common;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.MessageToByteEncoder;


public class MessageProtocolToByteEncoder extends MessageToByteEncoder<MessageProtocol> {
    @Override
    protected void encode(ChannelHandlerContext ctx, MessageProtocol msg, ByteBuf out) throws Exception {
        System.out.println("MessageProtocolToByteEncoder encode call");
        if (msg == null || msg.getLen() < 1 ||
                msg.getContent() == null || msg.getContent().length < 1) {
            System.out.println("msg == null or msg is empty");
            return;
        }
        out.writeInt(msg.getLen());
        out.writeBytes(msg.getContent());
    }
}
```

```java
// NettyStickyUnpackPackageSolveServer.java

package com.qqs.netty.server;

import com.qqs.netty.common.ByteToMessageProtocolDecoder;
import com.qqs.netty.common.MessageProtocolToByteEncoder;
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;

public class NettyStickyUnpackPackageSolveServer {
    public static void main(String[] args) {
        NioEventLoopGroup bossGroup = new NioEventLoopGroup(1);
        NioEventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 128)
                    .childOption(ChannelOption.SO_KEEPALIVE, true)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ChannelPipeline pipeline = ch.pipeline();
                            // 解码器
                            pipeline.addLast(new ByteToMessageProtocolDecoder());
                            // 编码器
                            pipeline.addLast(new MessageProtocolToByteEncoder());
                            // 业务处理器
                            pipeline.addLast(new NettyStickyUnpackPackageSolveServerHandler());
                        }
                    });
            ChannelFuture cf = bootstrap.bind(3208).sync();
            cf.channel().closeFuture().sync();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

```java
// NettyStickyUnpackPackageSolveServerHandler.java

package com.qqs.netty.server;

import com.qqs.netty.common.MessageProtocol;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

import java.nio.charset.StandardCharsets;
import java.util.UUID;

public class NettyStickyUnpackPackageSolveServerHandler extends ChannelInboundHandlerAdapter {

    private int count = 0;

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        MessageProtocol messageProtocol = (MessageProtocol) msg;
        System.out.println("server receive message len: " + messageProtocol.getLen());
        System.out.println("server receive message content: " + new String(messageProtocol.getContent()));
        System.out.println("server receive count: " + (++count));

        // 回复给客户端一个随机ID
        byte[] responseContent = UUID.randomUUID().toString().getBytes();
        int responseLen = responseContent.length;
        MessageProtocol responseMessageProtocol = new MessageProtocol();
        responseMessageProtocol.setLen(responseLen);
        responseMessageProtocol.setContent(responseContent);
        ctx.writeAndFlush(responseMessageProtocol);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

```java
// NettyStickyUnpackPackageSolveClient.java

package com.qqs.netty.client;

import com.qqs.netty.common.ByteToMessageProtocolDecoder;
import com.qqs.netty.common.MessageProtocolToByteEncoder;
import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;

public class NettyStickyUnpackPackageSolveClient {
    public static void main(String[] args) {
        NioEventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(group)
                    .channel(NioSocketChannel.class)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            ChannelPipeline pipeline = socketChannel.pipeline();
                            // 编码器
                            pipeline.addLast(new MessageProtocolToByteEncoder());
                            // 解码器
                            pipeline.addLast(new ByteToMessageProtocolDecoder());
                            // 业务处理器
                            pipeline.addLast(new NettyStickyUnpackPackageSolveClientHandler());
                        }
                    });
            ChannelFuture channelFuture = bootstrap.connect("127.0.0.1", 3208).sync();
            channelFuture.channel().closeFuture().sync();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            group.shutdownGracefully();
        }
    }
}
```

```java
// NettyStickyUnpackPackageSolveClientHandler.java

package com.qqs.netty.client;

import com.qqs.netty.common.MessageProtocol;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

import java.nio.charset.StandardCharsets;

public class NettyStickyUnpackPackageSolveClientHandler extends ChannelInboundHandlerAdapter {

    private int count = 0;

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        for (int i = 0; i < 10; i++) {
            byte[] content = ("hello,server " + i).getBytes(StandardCharsets.UTF_8);
            int len = content.length;
            MessageProtocol messageProtocol = new MessageProtocol();
            messageProtocol.setLen(len);
            messageProtocol.setContent(content);
            ctx.writeAndFlush(messageProtocol);
        }
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        MessageProtocol messageProtocol = (MessageProtocol) msg;
        System.out.println("client receive message len: " + messageProtocol.getLen());
        System.out.println("client receive message content: " + new String(messageProtocol.getContent(), StandardCharsets.UTF_8));
        System.out.println("client receive count: " + (++count));
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

