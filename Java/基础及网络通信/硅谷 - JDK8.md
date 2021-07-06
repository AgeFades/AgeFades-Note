[TOC]

# 硅谷 - JDK8

## Lambda 表达式

### 	简介

​		Lambda 是一个 匿名函数，可理解为 一段可以传递的代码

### 	语法

​		Jdk8 中引入了一个新的操作符 "->" ，该操作符称为箭头操作符

​		箭头操作符将 Lambda 表达式拆分成两部分

​			左侧 : Lambda 表达式的参数列表

​			右侧 : Lambda 表达式中所需执行的功能，即 Lambda 体

#### 		格式1

​			无参数，无返回值

```java
Runnable xx = () -> System.out.println("Hello Lambda!");
xx.run();
```

#### 		格式2

​			一个参数，无返回值（小括号可以忽略不写）

```java
Consumer<String> con = (x) -> System.out.println(x);
con.accept("Hello Lambda!");
```

#### 		格式3

​			两个或以上参数，有返回值，并且 Lambda 体中有多条语句

```java
Comparator<Interger> com = (x,y) -> {
    System.out.println("Hello Lambda!");
    return Integer.compare(x,y);
}
```

#### 		格式4

​			若 Lambda 体中只有一条语句，return 和 大括号都可以省略

```java
Comparator<Integer> com = (x,y) -> Integer.compare(x,y);
```

#### 		格式5

​			Lambda 表达式的参数列表的数据类型可以省略不写，因为 JVM 能 "类型推断"

```java
(Integer x,Integer y) -> Integer.compare(x,y);
```

#### 		总结

​			上联 : 左右遇一括号省

​			下联 : 左侧推断类型省

​			横批 : 能省则省

## 函数式接口

​	Lambda 表达式需要 "函数式接口" 的支持

​		函数式接口 : 接口中只有一个抽象方法的接口，称为函数式接口

​			可以使用 @FunctionalInterface 注解检查是否函数式接口

### 	四大内置核心函数式接口

​		Jdk8 内置的四大核心函数式接口

```java
Consumer<T> : // 消费型接口
	void accept(T t);
Supplier<T> : // 供给型接口
	T get();
Function<T,R> : // 函数型接口
	R apply(T t);
Predicate<T> : // 断言型接口
	boolean test(T t);
```

​	![UTOOLS1561543008368.png](https://i.loli.net/2019/06/26/5d13416126a7642261.png)

## 方法引用与构造器引用

### 	方法引用 

​		若 Lambda 体中的内容有方法已经实现了，可以使用 "方法引用"

​		（可以理解为方法引用是 Lambda 表达式的另外一种表达形式）

​		注意 : @Lambda 体中调用方法的参数列表与返回值类型，要与函数式接口中抽象方法的参数列表和返回值一致

​			  若 Lambda 参数列表中的第一参数是实例方法的调用者，第二个参数是实例方法的参数时，可以使用

​				ClassName :: method 

#### 		语法格式

​			对象 :: 实例方法名

```java
Consumer<String> con = System.out::println;
con.accept("Hello World!");
```

​			类 :: 静态方法名

```java
Comparator<Integer> com = Integer::compare;
```

​			类 :: 实例方法名

```java
BiPredicate<String,String> bp = (x,y) -> x.equals(y);
BiPredicate<String,String> bp2 = String::equals;
```

### 	构造器引用

#### 		语法格式

​			ClassName :: new

```java
Supplier<Employee> sup = () -> new Employee();
Supplier<Employee> sup2 = Employee::new;
```

### 	数组引用

#### 		语法格式

​			Type :: new

```java
Function<Integer,String[]> fun = String[]::new;
String[] str = fun.apply(10);
System.out.println(str2.length);
```

## Stream API

### 	介绍

​		Stream 是 JDK8 中处理集合的关键抽象概念，可以指定对集合的操作。

​		可以执行复杂查找、过滤、映射数据等操作

​		简而言之，类似 SQL 中 where、like、limit ... 等等操作

​		"集合讲数据，流讲计算！"

​		注意 :

​			Stream 自己不会存储元素

​			Stream 不会改变源对象，相反，会返回一个持有结果的新 Stream

​			Stream 操作是延迟执行的，等同于 懒加载。

### 	操作步骤

​		创建 Stream

​			一个数据源<集合、数组 ...> ，获取一个流

```java
// 1. 可以通过 Collection 系列集合提供的 stream() 或 parallelStream()
List<String> list = new ArrayList<>();
Stream<String> steam = list.stream();

// 2. 通过 Arrays 中的静态方法 stream() 获取数据组
Employee[] emps = new Employee[10];
Stream[Employee] steam2 = Arrays.stream(emps);

// 3. 通过 Stream 类中的静态方法 of()
Stream<String> stream3 = Stream.of("aa","bb","cc");

// 4. 创建无限流
// 迭代
Stream<Integer> stream4 = Stream.iterate(0,x -> x+2);
stream4.limit(10).forEach(System.out::println);

// 生成
Stream.generate(() -> Math.random())
      .limit(5)
      .forEach(System.out::println);
```

​		中间操作

​			一个中间操作链，对数据源的数据进行处理

​			多个 中间操作 可以连接起来成为一个流水线，除非流水线上触发终止操作

​				否则 中间操作 不会执行任何的处理，而在 终止操作 时一次性全部处理，

​				称为 "惰性求值"。

```java
/**
* 筛选与切片
* filter : 接收 Lambda，从流中排除某些元素
* limit : 截断流，使其元素不超过给定数量
* skip(n) : 跳过元素，返回一个扔掉了前 n 个元素的流。
*			若流中元素不足 n 个，返回空流，与 limit(n) 互补
* distinct : 筛选，通过流所生成元素的 hashCode() 和 equals() 去除重复元素
* map : 映射，接收 Lambda，将 元素转换成其他形式 或提取信息。
* 			 接收函数作为参数，该函数会被应用到每个元素上，并将其映射成一个新的元素
* flatMap : 接收一个函数作为参数,将流中的每个值都换成另一个流，然后把所有流合并成一个流
* sorted() : 自然排序(Comparable)
* sorted(Comparator com) : 定制排序(Comparator)
*/

// 内部迭代 : 迭代操作由 Stream API 完成
// 中间操作
Stream<Employee> stream = employees.stream()
    							   .filter( e -> e.getAge() > 35 );
// 终止操作
stream.forEach(System.out::println);

// map
List<String> list = Arrays.asList("aaa","bbb","ccc");
list.stream()
    .map((str) -> str.toUpperCase())
    .forEach(System.out::println);

employees.stream()
    	 .map(Employee::getName)
    	 .forEach(System.out::println);

// 自然排序
List<String> list = Arrays.asList("ccc","bbb","aaa");
list.stream()
    .sorted()
    .forEach(System.out::println);

// 定制排序
employees.stream()
    .sorted((e1,e2) -> {
        return e1.getAge().equals(e2.getAge()) ? e1.getName().compareTo(e2.getName()) 
            								   : e1.getAge().compareTo(e2.getAge())
    }).forEach(System.out::println);


```

​		终止操作

​			一个终止操作，执行中间操作链，产生结果。

```java
/**
* 查找与匹配
* allMatch : 检查是否匹配所有元素
* anyMatch : 检查是否至少匹配一个元素
* noneMatch : 检查是否没有匹配所有元素
* findFirst : 返回第一个元素
* findAny : 返回当前流中的任意元素
* count : 返回流中元素的总个数
* max : 返回流中最大值
* min : 返回流中最小值
*/

// 是否全部匹配
employees.stream()
    	 .allMatch( e -> e.getStatus().equals(Status.BUSY) );

// max
employees.stream()
    	.max((e1,e2) -> Double.compare(e1.getSalary(),e2.getSalary()));

// min
employees.stream()
    	 .map(Employee::getSalary)
    	 .min(Double::compare);
```

```java
/**
* 规约与收集
*
* 规约
* reduce(T identity,BinaryOperator) / reduce(BinaryOperator) :
* 	可以将流中元素反复结合起来，得到一个值
* map 与 reduce 的；连接通常称为 map-reduce 模式(映射规约)
*
* 收集
* collect : 将流转换为其他形式，接收一个 Collector 接口的实现，用于给 Stream 中元素汇总 
* 	Collectors.counting() : 总数
* 	Collectors.averagingDouble(Employee::getSalary()) : 平均值
* 	Collectors.summingDouble(Employee::getSalary())	: 总和
* 	Collectors.maxBy((e1,e2) -> Double.compare(e1.getSalary(),e2.getSalary())) : 最大值
* 	Collectors.joining(",") : 组合某字符串
*/

// 规约
List<Integer> list = Arrays.asList(1,2,3,4,5);
Integer sum = list.stream()
    			  .reduce(0,(x,y) -> x + y);
System.out.println(sum); // 15

Optional<Double> op = employees.stream()
    						   .map(Employee::getSalary)
    						   .reduce(Double::sum);
System.out.println(op.get()); // 员工工资总和

// 收集
List<String> list = employees.stream()
    						 .map(Employee::getName)
    						 .collect(Collector.toList()); 

HashSet<String> hs = employees.stream()
    						  .map(Employee::getName)
    						  .collect(Collectors.toCollection(HashSet::new));
							  .forEach(System.out::println);

// 分组
Map<Status,List<Employee>> map = employees.stream()    									 											  .collect(Collectors
									      .groupingBy(Employee::getStatus);

// 多级分组
Map<Status,Map<String,List<Employee>>> map = employees.stream()                                                                                                                                               .collect(Collectors																.groupingBy(Employee::getStatus,Collectors.groupingBy( e ->{
    if(((Employee) e).getAge() <= 35 ) return "青年";
    else if(((Employee) e).getAge() <= 50 ) return "中年";
    else return "老年";
})))
      
// 分区
Map<Boolean,List<Employee>> map = employees.stream()
                                           .collect(Collectors.partitioningBy( e -> 												   e.getSalary() >8000));                                                                                                    
                                                                                                                                               
// 数据收集
DoubleSummaryStatistices dss = employees.strean()
                                 		.collect(Collectors
                                	    .summarizingDouble(Employee::getSalary)); 
System.out.println(dss.getSum()); // or getAverage | getMax ...                                                                                                                                               
                                                                                                                                                                                                                                                                                                                                                                                                                         
                                                                                                                                              

```

### 	并行流与顺序流

​		JDK8 中将并行进行了优化，Stream API 可以声明性地通过 

​		parallel() 与 sequential() 在并行流与顺序流之间进行切换。

#### 		并行流

​			把一个内容分成多个数据块，并用不同的线程分别处理每个数据块的流。

#### 		Fork / Join 框架​			![UTOOLS1561602608462.png](https://i.loli.net/2019/06/27/5d142a31b9b2726965.png)

#### 		Fork / Join 与传统线程池的区别

​			采用 "工作窃取" 模式（work - stealing）:

​				当执行新任务时，拆分成更小的任务执行，并将小任务加到线程队列中

​				然后再从一个随机线程的队列中偷一个放到自己的队列中。

​			优势 : 

​				一般的线程池，如果一个线程正在执行的任务由于某些原因无法继续运行

​				该线程就处于等待状态。

​				Fork / Join 中，如某个子任务的线程等待另一个子任务的完成而阻塞

​				该线程会主动寻找其他未运行的子任务来运行

​				减少线程等待时间，提供性能和吞吐量。

## Optional 

### 	介绍

​		java.util.Optional : 容器类，代表一个值存在或不存在。

​		原来 null 表示一个值不存在，现在 Optional 可以更好的避免空指针异常。

### 	常用方法

​		Optional.of(T t) : 创建一个 Optional 实例

```java
/**
* 此处打印为 Employee[ xx =null ...]
* 如果 Optional.of(null), 还是会报空指针异常
*/
Optional<Employee> op = Optional.of(new Employee());
System.out.println(op.get());
```

​		Optional.empty() : 创建一个空的 Optional 实例

```java
/**
* 此处报错，NoSeachElement ....
*/
Optional<Employee> op = Optional.empty();
System.out.println(op.get());
```

​		Optional.ofNullable(T t) : 若 t 不为 null，创建 Otional 实例，否则创建空实例

​		isPresent() : 判断是否包含值

​		orElse(T t) : 如果调用对象包含值，返回该值，否则返回 t

​		orElseGet(Supplier s) : 如果调用对象包含值，返回该值，否则返回 s 获取的值

​		map(Function f) : 如果有值对其处理，并返回处理后的 Optional，否则返回 Optional.empty()

​		flatMap(Function mapper) : 与 map 类似，要求返回值必须是 Optional

## 接口中的默认方法与静态方法

### 	默认方法

​		类优先原则

​			若某接口中定义了一个默认方法，而另外一个父类或接口中又定义了一个同名的方法时

​			选择父类中的方法。如果一个父类提供了具体的实现，那么接口中具有相同名称和参数的

​				默认方法会被忽略

​			接口冲突。如果一个父接口提供一个默认方法，而另一个接口中也提供了一个具有相同名称

​			和参数列表的方法（不管方法是否有默认方法），那么必须覆盖该方法来解决冲突。

## 新时间日期 API

​	个人感觉这部分没什么太大的意义，时间日期类的处理都可参照 [Hutool.cn]()

## 其他新特性

### 	重复注解与类型注解

​		JDK8 对注解处理提供了两点改进

​			可重复的注解及可用于类型的注解

​		感觉也没啥用

## 总结

​	JDK8 最重要的两个更新

​	Lambda 表达式

​	Stream 流







