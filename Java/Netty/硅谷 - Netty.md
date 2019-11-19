# 硅谷 - Netty

## Netty 的介绍

```shell
# Netty 是由 JBOSS 提供的一个 Java 开源框架，现为 Github 上的独立项目

# Netty 是一个异步的、基于事件驱动的网络应用框架，用以快速开发高性能、高可靠性的网络 IO 程序

# Netty 主要针对在 TCP 协议下，面向 Clients 端的高并发应用，或者 Peer-to-Peer 场景下的大量数据持续传输的应用

# Netty 本质是一个 NIO 框架，适用于服务器通讯相关的多种应用场景

# TCP/IP
	# -> Java 的 IO 编程和网络编程
	# -> NIO
	# -> Netty
```

## Netty 的应用场景

```shell
# 在分布式系统中，各个节点之间需要远程服务调用，高性能的 RPC 框架必不可少，Netty 作为异步高性能的通信框架，往往作为基础通信组件被这些 RPC 框架使用。

# 典型的应用有 阿里巴巴分布式服务框架 Dubbo 的 RPC 框架使用 Dubbo 协议进行节点间通信，Dubbo 协议默认使用 Netty 作为基础通信组件，用于实现各进程节点之间的内部通信。

# Netty 作为高性能的基础通信组件，提供了 TCP/UDP 和 HTTP 协议栈，方便定制和开发私有协议栈，账号登录服务器。

# 地图服务器之间可以方便的通过 Netty 进行高性能的通信

# 经典的 Hadoop 的高性能通信和序列化组件<AVRO 实现数据文件共享> 的RPC 框架，默认采用 Netty 进行跨界点通信。

# 它的 Netty Service 基于 Netty 实现二次封装
```

## IO 模型

### IO 模型基本说明

```shell
# IO 模型简单的理解，就是用什么样的通道进行数据的发送和接收，很大程度上决定了程序通信的性能。

# Java 共支持 3 种网络编程模型: BIO、NIO、AIO

# Java BIO: <同步并阻塞>
	# 服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事，就会造成不必要的线程开销。
	
# Java NIO: <同步非阻塞>
	# 服务器实现模式为一个线程处理多个请求连接，即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮训到连接有 I/O 请求就进行处理
	
# Java AIO: <异步非阻塞>
	# AIO 引入了异步通道的概念，采用了 Proactor 模式，简化了程序编写，有效的请求才启动线程。
	# 特点是先由操作系统完成后才通知服务端程序启动线程去处理，一般适用于连接数较多且连续时间较长的应用
```

### BIO、NIO、AIO 适用场景分析

```shell
# BIO 方式适用于连接数目较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，JDK1.4 以前的唯一选择，但程序简单易理解。

# NIO 方式适用于连接数目多且连接比较短<轻操作> 的架构，比如聊天服务器，弹幕系统，服务器间通讯等，变成比较复杂，JDK1.4 开始支持。

# AIO 方式适用于连接数目多且连接比较长<重操作> 的架构，比如相册服务器，充分调用 OS 参与并发操作，编程比较复杂，JDK1.7 开始支持。
```

## BIO

### Java BIO 工作机制

![UTOOLS1574132186691.png](https://i.loli.net/2019/11/19/1el23zFKchSugBm.png)

## NIO

### NIO 基本介绍

```shell
# Java NIO<Non-Blocking IO>:
	# JDK1.4 开始，Java 提供的同步非阻塞 IO
	
# NIO 三大核心部分:
	# Channel<通道>:
	
	# Buffer<缓冲区>:
		
	# Selector<选择器>:
	
# NIO 是面向缓冲区，或者面向块 编程的，数据读取到一个它稍后处理的缓冲区，需要时可在缓冲区中前后移动，这就增加了处理过程中的灵活性，使用它可以提供非阻塞式的高伸缩性网络。 

# NIO 的非阻塞模式，使一个线程从某通道发送请求或者读取数据，但是它仅能得到目前可用的数据，如果目前没有数据可用时，就什么都不会获取，而不是保持线程阻塞，所以直至数据变得可以读取之前，该线程可以继续做其他的事情。非阻塞写也是如此，一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程同时可以去做别的事情。

# 通俗理解: NIO 是可以做到用一个线程来处理多个操作的吗。假设有 1W 个请求过来，根据实际情况，可以分配 50 或者 100 个线程来处理，不像之前的阻塞 IO 那样，非得分配 10000 个。

# HTTP2.0 使用了多路复用的技术，做到同一个连接并发处理多个请求，而且并发请求的数量比 HTTP1.1 大了好几个数量级。
```

![UTOOLS1574132996955.png](https://i.loli.net/2019/11/19/IhwzUvdi5WVjC6n.png)

### NIO 三大核心原理示意图

```shell
# 每个 Channel 都会对应一个 Buffer

# Selector 对应一个线程，一个线程可以对应多个 Channel<连接>

# Selector 切换到哪个 Channel 是由事件决定的，Event<重要概念: 事件>

# Selector 会根据不同的事件，在各个通道上切换

# Buffer 就是一个内存块，底层就是一个数组

# 数据的读取/写入，都是通过 Buffer，和 BIO 是有本质不同的。
	# BIO 中要么是输出流，要么是输入流，不是双向的。
	# NIO 的Buffer 是可以读/写的，需要 flip 方法切换
	
# Channel 是双向的，可以返回底层操作系统的情况，比如 Linux，底层的操作系统通道就是双向的。
```

### Channel

#### 基本介绍

```shell
# BIO 中的 Stream 是单向的，例如 FileInputStream 对象只能进行读取数据的操作，而 NIO 中的 Channel 是双向的。

# Channel 在 NIO 中时一个接口
public interface Channel extends Closeable{}

# 常用的 Channel 类:
	# FileChannel
		# 用于文件的数据读写
		
	# DatagramChannel
		# 用于 UDP 的数据读写
		
	# ServerSocketChannel
		# 用于 TCP 的数据读写
		
	# SocketChannel
		# 用于 TCP 的数据读写
		
# 常见的方法:
public int read(ByteBuffer dst) : 从通道读取数据并放到缓冲区中
public int write(ByteBuffer src) : 把缓冲区的数据写到通道中
public long transferFrom(ReadableByteChannel src, long position, long count) : 从目标通道中复制数据到当前通道
public long transferTo(long position, long count, WritableByteChannel target) : 把当前通道复制给目标通道
```

#### 代码案例

```java
public class NIODemo {

    public static void main(String[] args) throws Exception{
        String str = "Hello,AgeFades!";
        // 创建一个输出流 -> channel
        FileOutputStream fileOutputStream =
                new FileOutputStream("/Users/apple/Documents/Work/beluga/demo/Hello.txt");

        // 通过 fileOutputStream 获取 Channel
        FileChannel channel = fileOutputStream.getChannel();

        ByteBuffer buffer = ByteBuffer.allocate(1024);
        buffer.put(str.getBytes()).flip();
        channel.write(buffer);
        fileOutputStream.close();
    }

}
```

```java
public class NIODemo {

    public static void main(String[] args) throws Exception{
        File file = new File("/Users/apple/Documents/Work/beluga/demo/Hello.txt");
        // 创建一个输入流 -> channel
        FileInputStream fileInputStream =
                new FileInputStream(file);

        // 通过 fileOutputStream 获取 Channel
        FileChannel channel = fileInputStream.getChannel();

        ByteBuffer buffer = ByteBuffer.allocate((int)file.length());
        // 将通道的数据读入到 ByteBuffer
        channel.read(buffer);
        System.out.println(new String(buffer.array()));

        fileInputStream.close();
    }

}
```

```java
public class NIODemo {

    public static void main(String[] args) throws Exception{
        File file = new File("/Users/apple/Documents/Work/beluga/demo/Hello.txt");

        FileInputStream fileInputStream =
                new FileInputStream(file);
        FileChannel inputChannel = fileInputStream.getChannel();

        FileOutputStream fileOutputStream =
                new FileOutputStream("/Users/apple/Documents/Work/beluga/demo/Hello2.txt");
        FileChannel outputChannel = fileOutputStream.getChannel();

        ByteBuffer buffer = ByteBuffer.allocate(512);

        while (true) {
            buffer.clear();
            int read = inputChannel.read(buffer);
            if (read == -1){
                break;
            }
            buffer.flip();
            outputChannel.write(buffer);
        }

        fileOutputStream.close();
        fileInputStream.close();
    }

}
```

```java
public class NIODemo {

    public static void main(String[] args) throws Exception{
        File file = new File("/Users/apple/Documents/Work/beluga/demo/Hello.txt");

        FileInputStream fileInputStream =
                new FileInputStream(file);
        FileChannel inputChannel = fileInputStream.getChannel();

        FileOutputStream fileOutputStream =
                new FileOutputStream("/Users/apple/Documents/Work/beluga/demo/Hello3.txt");
        FileChannel outputChannel = fileOutputStream.getChannel();

        outputChannel.transferFrom(inputChannel,0,inputChannel.size());

        fileOutputStream.close();
        fileInputStream.close();
    }

}
```

#### Buffer 类型化和只读

```shell
# 关于Buffer 和Channel 的注意事项和细节
	# ByteBuffer 支持类型化的 put 和 get，put 放入的是什么数据类型，get 就应该使用相应的数据类型来取出，否则可能有 BufferUnderflowException 异常。
	
	# 可以将一个普通 Buffer 准成只读 Buffer
	
	# NIO 还提供了 MappedByteBuffer，可以让文件直接在内存<OS 本地内存> 中进行修改，而如何同步到文件由 NIO 来完成。
	
	# NIO 还支持通过多个 Buffer<即 Buffer 数组> 完成读写操作，即 Scattering 和 Gathering
```

