[TOC]

# 诸葛 - JVM 类加载

## 类加载运行全过程

```shell
# 当使用 Java 命令运行某个类的 main 程序启动程序时，
	# 首先需要通过 类加载器 把主类加载到 JVM。
```

```java
public class Math {
  public static final int initData = 666;
  public static User user = new User();
  
  // 一个方法对应一块栈帧内存区域
  public int compute() {
    int a = 1;
    int b = 2;
    int c = (a + b) * 10;
    return c;
  }
  
  public static void main(String[] args) {
    Math math = new Math();
    math.compute();
  }
}
```

```shell
# 通过 Java 命令执行代码的大体流程如下:
```

![UTOOLS1592214558710.png](https://user-gold-cdn.xitu.io/2020/6/15/172b76219d457d07?w=1266&h=804&f=png&s=239322)

```shell
# 其中 loadClass() 的类加载过程有如下几步:
	# 加载
	# -> 验证
	# -> 准备
	# -> 解析
	# -> 初始化
	# -> 使用
	# -> 卸载
```

### 加载

```shell
# 在磁盘上查找并通过 IO 读入字节码文件（这里的 Math.class），
	# 只有使用到该类时才会加载，例如调用 类的main()方法、new对象等等...
	
	# 在加载阶段会在内存中生成一个代表这个累的额 java.lang.Class 对象
		# 作为 方法区 这个类的各种数据的访问入口。
```

### 验证

```shell
# 校验字节码文件的正确性。

# 字节码文件以 cafe 开头。

# 如果篡改了这个class 文件，就不能被 JVM 加载。
```

![UTOOLS1592616681230.png](http://yanxuan.nosdn.127.net/6d7f890456dff1318ae99d592b5e1601.png)



### 准备

```shell
# 给类的静态变量分配内存，并赋予默认值。

# 在加载时，会将静态变量先赋值一个该类型的默认值。
	# 比如 boolean 就是 false
	# 比如 Integer 就是 0
	# 对象就是 null
	
# final 关键字修饰的常量会直接赋值。
```

### 解析

```shell
# 将 符号引用 替换为 直接引用，
	# 该阶段会把一些静态方法（符号引用，比如 main()方法）
		# 替换为指向数据所存内存的指针或句柄等（直接引用），
		# 这就是所谓的 静态链接 过程（类加载期间完成）。
		
	# 动态链接 是在程序运行期间完成的将符号引用替换为直接引用。
	
# 可以用 javap -v (字节码文件) 生成可读的 class 文件

# 类加载的过程是属于 静态链接，将我们写的 程序字面量 链接到数据所存内存的指针或句柄等。
```

### 初始化

```shell
# 对类的静态变量初始化为指定的值，执行静态代码块。

# 前面 准备 的过程，将静态变量赋值为类型默认值，
	# 这里就是将 静态变量 赋予实际值。
```

![UTOOLS1592216308102.png](https://user-gold-cdn.xitu.io/2020/6/15/172b77ccbd699234?w=1164&h=588&f=png&s=95006)

```shell
# 类被加载到方法区中主要包含:
	# 运行时常量池
	# 类型信息
	# 字段信息
	# 方法信息
	# 类加载的引用
	# 对应 class 实例的引用
	# ...
```

```shell
# 类加载器的引用:
	# 这个类到类加载器实例的引用
```

```shell
# 对应 class 实例的引用:
	# 类加载器在加载类信息放到方法区中后，
		# 会创建一个对应的 Class 类型的对象实例放到堆(Heap) 中，
		# 作为开发人员访问方法区中类定义的入口和切入点。
```

```shell
# 注意:
	# 主类在运行过程中如果使用到其他类，会逐步加载这些类。
	# jar 或 war 里的类不是一次性全部加载的，是使用到时才加载。
	# JVM 是属于懒加载。
	# 加载类时，会先加载类的静态代码块，再加载类的构造方法。
```

```java
public class TestDynamicLoad {
  
  static {
    System.out.println("load TestDynamicLoad");
  }
  
  public static void mian (String[] args) {
    new A();
    System.out.println("load test");
    B b = null;	// B 不会加载，除非这里执行 new B()
  }
  
  class A {
    static {
      System.out.println("load A");
    }
    
    public A() {
      System.out.println("initial A");
    }
  }
  
  class B {
    static {
      System.out.println("load B");
    }
    
    public B() {
      System.out.println("initial B");
    }
  }
  
}
```

```shell
# 运行结果
load TestDynamicLoad
load A
initial A
load test
```

## 类加载器和双亲委派机制

```shell
# 上面的类加载过程主要是通过类加载器来实现的，Java 有如下几种类加载器:

# 引导类加载器:
	# 负责加载支撑 JVM 运行的位于 JRE 的 lib 目录下的核心类库，
	# 比如 rt.jar、charsets.jar 等
	
# 扩展类加载器:
	# 负责加载支撑 JVM 运行的位于 JRE 的 lib 目录下的 ext 扩展目录中的 jar 类包。
	
# 应用程序类加载器:
	# 负责加载 ClassPath 路径下的类包，
	# 主要就是加载我们自己写的那些类。
	
# 自定义加载器:
	# 负责加载用户自定义路径下的类包。
```

### 类加载器实例

```java
public class TestJDKClassLoader {
  
  public static void main(String[] args) {
        System.out.println(String.class.getClassLoader());
        System.out.println(DESKeyFactory.class.getClassLoader().getClass().getName());
        System.out.println(TestJDKClassLoader.class.getClassLoader().getClass().getName());

        System.out.println();

        ClassLoader appClassLoader = ClassLoader.getSystemClassLoader();
        ClassLoader extClassLoader = appClassLoader.getParent();
        ClassLoader bootstrapLoader = extClassLoader.getParent();
        System.out.println("the bootstrapLoader: " + bootstrapLoader);
        System.out.println("the extClassLoader: " + extClassLoader);
        System.out.println("the appClassLoader: " + appClassLoader);

        System.out.println();
        System.out.println("bootstrapLoader 加载以下文件: ");
        URL[] urls = Launcher.getBootstrapClassPath().getURLs();
        Arrays.stream(urls).forEach(System.out::println);

        System.out.println();
        System.out.println("extClassloader 加载以下文件: ");
        System.out.println(System.getProperty("java.ext.dirs"));

        System.out.println();
        System.out.println("appClassLoader加载以下文件: ");
        System.out.println(System.getProperty("java.class.path"));
    }
  
}
```

![UTOOLS1592618597871.png](http://yanxuan.nosdn.127.net/48c5ffbc31a5b54cfda19729f2d15f0e.png)

### 类加载器初始化过程

```shell
# 参见类运行加载全过程图，可知其中会创建 JVM 启动器实例 sum.misc.Launcher。

# Launcher 初始化使用了单例设计模式，保证一个 JVM 虚拟机内只有一个 Launcher 实例。

# 在 Launcher 构造方法内部，其创建了两个类加载器，分别是:
	# ExtClassLoader
	
	# AppClassLoader
	
# JVM 默认使用 Launcher 的 getClassLoader() 方法返回 AppClassLoader 实例加载我们的应用程序。
```

```java
public Launcher() {
        Launcher.ExtClassLoader var1;
        try {
            var1 = Launcher.ExtClassLoader.getExtClassLoader();
        } catch (IOException var10) {
            throw new InternalError("Could not create extension class loader", var10);
        }

        try {
            this.loader = Launcher.AppClassLoader.getAppClassLoader(var1);
        } catch (IOException var9) {
            throw new InternalError("Could not create application class loader", var9);
        }

        Thread.currentThread().setContextClassLoader(this.loader);
        String var2 = System.getProperty("java.security.manager");
        if (var2 != null) {
            SecurityManager var3 = null;
            if (!"".equals(var2) && !"default".equals(var2)) {
                try {
                    var3 = (SecurityManager)this.loader.loadClass(var2).newInstance();
                } catch (IllegalAccessException var5) {
                } catch (InstantiationException var6) {
                } catch (ClassNotFoundException var7) {
                } catch (ClassCastException var8) {
                }
            } else {
                var3 = new SecurityManager();
            }

            if (var3 == null) {
                throw new InternalError("Could not create SecurityManager: " + var2);
            }

            System.setSecurityManager(var3);
        }

    }
```

![UTOOLS1592618862201.png](http://yanxuan.nosdn.127.net/b91c855e02fd48b2eac138b8367146b8.png)

![UTOOLS1592618949793.png](http://yanxuan.nosdn.127.net/12fc9690155a5d59ef169d0e5b0e54fb.png)

### 双亲委派机制

```shell
# JVM 类加载器是有亲子层级结构的，如下图:

# 为什么类加载是从 应用类加载器 往上委托加载，
	# 而不是从 引导类加载器 往下加载？
	
# 因为在我们写的 Web 程序中，95% 以上的代码是由 应用类加载器 加载，
	# 所以系统基础类加载只需要第一次向上委托加载，
	# 之后的类加载基本就在应用类加载器完成。
```

![UTOOLS1592383981764.png](https://user-gold-cdn.xitu.io/2020/6/17/172c17b49f664796?w=664&h=732&f=png&s=86618)

```shell
# 简单来说，就是先找父类加载器加载，所有父类加载器都没加载的话，最后才由自己加载。
```

```shell
# AppClassLoader 的 loadClass 方法最终会调用其父类 ClassLoader 的 loadClass 方法。

# 方法的大体逻辑如下:
	# 1. 首先检查指定名称的类是否已经加载过，如果加载过直接返回。
	
	# 2. 如果没有加载过，判断是否有父类加载器，如果有，则由父加载器加载，
		# 即调用 parent.loadClass(name, false);
		# 或者调用 bootstrap 类加载器来加载。
		
	# 3. 如果父加载器及 bootstrap 类加载器都没有找到指定的类，
		# 那么调用当前类加载器的 findClass 方法来完成类加载
```

```java
// ClassLoader 的 loadClass 方法，里面实现了双亲委派机制
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 首先检查当前类加载器是否已经加载了该类
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                      	// 如果父加载器不为空则委托父加载器加载该类
                        c = parent.loadClass(name, false);
                    } else {
                      	// 如果父加载器为空则委托启动类加载器加载该类
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                  	// 都会调用 URLClassLoader 的 findClass 方法在加载器的类路径里查找并加载该类
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {	// 不会执行
                resolveClass(c);
            }
            return c;
        }
    }
```

#### 为什么要设计双亲委派机制

```shell
# 沙箱安全机制:
	# 自己写的 java.lang.String.class 类不会被加载，
	# 这样便可以防止核心 API 库被随意纂改。
	
# 避免类的重复加载:
	# 当父加载器已经加载了该类时，就没有必要子加载器再加载一次，
	# 保证被加载类的唯一性。
```

#### 类加载示例

```java
package java.lang;

public class String {
    public static void main(String[] args) {
        System.out.println("My String Class");
    }
}
```

![UTOOLS1592385162926.png](https://user-gold-cdn.xitu.io/2020/6/17/172c18d5934fe3c0?w=1612&h=270&f=png&s=65199)

#### 全盘负责委托机制

```shell
# 全盘负责:
	# 指当一个 ClassLoader 装载一个类时，除非该类中依赖的类显示的使用另外一个 ClassLoader，
	# 否则该类所依赖及引用的类也都由这个 ClassLoader 载入。
```

#### 自定义类加载器示例

```shell
# 自定义类加载器只需要继承 java.lang.ClassLoader 类。

# 该类有两个核心方法:
	# loadClass(String boolan) : 实现双亲委派机制
	
	# findClass(): 加载类的方法，默认空实现，所以自定义类加载器主要是重写 findClass 方法
```

```java
public class MyClassLoaderTest {
  
  static class MyClassLoader extends ClassLoader {
    private String classPath;
    
    public MyClassLoader(String classPath) {
      this.classPath = classPath;
    }
    
    private byte[] loadByte(String name) throws Exception {
      name = name.replaceAll("\\.", "/");
      FileInputStream fis = new FileInputStream(classPath + "/" + name + ".class");
      int len = fis.available();
      byte[] data = new byte[len];
      fis.read(data);
      fis.close();
      return data;
    }
    
    protected Class<?> findClass(String name) throws ClassNotFoundException {
      try {
        byte[] data = loadByte(name);
        // defineClass 将一个字节数组转为 Class 对象，
        // 这个字节数组是 class 文件读取后最终的字节数组
        return defineClass(name, data, 0, data.length);
      } catch (Exception e) {
        e.printStackTrace();
        throw new ClassNotFoundException();
      }
    }
  }
  
  public static void main(String args[]) throws Exception {
    // 初始化自定义类加载器，会初始化父类 ClassLoader
    // 其中会把自定义类加载器的父类加载器，设置为应用程序类加载器 AppClassLoader
    MyClassLoader classLoader = new MyClassLoader("D:/test");
    // D 盘创建 test/com/swordsman/jvm 几级目录，将 User 类的复制类 User1 丢入该目录
    Class clazz = classLoader.loadClass("com.tuling.jvm.User1");
    Object obj = clazz.newInstance();
    Method method = clazz.getDeclaredMethod("sout", null);
    method.invoke(obj, null);
    
    // 这里因为双亲委派机制的原因，
    	// 如果 User1 也在 classpath 下，类加载器就是 AppClassLoader
    	// 只在 D:/test 下就是自定义加载器
    System.out.println(clazz.getClassLoader().getClass().getName());
  }
  
}
```

#### 打破双亲委派机制

```shell
# 其实就是打破 类加载器 向 父加载器委托加载的请求，而是直接自己加载类。

# 其实就是重写 loadClass() 方法

# 在原有的逻辑上加一层判断,通过类名控制是否需要双亲委派加载.
```

### Tomcat 打破双亲委派机制

```shell
# Tomcat 作为 Web 容器，一个 Tomcat 可能部署两个应用程序，
	# 不同的应用程序可能会 依赖同一个第三方类库的不同版本，
	# 所以如果使用 JDK 默认双亲委派机制，是会产生冲突的（比如A依赖Spring4，B依赖Spring5）
	# 所以 Tomcat 需要保证每个应用程序的类库都是独立的，保持相互隔离。
	
# 部署在同一个 Tomcat 容器中相同的类库相同的版本可以共享，
	# 否则，如果服务器有 10 个 app，就会有 10 份相同的类库加载进虚拟机。
	
# Tomcat 也有自己依赖的类库，不能与应用程序的类库混淆，
	# 基于安全考虑，应该让容器的类库和程序的类库隔离开来。
	
# Tomcat 还支持 jsp 编译成 class 文件运行后，还能修改编译达到热部署。
	# jsp 其实就是 class 文件，就算修改了，类名还是一样，类加载会直接取已经存在的，
	# 所以 Tomcat 是直接卸载掉这个 jsp 的类加载器，重新创建新的类加载器，加载新的 jsp 文件。
```

![UTOOLS1592643162974.png](http://yanxuan.nosdn.127.net/a0ec0847be5a64f260a0bcf59a590a7e.png)

```shell
# Tomcat 的几个主要类加载器: 

# commonLoader:
	# Tomcat最基本的类加载器，加载路径中的class可以被Tomcat容器本身以及各个Webapp服务
	
# catalinaLoader:
	# Tomcat容器私有的类加载器，加载路径中的class对于Webapp不可见。
  
# sharedLoader:
	# 各个Webapp共享的类加载器，加载路径中的class对于所有Webapp可见，
	# 但是对于Tomcat容器不可见。
  
# WebappClassLoader:
	# 各个Webapp私有的类加载器，加载路径中的class只对当前Webapp可见，
	# 比如加载war包里相关的类，每个war包应用都有自己的WebappClassLoader，
	# 实现相互隔离，比如不同war包应用引入了不同的spring版本， 
	# 这样实现就能加载各自的spring版本。
```

```shell
# 从图中的委派关系中可以看出: 
	# CommonClassLoader能加载的类，
	# 都可以被CatalinaClassLoader和SharedClassLoader使用，
  #	从而实现了公有类库的共用。
  
  # 而CatalinaClassLoader和SharedClassLoader自己能加载的类则 与对方相互隔离。 
  
  # WebAppClassLoader可以使用SharedClassLoader加载到的类，
  	# 但各个WebAppClassLoader 实例之间相互隔离。 
  	
  # 而JasperLoader的加载范围仅仅是这个JSP文件所编译出来的那一个.Class文件，
  # 它出现的目的 就是为了被丢弃:
  # 当Web容器检测到JSP文件被修改时，会替换掉目前的JasperLoader的实例， 
  # 并通过再建立一个新的Jsp类加载器来实现JSP文件的热加载功能。
```

```shell
# tomcat 为了实现隔离性，没有遵守双亲委派机制，

#	每个 webappClassLoader加载自己的目录下的class文件，
	# 不会传递给父类加载器，打破了双亲委派机制。
```

### 模拟实现打破双亲委派机制

```java
public class MyClassLoaderTest {
  
  static class MyClassLoader extends ClassLoader {
    private String classPath;
    
    public MyClassLoader(String classPath) {
      this.classPath = classPath;
    }
    
    private byte[] loadByte(String name) throws Exception {
      name = name.replaceAll("\\.", "/");
      FileInputStream fis = new FileInputStream(classPath + "/" + name + ".class");
      int len = fis.available();
      byte[] data = new byte[len];
      fis.read(data);
      fis.close();
      return data;
    }
    
    protected Class<?> findClass(String name) throws ClassNotFoundException {
      try {
        byte[] data = loadByte(name);
        // defineClass 将一个字节数组转为 Class 对象，
        // 这个字节数组是 class 文件读取后最终的字节数组
        return defineClass(name, data, 0, data.length);
      } catch (Exception e) {
        e.printStackTrace();
        throw new ClassNotFoundException();
      }
    }
    
    protected Class<?> loadClass(String name, boolean resolve) 
      throws ClassNotFoundException {
      
      synchronized (getClassLoadingLock(name)) {
        Class<?> c= findLoadClass(name);
      }
      
      if (c == null) {
        long t1 = System.nanoTime();
        
        // 非自定义的类继续走双亲委派机制
        if(!name.startsWith("com.swordsman.test")) {
          c = this.getParent().loadClass(name);
        } else {
          c = findClass(name)；
        }
        
        sun.misc.PrefCounter.getFindClassTime().addElaspedTimeFrom(t1);
        sun.misc.PrefCounter.getFindClasses().increment();
      }
      if (resolve) {
        resolveClass(c);
      }
      return c;
        
      }   
    }
  }
  
  public static void main(String args[]) throws Exception {
    // 初始化自定义类加载器，会初始化父类 ClassLoader
    // 其中会把自定义类加载器的父类加载器，设置为应用程序类加载器 AppClassLoader
    MyClassLoader classLoader = new MyClassLoader("D:/test");
    // D 盘创建 test/com/swordsman/jvm 几级目录，将 User 类的复制类 User1 丢入该目录
    Class clazz = classLoader.loadClass("com.tuling.jvm.User1");
    Object obj = clazz.newInstance();
    Method method = clazz.getDeclaredMethod("sout", null);
    method.invoke(obj, null);
    
    // 这里因为双亲委派机制的原因，
    	// 如果 User1 也在 classpath 下，类加载器就是 AppClassLoader
    	// 只在 D:/test 下就是自定义加载器
    System.out.println(clazz.getClassLoader().getClass().getName());
  }
  
}
```

