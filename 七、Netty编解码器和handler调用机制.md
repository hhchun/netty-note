# 基本说明

1. Netty的主要组件有：Channel、EventLoop、ChannelHandler、ChannelPipeline、ChannelFuture等。

2. ChannelHandler主要负责处理数据入站和出站的逻辑。ChannelInboundHandler、ChannelInboundHandlerAdapter负责处理入站，ChannelOutboundHandler、ChannelOutboundHandlerAdapter负责处理出站。

3. ChannelPipeline中包含ChannelHandler链。以客户端程序为例，如果事件的运动方向是从客户端到服务端，那么这些事件为出站，即客户端发送给服务端的数据会通过ChannelPipeline中的一系列ChannelOutboundHandler，并由这些ChannelOutboundHandler处理，反之则为入站。

   ![Handler入站和出站](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/Handler%E5%85%A5%E7%AB%99%E5%92%8C%E5%87%BA%E7%AB%99.svg)

# 编解码器

1. 当Netty发送或接收一个消息市，就会发生一次数据转换。如果入站消息会被解码（从字节码转为另一种格式），如果是出站消息会被编码成字节码。
2. Netty已提供一些列实用的编解码器，这些编解码器都实现了ChannelInboundHandler或ChannelOutboundHandler接口。以入站为例，从Channel中读取消息，channelRead方法会调用decode方法进行解码，并将解码后的数据传递给ChannelPipeline中的下一个ChannelInboundHandler。

# 解码器-ByteToMessageDecoder

## 基本介绍

1. 关系继承图

   ![ByteToMessageDecoder2](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/ByteToMessageDecoder2.png)

2. 由于数据接收方是无法感知接收到的数据是否是一次性发送一个完整的信息，TCP就已可能出现粘包、拆包的问题，这些问题的解决方法，后续的章节会单独进行说明。

## 举例说明

```java
package com.qqs.netty;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.ByteToMessageDecoder;

import java.util.List;

public class ByteToIntegerDecoder extends ByteToMessageDecoder {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        if (in.readableBytes() >= 4) {
            out.add(in.readInt());
        }
    }
}
```

**说明：**

1. 每次入站都从ByteBuf中读取4个字节，将其解码为一个int，然后把它添加到out（List）中。当没有更多的元素可以被添加到out中时，out的内容将会被传递给下一个ChannelInboundHandler。

2. 注意，在调用readint方法前必须先校验ByteBuf是否有足够的数据，不然就会抛出IndexOutOfBoundsException异常。

3. decode执行示意图：

   ![decode执行示意图](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/decode%E6%89%A7%E8%A1%8C%E7%A4%BA%E6%84%8F%E5%9B%BE.svg)

# 使用自定义编解码器对handler链调用机制进行简单说明

## 案例要求

1. 客户端发送long类型数据到服务端，服务端接收并打印到控制台。
2. 服务端发送long类型数据到客户端，客户端接收并打印到控制台。

## 案例代码

```java
// ByteToLongDecoder.java

package com.qqs.netty.codec;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.ByteToMessageDecoder;

import java.util.List;

/**
 * 解码器
 * byte => long
 */
public class ByteToLongDecoder extends ByteToMessageDecoder {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        System.out.println("ByteToLongDecoder decode call");
        if (in.readableBytes() >= 8) {
            out.add(in.readLong());
        }
    }
}
```

```java
// LongToByteEncoder.java

package com.qqs.netty.codec;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.MessageToByteEncoder;

/**
 * 编码器
 * long => type
 */
public class LongToByteEncoder extends MessageToByteEncoder<Long> {

    @Override
    protected void encode(ChannelHandlerContext ctx, Long msg, ByteBuf out) throws Exception {
        System.out.println("ByteToLongEncoder encode call");
        System.out.println("msg = " + msg);
        out.writeLong(msg);
    }
}

```

```java
// NettyCodecHandlerExplainServer.java

package com.qqs.netty.server;

import com.qqs.netty.codec.ByteToLongDecoder;
import com.qqs.netty.codec.LongToByteEncoder;
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;

public class NettyCodecHandlerExplainServer {
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
                            pipeline.addLast(new ByteToLongDecoder());

                            // 编码器
                            pipeline.addLast(new LongToByteEncoder());

                            // 业务处理器
                            pipeline.addLast(new NettyCodecHandlerExplainServerHandler());
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
// NettyCodecHandlerExplainServerHandler.java

package com.qqs.netty.server;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;


public class NettyCodecHandlerExplainServerHandler extends SimpleChannelInboundHandler<Long> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, Long msg) throws Exception {
        System.out.println("receiving client message: " + msg);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(98765L);
        System.out.println("NettyCodecHandlerExplainServerHandler send successful");
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }

}
```

```java
// NettyCodecHandlerExplainClient.java

package com.qqs.netty.client;

import com.qqs.netty.codec.ByteToLongDecoder;
import com.qqs.netty.codec.LongToByteEncoder;
import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.protobuf.ProtobufEncoder;

public class NettyCodecHandlerExplainClient {
    public static void main(String[] args) {
        NioEventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(group)
                    .channel(NioSocketChannel.class)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ChannelPipeline pipeline = ch.pipeline();
                            // 编码器
                            pipeline.addLast(new LongToByteEncoder());

                            // 解码器
                            pipeline.addLast(new ByteToLongDecoder());

                            // 业务处理器
                            pipeline.addLast(new NettyCodecHandlerExplainClientHandler());
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
// NettyCodecHandlerExplainClientHandler.java

package com.qqs.netty.client;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;


public class NettyCodecHandlerExplainClientHandler extends SimpleChannelInboundHandler<Long> {
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(123456L);
        System.out.println("NettyCodecHandlerExplainClientHandler send successful");
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, Long msg) throws Exception {
        System.out.println("client address: " + ctx.channel().remoteAddress());
        System.out.println("client reply message: " + msg);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

## 案例分析说明

**客户端发送long类型数据到服务端，服务端接收并打印到控制台。**

![使用自定义编解码器对handler链调用机制进行简单说明1](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E4%BD%BF%E7%94%A8%E8%87%AA%E5%AE%9A%E4%B9%89%E7%BC%96%E8%A7%A3%E7%A0%81%E5%99%A8%E5%AF%B9handler%E9%93%BE%E8%B0%83%E7%94%A8%E6%9C%BA%E5%88%B6%E8%BF%9B%E8%A1%8C%E7%AE%80%E5%8D%95%E8%AF%B4%E6%98%8E1.svg)

---

**服务端发送long类型数据到客户端，客户端接收并打印到控制台。**

![使用自定义编解码器对handler链调用机制进行简单说明2](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E4%BD%BF%E7%94%A8%E8%87%AA%E5%AE%9A%E4%B9%89%E7%BC%96%E8%A7%A3%E7%A0%81%E5%99%A8%E5%AF%B9handler%E9%93%BE%E8%B0%83%E7%94%A8%E6%9C%BA%E5%88%B6%E8%BF%9B%E8%A1%8C%E7%AE%80%E5%8D%95%E8%AF%B4%E6%98%8E2.svg)

---

**结论：**

1. 不论解码器还是编码器能处理的消息类型必须与待处理的消息类型一致，否则该处理器不会被执行。
2. 在解码器进行数据解码时，需要判断缓存区（ByteBuf）的数据是否足够，否则接收到的结果会期望结果可能不一致。

# 解码器-ReplayingDecoder

1. ReplayingDecoder 扩展 ByteToMessageDecoder，使用 ReplayingDecoder 不必调用readableBytes方法进行校验。

   <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/ReplayingDecoder.png" alt="ReplayingDecoder" style="zoom:50%;" />

2. **泛型S** 可用于指定用户状态管理的类型，其中Void代表不需要状态管理。

3. ReplayingDecoder 使用方便，但它也有一些局限性：

   1. 并不是所有的ByteBuf操作都被支持，如果调用一 个不被支持的方法，将会抛出UnsupportedOperationException异常。
   2. 在某些情况下可能稍慢于ByteToMessageDecoder，例如：网络缓慢并且消息格式复杂时，消息会被拆成多个碎片，速度变慢。

# 其他编解码器

| 编解码器                     | 描述                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| LineBasedFrameDecoder        | 使用行尾控制字符（\n 或者\r\n）作为分隔符来解析数据。        |
| DelimiterBasedFrameDecoder   | 使用自定义的特殊字符作为消息的分隔符。                       |
| HttpObjectDecoder            | HTTP数据解码器。                                             |
| LengthFieldBasedFrameDecoder | 通过指定长度来标识整包消息，这样就可以自动的处理黏包和半包（拆包）消息。 |

# Netty整合 Log4j

## Maven依赖

```xml
<dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j</artifactId>
    <version>1.2.17</version>
</dependency>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.25</version>
</dependency>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-log4j12</artifactId>
    <version>1.7.25</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-simple</artifactId>
    <version>1.7.25</version>
    <scope>test</scope>
</dependency>
```

## 配置文件

```properties
# resources/log4j.properties

log4j.rootLogger=DEBUG, stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=[%p] %C{1} - %m%n
```

