[TOC]

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

##### 代码案例

```java
/**
 * 抛出 BufferUnderflowException 异常
 */
public class NIODemo {

    public static void main(String[] args) throws Exception{

        ByteBuffer buffer = ByteBuffer.allocate(64);
        buffer.putInt(100);
        buffer.putLong(9L);
        buffer.putChar('A');
        buffer.putShort((short) 4);

        buffer.flip();

        System.out.println(buffer.getShort());
        System.out.println(buffer.getInt());
        System.out.println(buffer.getLong());
        System.out.println(buffer.getLong());

    }

}
```

```java
/**
 * 抛出 java.nio.ReadOnlyBufferException 异常
 */
public class NIODemo {

    public static void main(String[] args) throws Exception{

        ByteBuffer buffer = ByteBuffer.allocate(64);
        for (int i = 0; i < 64; i++) {
            buffer.put((byte)i);
        }

        buffer.flip();
        ByteBuffer readOnlyBuffer = buffer.asReadOnlyBuffer();
        readOnlyBuffer.put((byte)1);

    }

}
```

```java
/**
 * 直接在内存中修改文件内容，省去拷贝的性能消耗
 */
public class NIODemo {

    public static void main(String[] args) throws Exception{

        RandomAccessFile rw = new RandomAccessFile("/Users/apple/Documents/Work/beluga/demo/Hello.txt", "rw");
        FileChannel channel = rw.getChannel();
        MappedByteBuffer map = channel.map(FileChannel.MapMode.READ_WRITE, 0, rw.length());
        map.put(0,(byte)'h');

        rw.close();

    }

}
```

```java
public class NIODemo {

    public static void main(String[] args) throws Exception{

        ServerSocketChannel channel = ServerSocketChannel.open();
        InetSocketAddress address = new InetSocketAddress(7000);
        channel.socket().bind(address);

        ByteBuffer[] buffers = new ByteBuffer[2];
        buffers[0] = ByteBuffer.allocate(5);
        buffers[1] = ByteBuffer.allocate(3);

        SocketChannel socketChannel = channel.accept();
        int messageLength = 8;

        while (true) {
            int byteRead = 0;

            while (byteRead < messageLength){
                long read = socketChannel.read(buffers);
                byteRead += read;
                System.out.println("byteRead : " + byteRead);
                Arrays.asList(buffers).stream()
                        .map(buffer -> "position : " + buffer.position() + "limit : " + buffer.limit())
                        .forEach(System.out::println);
            }

            Arrays.asList(buffers).forEach(buffer -> buffer.flip());

            long byteWriter = 0;

            while (byteWriter < messageLength) {
                long write = socketChannel.write(buffers);
                byteWriter += write;
            }

            Arrays.asList(buffers).forEach(buffer -> buffer.clear());

            System.out.println("byteRead : " + byteRead + "byteWriter : "
                    + byteWriter + ", messageLength : " + messageLength);
        }

    }

}
```

### Selector

```shell
# NIO 中用一个线程，处理多个的客户端连接，就会使用到 Selector<选择器>

# Selector 能够检测多个注册的通道上是否有事件发生
	# 多个 Channel 以事件的方式可以注册到同一个 Selector
	# 如果有事件发生，便获取事件然后针对每个事件进行相应的处理
	# 这样就实现了只使用一个单线程去管理多个通道，也就是管理多个连接和请求
	
# 只有在 Channel 真正有读写事件发生时，才会进行读写，就大大的减少了系统开销，并且不必为每个 Channel 都创建一个线程，不用于维护多个线程。

# 避免了多线程之间的上下文切换导致的开销。
```

#### Selector 示意图和特点说明

```shell
# Netty 的 IO 线程 NioEventLoop 聚合了 Selector，可以同时并发处理成百上千个客户端连接

# 当线程从某客户端 Socket 通道进行读写数据时，若没有数据可用时，该线程可以进行其他任务。

# 线程通常将非阻塞 IO 的空闲时间用于在其他 Channel 上执行 IO 操作，所以单独的线程可以管理多个输入和输出 Channel

# 由于读写操作都是非阻塞的，这就可以充分提升 IO 线程的运行效率，避免由于频繁 I/O 阻塞导致的线程挂起
```

![UTOOLS1574228557688.png](https://i.loli.net/2019/11/20/xcXgzUFPWQVOvD4.png)

#### Selector 部分源码

```java
public abstract class Selector implements Closeable {
	
  public static Selector open(); // 得到一个选择器对象
  
  // 监控所有注册的通道，当其中有 IO 操作可以进行时，将对应的 SelectionKey 加入到内部集合中并返回，参数用来设置超时时间
  public int select(long timeout); 
  
  // 从内部集合中得到所有的 SelectionKey
  public Set<SelectionKey> selectedKeys(); 
  
}
```

#### Selector 注意事项

```shell
# NIO 中的 ServerSocketChannel 功能类似 ServerSocket，SocketChannel 功能类似 Socket 

# selector 相关方法说明
selector.select()  # 阻塞
selector.select(1000) # 阻塞 1000 毫秒，然后返回
selector.wakeup() # 唤醒 selector
selector.selectNow() # 不阻塞，立马返回
```

#### NIO 非阻塞网络编程原理分析图

```shell
# 当客户端连接时，会通过 ServerSocketChannel 得到 SocketChannel

# Selector 进行监听 select 方法，返回有事件发生的通道的个数

# 将 socketChannel 注册到 Selector 上，register(Selector sel, int ops)
	# 一个 selector 上可以注册多个 SocketChannel
	
# 注册后返回一个 Selectionkey，会和该 Selector 关联<集合>

# 进一步得到各个 SelectionKey<有事件发生>

# 在通过 SelectionKey 反向获取 SocketChannel，channel() 方法

# 通过得到的 channel，完成业务处理
```

##### ![UTOOLS1574228861414.png](https://i.loli.net/2019/11/20/3kUVFq2w6ovf79h.png)

##### 代码验证

```java
public class NIOServer {

    public static void main(String[] args) throws Exception{

        // 创建 ServerSocketChannel
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();

        // 创建 Selector
        Selector selector = Selector.open();

        // 绑定监听端口，在服务器端监听
        serverSocketChannel.socket().bind(new InetSocketAddress(7000));

        // 设置为非阻塞
        serverSocketChannel.configureBlocking(false);

        // 将 ServerSocketChannel 注册到 Selector 上，并监听 OP_ACCEPT 事件
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        // 循环等待客户端连接
        while (true) {

            if (selector.select(10000) == 0){
                System.out.println("服务器等待10s，无任何连接");
                continue;
            }

            // 获取相关的 SelectionKey, 表示已经获取到关注的事件
            Set<SelectionKey> selectionKeys = selector.selectedKeys();

            // 通过 selectionKeys 反向获取通道
            Iterator<SelectionKey> iterator = selectionKeys.iterator();
            while (iterator.hasNext()){

                // 获取到 SelectionKey
                SelectionKey key = iterator.next();
                // 根据 key 对应的通道发生的事件做相应处理
                // 新的客户端连接事件
                if (key.isAcceptable()){

                    // 给该客户端生成一个新的 SocketChannel
                    SocketChannel socketChannel = serverSocketChannel.accept();

                    socketChannel.configureBlocking(false);

                    System.out.println("客户端连接成功，生成了一个 socketChannel : " + socketChannel.hashCode());

                    // 将 socketChannel 注册到 selector 上，关注事件为 OP_READ，同时关联一个 Buffer
                    socketChannel.register(selector, SelectionKey.OP_READ, ByteBuffer.allocate(1024));

                }
                if (key.isReadable()){

                    // 通过 key 反向获取到 channel
                    SocketChannel socketChannel = (SocketChannel)key.channel();
                    // 获取到该 channel 关联的 Buffer
                    ByteBuffer buffer = (ByteBuffer)key.attachment();
                    socketChannel.read(buffer);
                    System.out.println("客户端发来消息: " + new String(buffer.array()));
										socketChannel.close();
                }

                // 手动从集合中移除当前 Key，防止重复操作
                iterator.remove();

            }

        }

    }

}

```

```java
public class NIOClient {

    public static void main(String[] args) throws Exception{

        // 得到一个网络通道
        SocketChannel socketChannel = SocketChannel.open();

        // 设置非阻塞模式
        socketChannel.configureBlocking(false);

        // 提供服务器端的 ip 和 port
        InetSocketAddress inetSocketAddress = new InetSocketAddress("127.0.0.1", 7000);

        // 连接服务器
        if (!socketChannel.connect(inetSocketAddress)) {

            while (!socketChannel.finishConnect()){
                System.out.println("因为连接需要时间，客户端不会阻塞...");
            }

        }

        // 连接成功，发送数据
        String str = "Hello! NIO 编程!";
        // 简化版 allocate 和 put
        ByteBuffer buffer = ByteBuffer.wrap(str.getBytes());
        socketChannel.write(buffer);
        System.in.read();

    }

}

```

##### SelectionKey ops

```shell
# SelectionKey, 表示 Selector 和网络通道的注册关系，共四种:
	# int OP_ACCEPT : 有新的网络连接可以 accept，值为 16
	
	# int OP_CONNECT : 代表连接已经建立，值为 8
	
	# int OP_READ : 代表读操作，值为 1
	
	# int OP_WRITE : 代表写操作，值为 4	
```

##### SelectionKey 相关方法

```java
public abstract class SelectionKey {
  
  public abstract Selector selector(); // 得到与之关联的 Selector 对象
  
  public abstract SelecableChannel channel(); // 得到与之关联的通道
  
  public final Object attachment(); // 得到与之关联的共享数据
  
  public abstract SelectionKey interestOps(int ops); // 设置或改变监听事件
  
  public final boolean isAcceptable(); // 是否可以 accept
  
  public final boolean isReadable(); // 是否可以读
  
  public final boolean isWritable(); // 是否可以写
}
```

##### ServerSocketChannel

```java
// ServerSocketChannel 在服务端监听新的客户端 Socket 连接
public abstract class ServerSocketChannel extends AbstractSelectableChannel implements NetworkChannel {
  
  // 得到一个 ServerSocketChannel 通道
  public static ServerSocketChannel open(); 
  
  // 设置服务器端端口号
  public final ServerSocketChannel bind(SocketAddress local);
  
  // 设置阻塞或非阻塞模式，取值 false 表示采用非阻塞模式
  public final SelectableChannel configureBlocking(boolean block);
  
  // 接受一个连接，返回代表这个连接的通道对象
  public SocketChannel accept();
  
  // 注册一个选择器并设置监听事件
  public final SelectionKey register(Selector sel, int ops);
  
}
```

##### SocketChannel

```java
// SocketChannel 网络 IO 通道，具体负责进行读写操作
public abstract class SocketChannel extends AbstractSelectableChannel implements ByteChannel, ScatteringByteChannel, GatheringByteChannnel, NetworkChannel {
  
  // 得到一个 SocketChannel 通道
  public static SocketChannel open();
  
  // 设置阻塞或非阻塞模式，取值 false 表示采用非阻塞模式
  public final SelectableChannel configureBlocking(boolean block);
  
  // 连接服务器
  public boolean connect(SocketAddress remote);
  
  // 如果上面的方法连接失败，接下来就要通过该方法完成连接操作
  public boolean finishConnect();
  
  // 往通道里写数据
  public int write(ByteBuffer src);
  
  // 从通道里读数据
  public int read(ByteBuffer dst);
  
  // 注册一个选择器并设置监听事件，最后一个参数可以设置共享数据
  public final SelectionKey register(Selector sel, int ops, Object att);
  
  // 关闭通道
  public final void close();
}
```

### NIO 实现群聊系统

#### 代码

```java
public class GroupChatServer {

    // 定义属性
    private Selector selector;
    private ServerSocketChannel listenChannel;
    private static final int PORT = 7000;

    // 初始化工作
    public GroupChatServer() {

        try {

            selector = Selector.open();
            listenChannel = ServerSocketChannel.open();
            listenChannel.socket().bind(new InetSocketAddress(PORT));
            listenChannel.configureBlocking(false);
            listenChannel.register(selector, SelectionKey.OP_ACCEPT);

        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    // 监听
    public void listen() {

        System.out.println("监听线程: " + Thread.currentThread().getName());
        try {
            while (true) {

                if (selector.select(10000) != 0) {
                    // 得到 selectionKey 集合
                    Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
                    while (iterator.hasNext()) {
                        // 取出 selectionKey
                        SelectionKey key = iterator.next();

                        // 监听到新客户端连接事件
                        if (key.isAcceptable()) {

                            // 根据 selectionKey 反向获取 socketChannel
                            SocketChannel sc = listenChannel.accept();
                            sc.configureBlocking(false);
                            // 将该 sc 注册到 selector
                            sc.register(selector, SelectionKey.OP_READ);

                            // 提示
                            System.out.println(sc.getRemoteAddress() + "上线");

                        }
                        // 通道发送 read 事件，即通道是可读的状态
                        if (key.isReadable()) {
                            readData(key);
                        }

                        // 删除当前 key，防止重复处理
                        iterator.remove();
                    }

                } else {
                    System.out.println("服务端初始化完毕，正在等待客户端连接...");
                }

            }

        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    // 读取客户端信息
    private void readData(SelectionKey key) throws Exception{

        SocketChannel channel = null;
        try {
            // 通过 key 反向得到 channel
            channel = (SocketChannel) key.channel();
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            int count = channel.read(buffer);

            if (count > 0){
                // 将缓存区中的数据转成字符串
                String msg = new String(buffer.array());
                // 输出该消息
                System.out.println("从客户端发来消息: " + msg);

                sendInfoToOtherClients(msg, channel);
            }
        } catch (IOException e){
            try {
                System.out.println(channel.getRemoteAddress() + "离线了...");
                // 取消注册
                key.cancel();
                channel.close();
            } catch (Exception e2){
                e2.printStackTrace();
            }

        }

    }

    // 转发消息给其他客户
    private void sendInfoToOtherClients(String msg, SocketChannel channel) throws IOException{

        System.out.println("服务器转发消息中...");
        System.out.println("服务器转发数据给客户端线程 : " + Thread.currentThread().getName());

        for (SelectionKey key : selector.keys()) {
            Channel target = key.channel();

            // 排除自己
            if (target instanceof SocketChannel && target != channel) {
                // 转型
                SocketChannel dest = (SocketChannel)target;
                // 将 msg 存储到 buffer
                ByteBuffer buffer = ByteBuffer.wrap(msg.getBytes());
                // 将 buffer 的数据写入通道
                dest.write(buffer);

            }
        }

    }

    public static void main(String[] args) {

        GroupChatServer groupChatServer = new GroupChatServer();
        groupChatServer.listen();

    }

}
```

```java
public class GroupChatClient {

    // 定义相关属性
    private final String HOST = "127.0.0.1";
    private final int PORT = 7000;
    private Selector selector;
    private SocketChannel socketChannel;
    private String username;

    // 构造器，完成初始化工作
    public GroupChatClient() throws IOException {

        selector = Selector.open();
        socketChannel = SocketChannel.open(new InetSocketAddress(HOST,PORT));
        socketChannel.configureBlocking(false);
        socketChannel.register(selector, SelectionKey.OP_READ);
        // 得到 username
        username = socketChannel.getLocalAddress().toString().substring(1);
        System.out.println(username + " is ok...");

    }

    // 向服务器发送消息
    public void sendInfo(String info) {
        info = username + " 说: " + info;

        try {
            socketChannel.write(ByteBuffer.wrap(info.getBytes()));
        } catch (Exception e){
           e.printStackTrace();
        }
    }

    // 读取从服务端回复的消息
    public void readInfo() {

        try {
            int readChannels = selector.select();

            if (readChannels > 0) { // 有可以用的通道
                Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
                while (iterator.hasNext()) {
                    SelectionKey key = iterator.next();
                    if (key.isReadable()) {
                        // 得到相关通道
                        SocketChannel sc = (SocketChannel)key.channel();
                        // 得到一个 Buffer
                        ByteBuffer buffer = ByteBuffer.allocate(1024);
                        // 读取
                        sc.read(buffer);
                        // 把读到的缓冲区的数据转成字符串
                        String msg = new String(buffer.array());
                        System.out.println(msg.trim());
                    }
                }
                // 删除当前的 selectionKey，防止重复操作
                iterator.remove();
            } else {
                System.out.println("没有可以用的通道...");
            }

        } catch (Exception e){
            e.printStackTrace();
        }

    }

    public static void main(String[] args) throws Exception {

        // 启动我们客户端
        GroupChatClient chatClient = new GroupChatClient();

        // 启动一个线程，每个3秒，读取从服务器发送数据
        new Thread(() -> {
            while (true) {
                chatClient.readInfo();
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();

        Scanner scanner = new Scanner(System.in);

        while (scanner.hasNextLine()) {
            String line = scanner.nextLine();
            chatClient.sendInfo(line);
        }

    }

}
```

### NIO 与零拷贝

#### 零拷贝基本介绍

```shell
# 零拷贝是网络编程的关键，很多性能优化都离不开。

# 在 Java 程序中，常用的零拷贝有 mmap(内存映射) 和 sendFile。
```

#### 传统IO 数据读写

```java
File file = new File("text.txt");
RandomAccessFile raf = new RandomAccessFile(file, "rw");

byte[] arr = new byte[(int) file.length()];
raf.read(arr);

Socket socket = new ServerSocket(8080).accept();
socket.getOutputStream().write(arr);
```

#### mmap 优化

```shell
# mmap 通过内存映射，将文件映射到内核缓冲区，同时，用户空间可以共享内核空间的数据。
	# 这样在进行网络传输时，就可以减少内核空间到用户空间的拷贝次数。
```

#### sendFile 优化

```shell
# Linux2.1 版本提供了 sendFile 函数。

# 基本原理:
	# 数据根本不经过用户态，直接从内核缓冲区进入到 Socket Buffer
	# 因为和用户态完全无关，就减少了一次上下文切换
	
# 在 Linux2.4 版本中，做了一些修改，避免了从内核缓冲区拷贝到 Socket Buffer 的操作，直接拷贝到协议栈，从而再一次减少了数据拷贝。
```

#### 零拷贝的再次理解

```shell
# 我们说的零拷贝，是从操作系统的角度来说的。因为内核缓冲区之间，没有数据是重复的（只有 kernel buffer 有一份数据）

# 零拷贝不仅仅带来更少的数据复制，还能带来其他的性能优势，例如更少的上下文切换，更少的 CPU 缓存伪共享以及无 CPU 校验和计算。
```

#### mmap 和 sendFile 的区别

```shell
# mmap 适合小数据量读写，sendFile 适合大文件传输。

# mmap 需要 4 次上下文切换，3 次数据拷贝

# sendFile 需要 3 次上下文切换，最少 2 次数据拷贝

# sendFile 可以利用 DMA 方式，减少 CPU 拷贝，mmap 则不能（必须从内核拷贝到 Socket 缓冲区）
```

#### NIO 零拷贝代码

```shell
# 核心就是 transferTo、transferFrom ...
```

## AIO

### AIO 基本介绍

```shell
# JDK7 引入了 Asynchronous I/O，即 AIO

# 在进行 I/O 编程中，常用到两种模式:
	# Reactor
		# NIO 就是 Reactor，当有事件触发时，服务器端得到通知，进行相应的处理
	
	# Proactor
		# AIO < NIO2.0，异步不阻塞的 IO >
		# 引入了异步通道的概念，采用 Proactor 模式，简化程序编写，有效的请求才启动线程。
		# 特点:
			# 先由操作系统完成后才通知服务端程序启动线程去处理，一般适用于连接数多且连接时间较长的应用。
			
# 目前 AIO 还没有广泛应用，不再详解 AIO。
```

## BIO、NIO、AIO 对比表

| 比较维度 | BIO      | NIO                  | AIO        |
| -------- | -------- | -------------------- | ---------- |
| IO 模型  | 同步阻塞 | 同步非阻塞<多路复用> | 异步非阻塞 |
| 编程难度 | 简单     | 复杂                 | 复杂       |
| 可靠性   | 差       | 好                   | 好         |
| 吞吐量   | 低       | 高                   | 高         |

## Netty

### 原生 NIO 存在的问题

```shell
# NIO 的类库和 API 繁杂，使用麻烦。
	# 需要熟练掌握 ServerSocketChannel、SocketChannel、ByteBuffer、Selector...
	
# 需要具备其他的额外技能。
	# 需要熟练Java 多线程编程，因为 NIO 编程设计到了 Reactor 模式。
	
# 开发工作量和难度都非常大。
	# 例如客户端面临断连重连、网络闪断、半包读写、失败缓存、网络拥塞和异常流的处理等等...
	
# JDK NIO 的Bug。
	# 例如臭名昭著的 Epoll Bug，它会导致 Selector 空轮询，最终导致 CPU 100%。
	# 知道 JDK1.7 版本该问题仍旧存在，没有被根本解决。
```

### Netty 官网说明

```shell
# 官网地址
https://netty.io/

# Netty 提供异步、基于事件驱动的网络应用程序框架，用以快速开发高性能、高可靠性的网络 IO 程序。

# Netty 简化和流程化了 NIO 的开发过程

# Netty 是目前最流行的 NIO 框架，ElasticSearch、Dubbo 框架内部都采用了 Netty
```

### Netty 的优点

```shell
# Netty 对 JDK 自带的 NIO 的 API 进行了封装，解决了上述问题。

# 设计优雅:
	# 适用于各种传输类型的统一的 API 阻塞和非阻塞 Socket
	# 基于灵活且可扩展的事件模型，可以清晰地分离关注点
	# 高度可定制的线程模型 - 单线程，一个或多个线程池
	
# 使用方便:
	# 详细记录的 javadoc，用户指南和示例，没有其他依赖项
	# JDK5<Netty3.x> 或 6<Netty4.x> 就足够了
	
# 高性能、吞吐量更高:
	# 延迟更低，减少资源消耗，最小化不必要的内存复制
	
# 安全:
	# 完整的 SSL/TLS 和 StartTLS 支持
	
# ...
```

### Netty 版本说明

```shell
# Netty 版本分为 Netty3.x、4.x、5.x

# 因为 Netty5 出现重大 Bug，已经被官网废弃了，目前推荐使用的是 Netty4.x 的稳定版本

# 此处使用 Netty4.1x 版本

# Netty 下载地址
https://bintray.com/netty/downloads/netty/
```

## Netty 高性能架构设计

### 线程模型基本介绍

```shell
# 目前存在的线程模型有:
	# 传统阻塞 I/O 服务模型
	# Reactor 模型
	
# 根据Reactor 的数量和处理资源池线程的数量不同，有3种典型的实现:
	# 单 Reactor 单线程
	# 单 Reactor 多线程
	# 主从 Reactor 多线程
	
# Netty 线程模式:
	# Netty 主要基于主从 Reactor 多线程模型做了一定改进，其中主从 Reactor 多线程模型有多个 Reactor
```

### 传统阻塞 I/O 服务模型

![UTOOLS1574391930248.png](https://i.loli.net/2019/11/22/cuSJh3YU8mzDCrE.png)

```shell
# 工作原理图
	# 黄色表示对象，蓝色表示线程，白色表示方法
	
# 模型特点
	# 采用阻塞 IO 模式获取输入的数据
	# 每个连接都需要独立的线程完成数据的输入，业务处理，数据返回
	
# 问题分析
	# 当并发数很大，就会创建大量的线程，占用很大系统资源
	# 连接创建后，如果当前线程暂时没有数据可读，该线程会阻塞在 read 操作，造成线程资源浪费
```

### Reactor 模式

```shell
# 差一张图
```

```shell
# Reactor 模式中核心组成:
	# Reactor:
		# Reactor 在一个单独的线程中运行，负责监听和分发事件，分发给适当的处理程序来对 IO 事件做出反应。
		# 类似 SpringMVC 中的 DispatchServlet
		
	# Handlers: 
		# 处理程序执行 I/O 事件要完成的实际事件。
		# Reactor 通过调度适当的处理程序来响应 I/O 事件，处理程序执行非阻塞操作。
```

#### 单 Reactor 单线程

![UTOOLS1574392242847.png](https://i.loli.net/2019/11/22/E8B6f7N2TKdYLja.png)

##### 方案说明

```shell
# Select 是前面 I/O 复用模型介绍的标准网络编程 API，可以实现应用程序通过一个阻塞对象监听多路连接请求

# Reactor 对象通过 Selector 监控客户端请求事件，收到事件后通过 Dispatch 进行分发

# 如果是建立连接请求事件，则由 Acceptor 通过 Accept 处理连接请求，然后创建一个 Handler 对象处理连接完成后的后续业务处理

# 如果不是建立连接事件，则 Reactor 会分发调用连接对应的 Handler 来响应

# Handler 会完成 Read -> 业务处理 -> Send 的完整业务流程
```

##### 缺点

```shell
# 服务器端用一个线程通过多路复用搞定所有的 IO 操作 <包括连接、读、写等...>

# 如果客户端连接数量较多，将无法支撑。

# 前面的 NIO 群聊案例就属于这种模型
```

#### 单 Reactor 多线程

![UTOOLS1574392320439.png](https://i.loli.net/2019/11/22/axhVybweSJNrMG8.png)

```shell
# 方案说明:
	# Reactor 对象通过 select 监控客户端请求事件 ->
		# 收到事件后，通过 dispatch 进行分发 ->
		# 如果建立连接请求，则由 Acceptor 通过 accept 处理连接请求 ->
		# 创建一个 Handler 对象处理完成连接后的各种事件 ->
		# 如果不是连接请求，则由 reactor 分发调用连接对应的 handler 来处理 ->
		# handler 只负责响应事件，不做具体的业务处理，通过 read 读取数据后，会分发给后面的 worker 线程池的某个线程处理业务 ->
		# worker 线程池会分配独立线程完成真正的业务，并将结果返回给 handler ->
		# handler 收到响应后，通过 send 将结果返回给 client
```

##### 方案优缺点分析

```shell
# 优点:
	# 可以充分的利用多核 cpu 的处理能力
	
# 缺点:
	# 多线程数据共享和访问比较复杂，reactor 处理所有的事件的监听和响应，在单线程运行，在高并发场景容易出现性能瓶颈。
```

#### 主从 Reactor 多线程

![UTOOLS1574393123063.png](https://i.loli.net/2019/11/22/AVgU5PiTzRW2vIy.png)

```shell
# 方案说明
	# Reactor 主线程 MainReactor 对象通过 select 监听连接事件，收到事件后，通过 Acceptor 处理连接事件 -> 
	# 当 Acceptor 处理连接事件后，MainReactor 将连接分配给 SubReactor ->
	# Subreactor 将连接加入到连接队列进行监听，并创建 handler 进行各种事件处理 ->
	# 当有新事件发生时，Subreactor 就会调用对应的 handler 处理 ->
	# handler 通过 read 读取数据，分发给后面的 worker 线程处理 ->
	# worker 线程池分配独立的 worker 线程进行业务处理，并返回结果 ->
	# handler 收到响应的结果后，再通过 Send 将结果返回给 Client ->
	# Reactor 主线程可以对应多个 Reactor 子线程，即 MainReactor 可以关联多个 SubReactor
```

##### 方案优缺点分析

```shell
# 优点:
	# 父线程与子线程的数据交互简单职责明确，父线程只需要接收新连接，子线程完成后续的业务处理。
	# React 主线程只需要把新连接传给子线程，子线程无需返回数据。
	
# 缺点:
	# 编程复杂度较高
```

##### 结合实例

```shell
# 这种模型在许多项目中广泛使用，包括 Nginx 主从 Reactor 多进程模型，Memcached 主从多线程，Netty 主从多线程模型的支持。
```

### Reactor 模式小结

```shell
# 单 Reactor 单线程:
	# 前台接待员和服务员是同一个人
	
# 单 Reactor 多线程:
	# 1个前台接待员，多个服务员
	
# 多从 Reactor 多线程:
	# 多个前台招待员，多个服务生
```

#### Reactor 模式具有如下的优点

```shell
# 响应快，不必为单个同步时间所阻塞，虽然 Reactor 本身依然是同步的

# 可以最大程度的避免复杂的多线程以及同步问题，并且避免了多线程/进程的切换开销

# 扩展性好，可以方便的通过增加 Reactor 实例个数来充分利用 CPU 资源

# 复用性好，Reactor 模型本身与具体事件处理逻辑无关，具有很高的复用性
```

## Netty 模型

![UTOOLS1574400521473.png](https://i.loli.net/2019/11/22/IpE2AymJTkaeB3X.png)

```shell
# Netyy 抽象出两组线程池:
	# BossGroup: 专门负责接收客户端的连接
	# WorkerGroup: 专门负责网络的读写
	# BossGroup 和 WorkerGroup 都是 NioEventLoopGroup
	
# NioEventLoopGroup:
	# 相当于一个事件循环组，这个组中含有多个事件循环，每一个循环事件是 NioEventLoop
	# 多个线程，即含有多个 NioEventLoop
	
# NioEventLoop:
	# 表示一个不断循环的执行处理任务的线程
	# 每个 NioEventLoop 都有一个 Selector，用于监听绑定其上的 Socket 的网络通信
	
# Boss NioEventLoop 循环执行的步骤:
	# 轮询 accept 事件 ->
		# 处理 accept 事件，与 client 建立连接 ->
		# 生成 NioSocketChannel，并将其注册到某个 Work NioEventLoop 上的 Selector 上 ->
		# 处理任务队列的任务，即 runAllTasks

# Worker NioEventLoop 执行的步骤:
	# 轮询 read、write 事件 —>
		# 处理 read、write 事件，在对应的 NioSocketChannel 上处理 ->
		# 处理任务队列的任务，即 runAllTasks
		
# Worker NioEventLoop 处理业务时，会使用 pipeline(管道)
	# pipeline 中包含了 channel，即通过 pipeline 获取到对应通道
	# pipeline 中还维护了很多的处理器
```

### Netty 快速入门实例 - TCP 服务

```shell
# 这里代码比较多了，就不往这里放了。
```

### 异步模型

#### 基本介绍

```shell
# 异步的概念和同步相对。
	# 当一个异步过程调用发出后，调用者不能立即得到结果。
	# 实际处理这个调用的组件在完成后，通过状态、通知和回调来通知调用者。
	
# Netty 中的 I/O 操作是异步的，包括 Bind、Write、Connect 等操作会简单的返回一个 ChannelFuture

# 调用者并不能立即获得结果，而是通过 Future-Listener 机制。
	# 用户可以方便的主动获取或者通过通知机制获得 IO 操作结果
	
# Netty 的异步模型是建立在 Future 和 Callback 之上的。
	# Callback 就是回调。
	# Future 的核心思想:
		# 假设一个方法，计算过程可能非常耗时，等待方法返回显然不合适。
		# 那可以在调用方法的时候，立即返回一个 Future。
		# 后续通过 Future 去监控方法的处理过程。（即: Future-Listener 机制）
```

#### 工作示意图

![UTOOLS1575526235291.png](https://img02.sogoucdn.com/app/a/100520146/f3143b7dae9b9b9313b7a98095adb78e)

```shell
# 在使用 Netty 进行编程时，拦截操作和转换出入栈数据只需要提供 callback 或利用 future 即可。
	# 这使得 链式操作 简单、高效，并有利于编写可重用的、通用的代码。
	
# Netty 框架的目标就是让业务逻辑从网络基础应用编码中分离出来。
```

#### Future - Listener 机制

```shell
# 当 Future 对象刚刚创建时，处于非完成状态，调用者可以通过返回的 ChannelFuture 来获取操作执行的状态，注册监听函数来执行完成后的操作。

# 常见操作:
	# isDone : 判断当前操作是否完成
	# isSuccess : 判断已完成的当前操作是否成功
	# getCause : 获取已完成的当前操作失败的原因
	# isCancelled : 判断已完成的当前操作是否被取消
	# addListener : 注册监听器，当操作已完成，将会通知指定的监听器
```

### Netty 快速入门实例 - HTTP 服务

```shell
# 代码略...
```

#### Http 服务过滤资源

```shell
# 通过 HttpObject Uri 对特定资源过滤
```

#### 阶段内容梳理

```shell
# 用户程序自定义的普通任务

# 用户自定义定时任务

# 非当前 Reactor 线程调用 Channel 的各种方法

# 例如在 推送系统 的业务线程里面，根据 用户的标识，找到对应的 Channel 引用
	# 然后调用 Write 类方法向该用户推送消息，就会进入到这种场景。
	# 最终的 Write 会提交到任务队列后被 异步消费
```

```shell
# Netty 抽象出两组线程池
	# BossGroup 专门负责接收客户端连接，WorkerGroup 专门负责网络读写操作
	
	# NioEventLoop 表示一个不断循环执行处理任务的线程
		# 每个 NioEventLoop 都有一个 selector,用于监听绑定在其上的 socket 网络通道。
		
	# NioEventLoop 内部采用串行化设计
		# 从消息的读取 -> 解码 -> 处理 -> 编码 -> 发送，使用由 IO 线程 NioEventLoop 负责
		
# NioEventLoopGroup 下包含多个 NioEventLoop
	# 每个 NioEventLoop 中包含有一个 Selector，一个 taskQueue
	
	# 每个 NioEventLoop 的 Selector 上可以注册监听多个 NioChannel
	
	# 每个 NioChannel 只会绑定在唯一的 NioEventLoop 上
	
	# 每个 NioChannel 都绑定一个自己的 ChannelPipeline
		

```

### ChannelPipeline

```shell
# ChannelPipeline 是一个 Handler 的集合
	# 它负责处理和拦截 inbound 或者 outbound 的事件和操作，相当于一个贯穿 Netty 的链。

# ChannelPipeline 实现了一种高级形式的拦截过滤器模式	
	# 使用户可以完全控制事件的处理方式，以及 Channel 中各个的 ChannelHandler 如何相互交互
```

```shell
# 在 Netty 中每个 Channel 都有且仅有一个 ChannelPipeline 与之对应

# 组成关系如下图:
```

![UTOOLS1575532207356.png](https://img02.sogoucdn.com/app/a/100520146/ce39b9a2b3be01ecbf51cdc6f4766926)

```shell
# 一个 Channel 包含了一个 ChannelPipeline
	# 而 ChannelPipeline 中又维护了一个由 ChannelHandlerContext 组成的双向链表
	# 并且每个 ChannelHandlerContext 中又关联着一个 ChannelHandler
	
# 入栈事件和出栈事件在一个双向链表中
	# 入栈事件会从链表 head 往后传递到最后一个入栈的 handler
	# 出栈事件会从链表 tail 往前传递到最前一个出栈的 handler
	# 两种类型的 handler 互不干扰
```

#### 常用方法

```shell
# ChannelPipeline addFirst(ChannelHandler... handlers) :
	# 把一个业务处理类（handler）添加到链中的第一个位置
	
# ChannelPipeline addLast(...)
	# 跟上面的相反
```

### Netty 核心模块组件梳理

#### BootStrap、ServerBootStrap

```shell
# BootStrap 意思是引导
	# 一个 Netty 应用通常由一个 BootStrap 开始
	# 主要作用是配置整个 Netty 程序，串联各个组件
	# Netty 中 BootStrap 类是客户端程序的启动引导类
	# ServerBootStrap 是服务端启动引导类
```

##### 常见方法

```java
// 该方法用于服务器端，用来设置两个 EventLoop
public ServerBootStrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup)
  
// 该方法用于客户端，用来设置一个 EventLoop
public B group(EventLoopGroup group)
  
// 该方法用来设置一个服务器端的通道实现
public B channel( Class<? extends C> channelClass)
  
// 用来给 ServerChannel 添加配置
public <T> B option( ChannelOption<T> option, T value )
  
// 用来给接收到的通道添加配置
public <T> ServerBootStrap childOption( ChannelOption<T> childOption, T value )

// 该方法用来设置业务处理类 (自定义的 handler)
public ServerBootStrap childHandler( ChannelHandler childHandler )
  
// 该方法用于服务器端，用来设置占用的端口号
public ChannelFuture bind( int inetPort )
  
// 该方法用于客户端，用来连接服务器
public ChannelFuture connect( String inetHost, int inetPort )
```

#### Future、ChannelFuture

```shell
# Netty 中所有的 IO 操作都是异步的，不能立刻得知消息是否被正确处理。
	# 但是可以过一会等它执行完成或者直接注册一个监听
	# 具体的实现就是通过 Future 和 ChannelFuture
	# 他们可以注册一个监听，当操作执行成功或失败时监听会自动触发注册的监听事件。
```

##### 常见方法

```java
// 返回当前正在进行 IO 操作的通道
public Channel channel()
  
// 等待异步操作执行完毕
public ChannelFuture sync()
```

#### Channel

```shell
# Netty 网络通信的组件，能够用于执行网络 I/O 操作。
	# 通过 Channel 可获得当前网络连接的通道的状态
	# 通过 Channel 可获得网络连接的配置参数（例如接收缓冲区大小）
	
# Channel 提供异步的网络 I/O 操作（如建立连接，读写，绑定端口）
	# 异步调用意味着任何 I/O 调用都将立即返回，并且不保证在调用结束时所请求的 I/O 操作已完成
	# 调用立即返回一个 ChannelFuture 实例
	# 通过注册监听器到 ChannelFuture 上，可以 I/O 操作成功、失败或取消时回调通知调用方
	
# 支持关联 I/O 操作与对应的处理程序
	
# 不同协议、不同的阻塞类型的连接都有不同的 Channel 类型与之对应
```

##### 常用 Channel 类型

```shell
# NioSocketChannel : 异步的客户端 TCP Socket 连接

# NioServerSocketChannel : 异步的服务端 TCP Socket 连接

# NioDatagramChannel : 异步的 UDP 连接

# NioSctpChannel : 异步的客户端 Sctp 连接

# NioSctpServerChannel : 异步的 Sctp 服务器端连接

# 这些通道涵盖了 UDP 和 TCP 网络 IO 以及文件 IO
```

#### Selector

```shell
# Netty 基于 Selector 对象实现 I/O 多路复用

# 通过 Selector 一个线程可以监听多个连接的 Channel 事件

# 当向一个 Selector 中注册 Channel 后，
	# Selector 内部的机制就可以自动不断地查询（select）这些注册的 Channel 是否有已就绪的 I/O 事件
	# 例如: 可读、可写、网络连接完成等
	# 这些程序就可以很简单地使用一个线程高效地管理多个 Channel
```

#### ChannelHandler 及其实现类

```shell
# ChannelHandler 是一个接口，处理 I/O 事件或拦截 I/O 操作
	# 并将其转发到其 ChannelPipeline（业务处理链）中的下一个处理程序
	
# ChannelHandler 本身并没有提供很多方法，因为这个接口有许多的方法需要实现
	# 方便使用者继承子类
	
# 我们经常需要自定义一个 Handler 类去继承 ChannelInboundHandlerAdapter
	# 通过重写相应方法实现业务逻辑
	# public void channelActive( ChannelHandlerContext ctx ) : 通道就绪事件
	# public void channelRead( ... ) : 通道读取数据事件
	# ...
	
# ChannelHandler 及其实现类一览图: 
```

![UTOOLS1575538310081.png](https://img04.sogoucdn.com/app/a/100520146/29fe90218c744679cf6dd95864056a8c)

#### ChannelHandlerContext

```shell
# 保存 Channel 相关的所有上下文信息，同时关联一个 ChannelHandler 对象

# 即 ChannelHandlerContext 中包含一个具体的事件处理器 ChannelHandler
	# 同时 ChannelHandlerContext 中也绑定了对应的 pipeline 和 Channel 的信息
	# 方便对 ChannelHandler 进行调用
```

##### 常用方法

```shell
# ChannelFuture close() : 关闭通道

# ChannelOutboundInvoker flush() : 刷新

# ChannelFuture writeAndFlush( Object msg ) : 
	# 将数据写到 ChannelPipeline 中当前 ChannelHandler 的下一个 ChannelHandler 开始处理
```

#### ChannelOption

```shell
# Netty 在创建 Channel 实例后，一般都需要设置 ChannelOption 参数

# ChannelOption 参数如下:
	# ChannelOption.SO_BACKLOG : 
		# 对应 TCP/IP 协议 listen 函数中的 backlog 参数，用来初始化服务器可连接队列大小。
		# 服务端处理客户端连接请求是顺序处理的
		# 所以同一时间只能处理一个客户端连接
		# 多个客户端来的时候，服务端将不能处理的客户端连接请求放在队列中等待处理
		# backlog 参数指定了队列的大小
		
	# ChannelOption.SO_KEEPALIVE
    # 一直保持连接活动状态
```

#### EventLoopGroup 和其实现类 NioEventLoopGroup

```shell
# EventLoopGroup 是一组 EventLoop 的抽象
	# Netty 为了更好的利用多核 CPU 资源，一般会有多个 EventLoop 同时工作
	# 每个 EventLoop 维护着一个 Selector 实例
	
# EventLoopGroup 提供 next 接口
	# 可以从组里面按照一定规则获取其中一个 EventLoop 来处理任务
	# 在 Netty 服务器端编程中，我们一般都需要提供两个 EventLoopGroup
	# 例如: BossEventLoopGroup 和 WorkerEventLoopGroup
	
# 通常一个服务端口即一个 ServerSocketChannel 对应一个 Selector 和一个 EventLoop 线程
	# BossEventLoop 负责接收客户端的连接并将 SocketChannel 交给 
		# WorkerEventLoopGroup 来进行 IO 处理
		
# 常用方法
public NioEventLoopGroup() : 构造方法
public Future<?> shutdownGracefully() : 断开连接，关闭线程
```

![UTOOLS1575539904374.png](https://img04.sogoucdn.com/app/a/100520146/0f698b6e2dbeaa91ed0544f7e0ac4f00)

#### Unpooled 类

```shell
# Netty 提供一个专门用来操作缓冲区（即 Netty 的数据容器）的工具类

# 常用方法:
public static ByteBuf copiedBuffer( CharSequence string, Charset charset ) :
	# 通过给定的数据和字符编码返回一个 ByteBuf 对象（类似于 NIO 中的 ByteBuffer 但有区别）
```

##### 代码案例

```shell
# 代码略...
```

### Netty 群聊系统

```shell
# 代码略...
```

### Netty 心跳机制实例

```shell
# 代码略...
```

### Netty 实现 WebSocket 长连接开发

```shell
# 代码略...
```

## Google Protobuf

### 编码和解码的基本介绍

```shell
# 编写网络应用程序时，因为数据在网络中传输的都是二进制字节码数据
	# 在发送数据时就需要编码，接收数据时就需要解码
	
# codec<编解码器> 的组成部分有两个:
	# decoder<解码器> : 负责把字节码数据转换成业务数据
	# encoder<编码器> : 负责把业务数据转换成字节码数据
```

### Netty 本身的编码解码的机制和问题分析

```shell
# Netty 自身提供了一些 codec
	# StringEncoder : 对字符串数据进行编码
	# ObjectEncoder : 对 Java 对象进行编码
	# 反之就是解码器...
	
# ObjectCodec 底层使用的是 Java 序列化，所以存在如下问题:
	# 无法跨语言
	# 序列化后的体积太大，是二进制编码的 5 倍多
	# 序列化性能太低
	
# 所以引入了新的解决方案 -> Google 的 Protobuf
```

### Protobuf

```shell
# Google 发布的开源项目，是一种轻便高效的结构化数据存储格式
	# 很适合做数据存储或 RPC 数据交换格式
	
# 目前很多公司:
	# http + json
	# tcp + protobuf
	
# Protobuf 是以 message 的方式来管理数据的，支持跨平台、跨语言
```

![UTOOLS1575597504431.png](https://img03.sogoucdn.com/app/a/100520146/00693bec36fd0c210359d3e82da3b5ec)

## Netty 编解码器和 handler 的调用机制

### 基本说明

```shell
# Netty 的主要组件:
	# Channel
	# EventLoop
	# ChannelFuture
	# ChannelHandler
	# ChannelPipe
	# ...
```

```shell
# ChannelHandler 充当了入栈和出栈数据的应用程序逻辑的容器。
	
# ChannelPipeline 提供了 ChannelHandler 链的容器
```

### 编码解码器

```shell
# 当 Netty 发送或者接收一个消息的时候，就会发生一次数据转换。

# Netty 提供了一系列使用的编解码器，都实现了 Handler 接口并重写 channelRead 方法

# 以入栈为例:
	# 每个从入栈 Channel 读取的消息，这个方法会被调用。
	# 随后，它将调用由解码器所提供的 decode() 方法进行解码，
	# 并将已经解码的字节转发给 ChannelPipeline 中的下一个 ChannelInboundHandler
```

### 解码器-ByteToMessageDecoder

#### 关系继承图

![UTOOLS1581385516387.png](http://yanxuan.nosdn.127.net/e3461543f4d7c87aa9a774cfa9a0cfa1.png)

```shell
# 由于不可能知道远程节点是否会一次性发送一个完整的信息
	# tcp 可能出现粘包拆包的问题，这个类会对入栈数据进行缓冲，知道它准备好被处理。
```

#### 实例分析

```java
public class ToIntegerDecoder extends ByteToMessageDecoder {
  @Override
  protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
    if (in.readableBytes() >= 4) {
      out.add(in.readInt());
    }
  }
}
```

```shell
# 说明:
	# 这个例子，每次入栈从 ByteBuf 中读取4字节，将其解码为一个int
		# 然后将它添加到下一个List 中。
	# 当没有更多元素可以被添加到该 List 中时，
		# 它的内容会被发送到下一个 ChannelInboundHandler。
	# int 在被添加到 List 中时，会被自动装箱为 Integer。
	# 在调用 readInt() 方法前，必须验证所输入的ByteBuf 是否有足够的数据
```

#### Decode 执行分析图

![UTOOLS1581385890513.png](http://yanxuan.nosdn.127.net/6bb3f8c9698633a1986764a5771f219c.png)

### Netty 的 handler 链的调用机制

![UTOOLS1581385959531.png](http://yanxuan.nosdn.127.net/0cc6247b688f89767538de72897e528e.png)

```shell
# 不论编码器 handler 还是解码器 handler
	# 接收的消息类型必须与待处理的消息类型一致，否则该 handler 不会被执行
	
# 在解码器进行数据解码时，需要判断缓冲区(ByteBuf) 的数据是否足够
	# 否则接收到的结果和期望结果可能不一致
```

### 解码器 ReplayingDecoder

```shell
# ReplayingDecoder 扩展了 ByteToMessageDecoder 类

# 使用这个类，不必调用 readableBytes() 方法

# 参数 S 指定了用户状态管理的类型，其中 Void 代表不需要状态管理
```

#### 应用实例

```java
public class MyByteToLongDecoder2 extends ReplayingDecoder<Void> {
  @Override
  protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
    // 在 ReplayingDecoder 不需要判断数据是否足够读取，内部会进行处理判断
    out.add(in.readLong());
  }
}
```

```shell
# 并不是所有的 ByteBuf 操作都被支持，如果调用了一个不被支持的方法，
	# 将会抛出 UnsupportedOperationException
	
# ReplayingDecoder 在某些情况下可能稍慢于 ByteToMessageDecoder,
	# 例如网络缓慢并且消息格式复杂时，消息会被拆成了多个碎片，速度变慢。
```

### 其他编解码器

#### 其他解码器

```shell
# LineBasedFrameDecoder:
	# 该类在 Netty 内部也有使用，它使用行位控制字符(\n 或者 \r\n ) 作为分隔符来解析数据。
	
# DelimiterBasedFrameDecoder:
	# 使用自定义的特殊字符作为消息的分隔符。
	
# HttpObjectDecoder:
	# 一个 HTTP 数据的解码器。
	
# LengthFieldBasedFrameDecoder:
	# 通过指定长度来标识整包消息，这样就可以自动的处理粘包和半包消息。
```

#### 其他编码器

![UTOOLS1581387311350.png](http://yanxuan.nosdn.127.net/56e9cb9939d57566e6ff3eeecad116a0.png)

## TCP 粘包和拆包及解决方案

### TCP 粘包和拆包基本介绍

```shell
# TCP 是面向连接的，面向流的，提供高可靠性服务。
	# 收发两端都要有一一成对的 socket

# 因此，发送端为了将多个发给接收端的包，更有效的发送给对方
	# 使用了优化方法(Nagle 算法)
	# 将多次间隔较小且数据量小的数据，合并成一个大的数据块，然后进行封包。
	# 这样虽然提高了效率，但是接收端就难于分辨出完整的数据包了。
	# 因为面向流的通信是无消息保护边界的。
	
# 由于 TCP 无消息保护边界，需要在接收端处理消息边界问题。
	# 也就是所谓的 粘包、拆包问题。
```

#### 图解拆包粘包

![UTOOLS1581390762353.png](http://yanxuan.nosdn.127.net/f7f31d5a08b6e56d1f8692f3159f7b0c.png)

```shell
# 假设客户端分别发送了两个数据包 D1 和 D2 给客户端，
	# 由于服务端一次读取到字节数是不确定的，故可能存在以下四种情况:
		# 服务端分两次读取D1、D2，没有粘包和拆包
		# 服务端一次接收到了两个数据包，D1 和 D2 粘合在一起，称之为 TCP 粘包
		# 两次读取，第一次完整的D1和部分D2，第二次部分 D2，称之为 TCP 拆包
		# 两次读取，第一次部分D1，第二次部分D1和完整D2，称之为 TCP 拆包
```

### TCP 粘包和拆包解决方案

```shell
# 使用自定义
```

