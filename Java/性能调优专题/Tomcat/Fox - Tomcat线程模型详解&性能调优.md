[TOC]

# Fox - Tomcat线程模型详解&性能调优

## 1. Tomcat I/O模型详解

### 1.1 Linux I/O模型详解

#### I/O 要解决什么问题

- I/O：**在计算机内存与外部设备之间拷贝数据的过程**
- 程序通过 CPU 向外部设备发出读指令，数据从外部设备拷贝至内存需要一段时间，这段时间CPU就是空闲的，程序有两种选择：
  - 让出CPU资源，让其干其他事情
  - 继续让CPU不停地查询数据是否拷贝完成
- 到底采取何种选择就是 I/O 模型需要解决的事情了。



- 以网络数据读取为例来分析，会涉及两个对象，一个是调用这个 I/O 操作的用户线程，另一个是操作系统内核。
  - 一个进程的地址空间分为用户空间和内核空间，基于安全上的考虑，**用户程序只能访问用户空间**，内核程序可以访问整个进程空间，只有内核可以直接访问各种硬件资源，比如磁盘和网卡。

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCE43df524dfb60fb3870a05b504710c5ae/63155)

- 当用户线程发起 I/O 调用后，网络数据读取操作会经历两个步骤：
  - **数据准备阶段：**用户线程等待内核将数据从网卡拷贝到内核空间
  - **数据拷贝阶段：**内核将数据从内核空间拷贝到用户空间（应用进程的缓冲区）

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCE93606ccc0377e367b75f70aa545a069d/63163)

- 不同的 I/O 模型对于这2个步骤有着不同的实现步骤

#### Linux的 I/O 模型分类

- Linux 系统下的 I/O 模型有5种：
  - 同步阻塞I/O（blocking I/O）
  - 同步非阻塞I/O（non-blocking I/O）
  - **I/O多路复用**（multiplexing I/O）
  - 信号驱动式I/O（signal-driven I/O）
  - 异步I/O（asynchronous I/O）
- 其中 信号驱动式I/O 在实际中并不常用

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCEda670373771a259cb84b2f1c78f32ff5/63170)

- 阻塞或非阻塞是指应用程序在发起 I/O 操作时，是立即返回还是等待
- 同步或异步是指应用程序在于内核通信时，数据从内核空间到应用空间的拷贝，是由内核主动发起还是由应用程序来触发。

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCE20a3a483f4d3e6950eb65b6914c5a6e4/63174)

### 1.2 Tomcat支持的I/O模型

| IO模型              | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| BIO （JIoEndpoint） | 同步阻塞式IO，即Tomcat使用传统的java.io进行操作。该模式下每个请求都会创建一个线程，对性能开销大，不适合高并发场景。优点是稳定，适合连接数目小且固定架构。Tomcat8.5.x开始移除BIO。 |
| NIO（NioEndpoint）  | 同步非阻塞式IO，jdk1.4 之后实现的新IO。该模式基于多路复用选择器监测连接状态再同步通知线程处理，从而达到非阻塞的目的。比传统BIO能更好的支持并发性能。Tomcat 8.0之后默认采用该模式。NIO方式适用于连接数目多且连接比较短（轻操作） 的架构， 比如聊天服务器， 弹幕系统， 服务器间通讯，编程比较复杂 |
| AIO (Nio2Endpoint)  | 异步非阻塞式IO，jdk1.7后之支持 。与nio不同在于不需要多路复用选择器，而是请求处理线程执行完成进行回调通知，继续执行后续操作。Tomcat 8之后支持。一般适用于连接数较多且连接时间较长的应用 |
| APR（AprEndpoint）  | 全称是 Apache Portable Runtime/Apache可移植运行库)，是Apache HTTP服务器的支持库。AprEndpoint 是通过 JNI 调用 APR 本地库而实现非阻塞 I/O 的。使用需要编译安装APR 库 |

- 注意：Linux 内核没有很完善地支持异步 I/O 模型，因此 JVM 没有采用原生的 Linux 异步I/O，
  - 而是在应用层面通过 epoll 模拟了异步 I/O 模型。
  - **因此在Linux平台上，Java NIO 和 Java NIO2  底层都是通过 epoll 来实现的**
  - Java NIO 更加简单高效

#### Tomcat I/O模型如何选型

- I/O 调优实际上是连接器类型的选择，**一般情况下默认都是NIO，在绝大部分情况下都是够用的**
  - 除非业务用到了 TLS 加密传输、而且对性能要求比较高，这时可以考虑 APR
  - 因为 APR 通过 OpenSSL 来处理 TLS 握手和加密/解密
  - OpenSSL 本身用C语言实现，对 TLS 通信做了优化，性能比 Java 要高
- 如果 Tomcat 跑在 Windows 上，可以考虑 NIO2，因为 **Windows从操作系统层面实现了真正意义上的异步 I/O** ，用 Http 传输的数据量比较大时，异步 I/O 的效果就能体现出来
- **如果 Tomcat 跑在 Linux 上，建议使用 NIO**
- 指定 I/O 模型只需修改 protocol 配置

```xml
<!-- 修改protocol属性, 使用NIO2 -->
<Connector port="8080" protocol="org.apache.coyote.http11.Http11Nio2Protocol"
           connectionTimeout="20000"
           redirectPort="8443" />
```

### 1.3 网络编程模型和Reactor线程模型

- Reactor模型是 **网络服务器端用来处理高并发网络 I/O 请求的一种编程模型**
- 该模型主要有三个关键角色来处理三类事件：
  - acceptor：负责连接事件
  - handler：负责读写事件
  - reactor：负责事件监听和事件分发

#### 单 Reactor 单线程

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCE6ca4ac87df30f83d1f7d25426865b8d2/63336)

#### 单 Reactor 多线程

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCEeee012a311a282f81decb350d719c27f/63348)

#### 主从 Reactor 多线程

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCE75fa23903351cb54590980fb913b99cc/63350)

### 1.4 Tomcat NIO实现

- 在 Tomcat 中，EndPoint 组件的主要工作就是处理 I/O，而 NioEndPoint 利用 Java NIO API 实现了多路复用 I/O 模型。
- **Tomcat的 NioEndPoint 是基于主从Reactor多线程模型设计的**

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCE59eecd198ba5072fe8e66fa3d6b4b888/62056)

- LimitLatch 是连接控制器，它负责控制最大连接数，NIO模式下默认是 10000（tomcat9中为8192）
  - 当连接数到达最大时阻塞线程，直到后续组件处理完一个连接后将连接数减1
  - 注意：**到达最大连接数后操作系统底层还是会接收客户端连接，但用户层已经不再接收**
- Acceptor 跑在一个单独的线程里，它在一个死循序里调用 accept 方法来接收新连接，一旦有新的连接请求到来，accept 方法返回一个 Channel 对象，接着把 Channel 对象交给 Poller 去处理

```java
#NioEndpoint#initServerSocket

serverSock = ServerSocketChannel.open();
//第2个参数表示操作系统的等待队列长度，默认100
//当应用层面的连接数到达最大值时，操作系统可以继续接收的最大连接数
serverSock.bind(addr, getAcceptCount());
//ServerSocketChannel 被设置成阻塞模式
serverSock.configureBlocking(true);
```

- ServerSocketChannel 通过 accept() 接受新的连接，accept() 方法返回获得 SocketChannel 对象，然后将 SocketChannel 对象封装在一个 PollerEvent 对象中，并将 PollerEvent 对象压入 Poller 的 SynchronizedQueue 里。
  - 这是个典型的 生产者-消费者 模式
  - Acceptor 与 Poller 线程之间通过 SynchronizedQueue 通信
  - Poller 的本质是一个 Selector，也跑在单独线程里。Poller 内部维护了一个 Channel 数组，它在一个死循环里不断检测 Channel 的数据就绪状态，一旦有 Channel 可读，就生成一个 SocketProcessor 任务对象扔给 Executor 去处理
  - ![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCE0924c56fe6481737ad0831a353496e3b/62057)
- Executor 就是线程池，负责运行 SocketProcessor 任务类，SocketProcessor 的 run 方法会调用 Http11Processor 来读取和解析请求数据
  - Http11Processor 是应用层协议的封装，它会调用容器获得响应，再把响应通过 Channel 写出

### 1.5 Tomcat 异步I/O实现

- NIO（同步） 和 NIO2（异步）
- 异步最大的特点：**应用程序不需要自己去触发数据从内核空间到用户空间的拷贝**

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCEa59fdb13f110cead7ad1636c7f7700ec/62058)

- Nio2Endpoint 中没有 Poller 组件，也就是没有 Selector
  - **在异步I/O模式下，Selector的工作交给内核来做了**

## 2. Tomcat 性能调优

- [Tomcat9参数配置](https://tomcat.apache.org/tomcat-9.0-doc/config/http.html)

### 2.1 如何监控 Tomcat 的性能

#### Tomcat 的关键指标

- 关键指标有 **吞吐量、响应时间、错误数、线程池、CPU以及JVM内存**
  - 前三个指标是最关心的业务指标，后三个是跟系统资源有关的
  - 当后三个某个资源出现瓶颈时，就会影响前面的业务指标
  - 比如：线程池中的线程数不足会影响吞吐量和响应时间，线程太多又会消耗大量CPU、影响吞吐量，内存不足时会频繁触发GC

#### 通过 JConsole 监控 Tomcat

- JConsole 是一款基于 JMX 的可视化监控和管理工具

##### 1. 开启 JMX 的远程监听端口

- 在 Tomcat 的 bin 目录下新建一个名为 setenv.sh（或者 setenv.bat，根据操作系统而定） 的文件

```shell
export JAVA_OPTS="${JAVA_OPTS} -Dcom.sun.management.jmxremote"
export JAVA_OPTS="${JAVA_OPTS} -Dcom.sun.management.jmxremote.port=8011"
export JAVA_OPTS="${JAVA_OPTS} -Djava.rmi.server.hostname=x.x.x.x"
export JAVA_OPTS="${JAVA_OPTS} -Dcom.sun.management.jmxremote.ssl=false"
export JAVA_OPTS="${JAVA_OPTS} -Dcom.sun.management.jmxremote.authenticate=false"
```

##### 2. 重启Tomcat

- 现在可以通过 JConsole 来连接这个端口

```shell
jconsole x.x.x.x:8011
```

- 主页面如下

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCE0a4d5dbf5e1d1d1905fe559bc36407a6/63253)

##### 吞吐量、响应时间、错误数

- 在 MBeans 标签页下选择 GlobalRequestProcessor，这里有 Tomcat 请求处理的统计信息
- 在看到的 Tomcat 各种连接器中，展开 **http-nio-8080**，就有该连接器上的统计信息，其中：
  - maxTime：最长响应时间
  - processingTime：平均响应时间
  - requestCount：吞吐量
  - errorCount：错误数

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCEe44a6f072de0fb15e8ff16404a85c834/63234)

##### 线程池

- 选择 **线程** 标签页，可以看到 Tomcat 进程中当前有多少线程

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCE8137bc0ebb07a97f32f0491a7e86fd2a/63244)

- 图左下方是线程列表，右边是线程的运行栈
  - 如果大量线程阻塞，通过观察线程栈，能看到线程阻塞在哪个函数，有可能是I/O等待，或者死锁

##### CPU

- 主界面可以找到CPU使用率指标，注意：这里的CPU使用率指的是Tomcat进程占用的CPU，不是主机总的CPU使用率

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCE2c3745bbb475a433a36078061136249b/63259)

##### JVM内存

- 选择 **内存** 标签页，可以查看 Tomcat 进程的 JVM 内存使用情况

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCEccc8053912ad5bf4d9b3017d3d4d308e/63251)

#### 命令行查看Tomcat指标

- 极端情况下，如果 Web 应用占用过多的 CPU 或者内存，又或者程序发生了死锁，导致 Web 应用对外没有响应，监控系统上看不到数据，这时候可以登陆到目标机器，通过命令行来查看各种指标



1. 首先通过 ps 命令找到 Tomcat 进程，拿到进程id

```shell
ps -ef | grep tomcat
```

2. 接着查看进程状态的大致信息

```shell
cat /proc/<pid>/status
```

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCE759f8543d56cf21692e73c0f57da5c8e/63264)

3. 监控进程的 CPU 和内存资源使用情况

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCE74835e3ba5b3dc40127ff91f2eb19851/63268)

4. 查看 Tomcat 的网络连接，比如 Tomcat 在8080端口上监听连接请求，通过下面的命令查看连接列表

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCEb32707027d5588d48fc22edd56f64422/63272)

- 还可以分别统计处在 **已连接** 状态和 **TIME_WAIT** 状态的连接数

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCEba45413107a55dd250462459fe78e1f4/63275)

5. 通过 ifstat 来查看网络流量，大致可以看出 Tomcat 当前的请求数和负载情况

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCE5c7550c5cdf0123ff1c09907d37670d4/63278)

### 2.2 线程池的并发调优

- 线程池调优指的是：**给Tomcat的线程池设置合适的参数，使得Tomcat能够更快更好地处理请求**

![](https://note.youdao.com/yws/public/resource/d404499987b0e4a7049aadc117569965/xmlnote/WEBRESOURCE5f5679e68d71576c34de8f38c0cb55f8/62061)

#### server.xml中配置线程池

```xml
<!--
namePrefix: 线程前缀
maxThreads: 最大线程数，默认设置 200，一般建议在 500 ~ 1000，根据硬件设施和业务来判断
minSpareThreads: 核心线程数，默认设置 25
prestartminSpareThreads: 在 Tomcat 初始化的时候就初始化核心线程
maxQueueSize: 最大的等待队列数，超过则拒绝请求 ，默认 Integer.MAX_VALUE
maxIdleTime: 线程空闲时间，超过该时间，线程会被销毁，单位毫秒
className: 线程实现类,默认org.apache.catalina.core.StandardThreadExecutor
-->
<Executor name="tomcatThreadPool" namePrefix="catalina-exec-Fox"
          prestartminSpareThreads="true"
          maxThreads="500" minSpareThreads="10"  maxIdleTime="10000"/>
          
<Connector port="8080" protocol="HTTP/1.1"  executor="tomcatThreadPool"
           connectionTimeout="20000"
           redirectPort="8443" URIEncoding="UTF-8"/>
```

- 这里最核心的就是 **如何确定 maxThreads 的值**
  - 如果设置小了，可能发生线程饥饿，大量请求处理在阻塞队列中等待，导致响应时间变长
  - 如果设置大了，可能因为CPU核数有限，线程数太多导致线程在 CPU 上来回切换，耗费大量的切换开销
- 理论上可以通过公式 **线程数 = CPU 核心数 *（1+平均等待时间/平均工作时间）**，计算出一个理想值
  - **实际场景中，需要基于该理想值的基础上进行压测，来获得最佳线程数**

## 3. SpringBoot中调整Tomcat参数

### 3.1 yml中配置

- 对应属性配置类：ServerProperties

```yaml
server:
  tomcat:
    threads:
      min-spare: 20
      max: 500
    connection-timeout: 5000ms
```

### 3.2 配置类配置

- SpringBoot 中的 TomcatConnectorCustomizer 类可用于对 Connector 进行定制化修改

```java
@Configuration
public class MyTomcatCustomizer implements
        WebServerFactoryCustomizer<TomcatServletWebServerFactory> {

    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        factory.setPort(8090);
        factory.setProtocol("org.apache.coyote.http11.Http11NioProtocol");
        factory.addConnectorCustomizers(connectorCustomizer());
    }

    @Bean
    public TomcatConnectorCustomizer connectorCustomizer(){
        return new TomcatConnectorCustomizer() {
            @Override
            public void customize(Connector connector) {
                Http11NioProtocol protocol = (Http11NioProtocol) connector.getProtocolHandler();
                protocol.setMaxThreads(500);
                protocol.setMinSpareThreads(20);
                protocol.setConnectionTimeout(5000);
            }
        };
    }

}
```

