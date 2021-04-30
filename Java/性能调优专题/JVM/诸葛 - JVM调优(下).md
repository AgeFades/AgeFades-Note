[TOC]

# 诸葛 - JVM调优(下)

## Arthas

### 简介

```shell
# 阿里巴巴 2018年9月 开源的 Java 诊断工具，支持 JDK1.6+，
	# 采用命令行交互模式，方便定位和诊断线上程序运行问题。
```

### 官网

[Arthas](https://alibaba.github.io/arthas)

### 使用

```shell
# 有兴趣的看着官方文档用就行。
```

## GC 日志

### Parllel

```shell
# 加上 JVM 启动参数，输出 GC 日志文件
java -jar -Xloggc:./gc-%t.log -XX:+PrintGCDetails +XX:+PrintGCTimeStamps +XX:+PrintGCCause -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 
-XX:GCLogFileSize=100M xxx.jar
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1597820670868.png)

### CMS

```shell
-Xloggc:./gc-cms-%t.log -Xms50M -Xmx50M -XX:MetaspaceSize=256M 
-XX:MaxMetaspaceSize=256M -XX:+PrintGCDetails -XX:+PrintGCDateStamps 
-XX:+PrintGCTimeStamps -XX:+PrintGCCause -XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M -XX:+UseParNewGC
-XX:+UseConcMarkSweepGC
```

### G1

```shell
-Xloggc:./gc-g1-%t.log -Xms50M -Xmx50M -XX:MetaspaceSize=256M 
-XX:MaxMetaspaceSize=256M -XX:+PrintGCDetails -XX:+PrintGCDateStamps 
-XX:+PrintGCTimeStamps -XX:+PrintGCCause -XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M -XX:+UseG1GC
```

### 日志分析

[GCEasy](https://gceasy.io)

## JVM参数汇总查看命令

```shell
java -XX:+PrintFlagInitial # 打印所有参数选项的默认值

java -XX:+PrintFlagsFinal # 打印出所有参数选项在运行程序时生效的值
```

## 常量池

### 静态常量池

```shell
# javac 编译后 java 文件得到 class 文件（16进制文件）
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1597821391441.png)

```shell
# 这些字节里，cafe babe 是模数，然后 主版本、次版本、常量池计数器

# 然后就是这个 class 文件的静态常量池,
	# 存放编译器生成的各种 字面量 和 符号引用。 
```

```shell
# 可以通过 javap 解读成可读的格式
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1597821518902.png)

#### 字面量

```shell
# 字面量: 由 字母、数字 等构成的字符串或者数值常量。
int a = 1;
int b = 2;
String c = "abc";
String d = "efg";
```

#### 符号引用

```shell
# 编译原理中的概念，相对于 直接引用，主要包括三类常量
	# 1. 类和接口的全限定名
	
	# 2. 字段的名称和描述符
	
	# 3. 方法的名称和描述符
	
# 这些都是符号引用: 
	# 上面的 a、b 就是字段名称

	# Math 类常量池里的 Lcom/tuling/jvm/Math 就是类的全限定名

	# main 和 compute 是方法名称
```

### 运行时常量池

```shell
# 当 class 文件被加载至 JVM 方法区中时，上面的 静态常量池 就成了运行时常量池，
	# 这些 符号引用 才有了真正对应的内存地址。
	
	# 对应的 符号引用 在 程序加载 或 运行时 会被转变为 被加载到内存区域的代码的直接引用，
		# 也就是所谓的 动态链接
		
	# 例如: compute() 这个符号引用，在运行时，
		# 就会被转变为 compute() 方法具体代码在内存中的地址，
		
		# 主要通过 对象头里的类型指针 去转换直接引用。
```

### 字符串常量池

#### 设计思想

```shell
# JVM 为了提高性能和减少内存开销，在 实例化 字符串对象 的时候，进行了一些优化。
	# 为字符串开辟一个字符串常量池，类似于缓存区。
	
	# 创建字符串常量时，首先查询字符串常量池是否存在该字符串。
	
	# 存在该字符串，直接返回引用实例，不存在，实例化该字符串并放入池中。
```

#### 三种字符串操作（JDK1.7+）

##### 直接赋值字符串

```java
// str 指向常量池中的引用
String str = "Swordsman";
```

```shell
# 这种方式创建的字符串对象，只会在常量池中。

# 创建 "Swordsman" 这个字面量，JVM 先去字符串常量池中通过 equals(key) 方法，
	# 如果有相同对象，直接返回该对象在字符串常量池中的引用。
	
	# 如果没有，则会在常量池中创建一个新对象，再返回引用。
```

##### new String()

```java
// str 指向内存中的对象引用
String str = new String("Swordsman");
```

```shell
# 这种方式创建的字符串对象，会保证 字符串常量池 和 堆 都有这个对象，
	# 没有就创建，最后返回堆内存中的对象引用。
	
# 先去 字符串常量池 中检查是否存在，
	# 不存在: 先去字符串常量池里创建一个字符串对象，再去堆内存中创建一个字符串对象
	
	# 存在: 直接去堆内存中创建一个字符串对象。
	
	# 最后，将内存中的引用返回。
```

##### intern 方法

```java
String s1 = new String("Swordsman");
String s2 = s1.intern();

// false
System.out.println(s1 == s2);
```

```shell
# String 中的 intern 方法是一个 native 方法，
	# 当调用 intern() 时，如果 字符串常量池 中已经包含一个等于此 String 对象的字符串，
	
	# 则返回 字符串常量池 中的字符串，
	
	# 否则，将 intern 返回的引用指向当前字符串 s1 
```

#### 字符串常量池位置

```shell
# JDK1.6之前，有永久代，运行时常量池在永久代，运行时常量池包含字符串常量池。

# JDK1.7: 有永久代，但已经逐步"去永久代"，字符串常量池从永久代里的运行时常量池分离到堆里。

# JDK1.8+: 无永久代，运行时常量池在元空间，字符串常量池依然在堆里。
```

```java
/**
 * JDK6: -Xms6M -Xmx6M -XX:PermSize=6M -XX:MaxPermSize=6M
 * JDK8: -Xms6M -Xmx6M -XX:MetaspaceSize=6M -XX:MaxMetaspaceSize=6M
 */
public class RuntimeConstantPoolOOMTest {
  public static void main(String[] args) {
    ArrayList<String> list = new ArrayList<String>();
    for (int i = 0; i < 1000000; i++) {
      String str = String.valueOf(i).intern();
      list.add(str);
    }
  }
}
```

```shell
# 运行结果:
	# JDK1.7+: java.lang.OutOfMemoryError: Java heap space
	
	# JDK1.6: java.lang.OutOfMemoryError: PermGen space
```

#### 字符串常量池设计原理

```shell
# 底层是 Hotspot C++ 实现，类似于一个 HashTable，保存字符串对象的引用。
```

#### 字符串常量池常见面试题

##### 一

```java
// 下面代码创建了多少个 String 对象
String s1 = new String("he") + new String("llo");
String s2 = s1.intern();
```

```shell
# 在 JDK1.6 下输出是 false，创建了 6 个对象

# 在 JDK1.7 及以上的版本输出是 true，创建了 5 个对象
```

```shell
# 变化原因:
	# JDK版本迭代，字符串常量池从永久代中脱离，进入堆中，intern() 方法也相应发生变化。
```

```shell
# JDK1.6 中，调用 intern() 首先在 字符串常量池 中寻找 equal() 相等的字符串，
	# 存在则返回字符串在字符串常量池中的引用，
	
	# 不存在时，JVM 重新在永久代上创建一个实例，将 StringTable 的一个表格指向它。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1597826478758.png)

```shell
# JDK1.7+，字符串常量池不在永久代了，intern() 做了一些修改，
	# 首先在 字符串常量池 中寻找 equal() 相等的字符串，
	
	# 存在时，直接返回对象引用
	
	# 不存在时，直接指向堆上的实例
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1597826647847.png)

##### 二

```java
String s0 = "AgeFades";
String s1 = "AgeFades";
String s2 = "Age" + "Fades";
// true
System.out.println(s0 == s1);
// true
System.out.println(s0 == s2);
```

```shell
# s0 和 s1 都是直接声明在 字符串常量池 中，在编译器就确定了 s0 == s1 为 true，

# s2 "Age" 和 "Fades" 也是字符串常量，
	# 当一个字符串由多个 字符串常量 连接而成时，它自己肯定也是字符串常量，
	
	# 所以 s2 在编译期同样被优化为一个字符串常量 "AgeFades"
	
	# 所以 s2 也是字符串常量池中 "AgeFades" 的一个引用，所以 s0 == s1 == s2 为 true
```

##### 三

```java
String s0 = "AgeFades";
String s1 = new String("AgeFades");
String s2 = "Age" + new String("Fades");
// false
System.out.println(s0 == s1);
// false
System.out.println(s0 == s2);
// false
System.out.println(s1 == s2);
```

```shell
# new String() 创建的字符串不是常量，不能在编译期就确定。

# 所以 new String() 创建的字符串不放入常量池中，而是放在堆里。

# s0 还是字符串常量池中的地址引用
	# s1 和 s2 new String("Fades") 都无法在编译期确定，
	
	# 所以 s1 和 s2 都是在堆内存中新的对象
	
	# 所以 s0 != s1 != s2 为 true
```

##### 四

```java
String a = "a1";
String b = "a" + 1;
// true
System.out.prinln(a == b);
  
String c = "atrue";
String d = "a" + "true";
// true
System.out.prinln(c == d);
  
String e = "a3.4";
String f = "a" + 3.4;
// true
System.out.prinln(e == f);
```

```shell
# JVM 对应字符串常量的 "+" 号连接，在程序编译期，就优化成了一个字符串常量
	# 所以上面的都是 true
```

##### 五

```java
String a = "ab";
String bb = "b";
String b = "a" + bb;
// false
System.out.prinln(a == b);
```

```shell
# 字符串 "+" 连接中，存在引用，而 JVM 无法在编译期确定引用的值，
	# 所以这里是 JVM 在程序运行时，动态分配并将连接后的新地址赋给b
	
	# 所以这里就是 false
```

##### 六

```java
String a = "ab";
final String bb = "b";
String b = "a" + bb;
// true
System.out.println(a == b);
```

```shell
# 和 五 相比，唯一不同的是 bb 字符串加了 final 修饰，
	# 对于 final 修饰的变量，在编译时被解析为常量值的一个本地拷贝，
	
	# 存储到自己的常量池中或嵌入到它的字节码流中。
	
	# 所以此时的 "a" + bb 和 "a" + "b" 效果是一样的。
```

##### 七

```java
String a = "ab";
final String bb = getBB();
String b = "a" + bb;
// false
System.out.prinltn(a == b);

private static String getBB() {
  return "b";
}
```

```shell
# JVM 对于字符串引用 bb，无法在编译期确定它的值，
	# 只有在程序运行期调用方法后，将方法的返回值和 "a" 来动态链接并分配地址为 b。
```

##### 八

```java
String s1 = new String("test");
// test作为字面量，放入了字符串常量池中
// new String() 指向的是堆中的地址、intern 指向的是字符串常量池中的地址
// false
System.out.println(s1 == s1.intern());

String s2 = new StringBuilder("abc").toString();
// false，原理同上
System.out.println(s2 == s2.intern());

String str1 = new StringBuilder("ja").append("va").toString();
// str1 指向的是堆中的地址
// 而 java 作为关键字，在 JVM 初始化相关类的时候就已经被放入字符串常量池了，
// 所以 intern 指向的是 字符串常量池 中的地址
// false
System.out.println(str1 == str1.intern());

String str2 = new StringBuilder("计算机").append("技术").toString();
// str2 指向的堆中对象的地址
// intern 先去字符串常量池找，只有 "计算机" "技术"，没有 "计算机技术"
// 所以 JDK1.7+ intern 在字符串常量池没有找到字符串常量，直接指向该对象的堆地址
// true
System.out.println(str2 == str2.intern());
```

#### 关于 String 是不可变的

```java
// 编译期就被优化成了 String s = "abc";
String s = "a" + "b" + "c";
String a = "a";
String b = "b";
String c = "c";

// 通过 java -p 查看字节码指令，发现 s1 的 "+" 操作都变成了
// StringBuilder temp = new StringBuilder();
// temp.append(a).append(b).append(c);
// String s1 = temp.toString;
String s1 = a + b + c;
```

### 八种基本类型的包装类和常量缓存池

```shell
# Java 中基本类型的包装类，大部分都实现了常量池技术，
	# 严格来说应该叫 对象池，在堆上。
	
# 这些类是:
	# Byte、Short、Interger、Long、Character、Boolean
	
	# 另外两种浮点数类型的包装类则没有实现，
	
	# Byte、Short、Interger、Long、Character 这五种包装类，
		# 常量缓存池的数值都是在 -128 - 127
```

```java
Integer i1 = 127;
Integer i2 = 127;
// 底层调用实际是 integer.valueOf(127)，用到了 IntergerCache
// true
System.out.println(i1 == i2)
  
Integer i3 = 128;
Integer i4 = 128;
// 大于 127，没有用到常量缓存池，在堆上创建新对象
// false
System.out.println(i3 == i4)
  
Integer i5 = new Integer(127);
Integer i6 = new Integer(127);
// 用 new 关键词新生成对象，不会使用常量缓存池
// false
System.out.println(i5 == i6)
  
Boolean bool1 = true;
Boolean bool2 = true;
// Boolean true 和 false 都在常量缓存池
// true
System.out.println(bool1 == bool2);

Double d1 = 1.0;
Double d2 = 1.0;
// 浮点类型的包装类没有实现常量缓存池技术
// false
System.out.println(d1 == d2);
```

