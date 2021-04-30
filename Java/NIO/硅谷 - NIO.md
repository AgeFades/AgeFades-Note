[TOC]

# 硅谷 - NIO

## Java NIO 简介

```shell
# Java NIO <New IO> 是从 Java1.4 版本开始引入的一个新的IO Api，可以替代标准的 Java IO Api。

# NIO 与原来的 IO 有同样的作用和目的，但是使用的方式完全不同，NIO 支持面向缓冲区的、基于通道的 IO 操作。 NIO 将以更加高效的方式进行文件的读写操作。
```

## NIO 与 IO 的主要区别

| IO                        | NIO                           |
| ------------------------- | ----------------------------- |
| 面向流（Stream Oriented） | 面向缓冲区（Buffer Oriented） |
| 阻塞IO（Bloking IO）      | 非阻塞IO（Non Bloking IO）    |
| （无）                    | 选择器（Selectors）           |

## 通道与缓冲区

```shell
# Java NIO 系统的核心

# 通道表示打开到 IO 设备<例如: 文件、套接字> 的连接。

# 使用 NIO，需要获取用于连接 IO 设备的通道以及用于容纳数据的缓冲区。

# 然后操作缓冲区，对数据进行处理。

# 简而言之: Channel 负责传输，Buffer 负责存储
```

### 缓冲区

```shell
# Buffer:
	# 一个用于特定基本数据类型的容器。由 java.nio 包定义，所有缓冲区都是 Buffer 抽象类的子类
	
# 主要用于与 NIO 通道进行交互，数据从通道读入缓冲区，从缓冲区写入通道中

# Buffer 就像一个数组，可以保存多个相同类型的数据。

# Buffer 常用子类:
	# ByteBuffer
	# CharBuffer
	# ShortBuffer
	# IntBuffer
	# LongBuffer
	# FloatBuffer
	# DoubleBuffer
	
# 所有Buffer 子类都采用相似的方法进行管理数据，只是各自的管理数据类型不同。

# 获取Buffer 对象的方法:
	# static XxxBuffer allocate(int capacity)
	# 创建一个容量为 capacity 的XxxBuffer 对象
```

#### Buffer 中的重要概念

```shell
# 容量<capacity>:
	# 表示 Buffer 最大数据容量，缓冲区容量不能为负，并且创建后不能更改。
	
# 限制<limit>:
	# 第一个不应该读取或写入的数据的索引，即位于 limit 后的数据不可读写。
	# 缓冲区的限制不能为负，并且不能大于其容量。
	
# 位置<position>: 
	# 下一个要读取或写入的数据的索引。
	# 缓冲区的位置不能为负，并且不能大于其限制。
	
# 标记与重置<mark And reset>
	# 标记是一个索引，通过 Buffer 中的 mark() 方法指定 Buffer 中一个特定的 position，之后可以通过调用 reset() 方法恢复到这个 position。
	
# 标记、位置、限制、容量 遵循以下不变式:
	# 0 <= mark <= position <= limit <= capacity
```

#### Buffer 的常用方法

```shell
# Buffer clear():
	# 清空缓冲区并返回对缓冲区的引用。
	
# Buffer flip():
	# 将缓冲区的界限设置为当前位置，并将当前位置重置为0
	
# int capacity():
	# 返回 Buffer 的 capacity 大小
	
# boolean hasRemaining():
	# 判断缓冲区中是否还有元素
	
# int limit():
	# 返回 Buffer 的界限<limit> 的位置
	
# Buffer limit(int n):
	# 将设置缓冲区界限为 n，并返回一个具有新 limit 的缓冲区对象
	
# Buffer mark():
	# 对缓冲区设置标记
	
# int position():
	# 返回缓冲区的当前位置 position
	
# Buffer position(int n):
	# 将设置缓冲区的当前位置为 n，并返回修改后的 Buffer 对象
	
# int reamining():
	# 返回 position 到 limit 之间的元素个数
	
# Buffer reset():
	# 将位置 position 转到以前设置的 mark 所在的位置
	
# Buffer rewind():
	# 将位置设为0，取消设置的 mark
```

#### 缓冲区的数据操作

```shell
# Buffer 所有子类提供了两个用于数据操作的方法:
	# get()
	# put()
	
# 获取 Buffer 中的数据:
	# get(): 读取单个字节
	# get(byte[] dst): 批量读取多个字节到 dst 中
	# get(int index): 读取指定索引位置的字节(不会移动 position)
	
# 放入数据到 Buffer 中:
	# put(byte b): 将给定单个字节写入缓冲区的当前位置
	# put(byte[] src): 将 src 中的字节写入缓冲区的当前位置
	# put(int index, byte b): 将指定字节写入缓冲区的索引位置(不会移动 position)
```

#### 直接与非直接缓冲区

```shell
# 字节缓冲区要么是直接的，要么是非直接的。

# 直接字节缓冲区:
	# JVM 会尽最大努力直接在此缓冲区上执行本机 I/O 操作。
	
	# 每次调用基础操作系统的一个本机 I/O 操作之前(或之后)，JVM 都会尽量避免将缓冲区的内容复制到中间缓冲区中。(或从中间缓冲区中复制内容)
	
	# 直接字节缓冲区可以通过此类的 allocateDirect() 工厂方法来创建，此方法返回的缓冲区进行分配和取消分配所需成本通常高于非直接缓冲区。
	
	# 直接缓冲区的内容可以驻留在 JVM 常规的垃圾回收堆之外，因此，它们对 Application 的内存需求量造成的影响可能并不明显。所以，建议将直接缓冲区主要分配给那些易受基础操作系统的本机 I/O 操作影响的大型、持久的缓冲区。
	
	# 一般情况下，最好仅在直接缓冲区能在程序性能方面带来明显好处时分配它们。
	
	# 直接字节缓冲区还可以通过 FileChannel 的 map() 方法将文件区域直接映射到内存中来创建。该方法返回 MappedByteBuffer，Java 平台的实现有助于通过 JNI 从本机代码创建直接字节缓冲区。如果以上这些缓冲区中的某个缓冲区实例指的是不可访问的内存区域，则试图放问该区域不会更改缓冲区的内容，并且将会在访问期间或稍后的某个时间导致抛出不确定的异常。
	
	# 字节缓冲区是直接缓冲区还是非直接缓冲区可通过调用其 isDirect() 方法来确定，提供此方法是为了能够在性能关键性代码中执行显示缓冲区管理。	
```

![UTOOLS1573182451353.png](https://i.loli.net/2019/11/08/jL6xh2YX7OnAHKZ.png)

![UTOOLS1573182474457.png](https://i.loli.net/2019/11/08/6PnoWKIQ2CEdT4l.png)

### 通道

```shell
# Channel:
	# java.nio.channels 包定义。
	# 表示 IO 源于目标打开的连接。
	# 类似传统的 "流"。只不过本身不能直接访问数据，只能与 Buffer 进行交互。
```

![UTOOLS1573183471709.png](https://i.loli.net/2019/11/08/vusPiVqpd5zF6GM.png)

#### 主要实现类

```shell
# FileChannel:
	# 用于读取、写入、映射和操作文件的通道。
	
# DatagramChannel:
	# 通过 UDP 读写网络中的数据通道。
	
# SocketChannel: 
	# 通过 TCP 读写网络中的数据。
	
# ServerSocketChannel:
	# 可以监听新进来的 TCP 连接，对每一个新进来的连接都会创建一个 SocketChannel。
```

#### 获取通道

```shell
# 获取通道的一种方式是对支持通道的对象调用 getChannel() 方法，支持通道的类有:
	# FileInputStream
	# FileOutputStream
	# RandomAccessFile
	# DatagramSocket
	# Socket
	# ServerSocket
	
# 获取通道的其他方式是使用 Files 类的静态方法，newByteChannel() 获取字节通道。或者通过通道的静态方法 open() 打开并返回指定通道。
```

#### 通道的数据传输

```shell
# 将 Buffer 中数据写入 Channel
	# 例如: int bytesWritten = inChannel.write(buf);
	
# 从 Channel 读取数据到 Buffer
	# 例如: int bytesRead = inChannel.read(buf);
```

#### 分散和聚集

```shell
# 分散读取<Scattering Reads> 是指从 Channel 中读取的数据分散到多个 Buffer 中
	# 注意: 按照缓冲区的顺序，从 Channel 中读取的数据依次将 Buffer 填满。
	
# 聚集写入<Gathering Writes> 是指将多个 Buffer 中的数据"聚集" 到 Channel。
	# 注意: 按照缓冲区的顺序，写入 position 和 limit 之间的数据到 Channel。
```

#### transferFrom()

```shell
# 将数据从源通道传输到其他 Channel 中:
```

```java
RandomAccessFile fromFile = new RandomAccessFile("data/fromFile.txt","rw");
FileChannel fromChannel = fromFile.getChannel();

RandomAccessFile toFile = new RandomAccessFile("data/toFile.txt","rw");
FileChannel toChannel = toFile.getChannel();

// 定义传输位置
long position = 0L;

// 最多传输的字节数
long count = fromChannel.size();

// 将数据从源通道传输到另一个通道
toChannel.transferFrom(fromChannel, count, position);
```

#### transferTo()

```shell
# 将数据从源通道传输到另一个通道
# 上述代码最后一句发生变化:
fromChannel.transferTo(position, count, toChannel);
```

#### FileChannel 的常用方法

| 方法                          | 描述                                         |
| ----------------------------- | -------------------------------------------- |
| int read(ByteBuffer dst)      | 从Channel 中读取数据到 ByteBuffer            |
| long read(ByteBuffer[] dsts)  | 将Channel 中的数据"分散"到 ByteBuffer[]      |
| int write(ByteBuffer src)     | 将ByteBuffer 中的数据写入到 Channel          |
| long write(ByteBuffer[] srcs) | 将ByteBuffer[] 中数据"聚集" 到 Channel       |
| long position()               | 返回此通道的文件位置                         |
| FileChannel position(long p)  | 设置此通道的文件位置                         |
| long size()                   | 返回此通道的文件的当前大小                   |
| FileChannel truncate(long s)  | 将此通道的文件截取为给定大小                 |
| void force(boolean metaData)  | 强制将所有对此通道的文件更新写入到存储设备中 |

## NIO 的非阻塞式网络通信

### 阻塞与非阻塞

```shell
# 传统的 IO 流都是阻塞式的，当一个线程调用 read() 或 write() 时，该线程被阻塞，直到有一些数据被读取或写入，该线程在此期间不能执行其他任务。

# 因此，在完成网络通信进行 IO 操作时，由于线程会阻塞，所以服务器端必须为每个客户端都提供一个独立的线程进行处理，当服务端需要处理大量客户端时，性能急剧下降。

# NIO 是非阻塞模式的。当线程从某通道进行读写数据时，若没有数据可用时，该线程可以进行其他任务。线程通常将非阻塞 IO 的空闲时间用于在其他通道上执行 IO 操作，所以单独的线程可以管理多个输入和输出管道。因此，NIO 可以让服务器端使用一个或有限几个线程来同时处理连接到服务器端的所有客户端。
```

### 选择器

```shell
# Selector 是 SelectableChannel 对象的多路复用器，Selector 可以同时监控多个 SelectableChannel 的IO 状况，也就是说，利用 Selector 可使一个单独的线程管理多个 Channel。Selector 是非阻塞 IO 的核心。
```

#### SelectableChannel 结构图

![UTOOLS1573197242137.png](https://i.loli.net/2019/11/08/cE4T9qNaxSWor1b.png)

#### 选择器的应用

```java
// 创建 Selector
Selector selector = Selector.open();

// 向选择器注册通道
// 创建一个 Socket 套接字
Socket socket = new Socket(InetAddress.getByName("127.0.0.1"),9898);

// 获取 SocketChannel
SocketChannel channel = socket.getChannel();

// 将 SocketChannel 切换到非阻塞模式
channel.configureBlocking(false);

/**
 * 向 Selector 注册 Channel
 * 当调用 register(Selector,int ops) 将通道注册选择器时，选择器对通道的监听事件，需要通过第
 * 二个参数 ops 指定。
 * 可以监听的事件类型(可使用 SelectionKey 的四个常量表示):
 *		读: SelectionKey.OP_READ
 *		写: SelectionKey.OP_WRITE
 *		连接: SelectionKey.OP_CONNECT
 *		接收: SelectionKey.OP_ACCEPT
 * 若注册时不止监听一个事件，则可以使用 | 操作符连接，例:
 *		int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
 */
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
```

#### SelectionKey

```shell
# SelectionKey:
	# 表示 SelectableChannel 和 Selector 之间的注册关系。每次向选择器注册通道时，就会选择一个事件(选择键)。
	
	# 选择键包含两个表示为整数值的操作集，操作集的每一位都表示该键的通道所支持的一类可选择操作。
```

| 方法                        | 描述                             |
| --------------------------- | -------------------------------- |
| int interestOps()           | 获取感兴趣时间集合               |
| int readyOps()              | 获取通道已经准备就绪的操作的集合 |
| SelectableChannel channel() | 获取注册通道                     |
| Selector selector()         | 返回选择器                       |
| boolean isReadable()        | 检测 Channel 中读事件是否就绪    |
| boolean isWritable()        | 检测 Channel 中写事件是否就绪    |
| boolean isConnectable()     | 检测 Channel 中连接是否就绪      |
| boolean isAcceptable()      | 检测 Channel 中接收是否就绪      |

#### Selector 的常用方法

| 方法                     | 描述                                                         |
| ------------------------ | ------------------------------------------------------------ |
| Set<SelectionKey> keys() | 所有的 SelectionKey 集合，代表注册在该 Selector 上的 Channel |
| selectedKeys()           | 被选择的SelectionKey 集合，返回此 Selector 的已选择键集      |
| int select()             | 监测所有注册的 Channel，当它们中间有需要处理的 IO 操作时，该方法返回，并将对应的 SelectionKey 加入被选择的 SelectionKey 集合中，该方法返回这些 Channel 的数量。 |
| int select(long timeout) | 可以设置超时时长的 select() 操作                             |
| int selectNow()          | 执行一个立即返回的 select() 操作，该方法不会阻塞线程         |
| Selector wakeup()        | 使一个还未返回的 select() 方法立即返回                       |
| void close()             | 关闭该选择器                                                 |

### SocketChannel

```shell
# Java NIO 中的 SocketChannel 是一个连接到 TCP 网络套接字的通道。

# 操作步骤:
	# 打开 SocketChannel
	# 读写数据
	# 关闭 SocketChannel
	
# NIO 中的 ServerSocketChannel 是一个可以监听新进来的 TCP 连接的通道，就像标准 IO 中的 ServerSocket 一样。
```

### DatagramChannel

```shell
# NIO 中的 DatagramChannel 是一个能收发 UDP 包的通道。

# 操作步骤:
	# 打开 DatagramChannel
	# 接收/发送数据
```

### 管道

```shell
# NIO 管道是2个线程之间的单向数据连接。

# Pipe 有一个 source 通道和一个 sink 通道。

# 数据会被写到 sink 通道，从 source 通道读取。
```

![UTOOLS1573199376680.png](https://i.loli.net/2019/11/08/PD1TC7h3UtV6NQg.png)

#### 向管道写数据

```java
@Test
public void test1() throws IOException{
    String str = "快乐的一只小青蛙";
    
    // 创建管道
    Pipe pipe = Pipe.open();
    
    // 向管道写输入
    Pipe.SinkChannel sinkChannel = pipe.sink();
    ByteBuffer buf = ByteBuffer.allocate(1024);
    buf.clear();
    buf.put(str.getBytes());
    buf.flip();
    
    while(buf.hasRemaining()){
        sinkChannel.write(buf);
    }
}
```

#### 从管道读取数据

```java
Pipe.SourceChannel sourceChannel = pipe.source();

ByteBuffer buf = ByteBuffer.allocate(1024);
sourceChannel.read(buf);
```

## NIO.2 - Path、Paths、Files

### NIO.2

```shell
# 随着 JDK7 的发布，Java 对 NIO 进行了极大的扩展，增强了对文件处理和文件系统特性的支持，因此被称为 NIO.2。
```

### Path 与 Paths

```shell
# java.nio.file.Path 接口代表一个平台无关的平台路径，描述了目录结构中文件的位置。

# Paths 提供的 get() 方法用来获取 Path 对象
	# Path get(String first,String ...more) : 用于将多个字符串串联成路径。
	
# Path 常用方法:
	# boolean endsWith(String path): 判断是否以 path 路径结束
	# boolean startsWith(String path): 判断是否以 path 路径开始
	# boolean isAbsolute(): 判断是否绝对路径
	# Path getFileName(): 返回与调用 Path 对象关联的文件名
	# Path getName(int idx): 返回的指定索引位置 idx 的路径名称
	# int getNameCount(): 返回 Path 根目录后面元素的数量
	# Path getParent(): 返回 Path 对象包含整个路径，不包含 Path 对象指定的文件路径
	# Path getRoot(): 返回调用 Path 对象的根路径
	# Path resolve(Path p): 将相对路径解析为绝对路径
	# Path toAbsolutePath(): 作为绝对路径返回调用 Path 对象
	# String toString(): 返回调用 Path 对象的字符串表示形式
```

### Files 类

```shell
# java.nio.file.Files 用于操作文件或目录的工具类

# Files 常用方法:
	# Path copy(Path src, Path dest, CopyOption ...how): 文件的复制
	# Path createDirectory(Path path,FileAttribute<?> ...attr): 创建一个目录
	# Path createFile(Path path,FileAttribute<?> ...attr): 创建一个文件
	# void delete(Path path): 删除一个文件
	# Path move(Path src, Path dest, CopyOption ...how): 将 src 移动到 dest位置
	# long size(Path path): 返回path 指定文件的大小
	
# Files 常用方法: 用于判断
	# boolean exists(Path path, LinkOption ...opts): 判断文件是否存在
	# boolean isDirectory(Path path, LinkOption ...opts): 判断是否是目录
	# boolean isExecutable(Path path): 判断是否是可执行文件
	# boolean isHidden(Path path): 判断是否是隐藏文件
	# boolean isReadable(Path path): 判断文件是否可读
	# boolean isWritable(Path path): 判断文件是否可写
	# boolean notExists(Path path, LinkOption ...opts): 判断文件是否不存在
	
# Files 常用方法: 用于操作内容
	# SeekableByteChannel newByteChannel(Path path,OpenOption ...how): 获取与指定文件的连接，how 指定打开方式。
	
	# DirectoryStream newDirectoryStream(Path path): 打开 path 指定的目录
	# InputStream newInputStream(Path path, OpenOption ...how): 获取 InputStream 对象
	# OutputStream newOutputStream(Path path, OpenOption ...how): 获取 OutputStream 对象
```

## 自动资源管理

```shell
# JDK7 增加了一个新特性，该特性提供了另外一种管理资源的方式，这种方式能自动关闭文件。
```

```java
/**
 * 自动资源管理基于 try 语句的扩展形式
 * 当 try 代码块结束时，自动释放资源，因此不需要显示的调用 close() 方法，
 * 注意:
 *		try 语句中声明的资源被隐式声明为 final，资源的作用局限于带资源的 try 语句
 *		可以在一条 try 语句中管理多个资源，每个资源以 ":" 隔开即可
 *		需要关闭的资源，必须实现了 AutoCloseable 接口或其子接口 Closeable
 */ 
try ("需要关闭的资源声明") {
    // 可能发生异常的语句
} catch ("异常类型 变量名") {
    // 异常的处理语句
}

......
    finally {
		// 一定执行的语句
    }
```



### 







