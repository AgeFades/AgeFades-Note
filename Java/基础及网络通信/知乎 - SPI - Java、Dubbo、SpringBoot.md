[TOC]

[知乎 - 原文链接](https://zhuanlan.zhihu.com/p/401662642)

# Java SPI+Dubbo SPI+SpringBoot SPI 原理解析

## **一、SPI 是什么？**

SPI(Service Provider Interface)：是一个可以被第三方扩展或实现的 API，它可以用来实现框架扩展和可替换的模块，优势是实现解耦。简单来说就是推荐模块之间基于接口编程，模块之间不对实现类进行硬编码。若在代码里涉及具体的实现类就违反了可挺拔的原则。从而 java SPI 提供了这种服务发现机制：为某个接口寻找服务实现的机制。

## **二、SPI 与 API 的区别**

- API 直接为提供了功能，使用 API 就能完成任务。
- API 和 SPI 都是相对的概念，差别只在语义上，API 直接被应用开发人员使用，SPI 被框架扩张人员使用。
- API 大多数情况下，都是实现方来制定接口并完成对接口的不同实现，调用方仅仅依赖却无权选择不同实现。SPI 是调用方来制定接口，实现方来针对接口来实现不同的实现。调用方来选择自己需要的实现方。

## **三、SPI 使用及示例**

1. 服务调用方通过 ServiceLoader.load 加载服务接口的实现类实例
2. 服务提供方实现服务接口后， 在自己Jar包的 META-INF/services 目录下新建一个接口名全名的文件， 并将具体实现类全名写入。

**示例：**

**1. 创建接口**：

```java
//缓存接口
public interface Cache {
	// 设置缓存
	void set(String key , String value);
	//获取缓存
	String get(String key);
}
```

**2. 创建 MemeCache 实现类**：

```java
//MemeCache 提供的缓存实现
public class MemeCache implements Cache{
	@Override
	public void set(String key, String value) {
		System.out.println("[MemeCache缓存] set");
	}
	@Override
	public String get(String key) {
		System.out.println("[MemeCache缓存] get");
		return "";
	}
}
```

**3. 创建 RedisCache 实现类**：

```java
//redis 提供的缓存实现
public class RedisCache implements Cache{
	@Override
	public void set(String key, String value) {
		System.out.println("[redis缓存] set");
	}
	@Override
	public String get(String key) {
		System.out.println("[redis缓存] get");
		return "";
	}
}
```

**4. META-INF/services 中创建接口全限定名文件：**[collection](eclipse-javadoc:☂=test-java/src.Cache ：

```text
collection.MemeCache
collection.RedisCache
```

**5. 测试类**：

```text
public static void main(String args[]) throws Exception {
	ServiceLoader<Cache> s = ServiceLoader.load(Cache.class);
	Iterator<Cache> iterator = s.iterator();
	while (iterator.hasNext()) {
		Cache c = iterator.next();
		c.get("");
	}
}

-----输出：-----
[MemeCache缓存] get
[redis缓存] get
```



## 四、源码分析

首先load 构造一个ServiceLoader实例：

```text
ServiceLoader<Cache> s = ServiceLoader.load(Cache.class);
```

load源码：

```java
public static <S> ServiceLoader<S> load(Class<S> service) {
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
}
```

直接调用ServiceLoader的静态load方法：

```java
public static <S> ServiceLoader<S> load(Class<S> service, ClassLoader loader)
{
    return new ServiceLoader<>(service, loader);
}
```

load静态方法：

```java
public void reload() {
    providers.clear();
    lookupIterator = new LazyIterator(service, loader);
}

private ServiceLoader(Class<S> svc, ClassLoader cl) {
    service = Objects.requireNonNull(svc, "Service interface cannot be null");
    loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
    reload();
}
```

主要就是靠 LazyIterator 这个类，先看下LazyIterator 的主要成员变量：

```java
 private class LazyIterator implements Iterator<S>  
     Class<S> service;  // 我们传入的接口class对象
     ClassLoader loader;  // 系统的加载器
     String nextName = null; //保存迭代到当前META-INF/services/下的接口文件里的实现类
}
```

LazyIterator 实现了Iterator，所有具有迭代的功能，我们看下迭代的主逻辑：

```java
private S nextService() {
    // hasNextService： 加载 META-INF/services/service.getName() 将类的全限定名赋值给nextName变量
    if (!hasNextService())
	throw new NoSuchElementException();
    String cn = nextName;  // META-INF/services目录下的接口文件里面的实现类（一个一个迭代）
    nextName = null;
    Class<?> c = null;
    try {
	c = Class.forName(cn, false, loader); // 加载实现类
    } catch (ClassNotFoundException x) {
	fail(service,
	     "Provider " + cn + " not found");
    }
    if (!service.isAssignableFrom(c)) {
	fail(service,
	     "Provider " + cn  + " not a subtype");
    }
    try {
	S p = service.cast(c.newInstance()); // 实例化实现类
	providers.put(cn, p); 
	return p;
    } catch (Throwable x) {
	fail(service,
	     "Provider " + cn + " could not be instantiated",
	     x);
    }
    throw new Error();          // This cannot happen
}
```

hasNextService方法里面是加载META-INF/services/接口文件文件逻辑，并且判断是不是将接口文件里的实现类迭代完了， 不需要纠结里面的实现。

## 源码思想很简单，总结步骤：

- ServiceLoader.load并没有实例化接口实现类，只是将接口和类加载器赋值，后面交给LazyIterator 迭代的时候进行懒加载实例化。
- LazyIterator 迭代其实就是 一个一个迭代 META-INF/services/接口文件， 迭代找到里面的实现类全路径。
- 通过Class.forName 类全路径得到Class，并通过反射newInstance 构造出实例返回。

## Dubbo SPI 解析

我们先看dubbo是如何玩spi的：

maven引入dubbo:

```java
	<dependency>
		<groupId>org.apache.dubbo</groupId>
		<artifactId>dubbo</artifactId>
		<version>2.7.0</version>
	</dependency>
```

同样定义缓存接口

```java
@SPI  // 必须定义为spi 接口才能玩
public interface Cache {
	// 设置缓存
	void set(String key , String value);
	//获取缓存
	String get(String key);
}
// 同样定义两个实现：RedisCache 和 memcache实现 我就不写了
```

新建接口文件 META-INF/dubbo/com.hadluo.test.spi.Cache :

```java
# 提供了两个实现
memeCache = com.hadluo.test.spi.MemeCache
redisCache = com.hadluo.test.spi.RedisCache
```

memeCache 为别名 ， 对应实现类为com.hadluo.test.spi.MemeCache

客户端使用：

```java
	public static void main(String[] args) {
		// 获取 loader 工厂类
		ExtensionLoader<Cache> loader = ExtensionLoader.getExtensionLoader(Cache.class);
		// 根据别名 获取redisCache 实现
		Cache redisCache = loader.getExtension("redisCache");
		redisCache.get("");
	}

//----结果----
[redis缓存] get
```

程序调用根据别名就可以调用到对应的视线，可谓非常方便！ 下面讲下主要源码实现。

首先是获取ExtensionLoader 类， 这时候并没有加载META-INF 下的接口也没有实例化接口。但是保存了Cache.class到ExtensionLoader的成员变量type 。

```java
ExtensionLoader.getExtensionLoader(Cache.class);
```

重点讲 loader.getExtension 方法：

```java
    public T getExtension(String name) { // name为传入的别名
        // 巧妙利用了缓存 【缓存存Holder ，Holder里面存真正的接口实例】
        Holder<Object> holder = cachedInstances.get(name);
        if (holder == null) {
            //缓存没有 ,new Holder
            cachedInstances.putIfAbsent(name, new Holder<Object>());
            holder = cachedInstances.get(name);
        }
        Object instance = holder.get();
        if (instance == null) {
            synchronized (holder) {  //双重锁
                instance = holder.get();
                if (instance == null) {
                    // 去读取META-INF ，找对应name的接口实现类，进行实例化
                    instance = createExtension(name);
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }
```

createExtension 方法：

```java
  private T createExtension(String name) {
        // 找对应name别名的 class
        Class<?> clazz = getExtensionClasses().get(name);
        try {
            //实例化
            instance = clazz.newInstance();
            injectExtension(instance);
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                    type + ")  could not be instantiated: " + t.getMessage(), t);
        }
    }
```

getExtensionClasses 方法主要返回一个Map ， 主要代码片段：

```java
final SPI defaultAnnotation = type.getAnnotation(SPI.class);

// key： 别名  对应的实现类class
Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
// 读取对应文件，解析行 ， 利用Class.forName得到实现类class
loadDirectory(extensionClasses, "META-INF/dubbo/internal/", type.getName());
loadDirectory(extensionClasses, "META-INF/dubbo/internal/", type.getName().replace("org.apache", "com.alibaba"));
loadDirectory(extensionClasses, "META-INF/dubbo/", type.getName());
loadDirectory(extensionClasses, "META-INF/dubbo/", type.getName().replace("org.apache", "com.alibaba"));
loadDirectory(extensionClasses, "META-INF/services/", type.getName());
loadDirectory(extensionClasses, "META-INF/services/", type.getName().replace("org.apache", "com.alibaba"));
```

也就是说 除了META-INF/dubbo/ ， 还加载META-INF/dubbo/internal，META-INF/services/ 目录。

Map也就是下面信息：

![img](https://pic2.zhimg.com/80/v2-68ad06f0aad21b0f249e1ce1c9a54d95_1440w.jpg)

接着回到createExtension 方法：

```java
 private T createExtension(String name) {
        // 找对应name别名的 class
        Class<?> clazz = getExtensionClasses().get(name);
        try {
            //实例化
            instance = clazz.newInstance();
            //当有set方法时，且是其他的SPI接口 ，也通过getExtension实例化注入值（类似spring bean注入）
            injectExtension(instance);
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                    type + ")  could not be instantiated: " + t.getMessage(), t);
        }
    }
```

**injectExtension 的意思**： 当你的SPI接口有set另外一个SPI 接口时，也就是依赖另一个SPI ， 会初始化另一个SPI ，并且调用set方法注入到当前SPI接口中。（参考sping的注入）。

总结

- dubbo spi 巧妙利用很多缓存来缓存实例了的SPI 。
- dubbo spi实例化DPI后，还会考虑到是否依赖其它SPI接口，依赖的话就加载，并且通过set方法注入。
- dubbo spi会扫描 三个目录下的配置文件：META-INF/services/、META-INF/dubbo/、META-INF/dubbo/internal



## SpringBoot中的类SPI扩展机制

在springboot的自动装配过程中，最终会加载META-INF/spring.factories文件，而加载的过程是由SpringFactoriesLoader加载的。

从CLASSPATH下的每个Jar包中搜寻所有META-INF/spring.factories配置文件，然后将解析properties文件，找到指定名称的配置后返回。

原理在下面这篇文章有讲到（我就不重复造轮子了）：



[罗政：不指定扫描包 框架第三方Jar包的Bean如何注入Spring （微服务监控实例）2 赞同 · 0 评论文章](https://zhuanlan.zhihu.com/p/354463297)

中间有讲 SpringBoot自动配置原理 ，就是利用的SPI机制。

SPI就讲到这里，谢谢观看！

## 强力推荐一个Java架构师修炼博客，全是干货 

[JAVA架构师修炼githubs.xyz/](https://link.zhihu.com/?target=https%3A//githubs.xyz/)