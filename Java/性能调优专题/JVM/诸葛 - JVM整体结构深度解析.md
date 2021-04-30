[TOC]

# 诸葛 - JVM整体结构深度解析

## JDK 体系结构

![UTOOLS1595989418324.png](http://yanxuan.nosdn.127.net/cb048a48338bcdda5c294c5c55da0c44.png)

## Java 语言的跨平台特性

![UTOOLS1595989522687.png](http://yanxuan.nosdn.127.net/989369db8e7d23a7118de5c05fa4eeb2.png)

## JVM 整体结构及内存模型

![UTOOLS1595989550611.png](http://yanxuan.nosdn.127.net/2e0ad9936867951929d12a22c9edff84.png)

### JVM 内存参数设置

![aU9leU.png](https://s1.ax1x.com/2020/08/03/aU9leU.png)

#### SpringBoot 的 JVM 参数设置格式

```shell
# Tomcat 启动直接加在 bin 目录下 catalina.sh 文件里
java -Xms2048M -Xmx2048M -Xss512k -XX:MetaspaceSize=256M 
-XX:MaxMetaspaceSize=256M -jar app.jar
```

#### 关于元空间的JVM参数

```shell
-XX:MaxMetaspaceSize=N
	# 设置元空间最大值，默认是-1，即不限制，或者说只受限于本地内存大小
	
-XX:MetaspaceSize=N
	# 指定元空间触发 Full GC 的初始阈值（元空间无固定初始大小）
	
	# 以字节为单位，默认是 21M，达到该值就会触发 Full GC 进行类型卸载
	
	# 同时 GC 收集器将对该值进行调整
		# 如果释放了大量的空间，就适当降低该值
		
		# 如果释放了很少的空间，那么在不超过-XX:MaxMetaspaceSize（如果设置了的话）的情况下，
			# 适当提高该值。
			
# 由于调整元空间的大小需要 Full GC，这是非常消耗性能的操作
	# 如果应用在启动的时候发生大量 Full GC，通常都是由于元空间发生了动态扩缩容
	
	# 基于这种情况，一般建议将 MetaspaceSize 和 MaxMetaspaceSize 设置成一样的值
		# 对于 8G 物理内存的机器来说，一般会将这两个值设置为 256M
```

#### StackOverflowError

```java
// JVM设置: -Xss128k（默认 1024k = 1M）
public class StackOverflowTest {
  static int count = 0;
  
  static void redo() {
    count++;
    redo();
  }
  
  public static void main(String[] args) {
    try {
      redo();
    } catch (Throwable t) {
      t.printStackTrace();
      System.out.println(count);
    }
  }
}
```

![aUKGz8.png](https://s1.ax1x.com/2020/08/03/aUKGz8.png)

##### 结论

```shell
# -Xss 设置越小 count 值越小
	# 说明一个线程栈里能分配的栈帧就越小
	# 但是对 JVM 整体来说能开启的线程数会更多
```

#### JVM内存参数设置示例

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596433908351.png)

```shell
Java -Xms3072M -Xmx3072M -Xss1M -XX:MetaspaceSize=512M -XX:MaxMetaspaceSize=512M -jar app.jar
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596434227231.png)

```shell
# 优化后
java -Xms3072M -Xmx3072M -Xmn2048M -Xss1M -XX:MetaspaceSize=256M -jar app.jar
```

##### 结论

```shell
# 通过上面的示例得出:
	# 尽可能让对象都在新生代里分配和回收
	# 尽量别让太多对象频繁进入老年代、避免对老年代进行垃圾回收
	# 同时给系统充足的内存大小，避免新生代频繁的进行垃圾回收。
```

