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

### 	函数式接口

​		Lambda 表达式需要 "函数式接口" 的支持

​		函数式接口 : 接口中只有一个抽象方法的接口，称为函数式接口

​			可以使用 @FunctionalInterface 注解检查是否函数式接口

## 函数式接口

## 方法引用与构造器引用

## Stream API

## 接口中的默认方法与静态方法

## 新时间日期 API

## 其他新特性