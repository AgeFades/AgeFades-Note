[TOC]

# Fox - Tomcat类加载机制及其热加载和热部署原理剖析

## 1. Tomcat类加载机制详解

### 1.1 JVM类加载器

- Java中有3个类加载器，另外也可以自定义类加载器
  - 引导（启动类）加载器：负责加载支撑JVM运行的位于JRE的lib目录下的核心类库，比如rt.jar、charsets.jar 等
  - 扩展类加载器：负责加载支撑JVM运行的位于JRE的lib目录下的ext扩展目录中的jar类包
  - 应用程序（系统）类加载器：负责加载ClassPath路径下的类包，主要就是加载自己写的那些类
  - 自定义加载器：负载加载用户自定义路径下的类包

```java
public class ClassLoaderDemo {

    public static void main(String[] args) {
        // BootstrapClassLoader
        System.out.println(ReentrantLock.class.getClassLoader());
        // ExtClassLoader
        System.out.println(ZipInfo.class.getClassLoader());
        // AppClassLoader
        System.out.println(ClassLoaderDemo.class.getClassLoader());

        // AppClassLoader
        System.out.println(ClassLoader.getSystemClassLoader());
        // ExtClassLoader
        System.out.println(ClassLoader.getSystemClassLoader().getParent());
        // BootstrapClassLoader
        System.out.println(ClassLoader.getSystemClassLoader().getParent().getParent());

    }
}
```

### 1.2 双亲委派机制

- JVM类加载器是有亲子层级结构的，如下图：

![](https://note.youdao.com/yws/public/resource/6b83c4d0509e7754b628846cd9a279bc/xmlnote/WEBRESOURCE4a78c96f4ed05927518e63d6110d99a2/62175)

- 先找父加载器加载、不行再由子加载器加载

#### ClassLoader#loadClass源码分析

- AppClassLoader的loadClass方法 大体逻辑如下：
  1. 首先，检查一下指定名称的类是否已经加载过，如果加载过了，就不重复加载，直接返回
  2. 如果没有加载过，先判断是否有父类加载器，如有则由父加载器加载，或是调用bootstrap加载器来加载
  3. 如果父加载器以及bootstrap类加载器都没有找到指定类，则调用当前类加载器的findClass方法来完成类加载

```java
public abstract class ClassLoader {
                                                                                                                                                                                               
    // 每个类加载器都有个父加载器
    private final ClassLoader parent;

    public Class<?> loadClass(String name) {
        // 查找一下这个类是不是已经加载过了
        Class<?> c = findLoadedClass(name);
        // 如果没有加载过
        if( c == null ){
          // 先委托给父加载器去加载，注意这是个递归调用
          if (parent != null) {
              c = parent.loadClass(name);
          }else {
              // 如果父加载器为空，查找 Bootstrap 加载器是不是加载过了
              c = findBootstrapClassOrNull(name);
          }
        }

        // 如果父加载器没加载成功，调用自己的 findClass 去加载
        if (c == null) {
            c = findClass(name);
        }
        return c；
    }

    protected Class<?> findClass(String name){
       //1. 根据传入的类名 name，到在特定目录下去寻找类文件，把.class 文件读入内存
          ...
       //2. 调用 defineClass 将字节数组转成 Class 对象
       return defineClass(buf, off, len)；
    }

    // 将字节码数组解析成一个 Class 对象，用 native 方法实现
    protected final Class<?> defineClass(byte[] b, int off, int len){
       ...
    }

}
```

![](https://note.youdao.com/yws/public/resource/6b83c4d0509e7754b628846cd9a279bc/xmlnote/WEBRESOURCE170b24090bded17bac75e81bc9abab9b/62176)

- 思考：**为什么要设计双亲委派机制？**
  - **沙箱安全机制：**用户自己写的java.lang.String.class之类的类不会被加载，这样可以防止核心API库被随意篡改
  - 避免类的重复加载：当父类已经加载了该类时，子类就不用再加载一次，保证被加载类的唯一性

### 1.3 Tomcat 如何打破双亲委派机制

- Tomcat 的自定义类加载器 **WebAppClassLoader** 打破了双亲委派机制
- **它首先会尝试自己去加载某个类，如果找不到再交给父类加载器，目的是优先加载Web应用自己定义的类**
- 具体实现就是重写 ClassLoader 的两个方法，**findClass 和 loadClass**

#### findClass()

```java
public Class<?> findClass(String name) throws ClassNotFoundException {
    ...
    Class<?> clazz = null;
    try {
            //1. 先在 Web 应用目录下查找类 
            clazz = findClassInternal(name);
    }  catch (RuntimeException e) {
           throw e;
    }
    
    if (clazz == null) {
    try {
            //2. 如果在本地目录没有找到，交给父加载器去查找
            clazz = super.findClass(name);
    }  catch (RuntimeException e) {
           throw e;
    }
    
    //3. 如果父类也没找到，抛出 ClassNotFoundException
    if (clazz == null) {
        throw new ClassNotFoundException(name);
     }
    return clazz;
}

```

- 在 findClass() 里，主要有三个步骤：
  1. 先在Web应用本地目录下查找要加载的类
  2. 如果没找到，交给父加载器（AppClassLoader）查找
  3. 如果父加载器也没找到，抛出 ClassNotFound

#### loadClass()

```java
public Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        Class<?> clazz = null;
        //1. 先在本地 cache 查找该类是否已经加载过
        clazz = findLoadedClass0(name);
        if (clazz != null) {
            if (resolve)
                resolveClass(clazz);
            return clazz;
        }

        //2. 从系统类加载器的 cache 中查找是否加载过
        clazz = findLoadedClass(name);
        if (clazz != null) {
            if (resolve)
                resolveClass(clazz);
            return clazz;
        }
 
        // 3. 尝试用 ExtClassLoader 类加载器类加载，为什么？
        ClassLoader javaseLoader = getJavaseClassLoader();
        try {
            clazz = javaseLoader.loadClass(name);
            if (clazz != null) {
                if (resolve)
                    resolveClass(clazz);
                return clazz;
            }
        } catch (ClassNotFoundException e) {
            // Ignore
        }

        // 4. 尝试在本地目录搜索 class 并加载
        try {
            clazz = findClass(name);
            if (clazz != null) {
                if (resolve)
                    resolveClass(clazz);
                return clazz;
            }
        } catch (ClassNotFoundException e) {
            // Ignore
        }

        // 5. 尝试用系统类加载器 (也就是 AppClassLoader) 来加载
            try {
                clazz = Class.forName(name, false, parent);
                if (clazz != null) {
                    if (resolve)
                        resolveClass(clazz);
                    return clazz;
                }
            } catch (ClassNotFoundException e) {
                // Ignore
            }
       }
    
    //6. 上述过程都加载失败，抛出异常
    throw new ClassNotFoundException(name);
}
```

- loadClass() 稍微复杂点，有六个步骤：
  1. 先在本地 cache 查找该类是否已经加载过（被Tomcat的类加载器加载过）
  2. 如果没有，就去看看系统类加载器是否加载过
  3. 如果还没有，**就让ExtClassLoad去加载，这一步比较关键，目的是防止Web应用自己的类覆盖JRE的核心类**
     - 因为Tomcat 需要打破双亲委派机制，假如Web应用里自定义了一个叫 Object 的类，如果自定义类加载器先加载了这个 Object，就会覆盖 JRE 里的 Object
     - 而优先使用 ExtClassLoad 去加载，就会委托给 BootStrapClassLoader 加载，加载完后直接返回给 Tomcat 的类加载器，这样 Tomcat 类加载器就不会去加载 Web 应用下的 Object 类了
  4. 如果 ExtClassLoader 加载器加载失败，也就是说 JRE 核心类中没有该类，就可以在本地 Web 应用目录下查找并加载
  5. 如果本地目录下没有该类，说明不是Web自定义的类，就交由系统类加载器去加载
     - 注意：Web应用是通过Class.forName调用交给系统类加载器的，因为Class.forName的默认加载器就是系统类加载器
  6. 如果上述加载全部失败，抛出 ClassNotFound 异常
- 从上面的过程我们可以看到，Tomcat 的类加载器打破了双亲委派机制，没有一上来就直接委托给父加载器，而是先在本地目录下加载，为了避免本地目录下的类覆盖 JRE 的核心类，先尝试用 JVM 扩展类加载器 ExtClassLoader 去加载。那为什么不先用系统类加载器 AppClassLoader 去加载？很显然，如果是这样的话，那就变成双亲委派机制了，这就是 Tomcat 类加载器的巧妙之处。

### 1.4 Tomcat如何隔离Web应用

- Tomcat作为Servlet容器，它负责加载Servlet类、以及Servlet所依赖的 jar 包，
- 并且 Tomcat 本身也是一个Java程序，所以也需要加载自己的类和所依赖的 jar包，思考以下问题：
  1. 假如 Tomcat 运行了两个 Web，这俩之间有同名的 Servlet，但功能不同，Tomcat 需要同时加载和管理这俩类，保证它们不会冲突，因此 Web 之间的类需要隔离
  2. 假如俩 Web 都依赖同一个第三方的 jar，Tomcat 要保证这俩 Web 能共享这个第三方的 jar（只被加载一次），否则随着依赖的第三方 jar 增多，jvm内存会急速膨胀
  3. 跟 JVM 一样，需要隔离 Tomcat 本身的类和 Web 应用的类

#### Tomcat类加载器的层次结构

- 为了解决上述问题，Tomcat 设计了类加载器的层次结构，它们关系图如下：

![](https://note.youdao.com/yws/public/resource/6b83c4d0509e7754b628846cd9a279bc/xmlnote/WEBRESOURCE05272a0215a1dfcdb46dd16a3ccf45b6/62264)

- commonClassLoader：
  - Tomcat 最基本的类加载器，加载路径中的class可以被Tomcat容器本身以及各个Web app访问
- CatalinaClassLoader：
  - Tomcat 容器私有的类加载器，加载路径中的class对于 Web app 不可见
- SharedClassLoader：
  - 各个Web app共享的类加载器，加载路径中的class对所有web app可见，但对Tomcat容器不可见
- WebappClassLoader：
  - 各个Web app私有的类加载器，加载路径中的class只对当前Web app可见，比如加载 war 包里相关的类，每个 war 包应用都有自己的 WebappClassLoader，实现相互隔离
  - 比如不同war包引用了不同版本的Spring版本，这就可以实现加载各自的Spring版本

#### WebAppClassLoader

- 看上述的第1个问题，假如使用 JVM 默认 AppClassLoader 来加载 Web 应用，那就只能加载一个 Servlet 类，在加载第二个同名 Servlet 时，会直接返回第一个的实例
- Tomcat 的解决方案：
  - **自定义一个类加载器 WebAppClassLoader，并给每个 Web 应用创建一个类加载器实例**
  - Context 容器组件对应一个 Web 应用，因此，每个Context容器负责创建和维护一个 WebAppClassLoader 加载器实例。
  - **背后原理就是：不同的类加载器实例加载的类被认为是不同的类，即使它们类名相同**

#### SharedClassLoader

- 看上述的第2个问题，本质需求是两个 Web app 之间怎么共享类库，并且不能重复加载相同的类
- **Tomcat的设计者又加了一个类加载器 SharedClassLoader，作为 WebAppClassLoader 的父加载器，专门来加载 Web app 之间共享的类**

#### CatalinaClassLoader

- 看上述的第3个问题，如何隔离 Tomcat 本身的类和 Web app 的类？
  - 要共享可以通过 父子关系，要隔离就需要 兄弟关系
- **兄弟关系就是指两个类加载器是平行的，它们可能拥有同一个父加载器，但俩兄弟类加载器加载的类是隔离的。基于此，Tomcat又设计了一个类加载器 CatalinaClassLoader，专门来加载 Tomcat 自身的类**
  - 这设计会有个问题，Tomcat 和各 Web app 之间需要共享一些类时，怎么办？

#### CommonClassLoader

- 老办法，再增加一个 CommonClassLoader 作为 CatalinaClassLoader 和 SharedClassLoader 的父加载器
- **CommonClassLoader 能加载的类都可以被 CatalinaClassLoader 和 SharadClassLoader 使用，而 CatalinaClassLoader 和 SharedClassLoader 能加载的类则与对方互相隔离**
- WebAppClassLoader 可以使用 SharedClassLoader 加载到的类，但各个 WebAppClassLoader 之间相互隔离

#### Spring的加载问题

##### 全盘负责委托机制

- 指当一个ClassLoader 装载一个类时，除非显示的使用另外一个ClassLoader，**否则该类所依赖及引用的类，也由这个ClassLoader载入**
- 比如 Spring 作为一个 bean 工厂，它需要创建业务类的实例，并在创建业务类实例之前，需要加载这些类。
  - Spring 是通过调用 Class.forName 来加载业务类的

```java
public static Class<?> forName(String className) {
    Class<?> caller = Reflection.getCallerClass();
    return forName0(className, true, ClassLoader.getClassLoader(caller), caller);
}
```

- 在 forName() 中，会用调用者也就是 Spring 的加载器去加载业务类
- 前面说到，Web app 之间共享的 jar 可以交给 SharedClassLoader 加载，从而避免重复加载
  - Spring 作为共享的第三方 jar，本身是由 SharedClassLoader 加载的，而 Spring 本身又要去加载业务类，按照前面的规则，加载 Spring 的类加载器也会用来加载业务类
  - 但是业务类在 web app 目录下，不在 SharedClassLoader 的加载路径下，怎么办？

##### 线程上下文加载器

- **一种类加载器传递机制**，该类加载器被保存在线程私有数据里，只要是同一个线程，一旦设置了线程上下文加载器，在线程后续执行过程中，就能把该类加载器取出来用
- 因此，**Tomcat为每个Web app创建一个 WebAppClassLoader 类加载器，并在启动 Web app 的县城里设置线程上下文加载器，这样 Spring 在启动时，就将线程上下文加载器取出来，用于加载bean**
- Spring 取线程上下文加载器的代码如下：

```java
cl = Thread.currentThread().getContextClassLoader();
```

- 线程上下文加载器不仅仅可以用在 Tomcat 和 Spring 类加载的场景中，核心框架类需要加载具体实现类时，都可以用到它，比如 JDBC 就是通过上下文类加载器来加载不同的数据库驱动的

## 2. Tomcat 热加载和热部署

- 在项目开发中，经常改动 Java/JSP 文件，但又不想重启 Tomcat，有两种方式：热加载和热部署

  - **热加载**：表示重新加载 class，它的执行主体是 Context

    - 在 server.xml -> context 标签中设置 reloadable = "true"

    - ```xml
      <Context docBase="D:\mvc" path="/mvc"  reloadable="true" />
      ```

  - **热部署**：表示重新部署应用，它的执行主体是 Host

    - 在 server.xml -> Host 标签中设置 autoDeploy = "true"

    - ```xml
      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">
      ```

- 两者区别是：

  - 热加载的实现方式是 Web 容器启动一个后台线程，定期检测类文件的变化，如果有变化，就重新加载类
    - 在这个过程中 **不会清空Session**，一般用在开发环境
  - 热部署原理类似，也是由后台线程定时检测Web app的变化，但它会重新加载整个Web app
    - 这种方式会清空 Session，比热加载更干净、彻底，一般用在生产环境

- **思考：Tomcat 是如何用后台线程来实现热加载和热部署的？**

### 2.1 Tomcat开启后台线程执行周期性任务

- Tomcat 通过开启后台线程 ContainerBase.ContainerBackgroundProcessor，使各个层次的容器组件都有机会完成一些周期性任务
  - 在实际工作中，往往也需要执行一些周期性任务，比如：监控程序周期性拉取系统的健康状态，就可以借鉴这种设计
- Tomcat9 是通过 ScheduledThreadPollExecutor 来开启后台线程的，它除了具有线程池的功能，还能够执行周期性的任务。

![](https://note.youdao.com/yws/public/resource/6b83c4d0509e7754b628846cd9a279bc/xmlnote/WEBRESOURCE5900291c20ba94697bb64b48bab03fb4/62178)

- 此后台线程会调用当前容器的 backgroundProcess()，以及递归调用子孙容器的 backgroundProcess()
  - backgroundProcess() 会触发容器的周期性任务

![](https://note.youdao.com/yws/public/resource/6b83c4d0509e7754b628846cd9a279bc/xmlnote/WEBRESOURCE0d34ac1744af2ae61843dac4b3b9881b/62179)

- 有了 ContainerBase 中的后台线程和 backgroundProcess 方法，各种子容器和通用组件不需要再各自弄一个后台线程来处理周期性任务了。

### 2.2 Tomcat 热加载实现原理

- 有了 ContainerBase 的周期性任务处理"框架"，作为具体容器子类，只需要实现自己的周期性任务即可。
- 而 **Tomcat的热加载，就是在Context容器中实现的**
  - Context 容器的 backgroundProcess() 如下：

```java
//  StandardContext#backgroundProcess

//WebappLoader 周期性的检查 WEB-INF/classes 和 WEB-INF/lib 目录下的类文件
// 热加载
Loader loader = getLoader();
if (loader != null) {
    loader.backgroundProcess();        
}
```

- **WebappClassLoader 实现热加载的逻辑：主要是调用了 Context 容器的 reload()，先 stop Context容器，再 start Context容器**，具体实现如下：
  1. 停止和销毁 Context 容器及其所有子容器，子容器其实就是 Wrapper，也就是说 Wrapper 里的 Servlet 实例也被销毁了
  2. 停止和销毁 Context 容器关联的 Listener 和 Filter
  3. 停止和销毁 Context 下的 Pipeline 和各种 Valve
  4. 停止和销毁 Context 的类加载器，以及类加载器加载的类文件资源
  5. 启动 Context 容器，在这个过程中会重新创建前面四步被销毁的资源
- 在这个过程中，类加载器发挥着关键作用。
  - **一个Context容器对应一个类加载器**，类加载器在销毁的过程中，会把它加载的所有类也全部销毁
  - Context 容器在启动过程中，会创建一个新的类加载器来加载新的类文件

### 2.3 Tomcat 热部署实现原理

- 热部署跟热加载的本质区别是，**热部署会重新部署Web app**，原来的Context对象会整个被销毁，因此这个 Context 所关联的一切资源都会被销毁，包括 Session
- Host 容器并没有在 backgroundProcessor() 中实现周期性检测的任务，而是通过监听器 HostConfig 来实现的

```java
// HostConfig#lifecycleEvent
// 周期性任务
if (event.getType().equals(Lifecycle.PERIODIC_EVENT)) {
    check();
}
protected void check() {
    if (host.getAutoDeploy()) {
        // Check for resources modification to trigger redeployment
        DeployedApplication[] apps = deployed.values().toArray(new DeployedApplication[0]);
        for (DeployedApplication app : apps) {
            if (tryAddServiced(app.name)) {
                try {
                    // 检查 Web 应用目录是否有变化
                    checkResources(app, false);
                } finally {
                    removeServiced(app.name);
                }
            }
        }
        // Check for old versions of applications that can now be undeployed
        if (host.getUndeployOldVersions()) {
            checkUndeploy();
        }

        // Hotdeploy applications
        //热部署
        deployApps();
    }
```

- HostConfig 会检查 webapps 目录下的所有 Web app
  - 如果原来 Web app 目录被删掉了，就把整个相应 Context 容器销毁掉
  - 是否有新的 Wen app 目录进来了，或有新的 war 包进来了，就部署相应的 web app
- 因此 HostConfig 做的事是比较 "宏观" 的
  - **它不会检查具体类文件或资源文件是否有变化，而是检查 web app 目录级别的变化**