# Bootstrap、ServerBootstrap

## 基本介绍

1. Bootstrap 意思是**引导**，一个 Netty 应用通常由一个 Bootstrap 开始，主要作用是配置 Netty 程序，串联各个组件。
2. Netty 中 Bootstrap 类是客户端启动引导类，ServerBootstrap 是服务端启动引导类。

## 常见方法

| 方法                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) | 用于服务端，设置两个EventLoop线程组                          |
| public B group(EventLoopGroup group)                         | 用于客户端，设置一个EventLoop线程组                          |
| public B channel(Class<? extends C> channelClass)            | 用于服务端，设置通道的实现类型                               |
| public < T > B option(ChannelOption option, T value)         | 给 ServerChannel 添加配置                                    |
| public < T > ServerBootstrap childOption(ChannelOption childOption, T value) | 给接收客户端连接的通道添加配置                               |
| public ServerBootstrap childHandler(ChannelHandler childHandler) | 设置业务处理类（通常为自定义的handler），针对workerGroup（childGroup） |
| public ServerBootstrap handler(ChannelHandler handler)       | 设置处理类，针对bossGroup（parentGroup）                     |
| public ChannelFuture bind(int inetPort)                      | 用于服务端，设置监听的端口号                                 |
| public ChannelFuture connect(String inetHost, int inetPort)  | 用于客户端，连接服务器端                                     |

# Future、ChannelFuture

## 基本介绍

1. Netty 中所有的 IO 操作都是**异步**的，不能立刻得知消息是否被正确处理。
2. 但可以等待它执行完成或者直接注册一个监听，具体的实现就是通过 Future 或 ChannelFuture 注册一个监听，当操作执行成功或失败时监听会自动触发注册的监听事件。

## 常见方法

| 方法                 | 描述                            |
| -------------------- | ------------------------------- |
| Channel channel()    | 获取当前正在进行 I/O 操作的通道 |
| ChannelFuture sync() | 等待异步操作执行完毕            |

# Channel

1. Netty 网络通信的组件，能够用于执行网络 I/O 操作。

2. 通过 Channel 可获取当前网络连接的**通道的状态**。

3. 通过 Channel 可获取网络连接的**配置参数**（例如：接收缓冲区大小）。

4. Channel 提供异步的网络 I/O 操作（如：建立连接，读写，绑定端口），异步调用意味着任何 I/O 调用都将立即返回，但不保证在调用结束时所请求的 I/O 操作已完成。

5. 调用立即返回一个 ChannelFuture 实例，通过**注册监听器**到 ChannelFuture 上，可以 I/O 操作成功、失败或取消时通过**回调**的方式通知调用方。

6. 支持关联 I/O 操作与对应的处理程序。

7. 不同协议、不同的阻塞类型的连接都有不同的 Channel 类型与之对应，常用的 Channel 类型：

   | 类型                   | 描述                         |
   | ---------------------- | ---------------------------- |
   | NioSocketChannel       | 异步的客户端 TCP Socket 连接 |
   | NioServerSocketChannel | 异步的服务端 TCP Socket 连接 |
   | NioDatagramChannel     | 异步的 UDP 连接              |
   | NioSctpChannel         | 异步的客户端 Sctp 连接       |
   | NioSctpServerChannel   | 异步的 Sctp 服务端连接       |

   > 这些通道涵盖了 UDP 和 TCP 网络 IO 以及文件 IO。

# Selector

1. Netty 基于 Selector 对象**实现I/O多路复用**，通过 Selector **一个**线程可以监听**多个**连接的 Channel 事件。
2. 当向一个 Selector 中注册 Channel 后，Selector 内部的机制就可以**自动**不断地查询（select）这些注册的 Channel 是否有**已就绪的I/O事件**（例如：可读，可写、网络连接完成等），这样就可以很简单的使用**一个线程管理多个 Channel**。

# ChannelHandler及其实现类

1. ChannelHandler 是一个**接口**，处理 I/O 事件或拦截 I/O 操作，并将其转发到其 ChannelPipeline（业务处理链）中的下一个处理程序（handler）。

2. ChannelHandler 本身并没有提供很多方法，因为这个接口有许多的方法需要实现，使用期间可以**继承它的子类**。

3. ChannelHandler 及其实现类一览图：

   ![ChannelHandler](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/ChannelHandler.png)

   | 类/接口                       | 描述                            |
   | ----------------------------- | ------------------------------- |
   | ChannelInboundHandler         | 用于处理入站 I/O 事件           |
   | ChannelOutboundHandler        | 用于处理出站 I/O 事件           |
   | ChannelInboundHandlerAdapter  | 用于处理入站 I/O 事件（适配器） |
   | ChannelOutboundHandlerAdapter | 用于处理出站 I/O 事件（适配器） |
   | ChannelDuplexHandler          | 用于处理入站和出站 I/O 事件     |

4. 在前面的很多案例中我们经常需要**自定义**一个handler类去**继承**ChannelInboundHandlerAdapter。

   ![image-20230731181837356](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230731181837356.png)

   ---

   通常需要通过重写相应的方法实现业务逻辑，一般都需要重写的方法有：

   | 方法                                                         | 描述                 |
   | ------------------------------------------------------------ | -------------------- |
   | public void channelRead(ChannelHandlerContext ctx, Object msg) | 处理通道就绪事件     |
   | public void channelActive(ChannelHandlerContext ctx)         | 处理通道读取数据事件 |

# Pipeline与ChannelPipeline

## 基本介绍

1. ChannelPipeline 是netty的一个**重点**。

2. ChannelPipeline 是一个 handler 的集合，它负责处理和拦截 inbound 或者 outbound 的事件和操作，相当于一个贯穿 Netty 的链。(也可以这样理解：ChannelPipeline 是保存 ChannelHandler 的 List，用于处理或拦截Channel 的入站事件和出站操作）。

3. ChannelPipeline 实现了一种高级形式的拦截过滤器模式，使开发者可以完全控制事件的处理方式，以及 Channel 中各个的 ChannelHandler 如何相互交互。

4. 在 Netty 中**每个 Channel 都有且仅有一个 ChannelPipeline 与之对应**，它们的组成关系如图：

   ![channel和pipeline的关系](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/channel%E5%92%8Cpipeline%E7%9A%84%E5%85%B3%E7%B3%BB.svg)

> 1. 一个Channel包含一个ChannelPipeline，而ChannelPipeline中又维护一个由ChannelHandlerContext组成的双向链表，并且每个ChannelHandlerContext中又关联一个ChannelHandler。
> 2. 入站事件和出站事件在一个双向链表中，入站事件会从链表head往后传递到最后一个入站的handler，出站事件会从链表tail往前传递到最前一个出站的handler，两种类型的handler互不干扰。

## 常见方法

| 方法                                               | 描述                                   |
| -------------------------------------------------- | -------------------------------------- |
| ChannelPipeline addFirst(ChannelHandler… handlers) | 添加channelHandler到链中的第一个位置   |
| ChannelPipeline addLast(ChannelHandler… handlers)  | 添加channelHandler到链中的最后一个位置 |

![image-20230731214646251](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230731214646251.png)

# ChannelHandlerContext

## 基本介绍

1. 保存 Channel 相关的所有上下文信息，同时关联一个 ChannelHandler 对象。
2. 即 ChannelHandlerContext 中包含一个具体的事件处理器 ChannelHandler， 同 时 ChannelHandlerContext 中也绑定了对应的 Pipeline 和 Channel 的信息，方便对 ChannelHandler 进行调用。

## 常见方法

| 方法                                    | 描述                                                         |
| --------------------------------------- | ------------------------------------------------------------ |
| Channel channel()                       | 获取当前连接的通道                                           |
| ChannelPipeline pipeline()              | 获取通道对应的管道                                           |
| ChannelFuture writeAndFlush(Object msg) | 将数据写入ChannelPipeline中当前ChannelHandler的下一个ChannelHandler进行处理（出站） |
| ChannelOutboundInvoker flush()          | 刷新通道                                                     |
| ChannelFuture close()                   | 关闭通道                                                     |

![image-20230731220158652](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230731220158652.png)

# ChannelOption

1. Netty 在创建 Channel 实例后，一般都需要设置 ChannelOption 参数。

2. **ChannelOption 常见参数**如下：

   | 参数                       | 描述                                                         |
   | -------------------------- | ------------------------------------------------------------ |
   | ChannelOption.SO_BACKLOG   | 对应TCP/IP协议listen函数中的backlog参数，用来初始化服务端可连接队列大小。服务端处理客户端连接请求是顺序处理的，所以同一时间只能处理一个客户端连接。多个客户端请求时，服务端将不能处理的客户端连接请求放到队列中等待处理，backlog参数指定这个等待队列的大小。 |
   | ChannelOption.SO_KEEPALIVE | 设置服务端一直保持连接活跃状态。                             |

# EventLoopGroup与NioEventLoopGroup

## 基本介绍

1. EventLoopGroup 是一组 EventLoop（就是对应线程） 的抽象，Netty 为了更好的利用多核 CPU 资源，一般会有多个 EventLoop同时工作，**每个 EventLoop 维护着一个 Selector 实例**。

2. EventLoopGroup 提供 next 接口，可以从组里面按照一定规则获取其中一个 EventLoop 来处理任务。

3. 在 Netty服务器端编程中，一般都需要提供两个 EventLoopGroup，例如：BossEventLoopGroup和WorkerEventLoopGroup。

   ![image-20230731222000137](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230731222000137.png)

4. 通常一个服务端口即一个 ServerSocketChannel 对应一个 Selector 和一个 EventLoop 线程。

5. 服务端中，BossEventLoopGroup 负责接收客户端的连接并将 SocketChannel 交给 WorkerEventLoopGroup 来进行IO处理，如图：

   ![BossEventLoopGroup与WorkerEventLoopGroup关系图3](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/BossEventLoopGroup%E4%B8%8EWorkerEventLoopGroup%E5%85%B3%E7%B3%BB%E5%9B%BE3.svg)

   > 1. BossEventLoopGroup通常是一个单线程的EventLoop，EventLoop维护着一个注册了ServerSocketChannel的Selector实例。BossEventLoopGroup不断轮询Selector将连接事件分离出来。
   > 2. 通常BossEventLoopGroup监听OP_ACCEPT事件，然后将接收到的SocketChannel交给WorkerEventLoopGroup进行监听处理。
   > 3. WorkerEventLoopGroup会由next选择其中一个EventLoop来将这个SocketChannel注册到其维护的Selector中，并对其后续的IO事件进行处理。

## 常见方法

| 方法                                  | 描述               |
| ------------------------------------- | ------------------ |
| public NioEventLoopGroup()            | 构造方法           |
| public Future<?> shutdownGracefully() | 断开连接，关闭线程 |

![image-20230731225319775](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230731225319775.png)

# Unpooled

## 基本介绍

1. Netty 提供一个专门用来**操作缓冲区**的工具类（Netty 的数据容器）。

2. 相比于NIO的ByteBuffer，Netty提供的ByteBuf不用考虑filp反转去操作数据的读写。ByteBuf对象内部包含一个byte类型的数组。

3. ByteBuf内部维护**readerIndex、writerIndex和capacity属性**，方便对数据的操作。

   ![ByteBuf3](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/ByteBuf3.svg)

## 常见方法

| 方法                                                         | 描述                                         |
| ------------------------------------------------------------ | -------------------------------------------- |
| public static ByteBuf copiedBuffer(CharSequence string, Charset charset) | 通过给定的数据和字符编码创建一个ByteBuf对象  |
| public static ByteBuf copiedBuffer(byte[] array)             | 通过给定字节数组创建一个ByteBuf对象          |
| public static ByteBuf copiedBuffer(ByteBuffer buffer)        | 通过给定ByteBuffer（nio）创建一个ByteBuf对象 |

## 举例说明

**例1：**

```java
// NettyByteBufExampleOne.java

package com.qqs.netty;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;

public class NettyByteBufExampleOne {
    public static void main(String[] args) {
        ByteBuf buffer = Unpooled.buffer(10);
        System.out.println("buffer capacity = " + buffer.capacity());

        for (int i = 0; i < 10; i++) {
            buffer.writeByte(i);
            System.out.println("buffer writable bytes = " + buffer.writableBytes());
        }

        for (int i = 0; i < buffer.capacity(); i++) {
            System.out.println("buffer index " + i + " value = " + buffer.readByte());
            System.out.println("buffer readable bytes = " + buffer.readableBytes());
        }
    }
}
```

**例2：**

```java
// NettyByteBufExampleTwo.java

package com.qqs.netty;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;

import java.nio.charset.StandardCharsets;

public class NettyByteBufExampleTwo {
    public static void main(String[] args) {
        ByteBuf buffer = Unpooled.copiedBuffer("hello,netty", StandardCharsets.UTF_8);
        System.out.println("buffer capacity = " + buffer.capacity());
        if (buffer.hasArray()) {
            byte[] content = buffer.array();
            System.out.println("buffer content = " + new String(content, 0, buffer.readableBytes(), StandardCharsets.UTF_8));
            System.out.println("buffer = " + buffer);
            System.out.println("buffer array offset = " + buffer.arrayOffset());
            System.out.println("buffer reader index = " + buffer.readerIndex());
            System.out.println("buffer writer index = " + buffer.writerIndex());
            System.out.println("buffer capacity = " + buffer.capacity());

            System.out.println("buffer index 0 value = " + buffer.getByte(0));
            System.out.println("buffer readable bytes = " + buffer.readableBytes());

            for (int i = 0; i < buffer.readableBytes(); i++) {
                System.out.println("buffer index " + i + " value = " + (char) buffer.getByte(i));
            }

            System.out.println("buffer 0-4 sequence = " + buffer.getCharSequence(0, 4, StandardCharsets.UTF_8));
            System.out.println("buffer 4-6 sequence = " + buffer.getCharSequence(4, 6, StandardCharsets.UTF_8));

        }
    }
}
```

# Netty快速入门案例（群聊系统）

## 案例要求

1. 编写一个 Netty 群聊系统，实现服务器端和客户端之间的数据简单通讯（非阻塞）。
2. 实现支持多人群聊。
3. 服务端：可监测用户上线、离线，并实现消息转发功能。
4. 客户端：通过 channel 无阻塞发送消息给其它所有用户，同时可以接受其它用户发送的消息（由服务端进行转发）。
5. 目的：进一步理解 Netty 非阻塞网络编程机制。

## 案例代码

```java
// NettyGroupChatServer.java

package com.qqs.netty.server;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;

public class NettyGroupChatServer {
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
                            // 向pipeline添加解码器
                            pipeline.addLast(new StringDecoder());
                            // 向pipeline添加编码器
                            pipeline.addLast(new StringEncoder());
                            // 业务处理
                            pipeline.addLast(new NettyGroupChatServerHandler());

                        }
                    });
            // 启动服务,监听端口
            ChannelFuture channelFuture = bootstrap.bind(3208).sync();
            // 监听关闭
            channelFuture.channel().closeFuture().sync();
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
// NettyGroupChatServerHandler.java

package com.qqs.netty.server;

import io.netty.channel.Channel;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.channel.group.ChannelGroup;
import io.netty.channel.group.DefaultChannelGroup;
import io.netty.util.concurrent.GlobalEventExecutor;

import java.text.SimpleDateFormat;
import java.util.Date;

public class NettyGroupChatServerHandler extends SimpleChannelInboundHandler<String> {

    private static final ChannelGroup CHANNEL_GROUP = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);
    private static final SimpleDateFormat SDF = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        Channel channel = ctx.channel();
        CHANNEL_GROUP.add(channel);
        CHANNEL_GROUP.writeAndFlush("client " + channel.remoteAddress() + " join group chat, date: " + SDF.format(new Date()));
    }

    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
        Channel channel = ctx.channel();
        CHANNEL_GROUP.writeAndFlush("client " + channel.remoteAddress() + " leave group, date: " + SDF.format(new Date()));
        System.out.println("CHANNEL_GROUP size: " + CHANNEL_GROUP.size());
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        Channel channel = ctx.channel();
        System.out.println("client " + channel.remoteAddress() + " online");
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        Channel channel = ctx.channel();
        System.out.println("client " + channel.remoteAddress() + " offline");
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
        Channel channel = ctx.channel();
        CHANNEL_GROUP.forEach(ch -> {
            if (ch == channel) {
                ch.writeAndFlush("self send: " + msg);
            } else {
                ch.writeAndFlush("client " + ch.remoteAddress() + " send" + msg);
            }
        });
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        // 出现异常,关闭通道
        ctx.close();
    }
}
```

```java
// NettyGroupChatClient.java

package com.qqs.netty.client;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.Channel;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;

import java.util.Scanner;

public class NettyGroupChatClient {
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
                            pipeline.addLast(new StringDecoder());
                            pipeline.addLast(new StringEncoder());
                            pipeline.addLast(new NettyGroupChatClientHandler());
                        }
                    });
            ChannelFuture channelFuture = bootstrap.connect("localhost", 3208).sync();
            Channel channel = channelFuture.channel();
            System.out.println(channel.localAddress() + " successfully");

            Scanner scanner = new Scanner(System.in);
            while (scanner.hasNextLine()) {
                String message = scanner.nextLine();
                channel.writeAndFlush(message);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            group.shutdownGracefully();
        }
    }
}
```

```java
// NettyGroupChatClientHandler.java

package com.qqs.netty.client;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;

public class NettyGroupChatClientHandler extends SimpleChannelInboundHandler<String> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
        System.out.println(msg);
    }
}
```

# Netty心跳检测机制案例

## 案例要求

1. 实现Netty心跳检测机制。
2. 当服务端超过3秒没有读操作时，就提示读空闲。
3. 当服务端超过5秒没有写操作时，就提示写空闲。
4. 当服务端超过7秒都没有读操作或写操作时，就提示读写空闲。

## 案例代码

```java
// NettyHeartbeatServer.java

package com.qqs.netty.server;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.timeout.IdleStateHandler;

import java.util.concurrent.TimeUnit;

public class NettyHeartbeatServer {
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
                            // IdleStateHandler: netty提供的处理空闲状态的处理器
                            // 参数说明:
                            // readerIdleTime: 指定多长时间没有读,就会发送心跳包检测连接状态
                            // writerIdleTime: 指定多长时间没有写,就会发送心跳包检测连接状态
                            // allIdleTime: 指定多长时间没有读写,就会发送心跳包检测连接状态
                            // 当IdleStateHandler触发事件后,会将事件传递到管道中的下一个handler的userEventTriggered方法
                            pipeline.addLast(new IdleStateHandler(3, 5, 7, TimeUnit.SECONDS));
                            // 业务处理
                            pipeline.addLast(new NettyHeartbeatServerHandler());
                        }
                    });
            // 启动服务,监听端口
            ChannelFuture channelFuture = bootstrap.bind(3208).sync();
            // 监听关闭
            channelFuture.channel().closeFuture().sync();
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
// NettyHeartbeatServerHandler.java

package com.qqs.netty.server;

import io.netty.channel.Channel;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.handler.timeout.IdleState;
import io.netty.handler.timeout.IdleStateEvent;

public class NettyHeartbeatServerHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent event = (IdleStateEvent) evt;
            IdleState state = event.state();
            Channel channel = ctx.channel();
            if (IdleState.READER_IDLE.equals(state)) {
                System.out.println("client "+ channel.remoteAddress() + " read idle");
            }
            if (IdleState.WRITER_IDLE.equals(state)) {
                System.out.println("client "+ channel.remoteAddress() + " writer idle");
            }
            if (IdleState.ALL_IDLE.equals(state)) {
                System.out.println("client "+ channel.remoteAddress() + " all idle");
            }
        }
    }
}
```

```java
// NettyHeartbeatClient.java

package com.qqs.netty.clien;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.Channel;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;

public class NettyHeartbeatClient {
    public static void main(String[] args) {
        NioEventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(group)
                    .channel(NioSocketChannel.class)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            System.out.println("NettyHeartbeatClient initChannel");
                        }
                    });
            ChannelFuture channelFuture = bootstrap.connect("localhost", 3208).sync();
            Channel channel = channelFuture.channel();
            System.out.println(channel.localAddress() + " successfully");
            channelFuture.channel().closeFuture().sync();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            group.shutdownGracefully();
        }
    }
}
```

# Netty通过WebSocket实现服务端与客户端长连接案例

## 案例要求

1. http协议是无状态的，浏览器与服务端的请求响应一次，下次请求会重新创建连接。
2. 实现基于WebSocket的长连接的全双工的交互，服务端可以发送消息到浏览器。
3. 客户端浏览器和服务端会互相感知，如：服务端关闭。

## 案例代码

```java
// NettyWebSocketServer.java

package com.qqs.netty;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.HttpServerCodec;
import io.netty.handler.codec.http.websocketx.WebSocketServerProtocolHandler;
import io.netty.handler.stream.ChunkedWriteHandler;

public class NettyWebSocketServer {
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
                            // 基于http协议,使用http的编解码器
                            pipeline.addLast(new HttpServerCodec());

                            // 由块的方式写,需要添加ChunkedWriteHandler处理器
                            pipeline.addLast(new ChunkedWriteHandler());

                            // http数据传输过程是分段的,HttpObjectAggregator用于将多个段聚合
                            // 这就是为什么,当浏览器发送大量数据时,会发生多次http请求
                            pipeline.addLast(new HttpObjectAggregator(8192));

                            // websocket的数据是以帧(frame)形式传递的
                            // WebSocketServerProtocolHandler 核心功能是将http协议升级为ws协议,保持长连接,是通过状态码101来完成的
                            pipeline.addLast(new WebSocketServerProtocolHandler("/netty-ws"));

                            // 业务处理器
                            pipeline.addLast(new NettyWebSocketServerHandler());
                        }
                    });
            // 启动服务,监听端口
            ChannelFuture channelFuture = bootstrap.bind(3208).sync();
            // 监听关闭
            channelFuture.channel().closeFuture().sync();
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
// NettyWebSocketServerHandler.java

package com.qqs.netty;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.handler.codec.http.websocketx.TextWebSocketFrame;

import java.text.SimpleDateFormat;
import java.util.Date;

public class NettyWebSocketServerHandler extends SimpleChannelInboundHandler<TextWebSocketFrame> {

    private static final SimpleDateFormat SDF = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, TextWebSocketFrame msg) throws Exception {
        System.out.println("server received message: " + msg.text());
        ctx.writeAndFlush(new TextWebSocketFrame("server date: " + SDF.format(new Date()) + " message: " + msg.text()));
    }

    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        System.out.println("handlerAdded call, channel id" + ctx.channel().id().asLongText());
    }

    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
        System.out.println("handlerRemoved call, channel id" + ctx.channel().id().asLongText());
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        System.out.println(cause.getMessage());
        ctx.close();
    }
}
```

## 案例运行

使用postman、apipost等工具进行连接测试。

![image-20230805011245268](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230805011245268.png)

![image-20230805011259751](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230805011259751.png)
