# I/O模型

1. I/O 模型简单的理解：就是用什么样的通道进行数据的发送和接收，很大程度上决定了程序通信的性能。

2. Java 共支持 3 种网络编程模型/IO 模式：BIO、NIO、AIO。

3. **Java BIO** ： **同步并阻塞(传统阻塞型)**，服务器实现模式**一个连接一个线程**，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销。

   ![bio](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/bio.svg)

4. **Java NIO** ：**同步非阻塞**，服务器实现模式为**一个线程处理多个请求(连接)**，即客户端发送的连接请求都会注册到**多路复用器**上，多路复用器轮询到连接有 I/O 请求就进行处理。

   ![nio](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/nio.svg)

5. **Java AIO(NIO.2)** ： **异步非阻塞**，AIO 引入异步通道的概念，采用了 Proactor 模式，简化了程序编写，有效的请求才启动线程，它的特点是先由操作系统完成后才通知服务端程序启动线程去处理，一般适用于**连接数较多且连接时间较长**的场景。

# BIO、NIO、AIO适用场景分析

| I/O模型 | 特点                                                         | 使用场景                                                     |
| ------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| BIO     | 同步并阻塞，一个连接对应一个线程，线程开销大                 | 连接数目比较小且固定的架构，服务器资源要求比较高，程序简单易理解 |
| NIO     | 同步非阻塞，一个线程处理多个请求(连接) ，多路复用器轮询到连接有 I/O 请求 | 连接数目多且连接比较短，编程比较复杂。                       |
| AIO     | 异步非阻塞，采用了 Proactor 模式                             | 连接数较多且连接时间较长，编程比较复杂。                     |

# Java BIO基本介绍

1. Java BIO 就是**传统的 Java io 编程**，其相关的类和接口在 java.io。
2. **BIO(blocking I/O)**： **同步阻塞**，服务器实现模式为**一个连接一个线程**，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销，可以通过线程池机制改善(实现多个客户连接服务器)。
3. BIO 方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，JDK1.4以前的唯一选择，程序简单易理解。

# Java BIO工作机制

![bio工作机制](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/bio%E5%B7%A5%E4%BD%9C%E6%9C%BA%E5%88%B6.svg)

# Java BIO 编程流程

1. 服务器端启动一个 ServerSocket。
2. 客户端启动 Socket 对服务器进行通信，默认情况下服务器端需要对每个客户建立一个线程与之通讯。
3. 客户端发出请求后, 先咨询服务器是否有线程响应，如果没有则会等待，或者被拒绝。
4. 如果有响应，客户端线程会等待请求结束后，在继续执行。

# Java BIO应用实例

## 实例说明

1. 使用 BIO 模型编写一个服务器端，监听 6666 端口，当有客户端连接时，就启动一个线程与之通讯。
2. 要求使用线程池机制改善，可以连接多个客户端.
3. 服务器端可以接收客户端发送的数据(telnet 方式即可)。

## 案例代码

```java
package com.qqs.bio;

import java.io.IOException;
import java.io.InputStream;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;


public class BIOServer {
    public static void main(String[] args) throws IOException {
        final ServerSocket server = new ServerSocket(6666);
        final ExecutorService executor = Executors.newCachedThreadPool();
        System.out.println("BIOServer start!");
        try {
            while (true) {
                System.out.println("current thread id =" + Thread.currentThread().getId() +
                        ",name = " + Thread.currentThread().getName());
                System.out.println("waiting for connection...");
                final Socket socket = server.accept();
                System.out.println("connect to client");
                executor.execute(() -> {
                    handler(socket);
                });
            }
        } finally {
            server.close();
        }
    }

    public static void handler(Socket socket) {
        try {
            System.out.println("current thread id =" + Thread.currentThread().getId() +
                    ",name = " + Thread.currentThread().getName());
            byte[] bytes = new byte[1024];
            InputStream inputStream = socket.getInputStream();
            while (true) {
                System.out.println("current thread id =" + Thread.currentThread().getId() +
                        ",name = " + Thread.currentThread().getName());
                int read = inputStream.read(bytes);
                if (read != -1) {
                    System.out.println(new String(bytes, 0, read));
                } else {
                    break;
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            System.out.println("close socket");
            try {
                socket.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

# Java BIO问题分析

1. **每个请求都需要创建独立的线程**，与对应的客户端进行数据 Read，业务处理，数据 Write 。

1. 当并发数较大时，需要**创建大量线程**来处理连接，系统资源占用较大。
2. 连接建立后，如果当前线程暂时没有数据可读，则线程就阻塞在 Read 操作上，造成线程**资源浪费**。