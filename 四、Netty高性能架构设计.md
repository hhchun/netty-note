# 线程模型的基本介绍

1. 不同的线程模式，对程序的性能有很大影响，为了搞清 Netty 线程模式，先系统的讲解下各个线程模式，最后看看 Netty 线程模型有什么优势。
2. 目前存在的线程模型有：
   1. 传统阻塞I/O服务模型。
   2. Reactor模型。
3. 根据Reactor的数量和处理资源线程的数量不同，有三种典型的实现：
   1. 单Reactor单线程。
   2. 但Reactor多线程。
   3. 主从Reactor多线程。

4. Netty模型：

   Netty模型主要基于主从 Reactor 多线程模型做了一定的改进，其中主从 Reactor 多线程模型有多个 Reactor。

# 传统阻塞I/O服务模型

## 工作原理图

![传统阻塞I:O服务模型](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E4%BC%A0%E7%BB%9F%E9%98%BB%E5%A1%9EI:O%E6%9C%8D%E5%8A%A1%E6%A8%A1%E5%9E%8B.svg)

## 模型特点

1. 采用阻塞I/O模式获取输入的数据。
2. 每个连接都需要独立的线程完成数据的输入，业务处理，数据返回。

## 问题分析

1. 当并发数很大，就会创建大量的线程，**占用很大系统资源**。
2. 连接创建后，如果当前线程暂时没有数据可读，该线程会阻塞在 read 操作，造成线程资源浪费。

# Reactor模式

## 解决方案：针对传统阻塞I/O服务模型的2个缺点

1. **基于I/O复用模型**：
   1. 多个连接共用一个阻塞对象，应用程序只需要在一个阻塞对象等待，无需阻塞等待所有连接。
   2. 当某个连接有新的数据可以处理时，操作系统通知应用程序，线程从阻塞状态返回，开始进行业务处理
   3. Reactor对应的叫法有：反应器模式、分发者模式（Dispatcher）、通知者模式（notifier）。

2. **基于线程池复用线程资源**：

   不必再为每个连接创建线程，将连接完成后的业务处理任务分配给线程进行处理，**一个线程可以处理多个连接**。

   ![基于线程池复用线程资源](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E5%9F%BA%E4%BA%8E%E7%BA%BF%E7%A8%8B%E6%B1%A0%E5%A4%8D%E7%94%A8%E7%BA%BF%E7%A8%8B%E8%B5%84%E6%BA%90.svg)

## I/O 复用结合线程池：Reactor 模式基本设计思想

![Reactor模式基本设计思想](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/Reactor%E6%A8%A1%E5%BC%8F%E5%9F%BA%E6%9C%AC%E8%AE%BE%E8%AE%A1%E6%80%9D%E6%83%B3.svg)

上图说明：

1. Reactor 模式，通过一个或多个输入同时传递给服务处理器的模式（**基于事件驱动**）。
2. 服务器端程序处理传入的多个请求，并将它们同步分派到相应的处理线程， **因此Reactor 模式也叫Dispatcher模式**。
3. Reactor模式使用**I/O复用监听事件**，收到事件后，**分发给某个线程（进程）**， 这点就是网络服务器高并发处理关键。

## Reactor模式核心组成

1. **Reactor**：Reactor在一个单独的线程中运行，负责监听和分发事件，分发给适当的处理程序来对 IO 事件做出反应。它就像公司的电话接线员，它接听来自客户的电话并将线路转移到适当的联系人。
2. **Handlers**：
   * **处理程序执行 I/O 事件要完成的实际事件**，类似于客户想要与之交谈的公司中的实际官员。
   * Reactor通过调度适当的处理程序来响应I/O事件，处理程序执行**非阻塞**操作。

## Reactor模式分类

根据Reactor的数量和处理资源线程的数量不同，有3种典型的实现：单Reactor单线程、单Reactor多线程、主从 Reactor 多线程。

# 单Reactor单线程

## 原理图

![单Reactor单线程](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E5%8D%95Reactor%E5%8D%95%E7%BA%BF%E7%A8%8B.svg)

## 说明

1. Select 是前面 I/O 复用模型介绍的标准网络编程 API，可以实现应用程序通过一个阻塞对象监听多路连接请求。
2. Reactor 对象通过 Select 监控客户端请求事件，收到事件后通过 Dispatch 进行分发。
3. 如果是建立连接请求事件，则由 Acceptor 通过 Accept 处理连接请求，然后创建一个 Handler 对象处理连接完成后的后续业务处理。
4. 如果不是建立连接事件，则 Reactor 会分发调用连接对应的 Handler 来处理响应。
5. Handler 会完成 Read→业务处理→Send 的完整业务流程 →再返回给Client。
6. 结论：服务器端用一个线程通过多路复用搞定所有的 IO 操作（包括连接，读、写等），编码简单，清晰明了，但是如果客户端连接数量较多，将无法支撑。前面的NIO案例就属于这种模型：[NIO网络编程案例（群聊系统）](./二、Java NIO#NIO网络编程案例（群聊系统）)。
7. 使用场景：客户端的数量有限，业务处理非常快速，比如 Redis 在业务处理的时间复杂度 O(1) 的情况。

## 优缺点

**优点**：

1. 模型简单，没有多线程、进程通信、竞争的问题，全部都在一个线程中完成。

---

**缺点**：

1. **性能问题**，只有一个线程，无法完全发挥多核 CPU 的性能。Handler 在处理某个连接上的业务时，整个进程无法处理其他连接事件，很容易导致性能瓶颈。
2. **可靠性问题**，线程意外终止，或者进入死循环，会导致整个系统通信模块不可用，不能接收和处理外部消息，造成节点故障。

# 单Reactor多线程

## 原理图

![单Reactor多线程](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E5%8D%95Reactor%E5%A4%9A%E7%BA%BF%E7%A8%8B.svg)

> Handler 将具体的业务处理 Worker 线程池分层出去，并通过开辟新的线程去完成。

## 说明

1. Reactor 对象通过 select 监控客户端请求事件，收到事件后，通过 dispatch 进行分发。
2. 如果建立连接请求，则右 Acceptor 通过accept 处理连接请求， 然后创建一个 Handler 对象处理完成连接后的各种事件。
3. 如果不是连接请求，则由 reactor 分发调用连接对应的 handler 来处理。
4. handler 只负责响应事件，不做具体的业务处理, 通过 read 读取数据后，会分发给后面的 worker 线程池的某个线程处理业务。
5. worker 线程池会分配独立线程完成真正的业务，并将结果返回给 handler。
6. handler 收到响应后，通过 send 将结果返回给 client。

## 优缺点

**优点**：

1. 以充分的利用多核 cpu 的处理能力。

---

**缺点**：

1. 多线程数据共享和访问比较**复杂**， reactor 处理所有的事件的监听和响应。
2. 在**单线程运行**，在高并发场景容易出现性能瓶颈。

# 主从Reactor多线程

## 原理图

![主从Reactor多线程](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E4%B8%BB%E4%BB%8EReactor%E5%A4%9A%E7%BA%BF%E7%A8%8B.svg)

> 1. 多加了一层派发层并采用新开线程（Reactor子线程，SubReactor），独立分为三层。
> 2. 针对单 Reactor 多线程模型中，Reactor 在单线程中运行，高并发场景下容易成为性能瓶颈，可以让 Reactor 在多线程中运行。

## 说明

1. Reactor 主线程 MainReactor 对象通过 select 监听连接事件，收到事件后，通过 Acceptor 处理连接事件。
2. 当 Acceptor 处理连接事件后，MainReactor 将连接分配给 SubReactor。
3. subreactor 将连接加入到连接队列进行监听，并创建 handler 进行各种事件处理。
4. 当有新事件发生时，subreactor 就会调用对应的 handler 处理。
5. handler 通过 read 读取数据，分发给后面的 worker 线程处理。
6. worker 线程池分配独立的 worker 线程进行业务处理，并返回结果。
7. handler 收到响应的结果后，再通过 send 将结果返回给 client。
8. Reactor 主线程可以对应多个 Reactor 子线程，即 MainRecator 可以关联多个 SubReactor。
9. 这种模型在许多项目中广泛使用，包括 Nginx 主从 Reactor 多进程模型，Memcached 主从多线程，Netty 主从多线程模型的支持。

## Scalable IO in Java对Multiple Reactors的原理图

![20210130132155683](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/20210130132155683.png)

## 优缺点

**优点**：

1. 父线程与子线程的数据交互简单职责明确，父线程只需要接收新连接，子线程完成后续的业务处理。
2. 父线程与子线程的数据交互简单，Reactor 主线程只需要把新连接传给子线程，子线程无需返回数据。

---

**缺点**：

1. 编程复杂度较高。

# Reactor总结

## 3种模式用生活案例来理解

1. 单 Reactor 单线程：前台接待员和服务员是同一个人，全程为顾客服。
2. 单 Reactor 多线程：1 个前台接待员，多个服务员，接待员只负责接待。
3. 主从 Reactor 多线程：多个前台接待员，多个服务生。

## Reactor 模式的优点

2. **响应快**，不必为单个同步时间所阻塞，虽然 Reactor 本身依然是同步的。
3. 可以最大程度的**避免复杂的多线程及同步问题**。
4. **避免了多线程/进程的切换开销**。
5. **扩展性好**，可以方便的通过增加 Reactor 实例个数来充分利用 CPU 资源。
6. **复用性好**，Reactor 模型本身与具体事件处理逻辑无关，具有很高的复用性。

# Netty模型

## 工作原理（简单版）

### 原理图

![工作原理图（简单版）1](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%9B%BE%EF%BC%88%E7%AE%80%E5%8D%95%E7%89%88%EF%BC%891.svg)

> Netty 主要基于主从 Reactors 多线程模型做了一定的改进，其中主从 Reactor 多线程模型有多个 Reactor。

### 原理图说明

1. BossGroup 线程维护 Selector ，只关注 Accecpt。
2. 当接收到 Accept 事件，获取到对应的 SocketChannel，封装成 NIOScoketChannel 并注册到 Worker 线程（事件循环）, 并进行维护。
3. 当 Worker 线程监听到 selector 中通道发生自己感兴趣的事件后，就进行处理。

## 工作原理（进阶版、详细版）

### 原理图

**进阶版**：

![工作原理图（进阶版）](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%9B%BE%EF%BC%88%E8%BF%9B%E9%98%B6%E7%89%88%EF%BC%89.svg)

---

**详细版**：

![工作原理图（详细版）](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%9B%BE%EF%BC%88%E8%AF%A6%E7%BB%86%E7%89%88%EF%BC%89.svg)

### 原理图说明

1. Netty 抽象出两组线程池，BossGroup 专门负责接收客户端的连接，WorkerGroup 专门负责网络的读写。
2. BossGroup 和 WorkerGroup 类型都是 NioEventLoopGroup。
3. NioEventLoopGroup 相当于一个事件循环组，这个组中含有多个事件循环 ，每一个事件循环是 NioEventLoop。
4. NioEventLoop 表示一个不断循环的执行处理任务的线程，每个 NioEventLoop 都有一个selector，用于监听绑定在其上的 socket 的网络通讯。
5. NioEventLoopGroup 可以有多个线程，即可以含有多个 NioEventLoop。
6. 每个 Boss NioEventLoop 循环执行的步骤有 3 步：
   1. 轮询 accept 事件。
   2. 处理 accept 事件，与 client 建立连接，生成 NioScocketChannel，并将其注册到某个 worker NIOEventLoop 上的 selector处理任务队列的任务，即 runAllTasks。
7. 每个 Worker NIOEventLoop 循环执行的步骤：
   1. 轮询 read，write 事件。
   2. 处理I/)事件，即 read / write 事件，在对应 NioScocketChannel 处理。
   3. 处理任务队列的任务 ， 即 runAllTasks。
8. 每个Worker NIOEventLoop 处理业务时，会使用pipeline（管道），pipeline 中包含channel，即通过 pipeline 可以获取到对应通道，管道中维护了很多处理器。

## Netty快速入门案例（TCP服务）

### 案例要求

1. Netty服务端监听3208端口，
2. 客户端发送 ”hello,netty server “ 到服务端。
3. 服务端回复 “hello,client”。

4. 目的：对Netty模型有一个初步认识，便于理解Netty模型理论。

### 案例代码

使用maven构建项目。

**pom.xml**：

```xml
<dependency>
  <groupId>io.netty</groupId>
  <artifactId>netty-all</artifactId>
  <version>4.1.86.Final</version>
</dependency>
```

---

**服务端**：

```java
// NettyTcpServer.java

package com.qqs.netty.server;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;

public class NettyTcpServer {
    public static void main(String[] args) {

        // 创建两个线程组 bossGroup 和 workerGroup
        // bossGroup只处理连接请求,由 workerGroup 完成对客户端的业务处理
        // 两个都是无限循环
        // bossGroup 和 workerGroup 的子线程（NioEventLoop）个数,默认线程数为: 实际cpu核心数 * 2
        NioEventLoopGroup bossGroup = new NioEventLoopGroup(1);
        NioEventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            // 设置两个线程组
            bootstrap.group(bossGroup, workerGroup)
                    // 使用NioServerSocketChannel作为服务端的通道实现
                    .channel(NioServerSocketChannel.class)
                    // 设置线程队列得到连接个数
                    .option(ChannelOption.SO_BACKLOG, 128)
                    // 设置保持活动连接状态
                    .childOption(ChannelOption.SO_KEEPALIVE, true)
                    // 设置通道初始化对象
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            // 给pipeline添加处理器
                            socketChannel.pipeline().addLast(new NettyTcpServerHandler());
                        }
                    });
            // 绑定端口,启动服务器
            ChannelFuture cf = bootstrap.bind(3208).sync();
            // 对关闭通道进行监听
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
// NettyTcpServerHandler.java

package com.qqs.netty.server;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.CharsetUtil;

public class NettyTcpServerHandler extends ChannelInboundHandlerAdapter {

    /**
     * 读取客户端发送的消息
     *
     * @param ctx 上下文对象,包含: 管道(pipeline)、通道(channel)、地址等
     * @param msg 客户端发送的数据,默认Object
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("ctx: " + ctx);
        System.out.println("client msg: " + ((ByteBuf) msg).toString(CharsetUtil.UTF_8));
        System.out.println("client address: " + ctx.channel().remoteAddress());
    }

    /**
     * 数据读写完毕
     * @param ctx 上下文对象
     */
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        // 将数据写入到缓存,并刷新缓存
        ctx.writeAndFlush(Unpooled.copiedBuffer("hello,client", CharsetUtil.UTF_8));
    }

    /**
     * 异常处理
     * @param ctx 上下文对象
     * @param cause 错误/异常
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

---

**客户端**：

```java
// NettyTcpClient.java

package com.qqs.netty.client;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;

public class NettyTcpClient {
    public static void main(String[] args) {
        NioEventLoopGroup group = new NioEventLoopGroup();
        try {
            // 客户端启动对象
            // 注意: 客户端使用的是Bootstrap而不是ServerBootstrap
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(group)
                    .channel(NioSocketChannel.class)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            socketChannel.pipeline().addLast(new NettyTcpClientHandler());
                        }
                    });
            ChannelFuture channelFuture = bootstrap.connect("127.0.0.1", 3208);
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
// NettyTcpClientHandler.java
package com.qqs.netty.client;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

import java.nio.charset.StandardCharsets;

public class NettyTcpClientHandler extends ChannelInboundHandlerAdapter {

    /**
     * 当通道就绪就会触发此方法
     */
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        System.out.println("cxt:  " + ctx);
        ctx.writeAndFlush(Unpooled.copiedBuffer("hello,server", StandardCharsets.UTF_8));
    }

    /**
     * 当通道有读事件时触发此方法
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        if (msg instanceof ByteBuf) {
            System.out.println("server reply: " + ((ByteBuf) msg).toString(StandardCharsets.UTF_8));
            System.out.println("server address: " + ctx.channel().remoteAddress());
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}
```

## 任务队列Task三种典型使用场景

**本章节中的举例代码是基于[Netty快速入门案例-TCP服务](#Netty快速入门案例（TCP服务）)进行修改的。**

### 自定义普通任务

```java
// NettyServerCommonTaskHandler.java

package com.qqs.netty.server.taskqueue;

import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.CharsetUtil;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.TimeUnit;

public class NettyServerCommonTaskHandler extends ChannelInboundHandlerAdapter {
    // 格式化时间
    private final static SimpleDateFormat SDF = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    /**
     * 读取客户端发送的消息
     *
     * @param ctx 上下文对象,包含: 管道(pipeline)、通道(channel)、地址等
     * @param msg 客户端发送的数据,默认Object
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        // 比如这里有一个非常耗时的业务，提交到channel对应的NioEventLoop的taskQueue中
        ctx.channel().eventLoop().execute(() -> {
            try {
                Thread.sleep(TimeUnit.SECONDS.toMillis(5));
                ctx.writeAndFlush(Unpooled.copiedBuffer("hello,client delay 5 seconds,"
                        + SDF.format(new Date()), CharsetUtil.UTF_8));
                System.out.println("channel hashCode: " + ctx.channel().hashCode());
                System.out.println("task current thread id: " + Thread.currentThread().getId());
                System.out.println("task current thread name: " + Thread.currentThread().getName());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        ctx.channel().eventLoop().execute(() -> {
            try {
                Thread.sleep(TimeUnit.SECONDS.toMillis(3));
                ctx.writeAndFlush(Unpooled.copiedBuffer("hello,client delay 3 seconds,"
                        + SDF.format(new Date()), CharsetUtil.UTF_8));
                System.out.println("channel hashCode: " + ctx.channel().hashCode());
                System.out.println("task current thread id: " + Thread.currentThread().getId());
                System.out.println("task current thread name: " + Thread.currentThread().getName());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        System.out.println("current thread id: " + Thread.currentThread().getId());
        System.out.println("current thread name: " + Thread.currentThread().getName());
    }
}
```

> 注意：handler需要添加到pipeline中才有效。
>
> ![image-20230730222129876](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230730222129876.png)

---

**运行结论**：

<img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230730222257092.png" alt="image-20230730222257092" style="zoom: 67%;" />

<img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230730222420005.png" alt="image-20230730222420005" style="zoom:60%;" />

### 自定义定时任务

```java
// NettyServerScheduleTaskHandler.java

package com.qqs.netty.server.taskqueue;

import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.CharsetUtil;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.TimeUnit;

public class NettyServerScheduleTaskHandler extends ChannelInboundHandlerAdapter {
    // 格式化时间
    private final static SimpleDateFormat SDF = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    /**
     * 读取客户端发送的消息
     *
     * @param ctx 上下文对象,包含: 管道(pipeline)、通道(channel)、地址等
     * @param msg 客户端发送的数据,默认Object
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        // 比如这里有一个非常耗时的业务，提交到channel对应的NioEventLoop的scheduledTaskQueue中
        ctx.channel().eventLoop().schedule(() -> {
            try {
                Thread.sleep(TimeUnit.SECONDS.toMillis(5));
                ctx.writeAndFlush(Unpooled.copiedBuffer("hello,client delay 5 seconds,"
                        + SDF.format(new Date()), CharsetUtil.UTF_8));
                System.out.println("channel hashCode: " + ctx.channel().hashCode());
                System.out.println("task current thread id: " + Thread.currentThread().getId());
                System.out.println("task current thread name: " + Thread.currentThread().getName());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, 5, TimeUnit.SECONDS);

        ctx.channel().eventLoop().schedule(() -> {
            try {
                Thread.sleep(TimeUnit.SECONDS.toMillis(3));
                ctx.writeAndFlush(Unpooled.copiedBuffer("hello,client delay 3 seconds,"
                        + SDF.format(new Date()), CharsetUtil.UTF_8));
                System.out.println("channel hashCode: " + ctx.channel().hashCode());
                System.out.println("task current thread id: " + Thread.currentThread().getId());
                System.out.println("task current thread name: " + Thread.currentThread().getName());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, 3, TimeUnit.SECONDS);
        System.out.println("current thread id: " + Thread.currentThread().getId());
        System.out.println("current thread name: " + Thread.currentThread().getName());
    }
}
```

> 注意：handler需要添加到pipeline中才有效。
>
> ![image-20230730224218913](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230730224218913.png)

**运行结论**：

<img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230730224140158.png" alt="image-20230730224140158" style="zoom:67%;" />

<img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230730224354891.png" alt="image-20230730224354891" style="zoom:55%;" />

### 非当前Reactor线程调用Channel的各种方法

**例如**：推送系统的业务线程里，根据用户标识找到对应的Channel引用，然后调用write方法向该用户推送消息，最终的write会提交到任务队列中被异步消费处理。

## Netty模型再说明

1. Netty 抽象出两组线程池，BossGroup 专门负责接收客户端连接，WorkerGroup 专门负责网络读写操作。
2. NioEventLoop 表示一个不断循环执行处理任务的线程，每个 NioEventLoop 都有一个 selector，用于监听绑定在其上的 socket 网络通道。

3. NioEventLoop 内部采用串行化设计，从消息的读取->解码->处理->编码->发送，始终由 IO 线程 NioEventLoop负责。
   1. NioEventLoopGroup 下包含多个 NioEventLoop。
   2. 每个 NioEventLoop 中包含有一个 selector，一个 taskQueue。
   3. 每个 NioEventLoop 的 selector 上可以注册监听多个 NioChannel。
   4. 每个 NioChannel 只会绑定在唯一的 NioEventLoop 上。
   5. 每个 NioChannel 都绑定有一个自己的 ChannelPipeline。

# 异步模型

## 基本介绍

1. 异步的概念和同步相对。当一个异步过程调用发出后，调用者不能立刻得到结果。实际处理这个调用的组件在完成后，通过状态、通知或回调来通知调用者。
2. Netty 中的 I/O 操作是异步的，包括 bind、write、connect 等操作会简单的返回一个 ChannelFuture。
3. 调用者并不能立刻获得结果，而是通过 Future-Listener 机制，可以方便的主动获取或者通过通知机制获得IO操作结果。

4. Netty 的异步模型是建立在 future 和 callback 之上。callback 就是回调。
5. **Future 核心思想是**：假设一个方法 fun，计算过程可能非常耗时，等待 fun 返回显然不合适。那么可以在调用 fun 的时候，立马返回一个 Future，后续可以通过 Future 去监控方法 fun 的处理过程（即 ： Future-Listener 机制）。

## Future说明

1. 表示**异步的执行结果**， 可以通过提供的方法来检测执行是否完成，比如检索计算等等。
2. ChannelFuture 是一个**接口** ：public interface ChannelFuture extends Future< Void >。
3. 可以**添加监听器，当监听的事件发生时，就会通知到监听器**。

## 工作原理图

![异步模型1](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E5%BC%82%E6%AD%A5%E6%A8%A1%E5%9E%8B1.svg)

---

![异步模型（详细版）](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E5%BC%82%E6%AD%A5%E6%A8%A1%E5%9E%8B%EF%BC%88%E8%AF%A6%E7%BB%86%E7%89%88%EF%BC%89.svg)

**说明**：

1. 在使用 Netty 进行编程时,拦截操作和转换出入站数据只需要您提供 callback 或利用 future 即可。这使得链式操作简单、高效，并有利于编写可重用的、通用的代码。
2. Netty 框架的目标就是让业务逻辑从网络基础应用编码中分离出来。

## ChannelFuture常见方法

| 方法                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| public boolean isDone()                                      | 判断当前操作是否完成。                                       |
| public boolean isSuccess()                                   | 判断已完成的当前操作是否成功。                               |
| public Throwable cause()                                     | 获取已完成的当前操作失败的原因。                             |
| public boolean isCancelled()                                 | 判断已完成的当前操作是否被取消。                             |
| Future<V> addListener(GenericFutureListener<? extends Future<? super V>> listener) | 注册监听器，当操作已完成（isDone 方法返回完成），将会通知指定的监听器，如果Future 对象已完成，则通知指定的监听器。 |

## Future-Listener机制

1. 当 Future 对象刚刚创建时，处于非完成状态，调用者可以通过返回的 ChannelFuture 来获取操作执行的状态，注册监听函数来执行完成后的操作。
2. 常见的操作：[ChannelFuture常见方法](#ChannelFuture常见方法)。

---

**举例代码**：

```java
ChannelFuture cf = bootstrap.connect("127.0.0.1", 3208).sync();

cf.addListener(new ChannelFutureListener() {
    public void operationComplete(ChannelFuture channelFuture) throws Exception {
        if (channelFuture.isSuccess()){
            System.out.println("监听3208端口成功");
        }else {
            System.out.println("监听3208端口失败");
        }
    }
});
```

# 快速入门案例（HTTP服务）

## 案例要求

1. Netty服务端监听2200端口。
2. 浏览器（客户端）请求 ”http://localhost:2200“ ，服务端回复消息给客户端 “hello,browser client”。 
3. 另外对特定请求资源进行过滤。
4. 目的：Netty可以做http服务开发，并理解handler实例与客户端及其请求的关系。

## 案例代码

使用maven构建项目。

```xml
<!-- pom.xml -->

<dependency>
  <groupId>io.netty</groupId>
  <artifactId>netty-all</artifactId>
  <version>4.1.86.Final</version>
</dependency>
```

```java
// NettyHttpServer.java

package com.qqs.netty.server;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;

public class NettyHttpServer {
    public static void main(String[] args) {
        NioEventLoopGroup bossGroup = new NioEventLoopGroup(1);
        NioEventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new NettyHttpServerInitializer());
            ChannelFuture channelFuture = bootstrap.bind(2200).sync();
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
// NettyHttpServerInitializer.java

package com.qqs.netty.server;

import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.socket.SocketChannel;
import io.netty.handler.codec.http.HttpServerCodec;

public class NettyHttpServerInitializer extends ChannelInitializer<SocketChannel> {
    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        // 向管道中添加处理器
        ChannelPipeline pipeline = ch.pipeline();
        // HttpServerCodec: netty提供处理http的编/解码器
        pipeline.addLast("simpleHttpServerCodec", new HttpServerCodec());
        // 自定义业务handler
        pipeline.addLast("simpleNettyHttpServerHandler", new NettyHttpServerHandler());
    }
}
```

```java
// NettyHttpServerHandler.java

package com.qqs.netty.server;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.handler.codec.http.*;

import java.nio.charset.StandardCharsets;
import java.util.Objects;

/**
 * SimpleChannelInboundHandler: ChannelInboundHandlerAdapter的子类
 * HttpObject: 客户端和服务端相互通讯的数据被封装成HttpObject类型
 */
public class NettyHttpServerHandler extends SimpleChannelInboundHandler<HttpObject> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, HttpObject msg) throws Exception {
        if (msg instanceof HttpRequest) {
            System.out.println("pipeline hashCode: " + ctx.pipeline().hashCode());
            System.out.println("msg class type: " + msg.getClass());
            System.out.println("client address: " + ctx.channel().remoteAddress());
            HttpRequest request = (HttpRequest) msg;
            if (Objects.equals("/favicon.ico", request.uri())) {
                System.out.println("针对/favicon.ico资源不进行响应处理");
                return;
            }
            ByteBuf content = Unpooled.copiedBuffer("hello,browser client", StandardCharsets.UTF_8);
            DefaultFullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1,
                    HttpResponseStatus.OK, content);
            response.headers().set(HttpHeaderNames.CONTENT_TYPE, "text/plain;charset=utf-8");
            response.headers().set(HttpHeaderNames.CONTENT_LENGTH, content.readableBytes());

            // 将response进行响应返回
            ctx.writeAndFlush(response);
        }
    }
}
```

**运行结论**：

<img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230731021825711.png" alt="image-20230731021825711" style="zoom:150%;" />

![image-20230731021729400](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230731021729400.png)

> http协议是短连接协议，因为每次请求完就会断开连接，所以每次请求都会分配一个新的pipeline对象与对应的channelSocket。
