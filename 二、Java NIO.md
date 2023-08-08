# Java NIO的基本介绍

1. **Java NIO** 全称 java non-blocking IO，是指 JDK 提供的新 API。从 JDK1.4 开始，Java 提供了一系列改进的输入/输出的新特性，被统称为 NIO(即 New IO)，是同步非阻塞的。
2. NIO 相关类都被放在 **java.nio** 包及子包下，并且对原 java.io 包中的很多类进行改写。
3. NIO 有三大核心部分：**Channel（通道），Buffer（缓冲区)） Selector（选择器）**。
4. NIO 是区面向**缓冲区**，向或者**面向块编程**的。数据读取到一个它稍后处理的缓冲区，需要时可在缓冲区中前后移动，这就增加了处理过程中的灵活性，使用它可以提供非阻塞式的高伸缩性网络。
5. Java NIO 的**非阻塞模式**，使一个线程从某通道发送请求或者读取数据，但是它仅能得到目前可用的数据，如果目前没有数据可用时，就什么都不会获取，而不是保持线程阻塞，所以直至数据变的可以读取之前，该线程可以继续做其他的事情。 非阻塞写也是如此，一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程同时可以去做别的事情。
6. 通俗理解：NIO 是可以做到用一个线程来处理多个操作的。 
7. HTTP2.0 使用了多路复用的技术，做到同一个连接并发处理多个请求，而且并发请求的数量比 HTTP1.1 大了好几个数量级。

# NIO与BIO比较

1. BIO 以流的方式处理数据,而 NIO 以块的方式处理数据,块 I/O 的效率比流 I/O 高很多。
2. BIO 是阻塞的，NIO 则是非阻塞的。
3. BIO 基于字节流和字符流进行操作，而 NIO 基于 Channel（通道）和 Buffer（缓冲区）进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。Selector（选择器）用于监听多个通道的事件（比如：连接请求，数据到达等），因此使用单个线程就可以监听多个客户端通道。

# NIO三大核心原理图

![nio核心原理图（简单版）](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/nio%E6%A0%B8%E5%BF%83%E5%8E%9F%E7%90%86%E5%9B%BE%EF%BC%88%E7%AE%80%E5%8D%95%E7%89%88%EF%BC%89.svg)

1. 每个 channel 都会对应一个 Buffer。
2. Selector 对应一个线程， 一个线程对应多个 channel（连接）。
3. 该图反应了有三个 channel 注册到 该 selector。
4. 程序切换到哪个 channel 是由事件决定的，Event 就是一个重要的概念。
5. Selector 会根据不同的事件，在各个通道上切换。
6. Buffer 就是一个内存块 ， 底层是一个数组。
7. 数据的读取写入是通过 Buffer，这个和 BIO ，BIO 中要么是输入流，或者是输出流，不能双向，但是 NIO 的 Buffer 是可以读也可以写，需要 flip 方法切换channel 是双向的，可以返回底层操作系统的情况，比如 Linux ， 底层的操作系统通道就是双向的。

# 缓冲区（Buffer）

## 基本介绍

缓冲区本质上是一个**可以读写数据的内存块**，可以理解成是一个容器对象，该对象提供了一组方法，可以更轻松地使用内存块，缓冲区对象内置了一些机制，能够跟踪和记录缓冲区的状态变化情况。Channel 提供从文件、网络读取数据的渠道，但是读取或写入的数据都必须经由 Buffer。

![nio缓冲区（Buffer）](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/nio%E7%BC%93%E5%86%B2%E5%8C%BA%EF%BC%88Buffer%EF%BC%89.svg)

## Buffer类及其子类

### 类的层级关系

![image-20230727162140053](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230727162140053.png)

1. ByteBuffer：存储字节类型数据的缓冲区。
2. CharBuffer：存储字符类型数据的缓冲区。
3. DoubleBuffer：存储双精度浮点数类型数据的缓冲区。
4. FloatBuffer：存储单精度浮点数类型数据的缓冲区。
5. IntBuffer：存储整数类型数据的缓冲区。
6. LongBuffer：存储长整数类型数据的缓冲区。
7. ShortBuffer：存储短整数类型数据的缓冲区。

### Buffer类的四个属性

Buffer 类定义了所有的缓冲区都具有的四个属性来提供关于其所包含的数据元素的信息：

![image-20230727163102200](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230727163102200.png)

| 属性     | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| mark     | 标记。                                                       |
| position | 位置，下一个要被读或写的元素的索引，每次读写缓冲区数据时都会改变，未下次读写做准备。 |
| limit    | 表示当前缓冲区的终点，不能对缓冲区超过限制的位置进行读写操作，且限制是可以修改的。 |
| capacity | 容量，即可容纳的最大数据量，在缓冲区创建时被设定后不能再改变。 |

### Buffer类的方法

<img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230727163710216.png" alt="image-20230727163710216" style="zoom:67%;" />

| 方法                                          | 描述                                                         |
| --------------------------------------------- | ------------------------------------------------------------ |
| public final int capacity()                   | 返回此缓冲区的容量。                                         |
| public final int position()                   | 返回此缓冲区的位置。                                         |
| public final Buffer position(int newPosition) | 设置此缓冲区的位置。                                         |
| public final int limit()                      | 返回此缓冲区的限制。                                         |
| public final Buffer limit(int newLimit)       | 设置此缓冲区的限制。                                         |
| public final Buffer mark()                    | 设置此缓冲区的位置标记。                                     |
| public final Buffer reset()                   | 重置此缓冲区以前标记的位置。                                 |
| public final Buffer clear()                   | 清除此缓冲区，即将各个标记恢复到初始状态，但数据并没有真正的擦除，新数据会覆盖掉旧数据。 |
| public final Buffer flip()                    | 反转缓冲区。                                                 |
| public final Buffer rewind()                  | 重绕此缓冲区。                                               |
| public final int remaining()                  | 返回当前位置与限制之间的元素个数。                           |
| public final boolean hasRemaining()           | 告知在当前位置与限制之间是否有元素。                         |
| public abstract boolean isReadOnly()          | 告知此缓冲区是否为只读缓冲区。                               |
| public abstract boolean hasArray()            | 告知此缓冲区是否具有可访问的底层实现的数组。                 |
| public abstract Object array()                | 返回此缓冲区的底层实现的数组。                               |
| public abstract int arrayOffset()             | 返回此缓冲区的底层实现的数组中第一个缓冲区元素的偏移量。     |
| public abstract boolean isDirect()            | 告知此缓冲区是否为直接缓冲区。                               |

### ByteBuffer类的主要方法

| 方法                                                         | 描述                                                       |
| ------------------------------------------------------------ | ---------------------------------------------------------- |
| public static ByteBuffer allocateDirect(int capacity)        | 创建缓冲区。                                               |
| public static ByteBuffer allocate(int capacity)              | 设置缓冲区的初始化容量。                                   |
| public static ByteBuffer wrap(byte[] array)                  | 将一个数组放入到缓存区中使用。                             |
| public static ByteBuffer wrap(byte[] array, int offset, int length) | 构造初始化位置offset和限制length的缓冲区。                 |
| public abstract byte get()                                   | 从当前位置（position）上获取数据，获取后position会自动+1。 |
| public abstract byte get(int index)                          | 从指定的绝对位置上获取数据。                               |
| public abstract ByteBuffer put(byte b)                       | 从当前位置（position）上添加数据，添加后position会自动+1。 |
| public abstract ByteBuffer put(int index, byte b)            | 从指定的绝对位置上添加数据。                               |

# 通道（Channel）

## 基本介绍

1. NIO 的通道类似于流，但有些区别如下：

   > - 通道可以**同时进行读写**，而流只能读或者只能写。
   > - 通道可以实现**异步读写**数据。
   > - 通道可以**从缓冲读数据**，也可以**写数据到缓冲**。

2. BIO 中的 stream 是单向的，例如 FileInputStream 对象只能进行读取数据的操作，而 NIO 中的通道（Channel）是**双向的**，**可以读操作，也可以写操作**。

3. Channel 在 NIO 中是一个接口：java.nio.channels.Channel

4. 常用的Channel类有 ：FileChannel、DatagramChannel、ServerSocketChannel、SocketChannel。

   > * ServerSocketChanne类似 ServerSocket，SocketChannel类似Socket。

   <img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230727205532341.png" alt="image-20230727205532341" style="zoom:75%;" />

5. FileChannel 用于文件的数据读写，DatagramChannel 用于 UDP 的数据读写，ServerSocketChannel 和 SocketChannel用于 TCP 的数据读写。

## FileChannel类

FileChannel 主要用来对本地文件进行 IO 操作，常见的方法：

| 方法                                                         | 描述                           |
| ------------------------------------------------------------ | ------------------------------ |
| public int read(ByteBuffer dst)                              | 从通道读取数据并放到缓冲区中   |
| public int write(ByteBuffer src)                             | 把缓冲区的数据写到通道中       |
| public long transferFrom(ReadableByteChannel src, long position, long count) | 从目标通道中复制数据到当前通道 |
| public long transferTo(long position, long count, WritableByteChannel target) | 把数据从当前通道复制给目标通道 |

## 案例一：本地文件写数据

案例要求：使用前面学习的ByteBuffer（缓冲区）和FileChannel（通道），将 "hello,nio" 写入到file01.txt中，如果文件不存在则创建。

 案例代码：

```java
package com.qqs.nio;

import java.io.FileOutputStream;
import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

public class LocalFileWrite {
    public static void main(String[] args) {
        String data = "hello,nio";
        FileOutputStream os = null;
        FileChannel channel = null;
        try {
            os = new FileOutputStream("/var/tmp/file01.txt");
            channel = os.getChannel();
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            buffer.put(data.getBytes());
            buffer.flip();
            channel.write(buffer);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (channel != null) {
                try {
                    channel.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if (os != null) {
                try {
                    os.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

运行结果：

![image-20230727215929035](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230727215929035.png)

## 案例二：本地文件读数据

案例要求：使用前面学习后的 ByteBuffer（缓冲） 和 FileChannel（通道），将 file01.txt 中的数据读入到程序，并打印在控制台。

案例代码：

```java
package com.qqs.nio;

import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

public class LocalFileRead {
    public static void main(String[] args) {
        FileInputStream is = null;
        FileChannel channel = null;
        try {
            File file = new File("/var/tmp/file01.txt");
            is = new FileInputStream(file);
            channel = is.getChannel();
            ByteBuffer buffer = ByteBuffer.allocate((int) file.length());
            channel.read(buffer);
            String data = new String(buffer.array());
            System.out.println("data = " + data);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (channel != null) {
                try {
                    channel.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if (is != null) {
                try {
                    is.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}

```

案例结果：

![image-20230727220809286](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230727220809286.png)

## 案例三：使用一个Buffer读写文件

案例要求：使用FileChannel（通道）的read、write方法，完成文件的拷贝。

![使用Buffer读写文件1](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/%E4%BD%BF%E7%94%A8Buffer%E8%AF%BB%E5%86%99%E6%96%87%E4%BB%B61.svg)

案例代码：

```java
package com.qqs.nio;

import sun.nio.ch.IOStatus;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

public class BufferCopyFile {
    public static void main(String[] args) {
        FileInputStream is = null;
        FileOutputStream os = null;
        try {
            is = new FileInputStream("/var/tmp/1.txt");
            os = new FileOutputStream("/var/tmp/2.txt");
            FileChannel inChannel = is.getChannel();
            FileChannel outChannel = os.getChannel();
            ByteBuffer buffer = ByteBuffer.allocate(512);
            while (true) {
                buffer.clear();
                int len = inChannel.read(buffer);
                System.out.println("len = " + len);
                if (len == IOStatus.EOF) {
                    break;
                }
                buffer.flip();
                outChannel.write(buffer);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (is != null) {
                try {
                    is.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if (os != null) {
                try {
                    os.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

## 案例四：使用transferFrom方法拷贝文件

案例要求：使用 FileChannel（通道）transferFrom方法，完成文件的拷贝。

案例代码：

```java
package com.qqs.nio;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.nio.channels.FileChannel;

public class TransferFromCopyFile {
    public static void main(String[] args) {
        FileInputStream is = null;
        FileOutputStream os = null;
        try {
            is = new FileInputStream("/var/tmp/1.txt");
            os = new FileOutputStream("/var/tmp/3.txt");
            FileChannel sourceChannel = is.getChannel();
            FileChannel destChannel = os.getChannel();
            destChannel.transferFrom(sourceChannel, 0, sourceChannel.size());
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (is != null) {
                try {
                    is.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if (os != null) {
                try {
                    os.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

## 关于Buffer和Channel的注意事项和细节 

1. ByteBuffer 支持类型化的 put 和 get， put **放入的是什么数据类型**，get 就应该**使用相应的数据类型来取出**，否则可能有 **BufferUnderflowException** 异常。

   ```java
   package com.qqs.nio;
   
   import java.nio.ByteBuffer;
   
   public class ByteBufferPutGet {
       public static void main(String[] args) {
           // 创建Buffer
           ByteBuffer buffer = ByteBuffer.allocate(64);
           // 类型化方式放入数据
           buffer.putInt(100);
           buffer.putLong(9);
           buffer.putChar('帅');
           buffer.putShort((short) 4);
   
           // 取出，顺序与放入的顺序一致，类型一致
           buffer.flip();
           System.out.println();
           System.out.println(buffer.getInt());
           System.out.println(buffer.getLong());
           System.out.println(buffer.getChar());
           System.out.println(buffer.getShort());
       }
   }
   ```

2. 可以将一个普通 Buffer 转成只读 Buffer

   ```java
   package com.qqs.nio;
   
   import java.nio.ByteBuffer;
   
   public class ReadOnlyBuffer {
       public static void main(String[] args) {
           // 创建buffer
           ByteBuffer buffer = ByteBuffer.allocate(64);
           for (int i = 0; i < 64; i++) {
               // 添加数据
               buffer.put((byte) i);
           }
           // 读取
           buffer.flip();
   
           // 获取只读Buffer
           ByteBuffer readOnlyBuffer = buffer.asReadOnlyBuffer();
           System.out.println(readOnlyBuffer.getClass());
           
           // 读取
           while (readOnlyBuffer.hasRemaining()) { // 判断是否还有数据
               // 取出，并给position+1
               System.out.println(readOnlyBuffer.get());
           }
   
           // 测试只能读取，不能在put写入
           readOnlyBuffer.put((byte) 100); // throws ReadOnlyBufferException
       }
   }
   ```

3. NIO 还提供了 MappedByteBuffer， 可以让文件直接在内存（堆外的内存）中进行修改， 如何同步到文件由 NIO 来自主完成。

   ```java
   package com.qqs.nio;
   
   import java.io.RandomAccessFile;
   import java.nio.MappedByteBuffer;
   import java.nio.channels.FileChannel;
   
   public class MappedByteBufferExample {
       public static void main(String[] args) throws Exception {
           RandomAccessFile randomAccessFile = new RandomAccessFile("/var/tmp/1.txt", "rw");
           // 获取对应的通道
           FileChannel channel = randomAccessFile.getChannel();
           // 参数1: FileChannel.MapMode.READ_WRITE 使用的读写模式
           // 参数2：0表示直接修改的起始位置（字节位置）
           // 参数3: 5表示映射到内存的大小（不是索引位置）,即将 1.txt 的多少个字节映射到内存
           // 那么可以直接修改的范围为: 0-5
           // mappedByteBuffer 的实际类型是 DirectByteBuffer
           MappedByteBuffer mappedByteBuffer = channel.map(FileChannel.MapMode.READ_WRITE, 0, 5);
           mappedByteBuffer.put(0, (byte) 'H');
           mappedByteBuffer.put(3, (byte) '9');
           mappedByteBuffer.put(5, (byte) 'Y'); // throws IndexOutOfBoundsException
           // 关闭资源
           randomAccessFile.close();
           channel.close();
       }
   }
   
   ```

   ![](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230727225757263.png)

4. NIO 还支持通过多个Buffer（即 Buffer 数组）完成读写操作，即 **Scattering** 和**Gathering**。

   ```java
   package com.qqs.nio;
   
   import java.net.InetSocketAddress;
   import java.nio.Buffer;
   import java.nio.ByteBuffer;
   import java.nio.channels.ServerSocketChannel;
   import java.nio.channels.SocketChannel;
   import java.util.Arrays;
   
   public class ScatteringAndGathering {
       public static void main(String[] args) throws Exception {
           // 使用ServerSocketChannel和SocketChannel
           ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
           InetSocketAddress inetSocketAddress = new InetSocketAddress(7000);
           // 绑定端口到socket，并启动
           serverSocketChannel.socket().bind(inetSocketAddress);
   
           // 创建buffer数组
           ByteBuffer[] byteBuffers = new ByteBuffer[2];
           byteBuffers[0] = ByteBuffer.allocate(5);
           byteBuffers[1] = ByteBuffer.allocate(3);
   
           // 等客户端连接（telnet）
           SocketChannel socketChannel = serverSocketChannel.accept();
           // 假定从客户端接收8个字节
           int messageLength = 8;
   
           // 循环的读取
           while (true) {
               long byteRead = 0;
               while (byteRead < messageLength) {
                   long l = socketChannel.read(byteBuffers);
                   byteRead += l; //累计读取的字节数
                   System.out.println("byteRead = " + byteRead);
                   // 打印观察当前buffer的position和limit
                   Arrays.stream(byteBuffers).map(buffer -> "position =" + buffer.position() + ", limit=" + buffer.limit())
                           .forEach(System.out::println);
               }
               // 将所有的buffer进行flip
               Arrays.asList(byteBuffers).forEach(Buffer::flip);
   
               // 将数据读出显示到客户端
               long byteWirte = 0;
               while (byteWirte < messageLength) {
                   long l = socketChannel.write(byteBuffers);
                   byteWirte += l;
               }
               // 将所有的buffer进行clear
               Arrays.asList(byteBuffers).forEach(Buffer::clear);
               System.out.println("byteRead = " + byteRead + " byteWrite = " + byteWirte + ", messageLength = " + messageLength);
           }
       }
   }
   ```

# 选择器（Selector）

## 基本介绍

1. Java 的 NIO，用非阻塞的 IO 方式。可以用一个线程，处理多个的客户端连接，就会使用到 **Selector（选择器）**。

2. **Selector 能够检测多个注册的通道上是否有事件发生（注意：多个 Channel 以事件的方式可以注册到同一个Selector）**，如果有事件发生，便获取事件然后针对每个事件进行相应的处理。这样就可以只用一个单线程去管理多个通道，也就是管理多个连接和请求。

3. 只有在 连接/通道 真正有读写事件发生时，才会进行读写，就大大地减少了系统开销，并且不必为每个连接都创建一个线程，不用去维护多个线程。

4. 避免了多线程之间的上下文切换导致的开销。

   ![selector](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/selector.svg)

> 1. Netty 的 IO 线程 NioEventLoop 聚合了 Selector(选择器，也叫多路复用器)，可以同时并发处理成百上千个客户端连接。
> 2. 当线程从某客户端 Socket 通道进行读写数据时，若没有数据可用时，该线程可以进行其他任务。
> 3. 线程通常将非阻塞 IO 的空闲时间用于在其他通道上执行 IO 操作，所以单独的线程可以管理多个输入和输出通道。
> 4. 由于**读写操作都是非阻塞**的，这就可以充分提升 IO 线程的运行效率，避免由于频繁 I/O 阻塞导致的线程挂起。
> 5. 一个 I/O 线程可以并发处理 N 个客户端连接和读写操作，这从根本上解决了传统同步阻塞 I/O 一连接一线程模型，架构的性能、弹性伸缩能力和可靠性都得到了极大的提升。

## Selector类常用方法

Selector类：**java.nio.channels.Selector**。

![image-20230727232010254](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230727232010254.png)

| 方法                                    | 描述                                                         |
| --------------------------------------- | ------------------------------------------------------------ |
| public static Selector open()           | 获取一个选择器对象                                           |
| public  Set<SelectionKey> keys()        | 获取注册的所有通道                                           |
| public Set<SelectionKey> selectedKeys() | 从内部集合中获取到所有的SelectionKey                         |
| public int select(long timeout)         | 监控所有注册的通道，当其中有I/O操作时，将对应的SelectionKey加入到内部集合中并返回，参数可设置超时时间 |

## 注意事项

1. NIO 中的 ServerSocketChannel 功能类似 ServerSocket，SocketChannel 功能类似 Socket。
2. selector 相关方法说明：
   - selector.select()：阻塞。
   - selector.select(1000)：阻塞 1000 毫秒，在 1000 毫秒后返回。
   - selector.wakeup()：唤醒 selector。
   - selector.selectNow()：不阻塞，立即返回。

# NIO非阻塞网络编程原理图

![NIO非阻塞](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/NIO%E9%9D%9E%E9%98%BB%E5%A1%9E.svg)

1. 将ServerSocketChannel注册到Selector中，只关心OP_ACCEPT事件。
2. Selector通过select方法进行监听，返回发生事件的通道个数。
3. 将SocketChannel注册到Selector中，一个Selector可以注册多个SocketChannel。
4. Selector会对已注册的通道进行监听，当通道有事件发生时，则可以通过selectedKeys方法获取到对应selectedKey的集合。
5. 可根据selectedKey判断不同类型的事件，针对不同的事件进行相应的业务处理。
6. 可以通过selectedKey反向获取对应的SocketChannel，完成业务处理。
7. 代码撑腰，code：[NIO非阻塞网络编程快速入门](#NIO非阻塞网络编程快速入门)

# NIO非阻塞网络编程快速入门

服务端：

```java
package com.qqs.nio.server;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.Iterator;
import java.util.Set;
import java.util.concurrent.TimeUnit;

public class NioQuickStartServer {
    public static void main(String[] args) throws IOException {
        // 创建ServerSocketChannel
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        // 监听3208端口
        serverSocketChannel.socket().bind(new InetSocketAddress(3208));
        // 设置为非阻塞
        serverSocketChannel.configureBlocking(false);

        // 创建Selector
        Selector selector = Selector.open();
        // 将serverSocketChannel注册到selector中，只关心OP_ACCEPT事件
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        // 循环等待客户端的连接或数据
        while (true) {
            if (selector.select(TimeUnit.SECONDS.toMillis(1)) == 0) {
                System.out.println("服务器已等待1秒,无连接");
                continue;
            }
            // 如果selector.select返回的结果大于0,表示已获取到关心的事件，
            // 可通过selector.selectedKeys获取关心事件的集合
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            // 使用迭代器遍历事件
            Iterator<SelectionKey> selectionKeyIterator = selectionKeys.iterator();
            while (selectionKeyIterator.hasNext()) {
                SelectionKey selectionKey = selectionKeyIterator.next();
                // 根据发生的事件不同进行相应的处理
                if (selectionKey.isAcceptable()) {
                    // OP_ACCEPT事件，有新的客户端连接
                    ServerSocketChannel channel = (ServerSocketChannel) selectionKey.channel();
                    SocketChannel socketChannel = channel.accept();
                    System.out.println("客户端连接成功，生成一个新的SocketChannel,hashcode = " + socketChannel.hashCode());
                    // 将socketChannel设置为非阻塞
                    socketChannel.configureBlocking(false);

                    // 将socketChannel注册到selector中，只关心OP_READ事件，同时给socketChannel关联一个Buffer
                    socketChannel.register(selector, SelectionKey.OP_READ, ByteBuffer.allocate(1024));
                }
                if (selectionKey.isReadable()) {
                    // OP_READ事件，客户端发送数据
                    // 通过selectionKey反向获取对应的channel
                    SocketChannel channel = (SocketChannel) selectionKey.channel();
                    // 获取到channel关联的buffer
                    ByteBuffer buffer = (ByteBuffer) selectionKey.attachment();
                    // 从channel中读取客户端发送的数据
                    channel.read(buffer);
                    System.out.println("从客户端中读取到数据: " + new String(buffer.array()));
                    buffer.clear();
                }
                // 移除selectionKey，防止重复操作
                selectionKeyIterator.remove();
            }
        }
    }
}
```

---

客户端：

```java
package com.qqs.nio.client;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;
import java.nio.charset.StandardCharsets;

public class NioQuickStartClient {
    public static void main(String[] args) throws IOException {
        // 创建网络通道
        SocketChannel socketChannel = SocketChannel.open();
        // 设置为非阻塞
        socketChannel.configureBlocking(false);
        // 设置服务端的ip和端口
        InetSocketAddress inetSocketAddress = new InetSocketAddress("127.0.0.1", 3208);
        // 连接服务端
        if (!socketChannel.connect(inetSocketAddress)) {
            while (!socketChannel.finishConnect()) {
                System.out.println("连接服务端需要时间，连接期间客户端不会阻塞，可以去做其他工作");
            }
        }

        // 连接成功,发送数据
        String data = "hello，NioQuickStartServer";
        ByteBuffer buffer = ByteBuffer.wrap(data.getBytes(StandardCharsets.UTF_8));
        socketChannel.write(buffer);

        // 阻塞客户端,防止客户端与客户端的连接挂掉
        System.in.read();
    }
}
```

# SelectionKey

## 类型

Selector与通道的注册关系类型，共四种：

| 属性       | 值   | 描述                 |
| ---------- | ---- | -------------------- |
| OP_ACCEPT  | 16   | 有新的连接可以accept |
| OP_CONNECT | 8    | 连接已建立           |
| OP_WRITE   | 4    | 写操作               |
| OP_READ    | 1    | 读操作               |

<img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230728021933439.png" alt="image-20230728021933439" style="zoom: 50%;" />

## SelectionKey相关方法

<img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230728022140780.png" alt="image-20230728022140780" style="zoom: 80%;" />

| 方法                                     | 描述                       |
| ---------------------------------------- | -------------------------- |
| public Selector selector()               | 获取与之关联的Selector对象 |
| public SelectableChannel channel()       | 获取与之关联的通道对象     |
| public final Object attachment()         | 获取与之关联的共享数据     |
| public SelectionKey interestOps(int ops) | 设置或改变监听的事件       |
| public final boolean isAcceptable()      | 获取是否可以accept         |
| public final boolean isReadable()        | 获取是否可以读             |
| public final boolean isWritable()        | 获取是否可以写             |

# ServerSocketChannel

<img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230728211057421.png" alt="image-20230728211057421" style="zoom:50%;" />

| 方法                                                         | 描述                                     |
| ------------------------------------------------------------ | ---------------------------------------- |
| public static ServerSocketChannel open()                     | 获取ServerSocketChannel通道              |
| public final ServerSocketChannel bind(SocketAddress local)   | 设置服务器端端口号                       |
| public SocketChannel accept()                                | 设置阻塞或非阻塞模式                     |
| public final SelectableChannel configureBlocking(boolean block) | 接受一个连接，返回代表这个连接的通道对象 |
| public final SelectionKey register(Selector sel, int ops)    | 将此通道注册到选择器中，并设置监听的事件 |

# SocketChannel

<img src="https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230728212305443.png" alt="image-20230728212305443" style="zoom:50%;" />

| 方法                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| public static SocketChannel open()                           | 获取SocketChannel通道                                        |
| public boolean connect(SocketAddress remote)                 | 连接服务端                                                   |
| public boolean finishConnect()                               | 如果connect方法连接失败，则需要通过该方法完成连接操作        |
| public int read(ByteBuffer dst)                              | 往通道里写入数据                                             |
| public int write(ByteBuffer src)                             | 从通道里读取数据                                             |
| public final SelectableChannel configureBlocking(boolean block) | 设置阻塞或非阻塞模式                                         |
| public final SelectionKey register(Selector sel, int ops,Object att) | 将此通道注册到选择器中，并设置监听事件，att参数可设置共享数据 |
| public final void close()                                    | 关闭通道                                                     |

# NIO网络编程案例（群聊系统）

## 案例要求

1. 实现服务端和客户端之间的数据简单通讯（非阻塞）。
2. 实现多人群聊。
3. 服务端：可以监控用户上线、离线，并实现消息转发功能。
4. 客户端：通过channel可以无阻塞发送消息给其它所有用户，同时也可以接受其它用户发送的消息（由服务端进行转发）。
5. 目的：进一步理解NIO非阻塞网络编程机制。

## 案例代码

服务端：

```java
package com.qqs.nio.server;


import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.Arrays;
import java.util.Iterator;

public class NioGroupChatServer {
    public static void main(String[] args) {
        Selector selector = null;
        ServerSocketChannel serverSocketChannel = null;
        try {
            // 创建选择器
            selector = Selector.open();
            // 创建通道
            serverSocketChannel = ServerSocketChannel.open();
            // 绑定端口
            serverSocketChannel.socket().bind(new InetSocketAddress(3209));
            // 设置非阻塞模式
            serverSocketChannel.configureBlocking(false);
            // 将通道注册到选择器中,并指定关心accept事件
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
            while (true) {
                int count = selector.select();
                if (count > 0) {
                    // 有事件需要处理
                    Iterator<SelectionKey> selectionKeyIterator = selector.selectedKeys().iterator();
                    // 使用迭代器遍历处理事件
                    while (selectionKeyIterator.hasNext()) {
                        SelectionKey selectionKey = selectionKeyIterator.next();

                        if (selectionKey.isAcceptable()) {
                            // 处理accept事件
                            ServerSocketChannel channel = (ServerSocketChannel) selectionKey.channel();
                            SocketChannel socketChannel = channel.accept();
                            socketChannel.configureBlocking(false);
                            // 将socketChannel注册到selector
                            socketChannel.register(selector, SelectionKey.OP_READ);
                            // 提示
                            System.out.println(socketChannel.getRemoteAddress() + " 上线");
                        }
                        if (selectionKey.isReadable()) {
                            // 处理read事件,即通道是可读状态
                            String message = handlerRead(selectionKey);
                            if (message != null) {
                                // 将读取到的消息发送给其他客户端
                                sendMessageToOtherClients(message, (SocketChannel) selectionKey.channel(), selector);
                            }
                        }
                        selectionKeyIterator.remove();
                    }
                } else {
                    System.out.println("等待...");
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (selector != null) {
                try {
                    selector.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if (serverSocketChannel != null) {
                try {
                    serverSocketChannel.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    /**
     * 将读取到的消息发送给其他客户端
     *
     * @param message  消息内容
     * @param self     排除无需发送的客户端
     * @param selector 选择器
     */
    private static void sendMessageToOtherClients(String message, SocketChannel self, Selector selector) {
        for (SelectionKey selectionKey : selector.keys()) {
            Channel channel = selectionKey.channel();
            // 排除无需发送的客户端
            if (channel instanceof SocketChannel && channel != self) {
                SocketChannel targetChannel = (SocketChannel) channel;
                ByteBuffer buffer = ByteBuffer.wrap(message.getBytes());
                try {
                    targetChannel.write(buffer);
                } catch (IOException e) {
                    e.printStackTrace();
                    try {
                        System.out.println(targetChannel.getRemoteAddress() + " 可能已离线");
                        // 取消注册
                        selectionKey.cancel();
                        // 关闭通道
                        targetChannel.close();
                    } catch (IOException ex) {
                        ex.printStackTrace();
                    }

                }
            }
        }
    }


    /**
     * 处理读
     *
     * @param selectionKey SelectionKey
     */
    private static String handlerRead(SelectionKey selectionKey) {
        SocketChannel channel = (SocketChannel) selectionKey.channel();
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        int len = 0;
        try {
            len = channel.read(buffer);
        } catch (IOException e) {
            try {
                System.out.println(channel.getRemoteAddress() + " 已离线");
                // 取消注册
                selectionKey.cancel();
                // 关闭通道
                channel.close();
            } catch (IOException ex) {
                ex.printStackTrace();
            }
        }
        if (len > 0) {
            // 将缓冲区中的数据转成字符串
            String message = new String(Arrays.copyOf(buffer.array(), len));
            // 打印输出消息内容
            System.out.println("服务端接收到消息: " + message);
            return message;
        }
        return null;
    }
}
```

---

客户端：

```java
package com.qqs.nio.client;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.util.Arrays;
import java.util.Iterator;
import java.util.Scanner;
import java.util.concurrent.TimeUnit;

public class NioGroupChatClient {
    public static void main(String[] args) {
        final String host = "127.0.0.1";
        final int port = 3209;
        try (
                // 创建选择器
                Selector selector = Selector.open();
                // 创建连接服务端对象
                SocketChannel socketChannel = SocketChannel.open(new InetSocketAddress(host, port))
        ) {
            // 设置非阻塞模式
            socketChannel.configureBlocking(false);
            socketChannel.register(selector, SelectionKey.OP_READ);
            String username = socketChannel.getLocalAddress().toString().substring(1);
            System.out.println(username + " 准备完成");

            // 启动一个子线程,每3秒读取从服务端发送的数据
            new Thread(() -> {
                while (true) {
                    try {
                        int count = selector.select();
                        if (count > 0) {
                            Iterator<SelectionKey> selectionKeyIterator = selector.selectedKeys().iterator();
                            while (selectionKeyIterator.hasNext()) {
                                SelectionKey selectionKey = selectionKeyIterator.next();
                                if (selectionKey.isReadable()) {
                                    // 处理读

                                    // 反向获取通道
                                    SocketChannel channel = (SocketChannel) selectionKey.channel();
                                    ByteBuffer buffer = ByteBuffer.allocate(1024);
                                    int len = channel.read(buffer);
                                    if (len > 0) {
                                        String message = new String(Arrays.copyOf(buffer.array(), len));
                                        System.out.println(message);
                                    }
                                }
                            }
                        }
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                    try {
                        Thread.currentThread().sleep(TimeUnit.SECONDS.toMillis(3));
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
            // 从控制台获取消息并发送给服务端
            Scanner scanner = new Scanner(System.in);
            while (scanner.hasNextLine()) {
                String message = scanner.nextLine();
                message = username + ": " + message;
                System.out.println(message);
                ByteBuffer buffer = ByteBuffer.wrap(message.getBytes());
                socketChannel.write(buffer);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

# NIO与零拷贝

## 零拷贝基本介绍

1. 零拷贝是网络编程的关键，很多性能优化都离不开。
2. 在 Java 程序中，常用的零拷贝有 mmap（内存映射）和 sendFile。那么，它们在 OS 里，到底是怎么样设计的。

## 传统I/O数据读写

```java
package com.qqs.nio;

import java.io.File;
import java.io.IOException;
import java.io.RandomAccessFile;
import java.net.ServerSocket;
import java.net.Socket;

public class MoresIoReadWrite {
    public static void main(String[] args) throws IOException {
        File file = new File("/var/tep/1.txt");
        RandomAccessFile raf = new RandomAccessFile(file, "rw");
        byte[] buffer = new byte[(int) file.length()];
        int len = raf.read(buffer);
        if (len > 0) {
            Socket socket = new ServerSocket(8080).accept();
            socket.getOutputStream().write(buffer);
        }
    }
}
```

## 传统I/O数据模型

**需要切换内核态&用户态3次，拷贝4次。**

![image-20230729001758059](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230729001758059.png)

> DMA：Direct Memory Access 直接内存拷贝（不使用CPU）。

## mmap优化

mmap 通过内存映射，将 文件映射到**内核缓冲区**，同时， 用户空间可以共享内核空间的数据。

这样，在进行网络传输时，就可以**减少内核空间到用户空间的拷贝次数**。

**但用户态&内核态的切换依然是3次**。

![image-20230729001540539](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230729001540539.png)

## sendFile优化

Linux 2.1 版本 提供了 sendFile 函数，其基本原理：数据根本不经过用户态，直接从内核缓冲区进入到Socket Buffer，同时，由于和用户态完全无关，就**减少了1次上下文切换**。

**减少了1次用户态内核态的切换，减少1次拷贝次数**。

![image-20230729004052481](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230729004052481.png)

零拷贝从操作系统角度，是没有 cpu 拷贝。

Linux 在 2.4 版本中，做了一些修改，避免了从 内核缓冲区拷贝到 Socket buffer 的操作，直接拷贝到协议栈，从而再一次减少了数据拷贝。

此时，**用户态内核态切换次数2次，拷贝次数1次**。

![image-20230729003852482](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230729003852482.png)

> * 这里其实有1次 cpu 拷贝kernel buffer -> socket buffer。
>
> * 但是，拷贝的信息很少，比如 lenght , offset , 消耗低，可以忽略。

## 零拷贝的再次理解

- 我们说零拷贝，是从**操作系统**的角度来说的。因为内核缓冲区之间，没有数据是重复的（只有 kernel buffer 有一份数据）。
- 零拷贝不仅仅带来更少的数据复制，还能带来其他的性能优势，例如**更少的上下文切换，更少的 CPU 缓存伪共享以及无 CPU 校验和计算**。

## mmap和sendFile的区别

1. mmap 适合小数据量读写，sendFile 适合大文件传输。
2. mmap 需要 4 次上下文切换，3 次数据拷贝；sendFile 需要 3 次上下文切换，最少 2 次数据拷贝。
3. sendFile 可以利用 DMA 方式，减少 CPU 拷贝，mmap 则不能（必须从内核拷贝到 Socket 缓冲区）。

## NIO零拷贝案例

案例要求：

1. 使用传统I/O方式传递一个大文件。
2. 使用NIO零拷贝的方式传递一个大文件。
3. 对比两种传递方式耗时时间分别是多少。

----

服务端：

```java
package com.qqs.nio.server;

import sun.nio.ch.IOStatus;

import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.nio.ByteBuffer;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;

public class NioZeroCopyServer {
    public static void main(String[] args) {
        InetSocketAddress address = new InetSocketAddress(3210);
        try (ServerSocketChannel serverSocketChannel = ServerSocketChannel.open()) {
            ServerSocket serverSocket = serverSocketChannel.socket();
            serverSocket.bind(address);

            ByteBuffer buffer = ByteBuffer.allocate(4096);
            while (true) {
                SocketChannel socketChannel = serverSocketChannel.accept();
                int len = 0;
                int total = 0;
                long startTime = System.currentTimeMillis();
                while (len != IOStatus.EOF) {
                    len = socketChannel.read(buffer);
                    total += len;
                    buffer.rewind();
                }
                long emdTime = System.currentTimeMillis();
                System.out.println("总字节数: " + total);
                System.out.println("总耗时: " + (emdTime - startTime));
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

---

客户端：

```java
package com.qqs.nio.client;

import java.io.File;
import java.io.FileInputStream;
import java.net.InetSocketAddress;
import java.nio.channels.FileChannel;
import java.nio.channels.SocketChannel;

public class NioZeroCopyTransferToClient {
    public static void main(String[] args) {
        File file = new File("/var/tmp/1.zip");
        try (
                SocketChannel socketChannel = SocketChannel.open();
                FileInputStream is = new FileInputStream(file);
                FileChannel fileChannel = is.getChannel()
        ) {
            socketChannel.connect(new InetSocketAddress("127.0.0.1", 3210));
            long startTime = System.currentTimeMillis();
            long len = fileChannel.transferTo(0, fileChannel.size(), socketChannel);
            long emdTime = System.currentTimeMillis();
            System.out.println("总字节数: " + len);
            System.out.println("总耗时: " + (emdTime - startTime));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

---

运行结果：

![image-20230729182951689](https://hhc-typora.oss-cn-shenzhen.aliyuncs.com/image-20230729182951689.png)

# Java AIO基本介绍

1. JDK 7 引入了 Asynchronous I/O，即 AIO。在进行 I/O 编程中，常用到两种模式：Reactor 和 Proactor。
2. Java 的NIO 就是 Reactor，当有事件触发时，服务器端得到通知，进行相应的处理。
3. AIO 即 NIO2.0，叫做异步不阻塞的 IO。
4. AIO 引入异步通道的概念，采用了 **Proactor 模式**，简化了程序编写，有效的请求才启动线程，它的特点是先由操作系统完成后才通知服务端程序启动线程去处理，一般适用于**连接数较多且连接时间较长的应用**。
5. 目前 AIO 还没有广泛应用，Netty 也是基于 NIO，而不是 AIO， 因此我们就不详解 AIO。
6. 有兴趣可参考 [<<Java 新 一 代 网 络 编 程 模 型 AIO 原 理 及 Linux 系 统 AIO 介 绍 >>]( http://www.52im.net/thread-306-1-1.html)。

# BIO、NIO、AIO对比表

|          | BIO      | NIO                    | AIO        |
| -------- | -------- | ---------------------- | ---------- |
| I/O模型  | 同步阻塞 | 同步非阻塞（多路复用） | 异步非阻塞 |
| 编程难度 | 简单     | 复杂                   | 复杂       |
| 可靠性   | 差       | 好                     | 好         |
| 吞吐量   | 低       | 高                     | 高         |
