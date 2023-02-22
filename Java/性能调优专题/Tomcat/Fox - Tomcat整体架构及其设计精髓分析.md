[TOC]

# Fox - Tomcat整体架构及其设计精髓分析

## 1. Tomcat介绍

[官方文档](https://tomcat.apache.org/tomcat-9.0-doc/index.html)

### 1.1 Tomcat概念

- Apache 开发的 **一款开源的Java Servlet容器**
  - 是一种Web服务器，用于在服务器端运行 Java Servlet 和 JSP
  - 可以为Java Web应用程序提供运行环境，并通过HTTP协议处理客户端请求
  - 也支持多种Web应用程序开发技术，如 JSF、JPA 等
- 总的来说，**Tomcat是一款高效、稳定和易于使用的Web服务器**
  - **Tomcat核心：Http服务器 + Servlet容器**

![](https://note.youdao.com/yws/public/resource/c3ffeb7afce8c3a27fbcf68d9cf27a1b/xmlnote/WEBRESOURCEd3c18e9b3387ed80329cbd1d70568dc3/61938)

### 1.2 Tomcat目录结构

![](https://note.youdao.com/yws/public/resource/c3ffeb7afce8c3a27fbcf68d9cf27a1b/xmlnote/WEBRESOURCE5e5152ea03bdd459be53024cd7bbbd74/61939)

- bin：
  - 主要用来存放tomcat的脚本，如 **startup.sh、shutdown.sh**
- conf：
  - catelina.policy：Tomcat安全策略文件，控制JVM相关权限，具体可参考 java.security.Permission
  - catalina.properties：Tomcat Catalina行为控制配置文件，比如Common ClassLoader
  - loggin.properties：Tomcat日志配置文件，JDK Logging
  - **server.xml：Tomcat Server配置文件**
  - GlobalNamingResources：全局JNDI资源
  - context.xml：全局Context配置文件
  - tomcat-users.xml：Tomcat角色配置文件
  - web.xml：Servlet标准的web.xml部署文件，Tomcat默认实现部分配置入内：
    - org.apache.catalina.servlets.DefaultServlet
    - org.apache.jasper.servlet.JspServlet
- lib：
  - 公共类库
- logs：
  - tomcat在运行过程中产生的日志文件
- **webapps**：
  - 用来存放应用程序，当Tomcat启动时会去加载webapps目录下的应用程序
- work：
  - 用来存放tomcat在运行时的编译后文件，例如JSP编译后的文件

### 1.3 web应用部署的三种方式

1. 拷贝到webapps目录下

```xml
<!-- 指定appBase -->
<Host name="localhost"  appBase="webapps" unpackWARs="true" autoDeploy="true">
```

2. server.xml的Context标签下配置Context
   - path：指定访问该Web应用的URL入口（context-path）
   - docBase：指定Web应用的文件路径，可以给定绝对路径，也可以给定相当于<Host>的appBase属性的相对路径
   - reloadable：如果这个属性设为true，tomcat服务器在运行状态下会监视在WEB-INF/classes和WEB-INF/lib目录下class文件的改动，如果检测到有class文件被更新的，服务器会自动重新加载Web应用

```xml
<Context docBase="D:\mvc" path="/mvc"  reloadable="true" />
```

3. 在$CATALINA_BASE/conf/[enginename]/[hostname]/目录下（默认conf/Catalina/localhost）创建xml文件，文件名就是contextPath
   - 比如创建mvc.xml，path就是/mvc
   - 注意：想要根路径访问，文件名为ROOT.xml

```xml
<Context docBase="D:\mvc"  reloadable="true" />
```

## 2. Tomcat整体架构分析

- Tomcat要实现2个核心功能：
  - 处理 Socket 连接，负责网络字节流与 Request 和 Response 对象的转化
  - 加载和管理 Servlet，以及具体处理 Request 请求
- 因此：**Tomcat设计了两个核心组件：连接器（Connector，负责对外交流）和 容器（Container，负责内部处理）**

### 2.1 Tomcat架构图

- Tomcat的架构分为以下几个部分：
  1. Connector：连接器，用于接收请求并将其发送给容器
  2. Container：容器，负责管理Servlet、JSP和静态资源的生命周期
  3. Engine：引擎，管理容器的生命周期和分配请求
  4. Host：主机，可以管理多个Web应用程序
  5. Context：上下文，用于管理单个Web应用程序的配置信息
  6. Servlet：负责处理请求并生成响应
  7. JSP：用于动态生成Web内容

![](https://note.youdao.com/yws/public/resource/c3ffeb7afce8c3a27fbcf68d9cf27a1b/xmlnote/WEBRESOURCE0a061e9c9a2d9e1f94b08378d4037a44/61882)

### 2.2 Tomcat核心组件详解

#### Server 组件

- 指整个Tomcat服务器，包含多组服务（Serivce），负责和管理启动各个Service，同时监听 8005 端口发过来的 shutdown 命令，用于关闭整个容器

#### Service组件

- **每个Service组件都包含了若干用于接收客户端消息的 Connector 组件和处理请求的 Engine 请求**
- 还包含若干 Executor 组件，每个 Executor 都是一个线程池，可以为 Service 内所有组件提供线程池执行任务
- **通常在 Tomcat 中配置多个 Service，可以实现通过不同的端口号来访问同一台机器上部署的不同应用**

![](https://note.youdao.com/yws/public/resource/c3ffeb7afce8c3a27fbcf68d9cf27a1b/xmlnote/WEBRESOURCE792ee72916715e109b4094d81104114c/61884)

#### Connector组件

- Tomcat与外部的连接器，监听固定端口接收外部请求，传递给 Container，并将 Container 处理的结果返回给外部。
- **连接器对Servlet容器屏蔽了不同的应用层协议及 I/O  模型，无论是 HTTP 还是 AJP，在容器中获取到的都是一个标准的 ServletRequest 对象**

#### Container组件

- 用于装载 Servlet
- **Tomcat通过一种分层的架构，使得Servlet容器具有很好的灵活性**
- Tomcat设置了4种容器，这4种容器不是平行关系，而是父子关系
  - Engine：引擎，顶层容器，用来管理多个虚拟站点，**一个 Service 最多只能有一个 Engine** 
  - Host：虚拟主机，**负责Web应用的部署和Context的创建**，可以给Tomcat配置多个虚拟主机地址，而一个虚拟主机下可以部署多个Web应用程序
  - Context：**Web应用上下文**，包含多个Wrapper，负责web配置的解析、管理所有的Web资源。**一个Context对应一个Web应用程序**
  - Wrapper：表示一个Serlvet，最底层的容器，**是对Servlet的封装，负责Servlet实例的创建、执行和销毁**

![](https://note.youdao.com/yws/public/resource/c3ffeb7afce8c3a27fbcf68d9cf27a1b/xmlnote/WEBRESOURCEac63b4bd6b97c6bbd80479abdde9b25d/61888)

### 2.3 结合Server.xml理解Tomcat架构

```xml
<Server>    <!-- 顶层组件，可以包括多个Service -->
    <Service>  <!-- 顶层组件，可包含一个Engine，多个连接器 -->
        <Connector/> <!-- 连接器组件，代表通信接口 -->           
        <Engine> <!-- 容器组件，一个Engine组件处理Service中的所有请求，包含多个Host -->
            <Host>  <!-- 容器组件，处理特定的Host下客户请求，可包含多个Context -->
                <Context/>  <!-- 容器组件，为特定的Web应用处理所有的客户请求 -->
        </Host>
        </Engine>
    </Service>    
</Server>    
```

- **Tomcat启动期间会通过解析 server.xml，利用反射创建相应的组件，所以xml中的标签和源码一一对应**
- 思考：Tomcat是怎么确定请求是由哪个 Wrapper 容器里的 Servlet 来处理的呢？

### 2.4 请求定位 Servlet 的过程

- **Tomcat是用Mapper组件来完成定位的**
- Mapper组件功能：**将用户请求的URL定位到一个Serlvet**
- **一个请求URL最后只会定位到一个 Wrapper 容器，也就是一个Servlet**

![](https://note.youdao.com/yws/public/resource/c3ffeb7afce8c3a27fbcf68d9cf27a1b/xmlnote/WEBRESOURCE3da9cac12fe00f6336a17b3ca37e48e5/61968)

## 3. Tomcat架构设计精髓分析

### 3.1 Connector高内聚低耦合设计

- 优秀的模块化设计应考虑 高内聚、低耦合：
  - 高内聚：指相关度比较高的功能要尽可能集中，不要分散
  - 低耦合：指两个相关模块要尽可能减少依赖的部分和降低依赖的程度，不要让两个模块产生强依赖
- Tomcat连接器需要实现的功能：
  1. 监听网络端口
  2. 接受网络连接请求
  3. 读取请求网络字节流
  4. 根据具体应用层协议（HTTP/AJP）解析字节流，生成统一的 Tomcat Request 对象
  5. 将 Tomcat Request 对象转成标准的 ServletRequest
  6. 调用 Servlet 容器，得到 ServletResponse
  7. 将 ServletResponse 转成 Tomcat Response 对象
  8. 将 Tomcat Response 转成网络字节流
  9. 将响应字节流写回到浏览器
- 分析完连接器的详细功能列表，我们会发现连接器需要完成3个高内聚的功能：
  - **网络通信**
  - **应用层协议解析**
  - **Tomcat Request/Response 与 ServletRequest/ServletResponse 的转化**
- 因此，Tomcat的设计者设计了3个组件来实现这3个功能：
  - **EndPoint：负责提供字节流给 Processor**
  - **Processor：负责提供 Tomcat Request 对象给 Adapter**
  - **Adapter：负责提供 ServletRequest 对象给容器**
  
- 组件之间通过抽象接口交互。这样做的好处是封装变化。这是面向对象设计的精髓，将系统中经常变化的部分和稳定的部分隔离，有助于增加复用性，并降低系统耦合度。
- 由于 I/O 模型和应用层协议可以自由组合，比如 NIO + HTTP 或者 NIO2 + AJP，
  - Tomcat的设计者将网络通信和应用层协议解析放在一起考虑，设计了一个叫 **ProtocolHandler** 的接口来封装这两种变化点
  - 各种协议和通信模型的组合有对应的具体实现类，比如：Http11NioProtocol 和 AjpNioProtocol

- 除了这些变化点，系统也存在一些相对稳定的部分，因此 **Tomcat设计了一系列抽象基类来封装这些稳定的部分**
  - 抽象基类 AbstarctProtocol 实现了 ProtocolHandler 接口。
  - 每一种应用层协议有自己的抽象基类，比如 AbstractAjpProtocol 和 AbstractHttp11Protocol
  - 具体协议的实现类扩展了协议层抽象基类


![](https://note.youdao.com/yws/public/resource/c3ffeb7afce8c3a27fbcf68d9cf27a1b/xmlnote/WEBRESOURCE6169d3e2f561732bc1b4540796678a26/61983)

#### ProtocolHandler

- **连接器用 ProtocolHandler 来处理网络连接和应用层协议**，包含了两个重要部件：EndPoint 和 Processor

![](https://note.youdao.com/yws/public/resource/c3ffeb7afce8c3a27fbcf68d9cf27a1b/xmlnote/WEBRESOURCE5a9babd06b970b6d2ee5bf2c60293f88/61984)

- **EndPoint 负责底层 Socket 通信**
- **Processor 负责应用层协议解析**
- **连接器通过适配器 Adapter 调用容器**

##### EndPoint

- **EndPoint是通信端点，即通信监听的接口**
  - 是具体的Socket接收和发送处理器，**是对传输层的抽象**，因此 EndPoint 是用来实现 TCP/IP 协议的
- EndPoint是一个接口，对应的抽象实现类是 AbstractEndpoint，而 AbstractEndpoint 的具体子类，比如 NioEndPoint 和 Nio2Endpoint 中，有两个重要的子组件：
  - **Acceptor：** 监听Socket连接请求
  - **SocketProcessor：**处理接收到的Socket请求，实现Runnable接口，在Run方法里调用协议处理组件Proccesor进行处理。
    - 为了提高处理能力，SocketProcessor 被提交到线程池来执行，**而这个线程池叫执行器（Executor）**

##### Processor

- Processor 用来实现 HTTP/AJP 协议，接收来自 EndPoint 的 Socket，读取字节流解析成 Tomcat Request 和 Response 对象，并通过 Adapter 将其提交到容器处理，**Processor 是对应用层协议的抽象**
- Processor 是一个接口，定义了请求的处理等方法。它的抽象实现类 AbstractProcessor 对一些协议共有的属性进行封装。
  - 具体的实现有 AJPProcessor、HTTP11Processor 等，这些具体实现类实现了特定协议的解析方法和请求处理方式。

![](https://note.youdao.com/yws/public/resource/c3ffeb7afce8c3a27fbcf68d9cf27a1b/xmlnote/WEBRESOURCE825831f6312311ed679c7955e6e1dc16/61985)

- EndPoint 接收到 Socket 连接后，生成一个 SocketProcessor 任务提交到线程池去处理
  - SocketProcessor 的 Run 方法会调用 Processor 组件去解析应用层协议
  - Proccesor 通过解析生成 Request 对象后，会调用 Adapter 的 Service 方法

##### Adapter

- 由于协议不同，客户端发过来的请求信息也不尽相同，Tomcat 定义了自己的 Request 类来 "存放" 这些请求信息。
- ProtocolHandler 接口负责解析请求并生成 Tomcat Request 类，但是该 Request 不是标准的 ServletRequest，也就意味着，不能用 Tomcat Request 作为参数来调用容器
- **Tomcat 设计者的解决方案是引入 CoyoteAdapter，这是适配器模式的经典运用**
  - 连接器调用 CoyoteAdapter 的 Service 方法，传入的是 Tomcat Request 对象，
  - CoyoteAdapter 负责将 Tomcat Request 转成 ServletRequest，再调用容器的 Service 方法

###### 设计复杂系统的基本思路

- **首先分析需求，根据高内聚低耦合的原则确定子模块，然后找出子模块中的变化点和不变点，**
  - **用接口和抽象基类去封装不变点，在抽象基类中定义模板方法，让子类自行实现抽象方法，也就是具体子类去实现变化点**

### 3.2 父子容器组合模式设计

**思考：Tomcat设计了4种容器，分别是Engine、Host、Context和Wrapper，Tomcat是怎么管理这些容器的？**

- **Tomcat采用组合模式来管理这些容器**。
- 具体实现方法：
  - 所有容器组件都实现了 Container 接口，因此组合模式可以使得用户对单容器对象和组合容器对象的使用，具有一致性。
- Container 接口定义如下：

```java
public interface Container extends Lifecycle {
    public void setName(String name);
    public Container getParent();
    public void setParent(Container container);
    public void addChild(Container child);
    public void removeChild(Container child);
    public Container findChild(String name);
}
```

### 3.3 Pipeline-Valve 责任链模式设计

- 连接器中的 Adapter 会调用容器的 Service 方法来执行 Servlet，
  - 最先拿到请求的是 Engine 容器，Engine 容器会对请求做一些处理后，把请求传给自己子容器 Host 继续处理
  - 依次类推，最后这个请求会传给 Wrapper 容器，Wrapper 会调用最终的 Servlet 来处理
- 这个调用过程具体实现就是使用 **Pipeline-Valve 管道**
- **Pipeline-Valve 是责任链模式**，责任链模式是指在一个请求处理的过程中，有很多处理者依次对请求进行处理，每个处理者负责做自己相应的处理，处理完之后将再调用下一个处理者继续处理
- **为什么要使用管道机制？**
  - 在复杂的大型系统中，如果一个对象或数据流需要进行繁杂的逻辑处理，我们可以选择在一个大的组件中直接处理这些繁杂的业务逻辑，这个方式虽然达到目的，但扩展性和可重用性较差。
  - **更好的解决方式是采用管道机制，用一条管道把多个对象（阀门部件）连接起来，整体看起来就像若干个阀门嵌套在管道中一样，而处理逻辑放在阀门上**

#### Vavle接口设计

- Valve 表示一个处理点，比如权限认证和记录日志

```java
public interface Valve {
    public Valve getNext();
    public void setNext(Valve valve);
    public void invoke(Request request, Response response) throws IOException, ServletException;
}
```

#### Pipeline接口设计

- 由于Pipeline是为容器设计的，所以在设计时加入了一个Containerd接口，就是为了制定当前Pipeline所属的容器

```java
public interface Pipeline extends Contained {

    // 基础的处理阀
    public Valve getBasic();
    public void setBasic(Valve valve);


    // 对节点（阀门）增删查
    public void addValve(Valve valve);
    public Valve[] getValves();
    public void removeValve(Valve valve);


    // 获取第一个节点，遍历的起点，所以需要有这方法
    public Valve getFirst();


    // 是否所有节点（阀门）都支持处理Servlet3异步处理
    public boolean isAsyncSupported();


    // 找到所有不支持Servlet3异步处理的阀门
    public void findNonAsyncValves(Set<String> result);
}
```

- Pipeline中维护了Valve链表，Valve可以插入到Pipeline中，对请求做某些处理。
  - 整个调用链的触发是由Valve来完成的，Valve完成自己的处理后，调用 getNext.invoke() 来触发下一个Valve调用。
  - **每一个容器都有一个Pipeline对象**，只要触发这个Pipeline的第一个Valve，这个容器里Pipeline中的Valve就会被调用到。
  - Basic Valve 处于 Valve 链表的末端，它是Pipeline中必不可少的一个Valve，负责调用下层容器的Pipeline里的第一个Valve

![](https://note.youdao.com/yws/public/resource/c3ffeb7afce8c3a27fbcf68d9cf27a1b/xmlnote/WEBRESOURCEe53733aae0cfb1aec9595b7c7ddaecdd/62017)

- 整个调用过程由连接器中的 Adapter 触发，它会调用 Engine 的第一个 Valve

```java
//org.apache.catalina.connector.CoyoteAdapter#service
// Calling the container
connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);
```

- Wrapper 容器的最后一个 Valve 会创建一个 Filter 链，并调用 doFilter() 方法，最终会调到 Servlet 的 service  方法

```java
//org.apache.catalina.core.StandardWrapperValve#invoke
filterChain.doFilter(request.getRequest(), response.getResponse());
```

#### Valve 和 Filter 的区别

- Valve 是 Tomcat 的私有机制，与 Tomcat 的基础架构 / API  是紧耦合的。
- Servlet API 是公有的标准，所有的 Web 容器包括 Jetty 都支持 Filter 机制。
- Valve 工作在 Web 容器级别，拦截所有应用的请求；
- 而 Servlet Filter 工作在应用级别，只能拦截某个 Web 应用的所有请求。

#### 对比两种责任链模式

| 管道/阀门                                            | 过滤器链/过滤器                                  |
| ---------------------------------------------------- | ------------------------------------------------ |
| 管道（Pipeline）                                     | 过滤器链（FilterChain）                          |
| 阀门（Valve）                                        | 过滤器（Filter）                                 |
| 底层实现为具有头（first）、尾（basic）指针的单向链表 | 底层实现为数组                                   |
| Valve的核心方法invoke(request, response)             | Filter核心方法doFilter(request, response, chain) |
| pipeline.getFirst().invoke(request, response)        | filterchain.doFilter(request, response)          |

### 3.4 Tomcat生命周期设计

- 如果想让 Tomcat 能够对外提供服务，我们需要创建、组装并启动Tomcat组件；
  - 在服务停止的时候，我们还需要释放资源，销毁Tomcat组件，这是一个动态的过程。
- **Tomcat 需要动态地管理这些组件的生命周期**
- 实际工作中，对大型复杂项目设计中，也需要考虑这几个问题：
  - **如何统一管理组件的创建、初始化、启动、停止和销毁？**
  - **如何做到代码逻辑清晰？**
  - **如何方便地添加或者删除组件？**
  - **如何做到组件启动和停止不遗漏、不重复？**

#### 一键式启停：LifeCycle接口

- **系统设计就是要找到系统的变化点和不变点**。
  - 这里的不变点就是每个组件都要经历创建、初始化、启动这几个过程，这些状态以及状态的转化是不变的
  - 而变化点是每个具体组件的初始化方法，也就是启动方法是不一样的。
- **因此，把不变点抽象出来成为一个接口，这个接口跟生命周期有关，叫做LifeCycle**

```java
public interface Lifecycle {
    /** 第1类：针对监听器 **/
    // 添加监听器
    public void addLifecycleListener(LifecycleListener listener);
    // 获取所有监听器
    public LifecycleListener[] findLifecycleListeners();
    // 移除某个监听器
    public void removeLifecycleListener(LifecycleListener listener);
    
    /** 第2类：针对控制流程 **/
    // 初始化方法
    public void init() throws LifecycleException;
    // 启动方法
    public void start() throws LifecycleException;
    // 停止方法，和start对应
    public void stop() throws LifecycleException;
    // 销毁方法，和init对应
    public void destroy() throws LifecycleException;
    
    /** 第3类：针对状态 **/
    // 获取生命周期状态
    public LifecycleState getState();
    // 获取字符串类型的生命周期状态
    public String getStateName();
}
```

- 在父组件的 init() 方法里需要创建子组件并调用子组件的 init() 方法。
  - 同样，在父组件的 start() 方法里也需要调用子组件的 start() 方法
  - 因此，调用者可以无差别的调用各组件的 init() 方法和 start() 方法，这就是 **组合模式** 的使用
  - 并且只要调用最顶层组件，也就是 Server 组件的 init() 和 start() 方法，整个Tomcat就被启动起来了。

#### 可扩展性：LifeCycle 事件

- 因为各个组件 init() 和 start() 方法的具体实现是复杂多变的，比如在 Host 容器的启动方法里需要扫描 webapps 目录下的 Web 应用，创建相应的 Context 容器
  - 如果将来需要增加新的逻辑，直接修改 start() 方法？这会违反开闭原则（**开闭原则：为了扩展系统的功能，不能直接修改系统中已有的类，但是可以定义新的类**）
  - 那如何解决这个问题呢？
- 组件的 init() 和 start() 调用，是由它的父组件的状态变化触发的，上层组件的初始化会触发子组件的初始化，上层组件的启动会触发子组件的启动，
  - 因此我们把组件的生命周期定义成一个个状态，把状态的转变看做是一个事件。
  - 而事件是有监听器的，在监听器里可以实现一些逻辑，并且监听器也可以方便的添加和删除，**这就是典型的观察者模式**。
- 具体来说，就是在 LifeCycle 接口里加入两个方法：添加监听器和删除监听器

![](https://note.youdao.com/yws/public/resource/c3ffeb7afce8c3a27fbcf68d9cf27a1b/xmlnote/WEBRESOURCE251a1474b53ad39ef862cbb458ba9c2c/62027)

- 还需要定义一个 Enum 来表示组件有哪些状态，以及处在什么状态会触发什么样的事件

![](https://note.youdao.com/yws/public/resource/c3ffeb7afce8c3a27fbcf68d9cf27a1b/xmlnote/WEBRESOURCE447a2c06dc70edd0411a247203b6a30b/62028)

- 组件生命周期状态变化如下：

![](https://note.youdao.com/yws/public/resource/c3ffeb7afce8c3a27fbcf68d9cf27a1b/xmlnote/WEBRESOURCE16df85e321dd53ec5c8415052f58d118/62029)

#### 重用性：LifeCycleBase 抽象基类

- **Tomcat定义一个基类 LifeCycleBase 来实现 LifeCycle 接口，把一些公共的逻辑放到基类中去，比如生命状态的转变与维护、生命周期事件的触发以及监听器的添加和删除等**
  - **而子类就负责实现自己的初始化、启动和停止等方法**
- 为了避免与基类方法同名，可以把具体子类的实现方法改个名字，在后面加上 Internal ..

![](https://note.youdao.com/yws/public/resource/c3ffeb7afce8c3a27fbcf68d9cf27a1b/xmlnote/WEBRESOURCE656da9422343dd5a0c17bb7d94bfb845/62030)

- LifeCycleBase 实现了 LifeCycle 接口中所有的方法，还定义了相应的抽象方法交给具体子类去实现，这是典型的 **模板设计模式（骨架抽象类和模板方法）**
- LifeCycleBase 的 init() 方法实现：

![](https://note.youdao.com/yws/public/resource/c3ffeb7afce8c3a27fbcf68d9cf27a1b/xmlnote/WEBRESOURCE4f1279c4b6318c0a850f9d092ed8ef1b/62031)

- **思考：是什么时候、谁把监听器注册进来的呢？**
  - Tomcat 自定义了一些监听器，这些监听器是父组件在创建子组件的过程中注册到子组件的。
  - 比如：MemoryLeakTrackingListener 监听器，用来监听 Context 容器中的内存泄漏，这个监听器是 Host 容器在创建 Context 容器时注册到 Context 中的。
  - 我们还可以在 server.xml 中定义自己的监听器，Tomcat 在启动时会解析 server.xml，创建监听器并注册到容器组件。

#### 生命周期总体类图

![](https://note.youdao.com/yws/public/resource/c3ffeb7afce8c3a27fbcf68d9cf27a1b/xmlnote/WEBRESOURCE4b539793bfb50d841e37aad61c759b28/62032)
