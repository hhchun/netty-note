# 编解码基本介绍

1. 在网络编程时，因为数据都是以二进制字节码的方式进行网络传输的。在发送数据时就需要编码，在接收数据时就需要解码。

2. codec（编解码器）的组成有两个：encoder（编码器）、decoder（解码器）。encoder负责将业务数据转换成字节码数据，decoder负责将字节码数据转换成业务数据。

   ![编解码基本介绍](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E7%BC%96%E8%A7%A3%E7%A0%81%E5%9F%BA%E6%9C%AC%E4%BB%8B%E7%BB%8D.svg)

# Netty的编解码的机制和问题分析

1. Netty本身已提供一些codec（编解码器），如：

   1. StringEncoder：对字符串数据进行编码。

   2. ObjectEncoder：对Java对象数据进行编码。

   3. StringDecoder：对字符串数据进行解码。

   4. ObjectDecoder：对Java对象数据进行解码。

      .....

2. Netty提供的 ObjectDecoder 和 ObjectEncoder 可以用来实现POJO对象或各种业务数据对象的编码和解码。但底层使用的是Java序列化技术，然而Java序列化技术本身效率并不高，存在一些问题，比如：无法跨语言、序列化后的体积过大，是二进制编码的五倍多、序列化性能太低。
3. 由此引出新的解决方案，如：hessian2、kryo、protobuf等。在此特别对protobuf进行简单使用介绍。

# Protobuf

## 基本介绍

1. Protobuf 是 Google 发布的开源项目，全称 Google Protocol Buffers，是一种轻便高效的结构化数据存储格式，可以用于结构化数据串行化，或者说序列化。它很适合做数据存储或RPC（远程过程调用 remote procedure call ）数据交换格式 。
2. 支持跨平台、跨语言（客户端和服务器端可以是不同的语言编写的）、高性能、高可靠性。
3. Protobuf 是以 message 的方式来管理数据的。
4. 使用参考文档：https://developers.google.com/protocol-buffers/docs/proto

## 使用示意图

![protobuf使用示意图](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/protobuf%E4%BD%BF%E7%94%A8%E7%A4%BA%E6%84%8F%E5%9B%BE.svg)

# Protobuf使用配置

## Windows环境配置

1. 下载protoc编译器[下载地址](https://github.com/protocolbuffers/protobuf/releases)，选择win版本下载后解压。

   <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230805174912267.png" alt="image-20230805174912267" style="zoom:50%;" />

   > protoc.exe编译器在解压后的bin目录下。

## Mac环境配置

1. mac环境下需要通过源码编译**protoc编译器**。

2. protoc源码下载[下载地址](https://github.com/protocolbuffers/protobuf/releases)，下载后解压。

   <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230805175823034.png" alt="image-20230805175823034" style="zoom:50%;" />

3. 编译源码：

   1. 进入到源码目录下。

   2. 检查当前系统下是否有gcc编译器，如果没有需要先安装，如何安装可自行百度。

      ```shell
      # zsh
      gcc -v
      ```

   3. 设置protobuf安装目录。

      ```shell
      # zsh
      ./configure --prefix=/usr/local/protobuf
      ```

      > 注意：执行时间较久，请耐心等待。

   4. 编译和安装

      ```shell
      # zsh
      make && make install
      ```

   5. 添加相关环境变量

      ```shell
      vim ~/.bash_profile
      ```

      ```shell
      # 将下面的内容添加到 ./bash_profile 文件最后
      export PROTOBUF=/usr/local/protobuf 
      export PATH=$PROTOBUF/bin:$PATH
      ```

      ```shell
      # 刷新环境变量使其生效
      source ~/.bash_profile
      ```

   6. 查看是否安装完成

      ```shell
      protoc --version
      ```

## Idea插件安装与配置

1. 安装Protobuf、GenProtobuf插件。

   <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230805181551903.png" alt="image-20230805181551903" style="zoom:50%;" />

2. 配置GenProtobuf插件，Tools --> Configure GenProtobuf。

   ![image-20230805181909259](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230805181909259.png)

   ![image-20230805182208235](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230805182208235.png)

   > protoc path：protoc编译器的路径，如果是win系统选择解压后protoc.exe的路径，如果是mac系统选择安装后的路径。protoc编译器安装参考：[win](#Windows环境配置)、[mac](#Mac环境配置)。
   >
   > Quick Gen：生成的类型。

3. GenProtobuf插件使用，选中proto文件右键使用即可。

   <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230805182922850.png" alt="image-20230805182922850" style="zoom: 50%;" />

# Protobuf快速入门案例-单类型

## 案例要求

1. 客户端发送Student对象到服务端，通过Protobuf进行编码。
2. 服务端接收Student对象，并打印输出对象的信息，通过Protobuf进行解码。

## 案例代码

```protobuf
// Student.proto

syntax = "proto3";

option java_outer_classname = "StudentEntity";

message Student {
    int32 id = 1;
    string name = 2;
}
```

> 使用idea的GenProtobuf插件生成StudentEntity.java，GenProtobuf插件配置参考：[Idea插件安装与配置](#Idea插件安装与配置)。

---

```java
// NettyProtobufSimpleServer.java

package com.qqs.netty.server;

import com.qqs.netty.entity.StudentEntity;
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.protobuf.ProtobufDecoder;

public class NettyProtobufSimpleServer {
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
                            // protobuf解码器
                            pipeline.addLast(new ProtobufDecoder(StudentEntity.Student.getDefaultInstance()));
                            pipeline.addLast(new NettyProtobufSimpleServerHandler());
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
// NettyProtobufSimpleServerHandler.java

package com.qqs.netty.server;

import com.qqs.netty.entity.StudentEntity;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;

import java.nio.charset.StandardCharsets;

public class NettyProtobufSimpleServerHandler extends SimpleChannelInboundHandler<StudentEntity.Student> {

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(Unpooled.copiedBuffer("I am the server", StandardCharsets.UTF_8));
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, StudentEntity.Student student) throws Exception {
        System.out.println("student id = " + student.getId());
        System.out.println("student name = " + student.getName());
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

```java
// NettyProtobufSimpleClient.java

package com.qqs.netty.client;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.protobuf.ProtobufEncoder;

public class NettyProtobufSimpleClient {
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
                            // protobuf编码器
                            pipeline.addLast(new ProtobufEncoder());
                            pipeline.addLast(new NettyProtobufSimpleClientHandler());
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
// NettyProtobufSimpleClientHandler.java

package com.qqs.netty.client;

import com.qqs.netty.entity.StudentEntity;
import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

import java.nio.charset.StandardCharsets;

public class NettyProtobufSimpleClientHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        StudentEntity.Student student = StudentEntity.Student.newBuilder()
                .setId(1001)
                .setName("jack")
                .build();
        ctx.writeAndFlush(student);
        System.out.println("student send successful");
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("client address: " + ctx.channel().remoteAddress());
        System.out.println("client reply message: " + ((ByteBuf) msg).toString(StandardCharsets.UTF_8));
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

# Protobuf快速入门案例-多类型

## 案例要求

1. 客户端发送Student、Teacher对象到服务端，通过Protobuf进行编码。
2. 服务端接收Student、Teacher对象，并打印输出对象的信息，通过Protobuf进行解码。

## 案例代码

```protobuf
// Person.proto

syntax = "proto3";

option optimize_for = SPEED;
// option java_package = "com.qqs.netty.entity";
option java_outer_classname = "PersonEntity";

// protobuf可以使用message来管理其他message
message Person {
    enum Type {
        STUDENT = 0;
        TEACHER = 1;
    }

    Type type = 1;

    oneof body {
        Student student = 2;
        Teacher teacher = 3;
    }

}

message Student {
    int32 id = 1;
    string name = 2;
}

message Teacher {
    int32 id = 1;
    string name = 2;
    int32 age = 3;
}
```

> 使用idea的GenProtobuf插件生成StudentEntity.java，GenProtobuf插件配置参考：[Idea插件安装与配置](#Idea插件安装与配置)。

---

```java
// NettyProtobufMultiServer.java

package com.qqs.netty.server;

import com.qqs.netty.entity.PersonEntity;
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.protobuf.ProtobufDecoder;

public class NettyProtobufMultiServer {
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
                            // protobuf解码器
                            pipeline.addLast(new ProtobufDecoder(PersonEntity.Person.getDefaultInstance()));
                            pipeline.addLast(new NettyProtobufMultiServerHandler());
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
// NettyProtobufMultiServerHandler.java

package com.qqs.netty.server;

import com.qqs.netty.entity.PersonEntity;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;

import java.nio.charset.StandardCharsets;

public class NettyProtobufMultiServerHandler extends SimpleChannelInboundHandler<PersonEntity.Person> {

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(Unpooled.copiedBuffer("I am the server", StandardCharsets.UTF_8));
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, PersonEntity.Person person) throws Exception {
        PersonEntity.Person.Type type = person.getType();
        if (type == PersonEntity.Person.Type.STUDENT) {
            PersonEntity.Student student = person.getStudent();
            System.out.println("received client student");
            System.out.println("student id = " + student.getId());
            System.out.println("student name = " + student.getName());
        } else if (type == PersonEntity.Person.Type.TEACHER) {
            PersonEntity.Teacher teacher = person.getTeacher();
            System.out.println("received client teacher");
            System.out.println("teacher id = " + teacher.getId());
            System.out.println("teacher name = " + teacher.getName());
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
  
}
```

```java
// NettyProtobufMultiClient.java

package com.qqs.netty.client;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.protobuf.ProtobufEncoder;

public class NettyProtobufMultiClient {
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
                            // protobuf编码器
                            pipeline.addLast(new ProtobufEncoder());
                            pipeline.addLast(new NettyProtobufMultiClientHandler());
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
// NettyProtobufMultiClientHandler.java

package com.qqs.netty.client;

import com.qqs.netty.entity.PersonEntity;
import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

import java.nio.charset.StandardCharsets;
import java.util.Random;

public class NettyProtobufMultiClientHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        Random random = new Random();
        int next = random.nextInt(3);
        if (next == 1) {
            PersonEntity.Student student = PersonEntity.Student.newBuilder()
                    .setId(1001)
                    .setName("jack")
                    .build();
            PersonEntity.Person person = PersonEntity.Person.newBuilder()
                    .setType(PersonEntity.Person.Type.STUDENT)
                    .setStudent(student)
                    .build();
            ctx.writeAndFlush(person);
        } else if (next == 2) {
            PersonEntity.Teacher teacher = PersonEntity.Teacher.newBuilder()
                    .setId(2001)
                    .setName("tom")
                    .build();
            PersonEntity.Person person = PersonEntity.Person.newBuilder()
                    .setType(PersonEntity.Person.Type.TEACHER)
                    .setTeacher(teacher)
                    .build();
            ctx.writeAndFlush(person);
        }
        System.out.println("student send successful");
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("client address: " + ctx.channel().remoteAddress());
        System.out.println("client reply message: " + ((ByteBuf) msg).toString(StandardCharsets.UTF_8));
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

