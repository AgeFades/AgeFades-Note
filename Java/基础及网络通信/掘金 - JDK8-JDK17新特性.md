[TOC]

# JDK8-JDK17新特性总结

[原文链接](https://juejin.cn/post/7250734439709048869)

[长安不及十里](https://juejin.cn/user/3228642192657389/posts)

2023-07-018,608阅读1小时+

前言：最近写项目用到了Spring3.0，JDK8不行了，一看原来JDK已经更新到了22了，下面是我总结的一些新特性 具体可以参考：[openjdk.org/projects/jd…](https://link.juejin.cn/?target=https%3A%2F%2Fopenjdk.org%2Fprojects%2Fjdk%2F)

# 一 JDK8 新特性

## 1.1 Lambda 表达式

Lambda表达式是Java 8引入的一个重要特性，它允许以更简洁的方式编写匿名函数，Lambda表达式可以看作是一种轻量级的函数式编程的语法表示 Lambda表达式的基本语法如下：

```rust
rust
复制代码(parameter1, parameter2, ..., parameterN) -> { 
    // 方法体
}
```

Lambda表达式包含以下几个部分：

- 参数列表：包含在圆括号内，可以是零个或多个参数。
- 箭头操作符：由箭头符号 "->" 表示，用于分隔参数列表和方法体。
- 方法体：包含在花括号内，实现具体的操作逻辑。

Lambda表达式的主要用途是简化代码，尤其是在使用函数式接口时非常方便。函数式接口是只包含一个抽象方法的接口，Lambda表达式可以与函数式接口@FunctionalInterface一起使用，从而实现函数式编程的效果。 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/71ff3fc9b9d54293b6b7f8f20227168f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

```typescript
typescript
复制代码package lambda;

/**
 * @description: 使用 lambda 表达式创建线程
 * @author: shu
 * @createDate: 2023/6/30 9:49
 * @version: 1.0
 */
public class Thread01 {
    public static void main(String[] args) {
        // 1.1 使用匿名内部类
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("Hello World! 1.1");
            }
        }).start();

        // 1.2 使用 lambda 表达式
        new Thread(() -> System.out.println("Hello World! 1.2 ")).start();

        // 1.3 使用 lambda 表达式和类型推导
        new Thread(() -> {
            System.out.println("Hello World! 1.3");
        }).start();
    }
}

```

Lambda表达式可以更简洁地表示匿名函数，避免了传统的匿名内部类的冗余代码，提高了代码的可读性和可维护性。它是Java 8引入的一个强大特性，广泛应用于函数式编程和Java 8以后的各种API中。

## 1.2 新的日期API

JDK 8引入了一个新的日期和时间API，名为`java.time`，它提供了更加灵活和易于使用的日期和时间处理功能。该API在设计上遵循了不可变性和线程安全性的原则，并且提供了许多方便的方法来处理日期、时间、时间间隔和时区等。 下面是`java.time`包中一些主要的类和接口：

1. `LocalDate`：表示日期，例如：年、月、日。
2. `LocalTime`：表示时间，例如：时、分、秒。
3. `LocalDateTime`：表示日期和时间，结合了`LocalDate`和`LocalTime`。
4. `ZonedDateTime`：表示带时区的日期和时间。
5. `Instant`：表示时间戳，精确到纳秒级别。
6. `Duration`：表示时间间隔，例如：小时、分钟、秒。
7. `Period`：表示日期间隔，例如：年、月、日。
8. `DateTimeFormatter`：用于日期和时间的格式化和解析。
9. `ZoneId`：表示时区。
10. `ZoneOffset`：表示时区偏移量。

这些类提供了一系列方法来执行日期和时间的计算、格式化、解析和比较等操作。而且，`java.time`包还提供了对日历系统的支持，包括对ISO-8601日历系统的全面支持。 以下是一个简单的示例，展示了如何使用新的日期API创建和操作日期和时间：

```java
java
复制代码package lambda;

import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;


/**
 * @description: 新的日期和时间 API 示例
 * @author: shu
 * @createDate: 2023/6/30 10:15
 * @version: 1.0
 */
public class DateDemo01 {
    public static void main(String[] args) {
        // 创建日期
        LocalDate date = LocalDate.now();
        System.out.println("Today's date: " + date);

        // 创建时间
        LocalTime time = LocalTime.now();
        System.out.println("Current time: " + time);

        // 创建日期和时间
        LocalDateTime dateTime = LocalDateTime.now();
        System.out.println("Current date and time: " + dateTime);

        // 格式化日期和时间
        String formattedDateTime = dateTime.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
        System.out.println("Formatted date and time: " + formattedDateTime);

        // 执行日期的计算操作
        LocalDate tomorrow = date.plusDays(1);
        System.out.println("Tomorrow's date: " + tomorrow);

        // 执行时间的比较操作
        boolean isBefore = time.isBefore(LocalTime.NOON);
        System.out.println("Is current time before noon? " + isBefore);
    }
}
```

## 1.3 使用Optional

JDK 8引入了一个新的类，名为`Optional`，用于解决在处理可能为null的值时出现的NullPointerException（空指针异常）问题。`Optional`类的设计目标是鼓励更好的编程实践，明确表示一个值可能为空，并鼓励开发人员在使用这些可能为空的值时进行显式的处理。 `Optional`类是一个容器对象，可以包含一个非空的值或者没有值。它提供了一系列方法来处理包含值和没有值的情况，例如：获取值、判断是否有值、获取默认值、执行操作等。 下面是`Optional`类的一些常用方法：

1. `of()`：创建一个包含非空值的Optional对象。
2. `empty()`：创建一个空的Optional对象。
3. `ofNullable()`：根据指定的值创建一个Optional对象，允许值为null。
4. `isPresent()`：判断Optional对象是否包含值。
5. `get()`：获取Optional对象中的值，如果没有值，则抛出NoSuchElementException异常。
6. `orElse()`：获取Optional对象中的值，如果没有值，则返回默认值。
7. `orElseGet()`：获取Optional对象中的值，如果没有值，则通过提供的Supplier函数生成一个默认值。
8. `orElseThrow()`：获取Optional对象中的值，如果没有值，则通过提供的Supplier函数抛出指定的异常。
9. `map()`：对Optional对象中的值进行转换，并返回一个新的Optional对象。
10. `flatMap()`：对Optional对象中的值进行转换，并返回一个新的Optional对象，该方法允许转换函数返回一个Optional对象。

以下是一个简单的示例，展示了如何使用`Optional`类来处理可能为空的值：

```arduino
arduino
复制代码import java.util.Optional;

/**
 * @description: Optional 示例
 * @author: shu
 * @createDate: 2023/6/30 10:20
 * @version: 1.0
 */
public class OptionalDemo {
    public static void main(String[] args) {
        String value = "Hello";

        // 创建一个包含非空值的Optional对象
        Optional<String> optional = Optional.of(value);

        // 判断Optional对象是否包含值
        if (optional.isPresent()) {
            // 获取Optional对象中的值
            String result = optional.get();
            System.out.println("Value: " + result);
        }

        // 获取Optional对象中的值，如果没有值，则返回默认值
        String defaultValue = optional.orElse("Default Value");
        System.out.println("Default Value: " + defaultValue);

        // 对Optional对象中的值进行转换
        Optional<String> transformed = optional.map(String::toUpperCase);
        if (transformed.isPresent()) {
            System.out.println("Transformed Value: " + transformed.get());
        }
    }
}
```

在上面的示例中，我们创建了一个包含非空值的Optional对象，并使用`isPresent()`和`get()`方法来判断和获取值。然后，我们使用`orElse()`方法来获取值或返回默认值，以及使用`map()`方法对值进行转换。 `Optional`类可以在代码中提供更加安全和清晰的处理空值的方式。然而，它并不适用于所有的场景，需要根据实际情况来判断是否使用`Optional`。请注意，过度使用`Optional`可能会导致代码冗余，因此需要权衡使用的利弊。

## 1.4 使用[Base64](https://link.juejin.cn/?target=https%3A%2F%2Fso.csdn.net%2Fso%2Fsearch%3Fq%3DBase64%26spm%3D1001.2101.3001.7020)

在JDK 8中，Java提供了对Base64编码和解码的支持，通过`java.util.Base64`类来实现。Base64是一种用于将二进制数据编码为文本的编码方式，常用于在网络传输中或存储数据时将二进制数据表示为可打印字符 下面是一些基本的使用示例：

1. 编码为Base64字符串：

```java
java
复制代码import java.util.Base64;

/**
 * @description: 编码示例
 * @author: shu
 * @createDate: 2023/7/1 9:23
 * @version: 1.0
 */
public class EncodedDemo {
    public static void main(String[] args) {
        // 原始数据
        String originalData = "Hello, World!";
        // 编码为Base64字符串
        String encodedData = Base64.getEncoder().encodeToString(originalData.getBytes());
        System.out.println("Encoded data: " + encodedData);
    }
}
```

1. 解码Base64字符串：

```java
java
复制代码import java.util.Base64;

/**
 * @description: 解码示例
 * @author: shu
 * @createDate: 2023/7/1 9:24
 * @version: 1.0
 */
public class DecodeDemo {
    public static void main(String[] args) {
        // Base64字符串
        String encodedData = "SGVsbG8sIFdvcmxkIQ==";
        // 解码为原始数据
        byte[] decodedData = Base64.getDecoder().decode(encodedData);
        System.out.println("Decoded data: " + new String(decodedData));
    }
}
```

在上述示例中，我们使用`Base64.getEncoder()`方法获取一个`Base64.Encoder`对象，并调用其`encodeToString()`方法将原始数据编码为Base64字符串。同样，我们使用`Base64.getDecoder()`方法获取一个`Base64.Decoder`对象，并调用其`decode()`方法解码Base64字符串为原始数据。 `java.util.Base64`类还提供了其他一些方法，例如：

- `Base64.getEncoder().encodeToString(byte[] src)`：将字节数组编码为Base64字符串。
- `Base64.getDecoder().decode(String src)`：将Base64字符串解码为字节数组。
- `Base64.getMimeEncoder().encodeToString(byte[] src)`：将字节数组编码为MIME类型的Base64字符串。
- `Base64.getMimeDecoder().decode(String src)`：将MIME类型的Base64字符串解码为字节数组。
- `Base64.getUrlEncoder().encodeToString(byte[] src)`：将字节数组编码为URL安全的Base64字符串。
- `Base64.getUrlDecoder().decode(String src)`：将URL安全的Base64字符串解码为字节数组。

这些方法可以根据不同的需求选择使用。 需要注意的是，Base64编码是不加密的，仅用于编码和解码数据，如果需要加密和解密数据，请使用加密算法，如AES、RSA等。

## 1.5 接口的默认方法和静态方法

JDK 8引入了接口的默认方法和静态方法，这使得在接口中添加新功能变得更加灵活。

1. 默认方法（Default Methods）： 默认方法允许在接口中定义具有默认实现的方法。这使得在不破坏现有实现类的情况下，可以向现有接口添加新的方法。

以下是默认方法的示例：

```typescript
typescript
复制代码public interface MyInterface {
    // 抽象方法
    void abstractMethod();

    // 默认方法
    default void defaultMethod() {
        System.out.println("This is a default method.");
    }
}

public class MyClass implements MyInterface {
    @Override
    public void abstractMethod() {
        System.out.println("Implementing abstractMethod() in MyClass.");
    }
}

/**
 * @description:
 * @author: shu
 * @createDate: 2023/7/1 9:29
 * @version: 1.0
 */
public class DefaultMethodsDemo {
    public static void main(String[] args) {
        MyClass obj = new MyClass();
        obj.abstractMethod();
        obj.defaultMethod();
    }
    
}
```

在上述示例中，`MyInterface`接口中定义了一个抽象方法`abstractMethod()`和一个默认方法`defaultMethod()`。`MyClass`类实现了`MyInterface`接口，并提供了对抽象方法的具体实现。此外，`MyClass`类可以继承默认方法`defaultMethod()`的默认实现。

1. 静态方法（Static Methods）： 静态方法是接口中的另一种类型的方法，它与特定的接口关联，并且只能通过接口名称来调用。

以下是静态方法的示例：

```csharp
csharp
复制代码public interface MyInterface {
    // 抽象方法
    void abstractMethod();

    // 静态方法
    static void staticMethod() {
        System.out.println("This is a static method.");
    }
}

public class MyClass implements MyInterface {
    @Override
    public void abstractMethod() {
        System.out.println("Implementing abstractMethod() in MyClass.");
    }
}

public class Main {
    public static void main(String[] args) {
        MyClass obj = new MyClass();
        obj.abstractMethod();
        MyInterface.staticMethod();
    }
}
```

在上述示例中，`MyInterface`接口中定义了一个抽象方法`abstractMethod()`和一个静态方法`staticMethod()`。`MyClass`类实现了`MyInterface`接口，并提供了对抽象方法的具体实现。`staticMethod()`可以通过接口名称直接调用。 通过默认方法和静态方法，接口的功能可以更加灵活地扩展，而无需破坏已有的实现类。这在JDK 8之前是不可能的。

## 1.6 方法引用格式

在JDK 8中，引入了方法引用（Method Reference）的概念，用于简化函数式接口（Functional Interface）的实现。方法引用允许您直接引用现有方法作为Lambda表达式的替代，使代码更加简洁和易读。在JDK 8中，有四种不同的方法引用格式：

1. 静态方法引用（Static Method Reference）：格式为`类名::静态方法名`。例如，`Integer::parseInt`表示引用Integer类的静态方法parseInt。
2. 实例方法引用（Instance Method Reference）：格式为`对象::实例方法名`。例如，`System.out::println`表示引用System.out对象的println方法。
3. 对象方法引用（Object Method Reference）：格式为`类名::实例方法名`。这种引用适用于无参实例方法。例如，`String::length`表示引用String类的length方法。
4. 构造方法引用（Constructor Method Reference）：格式为`类名::new`。例如，`ArrayList::new`表示引用ArrayList类的构造方法。

在使用方法引用时，要根据接口的抽象方法的参数个数和返回类型选择适当的方法引用格式。 以下是一个简单的示例，展示了这些方法引用格式的使用：

```java
java
复制代码import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

/**
 * @description: 方法引用示例
 * @author: shu
 * @createDate: 2023/7/1 9:32
 * @version: 1.0
 */
public class MethodReferenceDemo {
    public static void main(String[] args) {
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
        // 静态方法引用
        names.forEach(System.out::println);
        // 实例方法引用
        names.forEach(String::toUpperCase);
        // 对象方法引用
        names.forEach(String::length);
        // 构造方法引用
        List<Integer> lengths = names.stream()
                .map(Integer::new)
                .collect(Collectors.toList());
    }
}
```

## 1.7 [Stream](https://link.juejin.cn/?target=https%3A%2F%2Fso.csdn.net%2Fso%2Fsearch%3Fq%3DStream%26spm%3D1001.2101.3001.7020)类

JDK中的Stream类是Java 8引入的一个新概念，用于处理集合和数组等数据源的元素序列。Stream类提供了一种流式操作的方式，可以用于对数据进行过滤、映射、排序、聚合等各种操作，从而更加方便和高效地处理数据。 下面是一些Stream类的主要特点和用法：

1. 流的创建：可以通过集合、数组、I/O通道等多种方式创建流。例如，通过`Collection.stream()`方法可以从集合创建一个流，通过`Arrays.stream()`方法可以从数组创建一个流。
2. 中间操作：Stream类提供了一系列中间操作方法，用于对流进行转换、过滤、映射等操作，这些操作会返回一个新的Stream对象。常见的中间操作方法包括`filter()`、`map()`、`sorted()`等。
3. 终端操作：Stream类也提供了一系列终端操作方法，用于对流进行最终的处理，返回一个结果或产生一个副作用。常见的终端操作方法包括`forEach()`、`collect()`、`reduce()`等。
4. 惰性求值：Stream类的中间操作是惰性求值的，即在调用终端操作之前，中间操作并不会立即执行。这种方式可以优化性能，只对需要处理的元素进行操作。
5. 并行处理：Stream类可以支持并行处理，即在处理大量数据时，可以将操作并行化以提高处理速度。通过`parallel()`方法可以将流转换为并行流。

下面是一个简单的示例，展示了Stream类的使用：

```java
java
复制代码import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

/**
 * @description:
 * @author: shu
 * @createDate: 2023/7/1 9:36
 * @version: 1.0
 */
public class StreamDemo {
    public static void main(String[] args) {
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "Dave", "Eve");

        // 过滤长度大于3的元素
        names.stream()
                .filter(name -> name.length() > 3)
                .forEach(System.out::println);

        // 转换为大写并排序
        List<String> uppercaseNames = names.stream()
                .map(String::toUpperCase)
                .sorted()
                .collect(Collectors.toList());

        // 求长度之和
        int totalLength = names.stream()
                .mapToInt(String::length)
                .sum();
    }
}
```

具体参考我前面写的文章

## 1.8 注解相关的改变

在JDK 8中，注解相关的改变主要集中在两个方面：重复注解（Repeatable Annotations）和可使用的类型（Type Annotations）。

1. 重复注解（Repeatable Annotations）：在JDK 8之前，每个注解在一个元素上只能使用一次。而在JDK 8中，引入了重复注解的概念，允许在同一个元素上多次使用相同的注解。为了支持重复注解，新增了两个元注解（Meta-Annotations）：@Repeatable和@Retention。 @Repeatable注解用于注解声明，指定了注解的容器注解，该容器注解允许在同一个元素上多次使用相同的注解。 @Retention注解用于指定注解的生命周期，它可以应用于注解声明和容器注解。常用的生命周期包括@Retention(RetentionPolicy.SOURCE)、@Retention(RetentionPolicy.CLASS)和@Retention(RetentionPolicy.RUNTIME)。 通过使用重复注解，可以更灵活地在同一个元素上应用多个相同的注解，而不需要创建额外的容器注解。
2. 可使用的类型（Type Annotations）：在JDK 8之前，注解主要应用于类、方法、字段等元素的声明上。而在JDK 8中，引入了可使用的类型，使得注解可以应用于更多的位置，包括类型的使用上。 可使用的类型注解通过在类型前面添加注解，来对类型使用进行约束和标记。例如，可以在变量声明、方法参数、泛型类型参数等位置使用注解。 可使用的类型注解提供了更丰富的语义，可以帮助编译器和静态分析工具检查类型使用的合法性，并提供更精确的类型检查和约束。

这些改变使得注解在JDK 8中变得更加灵活和强大，可以更好地应用于代码的描述、分析和约束。重复注解和可使用的类型为开发人员提供了更多的选择和扩展性，使得注解在Java语言中的应用更加广泛和多样化。

## 1.9 支持并行（parallel）数组

在JDK 8中，引入了对并行数组操作的支持。这个功能由Arrays类和新的ParallelSorter接口提供 Arrays类中新增了一些用于并行操作数组的方法，其中最突出的是`parallelSort()`方法。该方法用于对数组进行并行排序，可以显著提高排序的性能。与传统的`sort()`方法相比，`parallelSort()`方法会将数组划分为多个小块，并在多个线程上并行进行排序。 以下是一个示例代码，展示了`parallelSort()`方法的使用：

```java
java
复制代码import java.util.Arrays;

/**
 * @description:
 * @author: shu
 * @createDate: 2023/7/1 9:39
 * @version: 1.0
 */
public class ParallelArrayDemo {
    public static void main(String[] args) {
        int[] numbers = {5, 2, 8, 1, 9, 3, 7, 6, 4};
        // 并行排序数组
        Arrays.parallelSort(numbers);
        // 打印排序结果
        for (int number : numbers) {
            System.out.println(number);
        }
    }
}
```

在上面的示例中，我们使用`parallelSort()`方法对整型数组进行并行排序，然后遍历数组打印排序结果。 需要注意的是，并行数组操作适用于一些可以被划分为独立块的操作，如排序、查找等。对于一些需要依赖前后元素关系的操作，可能不适合使用并行数组操作。 并行数组操作可以充分利用多核处理器的优势，提高处理大规模数据时的性能。但在使用并行操作时，也需要注意合理划分任务和数据的负载均衡，以避免线程竞争和效率下降。

## 1.10 对并发类（Concurrency）的扩展

在JDK 8中，对并发类（Concurrency）进行了一些扩展，以提供更强大、更灵活的并发编程能力。以下是几个主要的扩展：

1. CompletableFuture类：CompletableFuture类是一个实现了CompletableFuture接口的异步计算类，用于处理异步任务的结果。它提供了一系列方法，可以通过回调、组合和转换等方式处理异步任务的完成结果。CompletableFuture类使得异步编程更加方便和直观。

```java
java
复制代码import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ConcurrentHashMap;

/**
 * @description: CompletableFuture并发编程示例
 * @author: shu
 * @createDate: 2023/7/1 9:45
 * @version: 1.0
 */
public class ConcurrencyDemo {
    public static void main(String[] args) {
        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

        // 使用CompletableFuture进行异步计算
        CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
            // 模拟耗时操作
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return 42;
        });

        CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> {
            // 模拟耗时操作
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return 100;
        });

        // 使用CompletableFuture处理异步计算结果
        CompletableFuture<Void> combinedFuture = CompletableFuture.allOf(future1, future2);
        combinedFuture.thenRun(() -> {
            int result1 = future1.join();
            int result2 = future2.join();
            map.put("result1", result1);
            map.put("result2", result2);
            System.out.println("Results: " + map);
        });

        // 等待所有任务完成
        combinedFuture.join();
    }
}
```

1. LongAdder和DoubleAdder类：LongAdder和DoubleAdder类是对AtomicLong和AtomicDouble的改进。它们提供了更高的并发性能，在高并发场景下更适合使用。LongAdder和DoubleAdder类通过分解内部计数器，将更新操作分散到多个变量上，以减少竞争和锁争用。

```ini
ini
复制代码import java.util.concurrent.atomic.DoubleAdder;
import java.util.concurrent.atomic.LongAdder;

/**
 * @description:
 * @author: shu
 * @createDate: 2023/7/1 9:51
 * @version: 1.0
 */
public class AdderDemo {
    public static void main(String[] args) {
        LongAdder longAdder = new LongAdder();
        DoubleAdder doubleAdder = new DoubleAdder();

        // 多线程并发增加值
        Thread thread1 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                longAdder.increment();
                doubleAdder.add(0.5);
            }
        });

        Thread thread2 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                longAdder.increment();
                doubleAdder.add(0.5);
            }
        });

        thread1.start();
        thread2.start();

        try {
            thread1.join();
            thread2.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        // 输出结果
        System.out.println("LongAdder Result: " + longAdder.sum());
        System.out.println("DoubleAdder Result: " + doubleAdder.sum());
    }
}
```

1. StampedLock类：StampedLock类是一种乐观读写锁，用于优化读多写少的场景。与传统的读写锁相比，StampedLock类引入了乐观读模式，避免了获取锁的开销，提供更好的并发性能。

```csharp
csharp
复制代码import java.util.concurrent.locks.StampedLock;

/**
 * @description:
 * @author: shu
 * @createDate: 2023/7/1 9:52
 * @version: 1.0
 */
public class StampedLockDemo {
    private double x, y;
    private final StampedLock lock = new StampedLock();

    public void move(double deltaX, double deltaY) {
        long stamp = lock.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            lock.unlockWrite(stamp);
        }
    }

    public double distanceFromOrigin() {
        long stamp = lock.tryOptimisticRead();
        double currentX = x;
        double currentY = y;
        if (!lock.validate(stamp)) {
            stamp = lock.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }

    public static void main(String[] args) {
        StampedLockDemo example = new StampedLockDemo();
        example.move(3, 4);
        double distance = example.distanceFromOrigin();
        System.out.println("Distance from origin: " + distance);
    }
}
```

1. ConcurrentHashMap类的改进：ConcurrentHashMap类在JDK 8中进行了一些改进，提供了更好的并发性能和扩展性。改进包括分段锁（Segmented Locking）机制的优化和内部数据结构的改进，以减少竞争和提高并发性能。

```java
java
复制代码import java.util.concurrent.ConcurrentHashMap;

/**
 * @description:
 * @author: shu
 * @createDate: 2023/7/1 9:54
 * @version: 1.0
 */
public class ConcurrentHashMapDemo {
    public static void main(String[] args) {
        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

        // 多线程并发操作map
        Thread thread1 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                map.put("A" + i, i);
            }
        });

        Thread thread2 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                map.put("B" + i, i);
            }
        });

        thread1.start();
        thread2.start();

        try {
            thread1.join();
            thread2.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        // 输出map的大小和内容
        System.out.println("Map size: " + map.size());
        System.out.println("Map content: " + map);
    }
}
```

这些扩展提供了更多并发编程的工具和选择，使得在并发场景下编写高效、可靠的代码更加容易。开发人员可以根据具体需求选择合适的并发类，以提高应用程序的性能和可伸缩性。

# 二 JDK9 新特性

## 2.1 模块化

JDK 9 引入了一个重要的特性，即模块化系统（Module System），也称为 Java 平台模块系统（Java Platform Module System，JPMS）。该特性的目标是改善 Java 平台的可伸缩性、安全性和性能。 模块化系统的主要思想是将 Java 平台分解为一系列模块，每个模块都有自己的封装和依赖关系。这些模块可以被组合在一起，以构建更大的应用程序或库。 下面是一些 JDK 9 模块化的关键概念：

1. 模块（Module）：模块是一个独立的单元，它由一组相关的包和类组成，并且具有清晰的边界。模块可以指定它所依赖的其他模块，并且可以明确地声明它对外部世界提供的公共 API。
2. 模块化源代码（Module Source Code）：在 JDK 9 中，源代码被组织为一组模块。每个模块都有自己的目录结构，其中包含模块定义文件 `module-info.java`，用于声明模块的名称、依赖关系和导出的包等信息。
3. 模块路径（Module Path）：模块路径是一组目录或 JAR 文件的集合，用于加载和链接模块。与传统的类路径不同，模块路径允许显式地指定模块之间的依赖关系。
4. 模块化 JAR 文件（Modular JAR File）：在 JDK 9 中，可以使用新的 JAR 文件格式来创建模块化的 JAR 文件。这些 JAR 文件包含模块定义文件以及其他模块所需的类和资源。
5. 模块化命令（Module-Related Commands）：JDK 9 引入了一些新的命令，用于处理模块化系统。例如，`jdeps` 命令用于分析模块之间的依赖关系，`jlink` 命令用于构建自定义运行时镜像。

模块化系统带来了许多好处，包括：

- 更好的可维护性：模块的边界清晰，易于理解和维护。
- 更好的可扩展性：模块之间的依赖关系更明确，使得应用程序和库的组合更加灵活。
- 更好的性能：模块化系统可以进行更精确的依赖分析和懒加载，从而提高应用程序的启动时间和运行时性能。
- 更好的安全性：模块可以明确声明它们的对外部世界的公共 API，从而提供更细粒度的访问控制。

需要注意的是，尽管 JDK 9 引入了模块化系统，但并不是所有的 Java 库和应用程序都需要立即迁移到模块化系统。在 JDK 9 中，仍然可以使用传统的类路径来运行非模块化的代码。但是，模块化系统为那些需要更好的可伸缩性和安全性的项目提供了一个强大的工具。

> 案例一

来个案例：首先在模块化中和需要再跟目录下定义module-info.java，里面上面你需要导入或者暴露的模块

```java
java
复制代码/**
 * @description:
 * @author: shu
 * @createDate: 2023/6/30 11:20
 * @version: 1.0
 */
module Model {
    // TODO: 2023/6/30 导入我们需要的模块
    requires cn.hutool;
    // TODO: 2023/6/30 导出我们需要的模块
    exports  com.shu.model;
}
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/05007b07094948fda0ee1c476ff04cd7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 我们调用方法，如果没有导入模块你会发现没有发现该类

```java
java
复制代码import cn.hutool.core.convert.Convert;

/**
 * @description:
 * @author: shu
 * @createDate: 2023/6/30 11:21
 * @version: 1.0
 */
public class Test01 {
    public static void main(String[] args) {
        int a = 1;
        //aStr为"1"
        String aStr = Convert.toStr(a);
        long[] b = {1,2,3,4,5};
        //bStr为："[1, 2, 3, 4, 5]"
        String bStr = Convert.toStr(b);

    }
}
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3a1dd5e221a846578809f373d2b16316~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/092f461e43364ac1b630afd45aed6b14~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

> 案例二

当一个案例来说明 JDK 9 的模块化系统如何工作。 假设我们正在开发一个简单的图书管理应用程序，它由几个模块组成：`bookmanagement.core`、`bookmanagement.ui` 和 `bookmanagement.data`。其中，`bookmanagement.core` 模块提供核心业务逻辑，`bookmanagement.ui` 模块提供用户界面，`bookmanagement.data` 模块提供数据访问功能。 每个模块都有自己的目录结构，并且包含一个模块定义文件 `module-info.java`。 首先，让我们看一下 `bookmanagement.core` 模块的 `module-info.java` 文件：

```java
java
复制代码module bookmanagement.core {
    // 需要的模块
    requires bookmanagement.data;
    // 暴露的模块
    exports com.example.bookmanagement.core;
}
```

在这个文件中，我们声明了 `bookmanagement.core` 模块的名称，并且指定了它依赖于 `bookmanagement.data` 模块。我们还通过 `exports` 关键字明确声明了 `com.example.bookmanagement.core` 包对外的公共 API。 接下来，我们来看一下 `bookmanagement.ui` 模块的 `module-info.java` 文件：

```java
java
复制代码module bookmanagement.ui {
    requires bookmanagement.core;
    exports com.example.bookmanagement.ui;
    // 使用那个接口
    uses com.example.bookmanagement.core.BookService;
}
```

在这个文件中，我们声明了 `bookmanagement.ui` 模块的名称，并且指定了它依赖于 `bookmanagement.core` 模块。我们通过 `exports` 关键字声明了 `com.example.bookmanagement.ui` 包对外的公共 API，并且使用 `uses` 关键字声明了它使用了 `com.example.bookmanagement.core.BookService` 接口。 最后，我们来看一下 `bookmanagement.data` 模块的 `module-info.java` 文件：

```javascript
javascript
复制代码module bookmanagement.data {
    exports com.example.bookmanagement.data;
    provides com.example.bookmanagement.core.BookService
    with com.example.bookmanagement.data.DefaultBookService;
}
```

在这个文件中，我们声明了 `bookmanagement.data` 模块的名称，并且通过 `exports` 关键字声明了 `com.example.bookmanagement.data` 包对外的公共 API。我们还使用 `provides` 关键字声明了 `com.example.bookmanagement.core.BookService` 接口的实现类为 `com.example.bookmanagement.data.DefaultBookService`。 通过这些模块定义文件，我们明确了每个模块的名称、依赖关系和对外的公共 API。现在，我们可以使用 JDK 9 提供的命令进行编译、打包和运行。 例如，我们可以使用以下命令编译 `bookmanagement.core` 模块：

```bash
bash
复制代码javac -d out/bookmanagement.core src/bookmanagement.core/module-info.java src/bookmanagement.core/com/example/bookmanagement/core/*.java
```

然后，我们可以使用 `jar` 命令将 `bookmanagement.core` 模块打包为一个模块化 JAR 文件：

```css
css
复制代码jar --create --file=bookmanagement.core.jar --module-version=1.0 -C out/bookmanagement.core .
```

类似地，我们可以编译和打包 `bookmanagement.ui` 和 `bookmanagement.data` 模块。 最后，我们可以使用 `java` 命令来运行我们的应用程序，指定模块路径和主模块：

```arduino
arduino
复制代码java --module-path bookmanagement.core.jar;bookmanagement.ui.jar;bookmanagement.data.jar --module bookmanagement.ui/com.example.bookmanagement.ui.Main
```

这样，我们就可以通过模块化系统构建和运行我们的图书管理应用程序。模块化系统帮助我们管理模块之间的依赖关系，确保模块的封装性和对外公共 API 的可控性。它提供了更好的可维护性、可扩展性和安全性。

## 2.2 JShell

JDK 9 引入了一个名为 jshell 的交互式命令行工具，它提供了一个方便的方式来进行 Java 代码的实时交互式探索和实验。jshell 允许您在不需要编写完整的 Java 类或应用程序的情况下，直接在命令行中编写和执行代码片段。 以下是一些关键特性和用法说明：

1. 交互式编程：jshell 提供了一个交互式的命令行环境，您可以直接在命令行中输入和执行 Java 代码。您可以逐行输入代码片段，并立即查看结果。
2. 即时反馈：每次您输入一行代码，jshell 都会立即执行并显示结果。这样可以快速验证代码并获得实时反馈，无需编译和运行完整的 Java 程序。
3. 自动补全：jshell 提供了自动补全功能，可以帮助您快速输入代码和方法名。按下 Tab 键可以自动补全命令、类、方法等。
4. 历史记录：jshell 会记录您在交互式会话中输入的代码，并允许您在以后的会话中检索和重复使用之前的代码。
5. 异常处理：jshell 具有内置的异常处理机制，可以捕获并显示代码中的异常。它会提供有关异常的详细信息，以帮助您调试和修复问题。
6. 定义变量和方法：您可以在 jshell 中定义变量和方法，并在后续代码片段中使用它们。这使得您可以逐步构建复杂的逻辑和功能。
7. 多行输入：jshell 支持多行输入，您可以使用换行符（``）将一段代码分成多行。

要启动 jshell，您可以在命令行中输入 `jshell` 命令。然后，您可以开始输入 Java 代码并立即查看结果。以下是一个简单的示例会话：

```ini
ini
复制代码$ jshell
|  Welcome to JShell -- Version 9
|  For an introduction type: /help intro

jshell> int x = 10;
x ==> 10

jshell> int y = 20;
y ==> 20

jshell> int sum = x + y;
sum ==> 30

jshell> System.out.println("The sum is: " + sum);
The sum is: 30
```

在这个示例中，我们定义了两个变量 `x` 和 `y`，并计算它们的和。然后，我们使用 `System.out.println` 方法打印结果，jshell 对于快速测试代码片段、尝试新功能或进行教学和演示非常有用，它提供了一个轻量级且便捷的方式来进行 Java 代码的交互式探索和验证。

## 2.3 改进 Javadoc

Javadoc 工具可以生成 Java 文档， Java 9 的 javadoc 的输出现在符合兼容 HTML5 标准。

## 2.4 多版本兼容JAR

多版本兼容 JAR 是 JDK 9 中引入的一个功能，它允许您创建仅在特定版本的 Java 环境中运行的库程序，并通过使用 `--release` 参数来指定编译版本。 在 JDK 9 之前，使用旧版本的 JDK 编译的 JAR 文件在较新的 Java 环境中可能会遇到兼容性问题。通过多版本兼容 JAR，您可以使用较新的 JDK 编译您的库程序，并指定兼容的目标 Java 版本，以确保在该版本及更高版本的 Java 环境中运行。 以下是使用 `--release` 参数创建多版本兼容 JAR 的步骤：

1. 编写您的库程序代码，并使用适当的 JDK 版本进行编译。假设您正在使用 JDK 11 编译您的库程序。
2. 使用以下命令创建多版本兼容 JAR 文件：

```xml
xml
复制代码javac --release <target-version> -d <output-directory> <source-files>
```

其中：

- `<target-version>` 是您希望兼容的目标 Java 版本。例如，如果您希望兼容 Java 8，可以将 `<target-version>` 设置为 `8`。
- `<output-directory>` 是输出目录，用于存放编译生成的类文件。
- `<source-files>` 是您的源代码文件或源代码目录。

例如，要创建一个兼容 Java 8 的 JAR 文件，您可以运行以下命令：

```arduino
arduino
复制代码javac --release 8 -d output mylibrary/*.java
```

这将使用 JDK 11 编译您的库程序，并生成与 Java 8 兼容的类文件。

1. 使用以下命令创建 JAR 文件：

其中：

- `<jar-file>` 是要创建的 JAR 文件的名称。
- `<input-directory>` 是包含编译生成的类文件的目录。

例如，要创建名为 `mylibrary.jar` 的 JAR 文件，您可以运行以下命令：

```css
css
复制代码jar --create --file mylibrary.jar -C output .
```

这将创建一个包含编译生成的类文件的 JAR 文件，现在，您可以将生成的多版本兼容 JAR 文件提供给其他开发人员，以便他们可以在特定的 Java 环境中使用您的库程序。 需要注意的是，使用多版本兼容 JAR 并不会自动提供对较新 Java 版本的新功能和改进的支持。它仅确保您的库程序可以在指定的目标 Java 版本及更高版本的环境中运行，但不会利用较新版本的语言特性。因此，您需要根据您的目标 Java 版本的要求进行相应的编码和功能选择。

## 2.5 集合工厂方法

在 Java 9 中，为集合框架引入了一组新的工厂方法，使创建和初始化集合对象更加简洁和方便。这些工厂方法属于 `java.util` 包中的 `List`、`Set` 和 `Map` 接口，用于创建不可变的集合对象。 下面是 Java 9 中引入的集合工厂方法的示例用法：

1. List.of() 方法用于创建不可变的列表对象。例如：

```ini
ini
复制代码List<String> fruits = List.of("Apple", "Banana", "Orange");
```

在这个示例中，我们使用 `List.of()` 方法创建一个包含三个元素的不可变列表。

1. Set.of() 方法用于创建不可变的集合对象。例如：

```ini
ini
复制代码Set<Integer> numbers = Set.of(1, 2, 3, 4, 5);
```

在这个示例中，我们使用 `Set.of()` 方法创建一个包含五个元素的不可变集合。

1. Map.of() 方法用于创建不可变的键值对映射。例如：

```ini
ini
复制代码Map<String, Integer> studentScores = Map.of("Alice", 95, "Bob", 80, "Charlie", 90);
```

在这个示例中，我们使用 `Map.of()` 方法创建一个包含三对键值对的不可变映射。 这些集合工厂方法的特点是它们创建的集合对象都是不可变的，即无法修改集合中的元素。这种不可变性有助于编写更加健壮和可靠的代码，并提供更好的线程安全性。 需要注意的是，这些集合工厂方法创建的集合对象是不可变的，因此不能对其进行添加、删除或修改元素的操作。如果需要对集合进行修改操作，仍然可以使用传统的构造函数或其他方法来创建可变的集合对象。 另外，Java 9 中的集合工厂方法还提供了一系列的重载方法，用于处理不同数量的元素。您可以根据自己的需求选择适合的方法来创建集合对象。

## 2.6 私有接口方法

Java 9 引入了一项新的特性，即私有接口方法。这意味着接口可以包含私有的方法实现，这些方法只能在接口内部被调用，对于接口的实现类和其他类是不可见的。 私有接口方法可以帮助解决以下问题：

1. 代码重用：通过在接口中定义私有方法，可以在多个默认方法之间共享代码逻辑，避免代码重复。
2. 代码组织：私有接口方法可以将复杂的逻辑分解为更小的、可重用的方法，提高代码的可读性和维护性。

下面是一个示例，演示了如何在接口中定义和使用私有接口方法：

```csharp
csharp
复制代码public interface Calculator {
    // 公共的默认方法
    int add(int a, int b);

    default int subtract(int a, int b) {
        return add(a, negate(b));
    }

    default int multiply(int a, int b) {
        return a * b;
    }

    // 私有接口方法
    private int negate(int number) {
        return -number;
    }
}
```

在上面的示例中，`Calculator` 接口定义了三个默认方法 `add()`、`subtract()` 和 `multiply()`，以及一个私有接口方法 `negate()`。`subtract()` 方法内部调用了 `add()` 和 `negate()` 方法，而 `negate()` 方法是一个私有接口方法，只能在接口内部被调用。 通过私有接口方法，我们可以将 `negate()` 方法作为辅助方法来共享逻辑，并在多个默认方法中重复使用。 需要注意的是，私有接口方法不能被实现接口的类或其他类直接调用。它们仅用于在接口内部共享代码逻辑，提供更好的代码组织和代码重用。

## 2.7 该进的进程API

在 Java 9 中，对进程 API 进行了一些改进，以提供更好的控制和管理应用程序的进程。这些改进主要集中在 `java.lang.Process` 类和相关类的增强。 以下是 Java 9 中进程 API 的一些改进：

1. `ProcessHandle` 类：引入了一个新的 `ProcessHandle` 类，它代表一个本地操作系统进程的句柄。通过 `ProcessHandle` 类，您可以获取进程的 PID（进程标识符）、父进程的句柄、子进程的句柄以及其他进程相关的信息。
2. `ProcessHandle.Info` 类：`ProcessHandle` 类中的 `info()` 方法返回一个 `ProcessHandle.Info` 对象，它提供了有关进程的详细信息，如进程的命令行参数、启动时间、累计 CPU 时间等。
3. `ProcessBuilder` 类的改进：`ProcessBuilder` 类用于创建和启动新的进程。在 Java 9 中，`ProcessBuilder` 类添加了一些新的方法，例如 `inheritIO()` 方法，它允许子进程继承父进程的标准输入、输出和错误流。还添加了 `redirectInput()、redirectOutput() 和 redirectError()` 方法，用于重定向子进程的标准输入、输出和错误流。
4. `destroyForcibly()` 方法：`Process` 类中的 `destroyForcibly()` 方法用于强制终止进程，无论进程是否响应。这与 `destroy()` 方法的区别在于，`destroy()` 方法会尝试优雅地终止进程，而 `destroyForcibly()` 方法会强制终止进程。

这些改进使得在 Java 中管理和控制进程更加灵活和方便。您可以获取和操作正在运行的进程的信息，获取进程的 PID，以及更好地控制子进程的输入、输出和错误流。这些改进为进程管理和监控提供了更多的功能和选项。 请注意，Java 9 中的进程 API 的改进主要集中在进程管理方面，并没有引入类似于进程间通信的新功能。如果您需要进行进程间通信，可以使用其他 Java 平台提供的库，如 `java.nio.channels` 或第三方库。

```java
java
复制代码/**
 * @description:
 * @author: shu
 * @createDate: 2023/7/1 11:46
 * @version: 1.0
 */
public class ProcessInfoDemo {
    public static void main(String[] args) {
        // 获取当前进程的 ProcessHandle
        ProcessHandle currentProcess = ProcessHandle.current();

        // 获取当前进程的PID
        long pid = currentProcess.pid();
        System.out.println("当前进程的PID：" + pid);

        // 获取当前进程的信息
        ProcessHandle.Info processInfo = currentProcess.info();
        System.out.println("命令行：" + processInfo.command().orElse(""));
        System.out.println("启动时间：" + processInfo.startInstant().orElse(null));
        System.out.println("累计CPU时间：" + processInfo.totalCpuDuration().orElse(null));
    }

}
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5010ca4acbe1479a946537aa652ba3e1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

## 2.8 改进的Stream API

在Java 9中，Stream API得到了一些改进，以提供更多的操作和增强功能。下面是Java 9中改进的一些Stream API功能：

1. `takeWhile()` 和 `dropWhile()` 方法：引入了 `takeWhile()` 和 `dropWhile()` 两个新的Stream方法。`takeWhile()` 方法返回从流的开头开始的连续元素，直到遇到第一个不满足给定条件的元素。`dropWhile()` 方法返回从流的开头开始跳过满足给定条件的连续元素，直到遇到第一个不满足条件的元素。
2. `ofNullable()` 方法：Stream接口中新增了一个 `ofNullable()` 静态方法，它允许创建一个包含一个非空元素或空的Stream。如果提供的元素是非空的，将创建一个包含该元素的Stream；如果提供的元素为空，则创建一个空的Stream。
3. `iterate()` 方法的重载：`Stream.iterate()` 方法现在支持一个谓词（Predicate）作为第二个参数。这样，您可以定义在生成迭代元素时应该终止迭代的条件。
4. `Stream` 接口中的新方法：`Stream` 接口中添加了一些新的方法，如 `dropWhile()`、`takeWhile()`、`iterate()` 的重载版本，以及 `forEachOrdered()` 和 `toArray()` 方法的重载版本。

这些改进使得Stream API更加灵活和功能更强大。您可以使用新的方法来处理更多的流操作场景，例如根据条件获取部分元素、创建包含空元素的流等。 以下是一个示例，展示了Java 9中改进的Stream API的一些用法：

```java
java
复制代码import java.util.List;
import java.util.stream.Collectors;
import java.util.stream.Stream;

/**
 * @description:
 * @author: shu
 * @createDate: 2023/7/1 11:52
 * @version: 1.0
 */
public class StreamApiDemo {
    public static void main(String[] args) {
        // takeWhile() 方法示例
        List<Integer> numbers = Stream.of(1, 2, 3, 4, 5, 6)
                .takeWhile(n -> n < 4)
                .collect(Collectors.toList());
        System.out.println("takeWhile 示例：" + numbers); // 输出：[1, 2, 3]

        // dropWhile() 方法示例
        List<Integer> numbers2 = Stream.of(1, 2, 3, 4, 5, 6)
                .dropWhile(n -> n < 4)
                .collect(Collectors.toList());
        System.out.println("dropWhile 示例：" + numbers2); // 输出：[4, 5, 6]

        // ofNullable() 方法示例
        String name = null;
        List<String> names = Stream.ofNullable(name)
                .collect(Collectors.toList());
        System.out.println("ofNullable 示例：" + names); // 输出：[]

        // iterate() 方法的重载示例
        List<Integer> evenNumbers = Stream.iterate(0, n -> n < 10, n -> n + 2)
                .collect(Collectors.toList());
        System.out.println("iterate 重载示例：" + evenNumbers); // 输出：[0, 2, 4, 6, 8]

        //

        //Stream 接口中的新方法示例
        Stream.of("Java", "Python", "C++")
                .forEachOrdered(System.out::println); // 输出：Java Python C++
    }
}
```

这个示例展示了如何使用Java 9中改进的Stream API的一些方法，包括`takeWhile()`、`dropWhile()`、`ofNullable()`、`iterate()`和`forEachOrdered()`等。您可以运行这个示例并观察输出结果，以便更好地理解这些改进的Stream API功能。 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e7b3dbfc11204a3ca195c100feb02dd0~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

## 2.9 改进的 try-with-resources

在Java 7中引入了`try-with-resources`语句，用于在代码块结束时自动关闭实现`AutoCloseable`接口的资源。Java 9对`try-with-resources`进行了改进，使其更加便利和灵活。 Java 9中改进的`try-with-resources`语句具有以下特点：

1. 支持资源的匿名变量：在Java 9之前，`try-with-resources`语句中的资源必须是事先声明的变量。在Java 9中，可以在`try`关键字之后声明资源的匿名变量，并在`try`语句块中使用。
2. 资源的效率改进：在Java 9中，如果资源实现了`Closeable`接口，编译器会生成更高效的字节码，以减少关闭资源的开销。

下面是一个示例，演示了Java 9中改进的`try-with-resources`语句的用法：

```java
java
复制代码import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;

/**
 * @description:
 * @author: shu
 * @createDate: 2023/7/1 16:04
 * @version: 1.0
 */
public class TryWithResourcesDemo {
    public static void main(String[] args) {
        try (BufferedReader reader = new BufferedReader(new FileReader("example.txt"))) {
            String line;
            while ((line = reader.readLine()) != null) {
                System.out.println(line);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

在这个示例中，我们使用`try-with-resources`语句来自动关闭`BufferedReader`资源。在`try`关键字之后，我们创建了一个匿名的`BufferedReader`对象，并将其包装在`FileReader`中。在`try`语句块中，我们可以使用`reader`对象读取文件的内容。 当代码块结束时，无论是否发生异常，`reader`资源都会自动关闭。这样可以确保资源的正确释放，而无需显式调用`close()`方法。 需要注意的是，`try-with-resources`语句可以处理多个资源，只需将它们用分号分隔即可。所有的资源都会在代码块结束时自动关闭，按照资源的声明顺序逆序关闭。 通过使用Java 9中改进的`try-with-resources`语句，您可以更方便地处理资源的释放，并使代码更加简洁和易读。

## 2.10 改进的 @Deprecated 注解

在Java 9中，`@Deprecated`注解得到了一些改进，以提供更多的注释功能和精确的描述。以下是Java 9中改进的`@Deprecated`注解的特点：

1. `forRemoval`属性：`@Deprecated`注解新增了一个`forRemoval`属性，用于指示该元素是否被打算用于移除。设置`forRemoval=true`表示该元素将来会被移除，而设置`forRemoval=false`表示该元素被废弃但将保留。
2. `since`属性：`@Deprecated`注解中的`since`属性用于指定元素被废弃的版本。通过指定`since`属性，您可以明确说明从哪个版本开始该元素被废弃。
3. 更多注释说明：Java 9为`@Deprecated`注解添加了更多的注释说明，使得可以提供更详细和精确的描述，以便开发人员了解为什么该元素被废弃以及推荐使用的替代方法。

下面是一个示例，演示了Java 9中改进的`@Deprecated`注解的用法：

```typescript
typescript
复制代码public class DeprecatedExample {
    @Deprecated(since = "1.5", forRemoval = true)
    public void oldMethod() {
        // 旧的方法实现
    }

    @Deprecated(since = "2.0", forRemoval = false)
    public void deprecatedMethod() {
        // 废弃的方法实现
    }

    public static void main(String[] args) {
        DeprecatedExample example = new DeprecatedExample();

        // 调用旧的方法，会收到编译器警告
        example.oldMethod();

        // 调用废弃的方法，不会收到编译器警告
        example.deprecatedMethod();
    }
}
```

在这个示例中，我们定义了一个`DeprecatedExample`类，其中包含了两个方法：`oldMethod()`和`deprecatedMethod()`。我们使用改进的`@Deprecated`注解对这两个方法进行注释。 在`oldMethod()`方法上，我们设置了`@Deprecated`注解的`since`属性为"1.5"，`forRemoval`属性为`true`，表示该方法从版本1.5开始被废弃，并且将来会被移除。 在`deprecatedMethod()`方法上，我们设置了`@Deprecated`注解的`since`属性为"2.0"，`forRemoval`属性为`false`，表示该方法从版本2.0开始被废弃，但是会保留。 在`main()`方法中，我们实例化`DeprecatedExample`对象，并分别调用了旧的方法和废弃的方法。编译器会对调用旧的方法产生警告，而对调用废弃的方法不会产生警告。 通过使用Java 9中改进的`@Deprecated`注解，您可以提供更多的注释信息，包括指定废弃的版本和是否打算 移除，以便开发人员了解废弃元素的详细情况，并采取适当的行动。

## 2.11 钻石操作符

钻石操作符（Diamond Operator）是Java 7中引入的语法糖，用于在实例化泛型类时省略类型参数的重复声明。Java 9对钻石操作符进行了改进，使其在更多的情况下可以使用。 在Java 9之前，钻石操作符只能用于变量声明的右侧，并且只能省略类型参数的声明，不能省略类型参数的具体实例化。例如：

```arduino
arduino
复制代码List<String> list = new ArrayList<>(); // 使用钻石操作符，省略了类型参数的声明
```

在Java 9中，钻石操作符的适用范围得到了扩展。现在，钻石操作符不仅可以用于变量声明的右侧，还可以用于匿名类的实例化、显式的构造函数调用、显式的方法调用等更多的情况。例如：

```arduino
arduino
复制代码// 在匿名类的实例化中使用钻石操作符
Map<String, Integer> map = new HashMap<>() {
    // 匿名类的实现
};

// 在显式的构造函数调用中使用钻石操作符
MyClass obj = new MyClass<>();

// 在显式的方法调用中使用钻石操作符
myMethod(new ArrayList<>());
```

通过在实例化时使用钻石操作符，可以使代码更简洁、更易读。编译器会根据上下文推断出类型参数，并自动进行类型推断。 需要注意的是，钻石操作符只能用于具有泛型类型的类的实例化。对于原始类型或无类型参数的类，仍需要显式地声明类型参数。 总结来说，Java 9对钻石操作符进行了改进，使其可以在更多的情况下使用，包括变量声明、匿名类的实例化、显式的构造函数调用、显式的方法调用等。这使得代码更加简洁和易读，同时不影响类型安全性。

## 2.12 改进的 Optional 类

在Java 8中引入的Optional类提供了一种用于处理可能为null的值的封装。Java 9对Optional类进行了一些改进，以提供更多的实用方法和增强功能 下面是Java 9中改进的Optional类的特点：

1. `ifPresentOrElse()`方法：新增了`ifPresentOrElse()`方法，用于在Optional对象有值时执行一个操作，否则执行一个备选操作。这样可以避免使用传统的`if-else`语句来处理Optional对象的值。
2. `stream()`方法：Optional类中新增了`stream()`方法，用于将Optional对象转换为一个包含单个元素的Stream。如果Optional对象有值，则返回一个包含该值的Stream，否则返回一个空Stream。
3. `or()`方法的重载：`Optional.or()`方法现在支持Supplier函数接口，用于提供备选值。如果Optional对象为空，则使用Supplier提供的备选值。
4. `ifPresentOrElse()`方法的重载：`ifPresentOrElse()`方法现在支持两个Consumer函数接口，分别用于处理Optional对象有值时的情况和没有值时的情况。这样可以提供更多的灵活性和定制化的处理逻辑。

下面是一个示例，演示了Java 9中改进的Optional类的用法：

```typescript
typescript
复制代码import java.util.Optional;

/**
 * @description:
 * @author: shu
 * @createDate: 2023/7/1 16:04
 * @version: 1.0
 */
public class OptionalDemo {
    public static void main(String[] args) {
        Optional<String> optionalValue = Optional.of("Hello");

        // 使用ifPresentOrElse()方法执行操作
        optionalValue.ifPresentOrElse(
                value -> System.out.println("Value: " + value),
                () -> System.out.println("No value present")
        );

        // 使用stream()方法将Optional转换为Stream
        optionalValue.stream()
                .forEach(value -> System.out.println("Stream value: " + value));

        // 使用or()方法提供备选值
        Optional<String> emptyOptional = Optional.empty();
        String result = emptyOptional.or(() -> Optional.of("Default Value"))
                .orElse("Fallback Value");
        System.out.println("Result: " + result);
    }
}
```

在这个示例中，我们使用Optional类创建了一个包含值的Optional对象。然后，我们使用改进的方法对Optional对象进行操作。 使用`ifPresentOrElse()`方法，我们指定了一个Consumer函数来处理Optional对象有值时的情况，并指定一个备选操作来处理Optional对象为空的情况。 使用`stream()`方法，我们将Optional对象转换为一个包含单个元素的Stream，并对每个元素执行操作。 在`or()`方法示例中，我们创建了一个空的Optional对象，并使用Supplier函数提供了一个备选值。如果Optional对象为空，则使用备选值。 通过使用Java 9中改进的Optional类，我们可以更方便地处理Optional对象，提供更多的处理选项，并使代码更简洁和可读。这些改进使得Optional类更加实用和强大。

# 三 JDK10 新特性

参考网站：[openjdk.org/projects/jd…](https://link.juejin.cn/?target=https%3A%2F%2Fopenjdk.org%2Fprojects%2Fjdk%2F)

```makefile
makefile
复制代码286: Local-Variable Type Inference 局部变量类型推断
 
296: Consolidate the JDK Forest into a Single Repository JDK库的合并
 
304: Garbage-Collector Interface 统一的垃圾回收接口
 
307: Parallel Full GC for G1 为G1提供并行的Full GC
 
310: Application Class-Data Sharing 应用程序类数据（AppCDS）共享
 
312: Thread-Local Handshakes ThreadLocal握手交互
 
313: Remove the Native-Header Generation Tool (javah) 移除JDK中附带的javah工具
 
314: Additional Unicode Language-Tag Extensions 使用附加的Unicode语言标记扩展
 
316: Heap Allocation on Alternative Memory Devices 能将堆内存占用分配给用户指定的备用内存设备
 
317: Experimental Java-Based JIT Compiler 使用基于Java的JIT编译器
 
319: Root Certificates 根证书
 
322: Time-Based Release Versioning 基于时间的发布版本
```

## 3.1 局部变量类型推断

JDK 10 引入了局部变量类型推断，通过使用 `var` 关键字，可以让编译器根据上下文自动推断局部变量的类型。这个特性可以使代码更加简洁和易读。 下面是一个示例：

```ini
ini
复制代码var str = "Hello, World!"; // 推断为 String 类型
var list = new ArrayList<String>(); // 推断为 ArrayList<String> 类型
```

在上面的代码中，`var` 关键字用于声明局部变量 `str` 和 `list`，编译器根据右侧的表达式自动推断变量的类型为 `String` 和 `ArrayList<String>`。 需要注意的是，局部变量类型推断只能用于局部变量的声明和初始化，不能用于方法的参数、方法的返回类型、类的字段等。并且推断的类型是在编译时确定的，运行时变量的类型仍然是具体的类型，这个特性并不会影响 Java 的静态类型检查。 局部变量类型推断可以简化代码书写，特别是在使用泛型、匿名类和复杂类型时能够减少冗余的类型声明。然而，为了保持代码的可读性和清晰性，建议在使用 `var` 关键字时仍然给变量赋予有意义的名称。 请注意，这个特性是在 JDK 10 中引入的，如果你使用的是更早的 JDK 版本，将无法使用局部变量类型推断。

## 3.2 引入并行 Full GC算法

- G1 是设计来作为一种低延时的垃圾回收器。G1收集器还可以进行非常精确地对停顿进行控制。从JDK7开始启用G1垃圾回收器，在JDK9中G1成为默认垃圾回收策略。截止到ava 9，G1的Full GC采用的是单线程算法。也就是说G1在发生Full GC时会严重影响性能。
- JDK10又对G1进行了提升，G1 引入并行 Full GC算法，在发生Full GC时可以使用多个线程进行并行回收。能为用户提供更好的体验。

## 3.3 应用程序类数据共享

应用程序类数据共享（Application Class-Data Sharing）是 JDK 10 引入的一项特性，它旨在改善 Java 应用程序的启动性能和内存占用。 在传统的 Java 应用程序中，每次启动都需要加载和解析大量的类文件，这会消耗较多的时间和内存资源。应用程序类数据共享通过将类元数据和字节码预先计算和存储在共享的存档文件中，使得多个 Java 进程可以共享这些数据，从而减少了类加载和解析的时间和内存开销。 具体来说，应用程序类数据共享包括以下步骤：

1. 构建共享的类数据存档（Shared Class-Data Archive，CDS）：使用 `java -Xshare:dump` 命令构建共享的类数据存档。这个命令会启动应用程序，执行一些预热操作，然后生成一个包含类元数据和字节码的存档文件。
2. 启用应用程序类数据共享：使用 `java -Xshare:on` 命令启用应用程序类数据共享。在启用后，Java 进程将使用共享的类数据存档，从而减少类加载和解析的时间和内存占用。

通过应用程序类数据共享，可以显著缩短 Java 应用程序的启动时间和减少内存占用，尤其对于较大的应用程序或需要频繁启动的场景更为有效。 需要注意的是，应用程序类数据共享仅适用于具有相同类路径和相同类加载器结构的 Java 进程。因此，它主要适用于服务器端的 Java 应用程序，而对于客户端或桌面应用程序可能不太适用。 此外，应用程序类数据共享是 JDK 中的商业特性（Commercial Feature），只在 Oracle JDK 和 Oracle OpenJDK 中可用。在其他发行版或替代的 JDK 实现中，可能没有该特性或有所不同的实现方式。

## 3.4 线程本地握手

- Safepoint是Hotspot JVM中一种让应用程序所有线程停止的机制。为了要做一些非常之安全的事情，需要让所有线程都停下来它才好做。比如菜市场，人来人往，有人忽然要清点人数，这时候，最好就是大家都原地不动，这样也好统计。Safepoint起到的就是这样一个作用。
- JVM会设置一个全局的状态值。应用线程去观察这个值，一旦发现JVM把此值设置为了“大家都停下来”。此时每个应用线程就会得到这个通知，然后各自选择一个point（点）停了下来。这个点就叫Safepoint。待所有的应用线程都停下来了。
- JVM就开始做一些安全级别非常高的事情了，比如下面这些事情：垃圾清理暂停。类的热更新。偏向锁的取消。各种debug操作。
- 然而，让所有线程都到就近的safepoint停下来本是需要较长的时间。而且让所有线程都停下来显得有些粗暴。为此Java10就引入了一种可以不用stop all threads的方式，就是Thread Local Handshake（线程本地握手）。该新特性的效果线程本地握手是在 JVM 内部相当低级别的更改，修改安全点机制，允许在不运行全局虚拟机安全点的情况下实现线程回调，使得部分回调操作只需要停掉单个线程，而不是停止所有线程。

## 3.5 JDK库的合并

JDK 10 引入了一项名为 "JDK 库的合并"（Consolidate the JDK Forest into a Single Repository）的重要特性。在此之前，JDK 的源代码分布在多个 Mercurial 代码仓库中，而 JDK 10 将这些代码仓库合并为一个单一的 Git 代码仓库。 JDK 库的合并旨在简化 JDK 开发和维护过程，提高开发效率和代码管理的一致性。这项特性将 JDK 代码库从 Mercurial 迁移到 Git，并将所有相关代码合并到一个统一的 Git 仓库中，以便更方便地进行代码的版本控制、分支管理和协作开发。 通过将 JDK 代码库合并为一个 Git 仓库，开发者可以更轻松地浏览和查找 JDK 的源代码，同时更容易参与 JDK 的开发和贡献。此外，这也为社区提供了更便利的方式来提交错误报告、贡献补丁和参与 JDK 开发的讨论。 JDK 库的合并是 JDK 10 中一个重要的基础设施变更，对于 JDK 的开发者和维护者来说具有重要的影响。这项特性的引入标志着 JDK 开发过程中的一项重要改进，并为未来的 JDK 版本提供了更灵活和高效的开发基础。 请注意，以上信息基于 JDK 10 的版本。在更高版本的 JDK 中，可能会有一些变化和进一步的改进。建议查阅官方文档或相关资源以获取最新的信息和详细的说明。

## 3.6 实验型的垃圾回收接口

JDK 10 的主要特性之一是引入了一种实验性的垃圾回收器接口，称为 "GC 接口"（GC Interface）。该接口的目标是提供一种标准化的方式，使得开发者可以更方便地实现自定义的垃圾回收器。 GC 接口允许开发者基于 JDK 的垃圾回收框架构建自定义的垃圾回收器。通过实现 GC 接口，开发者可以插入自己的垃圾回收算法和策略，并与 JDK 的其他部分进行集成。 然而，需要注意的是，GC 接口在 JDK 10 中仍然是一个实验性的功能，并且不建议在生产环境中使用。这个接口可能在未来的 JDK 版本中进行改进或变化，或者可能被更稳定的替代方案所取代。

## 3.7 移除JDK中附带的javah工具

这个工具已经被 javac 中的高级功能所取代，这些功能是在 JDK 8(JDK-7150368)中添加的。此功能提供了在编译 Java 源代码时编写本机头文件的能力，从而消除了对单独工具的需求。

## 3.8 备⽤内存设备上的堆分配

Java 的堆内存通常是分配在主内存中的，并由 JVM 进行管理。备用内存设备（如 SSD、NVMe 等）通常用于存储持久化数据或作为辅助存储设备，而不是用于直接的堆内存分配。 然而，对于大型数据处理、高性能计算等特定场景，可以使用一些特殊的技术和工具，例如使用内存映射文件（Memory-mapped Files）将部分堆内存映射到备用存储设备上，以扩展可用的堆内存空间。这种方法可以提供更大的内存容量，但需要谨慎考虑性能和数据访问的开销。

## 3.9 基于 Java 的实验性 JIT 编译器

Java 10 引入了一个实验性的 JIT（Just-In-Time）编译器，称为 Graal 编译器。Graal 编译器是基于 Java 实现的，它提供了一种替代 HotSpot JIT 编译器的选择。 Graal 编译器具有以下一些特点和优势：

1. 高性能：Graal 编译器通过优化和即时编译技术提供了更好的性能表现，尤其在特定类型的工作负载上可能比 HotSpot JIT 编译器更快。
2. 低延迟：Graal 编译器的即时编译能力可以减少应用程序的停顿时间，从而提供更低的延迟和更高的响应性。
3. 兼容性：Graal 编译器与现有的 Java 代码和库兼容，并支持在现有的 JVM 环境中使用。

需要注意的是，尽管 Graal 编译器在性能和延迟方面可能带来优势，但它仍然是一个实验性的特性，并且在某些情况下可能与特定的代码或库不兼容。此外，Graal 编译器在编译速度和内存消耗方面可能会有一些权衡，具体取决于应用程序的特点和配置。 为了启用 Graal 编译器，您可以在 JDK 10+ 的启动参数中添加 `-XX:+UnlockExperimentalVMOptions -XX:+UseJVMCICompiler`。这将激活 Graal 编译器并使用它来编译 Java 代码。

## 3.10 根证书

JDK 10 中的根证书库与之前的版本类似，仍然包含在 JDK 安装目录下的 "jre/lib/security" 目录中的 "cacerts" 文件中。 根证书库（cacerts）中包含了一系列受信任的根证书，用于验证 SSL/TLS 连接和其他安全通信。这些根证书由各种受信任的证书颁发机构（CA）签发，包括常见的公共 CA 如 VeriSign、Thawte、DigiCert 等。 在 JDK 10 中，根证书库可能会有更新，以反映最新的根证书颁发机构和信任链。由于根证书库的安全性至关重要，因此 Oracle 公司会定期更新 JDK 发布版本中的根证书库，以确保其中包含最新的根证书。 可以使用 JDK 提供的 "keytool" 工具来执行与根证书库相关的操作，例如查看证书、添加新的根证书、删除根证书等。这可以用于管理 JDK 10 中的根证书库，并确保其与最新的证书颁发机构保持同步。 需要注意的是，根证书库的管理需要谨慎操作，确保只信任可靠和受信任的证书颁发机构，并避免操纵根证书库以防止安全风险。

# 四 JDK11 新特性

参考官网：[openjdk.org/projects/jd…](https://link.juejin.cn/?target=https%3A%2F%2Fopenjdk.org%2Fprojects%2Fjdk%2F11%2F)

```yaml
yaml
复制代码181: Nest-Based Access Control
181: 基于嵌套的访问控制
309: Dynamic Class-File Constants
309: 动态类文件常数
315: Improve Aarch64 Intrinsics
315: 改进 Aarch64内部结构
318: Epsilon: A No-Op Garbage Collector
318: Epsilon: 一个不可操作的垃圾收集器
320: Remove the Java EE and CORBA Modules
320: 删除 JavaEE 和 CORBA 模块
321: HTTP Client (Standard)
321: HTTP 客户端(标准)
323: Local-Variable Syntax for Lambda Parameters
323: Lambda 参数的局部变量语法
324: Key Agreement with Curve25519 and Curve448
324: 与 Curve25519和 Curve448的关键协议
327: Unicode 10
328: Flight Recorder
328: 飞行记录器
329: ChaCha20 and Poly1305 Cryptographic Algorithms
329: ChaCha20和 Poly1305密码算法
330: Launch Single-File Source-Code Programs
330: 启动单文件源代码程序
331: Low-Overhead Heap Profiling
331: 低开销堆分析
332: Transport Layer Security (TLS) 1.3
332: 传输层安全(TLS)1.3
333: ZGC: A Scalable Low-Latency Garbage Collector
   (Experimental)
333: ZGC: 一个可伸缩的低延迟垃圾收集器(实验)
335: Deprecate the Nashorn JavaScript Engine
335: 废弃 Nashorn JavaScript 引擎
336:
Deprecate the Pack200 Tools and API废弃 Pack200工具和 API
```

## 4.1 Lambda 参数的局部变量语法

在JDK 11及更高版本中，Lambda表达式的参数可以使用var关键字来声明局部变量。使用var关键字可以让编译器根据上下文推断参数的类型。以下是Lambda参数的局部变量语法示例：

```csharp
csharp
复制代码interface MyInterface {
    void myMethod(String name, int age);
}

public class Main {
    public static void main(String[] args) {
        MyInterface myLambda = (var name, var age) -> {
            System.out.println("Name: " + name);
            System.out.println("Age: " + age);
        };
        
        myLambda.myMethod("John", 25);
    }
}
```

在上述示例中，我们使用Lambda表达式实现了`MyInterface`接口的`myMethod`方法。Lambda表达式的参数使用`var`关键字声明为局部变量。编译器会根据上下文推断参数的类型。 请注意，Lambda参数的类型推断只适用于局部变量，而不适用于方法的参数类型、返回类型或字段类型。

## 4.2 HTTP 客户端（标准）

在JDK 11中，引入了标准的HTTP客户端API，用于发送HTTP请求和处理响应。这个API提供了一种原生的方式来进行HTTP通信，不再需要使用第三方库。以下是一个简单的示例：

```java
java
复制代码import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.concurrent.CompletableFuture;

public class HttpClientExample {
    public static void main(String[] args) throws Exception {
        // 创建HTTP客户端
        HttpClient httpClient = HttpClient.newHttpClient();

        // 创建HTTP请求
        HttpRequest httpRequest = HttpRequest.newBuilder()
                .uri(URI.create("https://api.example.com/data"))
                .build();

        // 发送HTTP请求并异步获取响应
        CompletableFuture<HttpResponse<String>> future = httpClient.sendAsync(httpRequest, HttpResponse.BodyHandlers.ofString());

        // 处理响应
        future.thenAccept(response -> {
            int statusCode = response.statusCode();
            String responseBody = response.body();
            System.out.println("Status code: " + statusCode);
            System.out.println("Response body: " + responseBody);
        });

        // 等待异步请求完成
        future.join();
    }
}
```

在上述示例中，我们首先创建一个`HttpClient`对象，然后构建一个`HttpRequest`对象，指定请求的URI。接下来，使用`sendAsync`方法发送异步请求，并指定响应的处理方式（这里使用`HttpResponse.BodyHandlers.ofString()`将响应体解析为字符串）。 通过`CompletableFuture`的`thenAccept`方法，我们可以处理异步请求完成后的响应。在回调函数中，我们获取响应的状态码和响应体，并进行相应的处理。 最后，我们使用`future.join()`等待异步请求完成。注意，此处的异步请求是非阻塞的，可以继续执行其他操作。

## 4.3 新的 Collection.toArray()

在JDK 11中，`Collection`接口新增了一个重载的`toArray`方法，用于将集合转换为数组。该方法的签名如下：

```scss
scss
复制代码default <T> T[] toArray(IntFunction<T[]> generator)
```

这个方法接受一个`IntFunction<T[]>`类型的参数，该参数用于提供一个生成指定类型数组的函数。函数的输入参数是集合的大小，输出是一个新的数组。 下面是一个示例代码，演示如何使用JDK 11中的`Collection.toArray`方法：

```typescript
typescript
复制代码import java.util.ArrayList;
import java.util.List;

public class CollectionToArrayExample {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        list.add("Java");
        list.add("Python");
        list.add("C++");

        // 使用Collection.toArray方法转换为数组
        String[] array = list.toArray(String[]::new);

        // 打印数组元素
        for (String element : array) {
            System.out.println(element);
        }
    }
}
```

在上述示例中，我们创建了一个`ArrayList`对象并向其中添加一些元素。然后，我们使用`toArray`方法将`ArrayList`转换为`String`类型的数组。注意到，我们使用了`String[]::new`作为`IntFunction`参数，这样就会生成一个与集合大小相同的新数组。 最终，我们遍历数组并打印每个元素，这个新的`toArray`方法简化了集合到数组的转换过程，并且避免了类型转换的麻烦。

## 4.4 新的字符串方法

JDK 11引入了一些新的字符串方法，以提供更方便和强大的字符串操作功能。以下是一些JDK 11中新增的字符串方法：

1. `**String.isBlank()**`：该方法用于检查字符串是否为空白字符串。它返回一个布尔值，指示字符串是否为空白。如果字符串是空白字符串（仅由空格、制表符、换行符等字符组成），则返回`true`；否则返回`false`。
2. `**String.strip()**`：该方法用于去除字符串的首尾空白字符。它返回一个新的字符串，其中去除了原始字符串的首尾空白字符。与`trim()`方法不同的是，`strip()`方法可以正确处理Unicode空白字符。
3. `**String.stripLeading()**`**和**`**String.stripTrailing()**`：这两个方法分别用于去除字符串的前导空白字符和尾随空白字符。它们返回一个新的字符串，其中去除了原始字符串的前导或尾随空白字符。
4. `**String.lines()**`：该方法返回一个包含字符串中所有行的`Stream`。它根据行终止符将字符串拆分成多个行，并返回一个`Stream`，每个元素代表一行。
5. `**String.repeat(int count)**`：该方法用于重复字符串指定次数，并返回一个新的字符串。它接受一个整数参数`count`，指定要重复的次数。

这些方法都是在`java.lang.String`类中新增的，可以直接在字符串对象上调用。它们提供了更直观和方便的字符串操作，简化了对字符串的处理和转换。请注意，这些方法在JDK 11及更高版本中可用。

## 4.5 新的文件方法

JDK 11引入了一些新的文件方法，以提供更便捷和强大的文件操作功能。以下是一些JDK 11中新增的文件方法：

1. `**Files.writeString()**`：该方法用于将字符串写入文件。它接受一个文件路径和要写入的字符串，可以指定编码格式和可选的文件选项。如果文件不存在，则创建新文件；如果文件已存在，则覆盖原有内容。
2. `**Files.readString()**`：该方法用于读取文件内容并以字符串形式返回。它接受一个文件路径，可以指定编码格式和可选的文件选项。
3. `**Files.writeStringList()**`：该方法用于将字符串列表逐行写入文件。它接受一个文件路径和字符串列表，可以指定编码格式和可选的文件选项。
4. `**Files.readStringList()**`：该方法用于逐行读取文件内容，并以字符串列表的形式返回。它接受一个文件路径，可以指定编码格式和可选的文件选项。
5. `**Files.mismatch()**`：该方法用于比较两个文件的内容。它接受两个文件路径，并返回第一个不匹配的字节的位置。如果文件完全相同，则返回-1。

这些方法都是在`java.nio.file.Files`类中新增的，用于处理文件的读写和比较操作。它们提供了更便捷和高效的方式来操作文件内容。 以下是一个示例代码，演示如何使用JDK 11的新文件方法读取和写入文件：

```java
java
复制代码import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardOpenOption;
import java.util.List;

public class FileOperations {
    public static void main(String[] args) throws Exception {
        String filePath = "example.txt";

        // 写入文件
        String content = "Hello, world!";
        Files.writeString(Path.of(filePath), content);

        // 读取文件
        String fileContent = Files.readString(Path.of(filePath));
        System.out.println("文件内容：\n" + fileContent);

        // 逐行写入文件
        List<String> lines = List.of("Line 1", "Line 2", "Line 3");
        Files.writeStringList(Path.of(filePath), lines, StandardOpenOption.APPEND);

        // 逐行读取文件
        List<String> fileLines = Files.readStringList(Path.of(filePath));
        System.out.println("文件行数：" + fileLines.size());
        System.out.println("文件内容：");
        for (String line : fileLines) {
            System.out.println(line);
        }
    }
}
```

在上述示例中，我们首先使用`Files.writeString`方法将字符串写入文件。然后使用`Files.readString`方法读取文件内容，并打印到控制台。 接下来，我们使用`Files.writeStringList`方法逐行写入字符串列表到文件。然后使用`Files.readStringList`方法逐行读取文件内容，并打印到控制台。

## 4.6 一个无操作垃圾收集器

JDK 11引入了一个名为"ZGC"（Z Garbage Collector）的新的垃圾收集器，它被设计为一种无操作垃圾收集器。这意味着它在大部分情况下几乎不会对应用程序的执行造成明显的停顿 ZGC是一种低延迟的垃圾收集器，旨在实现非常短暂的停顿时间。它的目标是保持最大15毫秒的停顿时间，并限制不超过10%的吞吐量损失。这使得ZGC适合那些对低延迟和高吞吐量要求都很高的应用程序。 ZGC使用了一些创新的技术来实现其目标。它使用了一种称为"Region"的内存布局，将堆内存划分为一系列大小固定的区域，使得垃圾收集可以在不停止整个应用程序的情况下并发进行。此外，ZGC还使用了写屏障技术来跟踪对象的引用变化，并在后台处理未访问的对象。 需要注意的是，ZGC在JDK 11中被标记为实验性特性，并且默认情况下并不启用。要使用ZGC收集器，需要通过JVM参数显式地启用。可以使用以下参数启用ZGC收集器：

```ruby
ruby
复制代码-XX:+UseZGC
```

使用ZGC收集器的示例如下：

```typescript
typescript
复制代码public class ZGCExample {
    public static void main(String[] args) {
        // 设置ZGC作为垃圾收集器
        System.setProperty("java.vm.useZGC", "true");

        // 应用程序代码...
    }
}
```

需要注意的是，ZGC仅适用于支持64位操作系统和64位JVM的平台。并且在某些情况下，它可能与一些JVM选项不兼容，例如启用了某些特定的实验性特性。

## 4.7 启动单文件源代码程序

JDK 11引入了一项新功能，允许直接启动单个源代码文件而无需先将其编译为字节码文件。这个功能称为"单文件源代码程序"（Single-File Source-Code Programs）。 要在JDK 11中启动单文件源代码程序，可以使用以下命令：

```xml
xml
复制代码java <options> <source-file>.java
```

其中，`<options>`是可选的JVM选项，`<source-file>.java`是要执行的源代码文件。 以下是一个示例，演示如何使用JDK 11启动单文件源代码程序：

```typescript
typescript
复制代码public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```

保存上述代码为`HelloWorld.java`文件。 然后，通过以下命令运行该源代码文件：

```
复制代码java HelloWorld.java
```

注意，在执行单文件源代码程序时，JDK 11会在内部将源代码文件编译为字节码，并在运行时执行。这使得我们能够更方便地运行和测试简单的Java程序，而无需事先编译为.class文件。 需要注意的是，单文件源代码程序主要适用于简单的小型程序或测试目的。对于复杂的项目，仍然建议将源代码编译为字节码文件，并使用传统的方式执行。

## 4.8 基于嵌套的访问控制

JDK 11引入了一项新的语言特性，即基于嵌套的访问控制（Nested Access Control）。这个特性旨在提供更细粒度的访问控制机制，以保护类的内部结构，并更好地支持模块化开发。 在传统的Java访问控制中，类的成员（字段、方法、内部类等）可以被声明为`public`、`protected`、`private`或默认（不写访问修饰符）。这些修饰符控制着类的成员在不同上下文中的可见性。 而基于嵌套的访问控制引入了两个新的访问修饰符：`private`和`protected`的嵌套形式。这些修饰符用于限制对嵌套类的访问，使得只有特定的外部类可以访问其嵌套类的成员。 具体而言，以下是基于嵌套的访问控制修饰符的规则：

1. `**private**`**的嵌套形式**（private nested）：内部类声明为`private`的嵌套形式时，只有外部类可以访问该嵌套类。
2. `**protected**`**的嵌套形式**（protected nested）：内部类声明为`protected`的嵌套形式时，只有外部类及其子类可以访问该嵌套类。

这些新的访问修饰符对于模块化开发非常有用，可以更好地封装类的内部结构，避免不必要的访问和依赖关系。 以下是一个示例，展示了基于嵌套的访问控制的使用：

```java
java
复制代码public class OuterClass {
    private int privateField;

    private static class PrivateNestedClass {
        private void privateMethod() {
            OuterClass outer = new OuterClass();
            outer.privateField = 10;
        }
    }

    protected static class ProtectedNestedClass {
        protected void protectedMethod() {
            OuterClass outer = new OuterClass();
            outer.privateField = 20;
        }
    }

    public static void main(String[] args) {
        PrivateNestedClass privateNested = new PrivateNestedClass();
        privateNested.privateMethod();

        ProtectedNestedClass protectedNested = new ProtectedNestedClass();
        protectedNested.protectedMethod();
    }
}
```

在上述示例中，`PrivateNestedClass`是一个私有的嵌套类，只有`OuterClass`内部可以访问它。`ProtectedNestedClass`是一个受保护的嵌套类，只有`OuterClass`及其子类可以访问它。 在`main`方法中，我们可以实例化并调用这些嵌套类的方法，因为它们的访问限制在其声明的上下文中是有效的。 需要注意的是，基于嵌套的访问控制修饰符只适用于嵌套类，而不适用于顶级类（即没有外部类的类）。此外，仍然可以使用传统的访问修饰符（`public`、`protected`、`private`和默认）来控制顶级类的访问性。

## 4.9 Java 飞行记录器

JDK 11引入了一个名为"Java飞行记录器"（Java Flight Recorder，JFR）的功能，它是一个事件记录和分析引擎，用于在运行时收集和分析Java应用程序的运行数据。 Java飞行记录器可以捕获各种事件，包括方法调用、垃圾收集、线程活动、I/O操作等。它还可以收集各种度量指标，如CPU使用率、内存使用量、线程数量等。这些事件和度量指标可以用于分析和优化应用程序的性能、诊断问题和进行故障排除。 使用Java飞行记录器需要以下步骤：

1. **启用Java飞行记录器**：要使用Java飞行记录器，首先需要在JVM参数中启用它。可以使用以下参数启用JFR：

```ruby
ruby
复制代码-XX:+UnlockCommercialFeatures -XX:+FlightRecorder
```

注意，Java飞行记录器属于商业特性，在某些Java发行版中可能需要适当的许可证。

1. **配置和启动飞行记录器**：一旦启用了Java飞行记录器，你可以使用命令行工具（如`jcmd`、`jfr`）或编程方式来配置和启动飞行记录器。你可以指定要记录的事件类型、持续时间等。
2. **收集和分析记录数据**：在应用程序运行期间，Java飞行记录器将会收集指定的事件和度量指标。收集的数据可以保存到文件中。然后，你可以使用Java Mission Control（JMC）或其他工具来加载和分析这些记录文件，以获得有关应用程序性能和行为的详细信息。

Java飞行记录器提供了强大的性能分析和故障排除能力，能够帮助开发人员识别和解决应用程序中的性能问题和异常情况。它是JDK 11中一个重要的调试和优化工具。

## 4.10 低开销的堆分析

在JDK 11中，引入了一项名为"低开销的堆分析"（Low Overhead Heap Profiling）的功能，它允许在应用程序运行时收集堆分析数据，以便更好地理解和调试内存使用情况。 传统的堆分析工具通常会对应用程序的内存进行全面的快照，以获取详细的对象分配和引用关系信息。然而，这种全面的堆快照收集过程可能会对应用程序的性能产生较大的开销，特别是在大型应用程序中。 低开销的堆分析通过减少采样频率和记录粒度，以及使用一些技术手段来减少开销，从而提供了一种更轻量级的堆分析方法。它可以在应用程序运行时进行堆分析，对内存使用情况进行采样，并生成堆分析报告。 要使用低开销的堆分析功能，可以使用以下JVM参数：

```ruby
ruby
复制代码-XX:StartFlightRecording=heap=low
```

这将启用低开销的堆分析，并将堆分析数据记录到默认的JFR文件中。 然后，可以使用Java Mission Control（JMC）或其他JFR分析工具加载和分析生成的JFR文件，以获得有关应用程序内存使用的详细信息。JFR分析工具提供了堆分配热点、对象分布、对象生命周期等信息，帮助开发人员识别内存泄漏、过度分配和其他内存相关问题。 需要注意的是，低开销的堆分析是一种近似的分析方法，它可能会在某些情况下丢失一些细节信息。因此，在某些情况下，仍然建议使用传统的全面堆分析方法来获取更准确的堆快照。

## 4.11 从 Oracle JDK 中移除 JavaFX

从 Java 11 开始，JavaFX（以及相关的 javapackager 工具）不再随 JDK 一起提供。相反，我们可以从 JavaFX 主页将其作为单独的 SDK 下载。

## 4.12 删除模块

JEP 320 从 JDK 中删除了以下模块：

- java.xml.ws (JAX-WS)
- java.xml.bind (JAXB)
- java.activation (JAF)
- java.xml.ws.annotation（通用注解）
- java.corba (CORBA)
- java.transaction (JTA)
- java.se.ee（前面提到的六个模块的聚合器模块）
- jdk.xml.ws（JAX-WS 工具）
- jdk.xml.bind（JAXB 工具）

其他请参考官方文档

# 五 JDK12 新特性

参考：[openjdk.org/projects/jd…](https://link.juejin.cn/?target=https%3A%2F%2Fopenjdk.org%2Fprojects%2Fjdk%2F12%2F)

```makefile
makefile
复制代码
189:
Shenandoah: A Low-Pause-Time Garbage Collector (Experimental)Shenandoah: 低暂停时间垃圾收集器(实验)
230:
Microbenchmark Suite微基准测试套件
325:
Switch Expressions (Preview)开关表达式(预览)
334:
JVM Constants APIJVM 常量 API
340:
One AArch64 Port, Not Two一个 AArch64端口，不是两个
341:
Default CDS Archives默认的 CDS 存档
344:
Abortable Mixed Collections for G1可流产的 G1混合集合
346:
Promptly Return Unused Committed Memory from G1从 G1及时返回未使用的提交内存
```

## 5.1 Switch表达式（Preview）

JDK 12引入了一个名为"Switch表达式（Preview）"的功能，它对Switch语句进行了改进，使其更加灵活和易用。Switch表达式允许在Switch语句中使用Lambda风格的语法进行模式匹配，并直接返回值。 在传统的Switch语句中，每个case分支都需要使用`break`语句来防止掉落到下一个分支。而在Switch表达式中，我们可以使用箭头(`->`)来直接返回值，而不需要使用`break`语句。 以下是Switch表达式的示例：

```arduino
arduino
复制代码int day = 3;
String dayName = switch (day) {
    case 1, 2, 3, 4, 5 -> "Weekday";
    case 6, 7 -> "Weekend";
    default -> "Invalid day";
};
```

在上述代码中，Switch表达式根据`day`的值进行模式匹配，然后返回相应的字符串。在case分支中，可以使用逗号将多个值组合在一起，表示它们共享相同的处理逻辑。在最后的default分支中，如果没有匹配的情况，将返回"Invalid day"。 Switch表达式的语法比传统的Switch语句更简洁和易读。它可以减少重复的代码和易错的`break`语句，使得代码更加清晰和紧凑。 需要注意的是，Switch表达式在JDK 12中是作为预览功能引入的，意味着它是一项实验性的功能，可能会在后续的JDK版本中进行调整和改进。要使用Switch表达式，需要确保在编译器选项中启用了预览功能，例如使用`--enable-preview`选项。

## 5.2 微基准测试套件

JDK 12引入了Microbenchmark Suite（JEP 230），它是一个用于编写和运行微基准测试的套件。微基准测试是一种专门用于测量小块代码片段的性能的测试方法。 Microbenchmark Suite提供了一组工具和库，帮助开发人员编写和执行微基准测试，并提供准确的性能测量结果。以下是Microbenchmark Suite的一些关键特点：

1. **@Benchmark注解**：Microbenchmark Suite通过`@Benchmark`注解标记要进行性能测试的方法。这样，开发人员可以将注解添加到他们想要测试的方法上，以指示该方法是一个微基准测试。
2. **Blackhole类**：Microbenchmark Suite提供了`Blackhole`类，用于消耗被测试方法的结果，以防止JVM进行优化和消除未使用的代码。这样可以更准确地测量方法的性能。
3. **测试运行器**：Microbenchmark Suite提供了一个测试运行器，用于执行微基准测试，并生成关于每个测试的性能测量结果。测试运行器可以配置以控制测试的参数和执行方式。

通过使用Microbenchmark Suite，开发人员可以编写和运行精确的微基准测试，以测量特定代码片段的性能，并进行性能优化。微基准测试可以帮助开发人员识别潜在的性能问题、比较不同实现的性能差异，并确定性能优化的方向。 需要注意的是，微基准测试需要谨慎使用，因为性能测试结果可能会受到多种因素的影响，如硬件环境、垃圾收集器、JVM参数等。因此，在进行微基准测试时，需要了解其原理、正确使用工具和库，并进行合理的结果解释和分析。

## 5.3 JVM 常量 API

在 JDK 12 中，Java 虚拟机（JVM）引入了一些常量 API，以便在运行时获取 JVM 的一些常量信息。这些常量 API 主要位于 `java.lang.constant` 包中，提供了一种安全和类型安全的方式来访问这些常量。 以下是 JDK 12 中一些常量 API 的主要类和接口：

1. `java.lang.constant.Constable`：这是一个标记接口，表示一个常量值可以在编译时被抽象为常量。这个接口有助于在编译时执行常量折叠和内联优化。
2. `java.lang.constant.ConstantDesc`：这是一个通用的常量描述符接口，用于表示各种类型的常量。它是所有常量类型的父接口。
3. `java.lang.constant.ClassDesc`：这是一个常量描述符接口，表示一个类类型的常量。它提供了获取类名、包名等信息的方法。
4. `java.lang.constant.MethodTypeDesc`：这是一个常量描述符接口，表示方法类型的常量。它提供了获取方法参数类型和返回类型的方法。
5. `java.lang.constant.DynamicCallSiteDesc`：这是一个常量描述符接口，表示一个动态调用点（Dynamic Call Site）的常量。它提供了获取调用点信息的方法。
6. `java.lang.constant.DirectMethodHandleDesc`：这是一个常量描述符接口，表示一个直接方法句柄（Direct Method Handle）的常量。它提供了获取方法句柄信息的方法。
7. `java.lang.constant.ConstableDesc`：这是一个常量描述符接口，表示一个常量值的常量。它提供了获取常量值的方法。

除了上述接口，还有一些其他类和接口，用于支持不同类型的常量，如字段常量、模块常量等。 这些常量 API 的引入使得开发人员可以更方便地在运行时访问和处理 JVM 中的常量信息，从而实现更高效和更安全的代码。

## 5.4 默认 [CDS](https://link.juejin.cn/?target=https%3A%2F%2Fso.csdn.net%2Fso%2Fsearch%3Fq%3DCDS%26spm%3D1001.2101.3001.7020) 档案

JDK 12 引入了一个新功能，称为默认 CDS（Class Data Sharing）档案。CDS 是一项技术，它允许在 JVM 启动时将类的元数据和字节码预先加载到共享存档文件中，以提高应用程序的启动时间和内存占用。 在 JDK 12 中，默认 CDS 提供了一种简化的方法来创建和使用 CDS 档案。它允许您使用默认的配置文件来自动创建 CDS 档案，而无需手动指定类列表。 以下是使用默认 CDS 档案的步骤：

1. 运行应用程序时，添加 `-XX:DumpLoadedClassList=<classlist.txt>` 参数。这将在应用程序退出时生成一个类列表文件 `classlist.txt`，其中包含了应用程序加载的所有类。
2. 使用 `jlink` 命令创建一个包含默认 CDS 档案的自定义运行时映像。添加 `--class-list=<classlist.txt>` 参数，其中 `<classlist.txt>` 是第一步生成的类列表文件的路径。
3. 在启动应用程序时，使用 `java` 命令指定 `-Xshare:dump` 参数来生成默认的 CDS 档案。这将根据默认的配置文件（`<JDK>/lib/classlist`）自动创建 CDS 档案。
4. 在后续启动应用程序时，使用 `-Xshare:on` 参数来启用默认 CDS 档案。这将使用预先生成的 CDS 档案来加速应用程序的启动。

需要注意的是，默认 CDS 档案功能在不同的平台和 JDK 发行版中可能有所差异。在一些平台上，CDS 功能可能需要特定的许可证。此外，CDS 档案的创建和使用可能会因应用程序的特性而有所不同，因此建议参考相关的 JDK 文档和文档以获取详细信息和指导。

## 5.5 G1 的可流动混合收集

G1的目标之一是满足用户提供的暂停时间目标以暂停其收集暂停。G1使用高级分析引擎来选择在集合期间 要完成的工作量（这部分基于应用程序行为）。此选择的结果是一组称为集合集的区域。一旦确定了集合集并且 已经开始集合，则G1必须在不停止的情况下收集集合集的所有区域中的所有活动对象。如果启发式选择过大的收 集，则此行为可导致G1超过暂停时间目标，例如，如果应用程序的行为发生变化，以致启发式工作在“陈旧”数据 上，则可能发生这种情况。特别是在混合集合期间可以观察到这种情况，其中集合集通常可以包含太多旧区域。 需要一种机制来检测启发式方法何时反复为集合选择错误的工作量，如果是，则让G1逐步递增地执行收集工作， 其中集合可以在每个步骤之后中止。这种机制将允许G1更频繁地满足暂停时间目标。 其他详细信息请参考官网

# 六 JDK13 新特性

参考官网：[openjdk.org/projects/jd…](https://link.juejin.cn/?target=https%3A%2F%2Fopenjdk.org%2Fprojects%2Fjdk%2F13%2F)

```makefile
makefile
复制代码
350:
Dynamic CDS Archives动态 CDS 存档
351:
ZGC: Uncommit Unused Memory取消未使用内存
353:
Reimplement the Legacy Socket API重新实现遗留套接字 API
354:
Switch Expressions (Preview)开关表达式(预览)
355:
Text Blocks (Preview)文本块(预览)
```

## 6.1 动态CDS归档（Dynamic CDS Archiving）

在JDK 13中，引入了一项名为动态CDS归档（Dynamic CDS Archiving）的新功能。CDS（Class Data Sharing）是一种技术，它允许将类的元数据和字节码预先加载到共享归档文件中，以提高应用程序的启动时间和内存占用。动态CDS归档进一步扩展了CDS的能力，使得在应用程序运行时可以动态地创建和更新CDS归档。 动态CDS归档的主要优势在于，它允许在应用程序运行时收集和记录正在使用的类和库，并将它们添加到已存在的CDS归档中，从而实现动态的类共享。这样，在下次启动应用程序时，可以使用包含动态更新的CDS归档，进一步加速应用程序的启动。 以下是使用动态CDS归档的一般步骤：

1. 使用JDK 13构建应用程序，并确保启用CDS。可以通过使用以下参数来启用CDS：`-XX:ArchiveClassesAtExit=<archive-file>`。
2. 运行应用程序，让它进行正常操作。这将导致动态CDS归档文件被创建。
3. 在下次启动应用程序时，使用以下参数来启用CDS并指定先前创建的动态CDS归档文件：`-XX:SharedArchiveFile=<archive-file>`。

通过这种方式，应用程序可以利用先前收集的动态CDS归档文件，从而加快启动时间和减少内存消耗。 需要注意的是，动态CDS归档功能在JDK 13中是作为实验性功能引入的。因此，在使用时应谨慎评估其在特定应用程序和环境中的效果，并确保遵循官方文档和最佳实践。

## 6.2 重新实现遗留套接字 API

java.net.Socket和java.net.ServerSocket API将所有套接字操作委托给java.net.SocketImpl，这是自JDK 1.0起已经存在的服务提供程序接口（SPI）机制。内置的实现称为“普通”实现，由非公开的PlainSocketImpl通过支持类SocketInputStream和SocketOutputStream实施。 PlainSocketImpl由其他两个JDK内部实现扩展，这些实现支持通过SOCKS和HTTP代理服务器的连接。默认情况下，使用基于SOCKS的SocketImpl创建Socket和ServerSocket（有时是延迟的）。在ServerSocket的情况下，使用SOCKS实现是一个古怪的事情，可以追溯到对JDK 1.4中的代理服务器连接的实验性（并且自从删除以来）支持。 新的实现NioSocketImpl替代了PlainSocketImpl。它被开发为易于维护和调试。它与新I / O（NIO）实现共享相同的JDK内部基础结构，因此不需要自己的本机代码。它与现有的缓冲区高速缓存机制集成在一起，因此不需要将线程堆栈用于I / O。它使用java.util.concurrent锁而不是同步方法，以便将来可以在fibers上很好地使用。在JDK 11中，大多数NIO SocketChannel和其他SelectableChannel实现都是在实现相同目标的情况下重新实现的。

## 6.3 Switch表达式

在JDK 13中，Switch表达式引入了一些新的语法和功能，使得在Switch语句中编写更简洁和灵活的代码成为可能。下面是一些JDK 13中Switch表达式的示例：

1. 简单的Switch表达式：

```arduino
arduino
复制代码int day = 3;
String dayName = switch (day) {
    case 1 -> "Monday";
    case 2 -> "Tuesday";
    case 3 -> "Wednesday";
    case 4 -> "Thursday";
    case 5 -> "Friday";
    default -> "Invalid day";
};

System.out.println(dayName);  // 输出: Wednesday
```

在这个例子中，我们使用Switch表达式根据给定的`day`值返回对应的星期几名称。

1. 表达式和语句的组合：

```csharp
csharp
复制代码int day = 5;
String dayType = switch (day) {
    case 1, 2, 3, 4, 5 -> {
        yield "Weekday";  // 使用yield返回一个值
    }
    case 6, 7 -> {
        System.out.println("It's a weekend!");  // 执行语句
        yield "Weekend";
    }
    default -> {
        yield "Invalid day";
    }
};

System.out.println(dayType);  // 输出: Weekend
```

在这个例子中，我们根据给定的`day`值返回对应的日期类型。如果是工作日（1-5），我们使用`yield`返回一个字符串值。如果是周末（6-7），我们首先打印一条消息，然后返回一个字符串值。

1. 表达式的返回类型推断：

```arduino
arduino
复制代码String dayName = switch (getDayOfWeek()) {
    case 1 -> "Monday";
    case 2 -> "Tuesday";
    case 3 -> "Wednesday";
    case 4 -> "Thursday";
    case 5 -> "Friday";
    default -> "Invalid day";
};

System.out.println(dayName);
```

在这个例子中，`getDayOfWeek()`方法返回一个整数表示星期几。在Switch表达式中，我们根据这个返回值进行匹配，并返回对应的星期几名称。注意，我们没有显式地指定Switch表达式的返回类型，而是使用类型推断，编译器会自动推断返回类型为`String`。 这些示例展示了JDK 13中Switch表达式的一些用法。Switch表达式的引入使得编写简洁、灵活的条件分支逻辑变得更加容易。

## 6.4 文本块

在JDK 13中，引入了文本块（Text Blocks）功能，它提供了一种更直观和易读的方式来定义多行字符串。文本块使得在Java代码中编写包含换行符和缩进的长字符串变得更加方便。下面是一些JDK 13中文本块的示例：

1. 基本的文本块：

```ini
ini
复制代码String htmlContent = """
    <html>
        <body>
            <h1>Hello, JDK 13!</h1>
        </body>
    </html>
""";
```

在这个例子中，我们使用文本块定义了一个包含HTML标记的字符串。文本块使用三个双引号（"""）作为起始和结束标记，使得可以在字符串中包含多行内容，并且保留了换行符和缩进。

1. 转义字符的处理：

```ini
ini
复制代码String message = """
    This is a multiline string \
    with line continuation.
""";
```

在这个例子中，我们使用文本块定义了一个包含转义字符的多行字符串。在文本块中，使用反斜杠（\）来表示行连接符，可以在多行字符串中实现行的延续。

1. 引号的处理：

```ini
ini
复制代码String quote = """
    She said, "Java is awesome!"
""";
```

在这个例子中，我们使用文本块定义了一个包含引号的多行字符串。在文本块中，可以直接使用引号而无需进行转义。

1. 保留缩进的控制：

```ini
ini
复制代码String indentedString = """
    This is an indented string.
        It has nested indentation.
    """;
```

在这个例子中，我们使用文本块定义了一个具有保留缩进的字符串。文本块中的每行会保留与起始双引号的缩进级别一致的空格。

## 6.5 ZGC改进

在JDK 13中，ZGC（Z Garbage Collector）垃圾收集器进行了一些改进，以进一步降低垃圾收集的停顿时间并提高吞吐量。以下是一些JDK 13中ZGC的改进：

1. 停顿时间优化：ZGC通过引入更多并发操作来减少垃圾收集的停顿时间。JDK 13对ZGC进行了一些优化，以减少并发标记和并发整理阶段的停顿时间。这意味着应用程序在运行时不会因为垃圾收集而出现明显的停顿。
2. 吞吐量改进：ZGC的目标之一是在减少停顿时间的同时提高吞吐量。JDK 13引入了一些吞吐量方面的改进，以提高应用程序的整体性能。这些改进包括并发垃圾收集的并行处理，以及更高效的内存分配和回收策略。
3. 栈上分配：ZGC在JDK 13中引入了栈上分配（Stack Allocation）的优化。栈上分配是一种优化技术，将一些短暂的对象分配到线程的栈上，而不是堆上。这样可以减少垃圾收集的工作量，并且可以更快地回收这些短暂对象。
4. 并发Class Unloading：JDK 13中的ZGC引入了并发类卸载（Concurrent Class Unloading）的功能。这意味着当类不再使用时，可以在应用程序运行时卸载类，而无需停止应用程序的执行。这有助于减少内存占用，并提高应用程序的可伸缩性。

这些改进使得ZGC在JDK 13中更加适用于处理大型堆内存和具有低延迟要求的应用程序。然而，需要注意的是ZGC并非适用于所有场景，具体的垃圾收集器选择应该根据应用程序的特点和需求进行评估和选择。 请注意，ZGC是在JDK 11中首次引入的，而在JDK 13中进行了一些改进和优化。

# 七 JDK14 新特性

参考网站：[openjdk.org/projects/jd…](https://link.juejin.cn/?target=https%3A%2F%2Fopenjdk.org%2Fprojects%2Fjdk%2F14%2F)

```makefile
makefile
复制代码
305:
Pattern Matching for instanceof (Preview)Instanceof 的模式匹配(预览)
343:
Packaging Tool (Incubator)包装工具(孵化器)
345:
NUMA-Aware Memory Allocation for G1基于 NUMA 的 G1内存分配
349:
JFR Event StreamingJFR 事件流
352:
Non-Volatile Mapped Byte Buffers非易失性映射字节缓冲区
358:
Helpful NullPointerExceptions有用的 NullPointerException
359:
Records (Preview)档案(预览)
361:
Switch Expressions (Standard)开关表达式(标准)
362:
Deprecate the Solaris and SPARC Ports废弃 Solaris 和 SPARC 端口
363:
Remove the Concurrent Mark Sweep (CMS) Garbage Collector删除并发标记扫描(CMS)垃圾收集器
364:
ZGC on macOS
365:
ZGC on WindowsWindows 上的 ZGC
366:
Deprecate the ParallelScavenge + SerialOld GC Combination不推荐使用并行清除 + SerialOld GC 组合
367:
Remove the Pack200 Tools and API删除 Pack200工具和 API
368:
Text Blocks (Second Preview)文本块(第二次预览)
370:
Foreign-Memory Access API (Incubator)外部内存访问 API (孵化器)
```

## 7.1 instanceof模式匹配

`instanceof`模式匹配是JDK 14引入的一项新特性，用于简化对类型的检查和转换操作。在以往的Java版本中，使用`instanceof`进行类型检查后，需要进行显式的类型转换才能使用该类型的特定方法或属性。而使用`instanceof`模式匹配后，可以在同一表达式中进行类型检查和类型转换。 下面是一个使用`instanceof`模式匹配的示例：

```scss
scss
复制代码public void processShape(Shape shape) {
    if (shape instanceof Circle c) {
        double area = Math.PI * c.getRadius() * c.getRadius();
        System.out.println("Circle area: " + area);
    } else if (shape instanceof Rectangle r) {
        double area = r.getWidth() * r.getHeight();
        System.out.println("Rectangle area: " + area);
    } else if (shape instanceof Triangle t) {
        double area = 0.5 * t.getBase() * t.getHeight();
        System.out.println("Triangle area: " + area);
    } else {
        System.out.println("Unknown shape");
    }
}
```

在上述示例中，`processShape`方法接收一个`Shape`对象作为参数，并使用`instanceof`模式匹配来检查对象的具体类型。如果对象是`Circle`类型，则在`if`语句中的`instanceof`后声明一个新的变量`c`，表示类型为`Circle`的对象，然后可以直接使用`c`的方法和属性。同样地，对于`Rectangle`和`Triangle`类型，也可以在对应的`instanceof`语句中声明新的变量并使用。 这种模式匹配的语法形式为：`instanceof 类型 变量`，其中`类型`是要检查的类型，`变量`是在该分支中表示该类型的对象。这样可以避免了显式的类型转换，使代码更加简洁和可读性更高。 需要注意的是，`instanceof`模式匹配只在对应的`if`语句分支中有效，如果想在后续的代码中继续使用已经声明的变量，需要将其提升到外部作用域。

## 7.2 Records（记录类型）

Records（记录类型）是JDK 14引入的一项新特性，它简化了创建不可变（immutable）数据对象的过程。通过使用`record`关键字，可以定义一个记录类型，该类型自动生成了一些标准的方法，如构造函数、getter方法、equals()、hashCode()和toString()等方法。 以下是一个使用记录类型的示例：

```arduino
arduino
复制代码public record Person(String name, int age) {
    // 自动生成了构造函数和getter方法
}

// 创建记录类型的实例
Person person = new Person("John Doe", 30);

// 访问记录类型的属性
String name = person.name();
int age = person.age();

// 自动生成的toString()方法
System.out.println(person);
```

在上述示例中，使用`record`关键字定义了一个名为`Person`的记录类型，它有两个属性：`name`和`age`。通过定义记录类型，我们可以避免手动编写构造函数和getter方法，这些方法会自动生成并与记录类型绑定。 我们可以使用记录类型的构造函数来创建实例，然后通过自动生成的getter方法访问属性。例如，`person.name()`返回记录类型实例的`name`属性值。 此外，记录类型还提供了自动生成的`equals()`和`hashCode()`方法，用于比较记录类型的相等性和生成哈希码。同时，它还提供了自动生成的`toString()`方法，用于以字符串形式表示记录类型的内容。 需要注意的是，记录类型是不可变的，即一旦创建，就不能修改其属性的值。如果需要修改属性，需要创建一个新的记录类型实例。 记录类型的引入简化了创建简单数据对象的过程，减少了样板代码，提高了代码的可读性和可维护性，尤其适用于那些只包含数据的简单对象。

## 7.3 Switch表达式扩展

JDK 14对switch表达式进行了扩展，使其更加强大和灵活。以下是一些switch表达式的扩展特性：

1. 使用箭头语法（Arrow Syntax）：在JDK 14之前，switch语句使用冒号（:）来分隔标签和相应的代码块。但在JDK 14中，引入了箭头（->）来替代冒号，使得switch语句更加简洁和易读。 示例：

```arduino
arduino
复制代码int dayOfWeek = 3;
String dayType = switch (dayOfWeek) {
    case 1, 2, 3, 4, 5 -> "Weekday";
    case 6, 7 -> "Weekend";
    default -> throw new IllegalArgumentException("Invalid day of the week: " + dayOfWeek);
};
```

1. 使用多个标签（Multiple Labels）：在JDK 14中，可以在一个case语句中使用多个标签，以逗号分隔。这样可以将多个标签映射到同一段代码，避免了重复的代码块。 示例：

```csharp
csharp
复制代码int number = 2;
switch (number) {
    case 1, 2, 3:
        System.out.println("Number is between 1 and 3");
        break;
    case 4, 5, 6:
        System.out.println("Number is between 4 and 6");
        break;
    default:
        System.out.println("Number is out of range");
        break;
}
```

1. 使用yield语句返回值（Yield Statement）：在JDK 14中，可以在switch表达式的每个分支中使用yield语句返回一个值。这样可以更方便地从switch表达式中返回结果。 示例：

```arduino
arduino
复制代码int number = 2;
String numberType = switch (number) {
    case 0:
    case 1:
        yield "Even";
    case 2:
    case 3:
        yield "Odd";
    default:
        yield "Unknown";
};
```

这些扩展使得switch表达式更加强大和灵活，能够更清晰地表达代码逻辑，并减少冗余代码。

## 7.4 垃圾回收器（Garbage Collectors）增强

在JDK 14中，引入了两个新的垃圾回收器：ZGC（Z Garbage Collector）和Shenandoah，以增强Java应用程序的垃圾回收性能。以下是对它们的简要介绍：

1. ZGC（Z Garbage Collector）：ZGC是一种低延迟的垃圾回收器，旨在减少Java应用程序的停顿时间。它是一种并发垃圾回收器，可以在非常短暂的暂停时间内执行大部分的垃圾回收操作。ZGC的目标是使得垃圾回收停顿时间不超过10毫秒，并且在几百兆字节至几个太字节的堆大小下具有可扩展性。
2. Shenandoah：Shenandoah是另一种低延迟的垃圾回收器，专注于最小化应用程序的停顿时间。它是一种并发标记-清除垃圾回收器，它的主要目标是在堆大小为几百兆字节到几个太字节之间，提供可接受的垃圾回收停顿时间。Shenandoah采用了一种全局并发标记算法，使得标记阶段与应用程序线程并发执行，从而减少了停顿时间。

这两个垃圾回收器的共同目标是减少Java应用程序的停顿时间，以提高应用程序的响应性能。它们采用了不同的垃圾回收算法和技术，适用于不同的应用场景和堆大小。选择使用哪个垃圾回收器取决于应用程序的特性、性能需求以及硬件配置。 需要注意的是，ZGC和Shenandoah在JDK 14中被标记为实验性功能，可以通过命令行选项来启用它们。在后续的JDK版本中，这些垃圾回收器可能会继续改进和优化，以提供更好的性能和稳定性。

## 7.5 Numeral Formatting

在JDK 14中，引入了对Numeral Formatting的增强，主要通过`java.text.NumberFormat`类的新方法和功能进行改进。以下是JDK 14中Numeral Formatting的一些新特性：

1. 使用`CompactNumberFormat`类： JDK 14引入了`CompactNumberFormat`类，它是`NumberFormat`的子类，用于在格式化数字时以紧凑形式显示。它可以根据数字的大小自动选择合适的单位（如K、M、B等）来显示数字。 示例：

```ini
ini
复制代码double value = 12345678;
NumberFormat nf = NumberFormat.getCompactNumberInstance();
String formattedValue = nf.format(value);
System.out.println(formattedValue); // 输出：12M
```

1. 新的`toLocalizedPattern()`和`toPattern()`方法： 在JDK 14中，`NumberFormat`类新增了`toLocalizedPattern()`和`toPattern()`方法，用于获取数字格式化模式的本地化字符串表示和原始字符串表示。 示例：

```ini
ini
复制代码NumberFormat nf = NumberFormat.getInstance();
String localizedPattern = nf.toLocalizedPattern();
String pattern = nf.toPattern();
System.out.println(localizedPattern);
System.out.println(pattern);
```

1. 增强的小数位数控制： 在JDK 14中，`NumberFormat`类的`setMaximumFractionDigits()`和`setMinimumFractionDigits()`方法支持负数参数，用于设置小数位数的上限和下限。负数值表示保留尽可能多的小数位数，但至少保留指定的数目。 示例：

```ini
ini
复制代码double value = 1234.56789;
NumberFormat nf = NumberFormat.getInstance();
nf.setMaximumFractionDigits(-1); // 保留尽可能多的小数位数
nf.setMinimumFractionDigits(2); // 至少保留2位小数
String formattedValue = nf.format(value);
System.out.println(formattedValue); // 输出：1,234.56789
```

这些增强使得在JDK 14中进行Numeral Formatting更加方便和灵活。可以根据需求使用紧凑形式、获取格式化模式的本地化字符串表示以及对小数位数进行更精细的控制。

# 八 JDK15新特性

参考网站：[openjdk.org/projects/jd…](https://link.juejin.cn/?target=https%3A%2F%2Fopenjdk.org%2Fprojects%2Fjdk%2F15%2F)

```makefile
makefile
复制代码
339:
Edwards-Curve Digital Signature Algorithm (EdDSA)曲线数字签名算法
360:
Sealed Classes (Preview)密封类(预览)
371:
Hidden Classes隐藏类别
372:
Remove the Nashorn JavaScript Engine删除 Nashorn JavaScript 引擎
373:
Reimplement the Legacy DatagramSocket API重新实现遗留 DatagramSocket API
374:
Disable and Deprecate Biased Locking禁用和取消偏差锁定
375:
Pattern Matching for instanceof (Second Preview)模式匹配(第二次预览)
377:
ZGC: A Scalable Low-Latency Garbage CollectorZGC: 一个可伸缩的低延迟垃圾收集器
378:
Text Blocks文本块
379:
Shenandoah: A Low-Pause-Time Garbage CollectorShenandoah: 一个低暂停时间的垃圾收集器
381:
Remove the Solaris and SPARC Ports删除 Solaris 和 SPARC 端口
383:
Foreign-Memory Access API (Second Incubator)外部内存访问 API (第二个孵化器)
384:
Records (Second Preview)纪录(第二次预览)
385:
Deprecate RMI Activation for Removal不推荐激活 RMI 以便删除
```

## 8.1 Sealed Classes（密封类）

密封类（Sealed Classes）是JDK 15引入的一项重要特性，它是一种新的类和接口修饰符，用于控制类的继承关系。通过将类或接口声明为密封类，可以限制哪些类可以继承或实现该密封类 密封类的主要目的是提供更严格的访问控制，以确保继承层次结构的完整性和安全性。在使用密封类时，开发者可以明确指定允许继承的类的范围，这样可以防止不受控制的扩展和继承。 要将一个类声明为密封类，需要使用`sealed`关键字进行修饰。例如：

```kotlin
kotlin
复制代码sealed class Shape permits Circle, Square, Triangle {
    // Class body
}
```

在上面的例子中，`Shape`是一个密封类，它允许继承的类有`Circle`、`Square`和`Triangle`。这意味着只有这三个类可以直接继承`Shape`，其他类无法继承它。如果其他类尝试继承`Shape`，编译器将报错。 通过使用密封类，可以有效地控制继承关系，并减少意外的扩展。这对于框架和库的设计非常有用，可以确保只有受信任的类可以扩展或实现密封类。 此外，密封类与模式匹配（Pattern Matching）功能相结合，可以更方便地处理密封类的实例。模式匹配可以根据实例的类型进行匹配，并根据匹配的结果执行相应的操作，从而简化了类型检查和转换的代码。 总之，密封类是JDK 15引入的一项重要特性，它通过控制继承关系来提供更严格的访问控制和安全性。它可以帮助开发者更好地设计类层次结构，并减少意外的扩展。

## 8.2 Text Blocks（文本块）

文本块（Text Blocks）是JDK 15引入的一项重要特性，它提供了一种更自然、更易读的多行字符串表示形式，以减少在代码中编写长字符串时的转义字符和格式化问题。

在传统的Java中，如果要编写一个包含多行文本的字符串，需要使用转义字符（例如`\n`）或连接操作符（`+`）将多行字符串拼接在一起。这样的写法不仅冗长而且难以阅读和维护。

使用文本块，可以更简洁地表示多行字符串，而无需使用转义字符或连接操作符。文本块使用三个引号（"""）将多行字符串包裹起来，并通过缩进来定义字符串的内容。例如：

```ini
ini
复制代码String textBlock = """
    Hello,
    This is a multi-line
    text block.
    """;
```

在上面的例子中，`textBlock`是一个文本块，包含了三行文本。通过使用文本块，我们可以直观地看到字符串的格式和结构，而无需担心转义字符和格式化问题。 文本块还支持一些额外的特性，例如去除前导空格和尾随空格。可以使用`~`字符来指定文本块的缩进级别，从而控制文本块中的空格。

```ini
ini
复制代码String indentedBlock = """
        This is an indented block
        with leading and trailing spaces.
        """.stripIndent();
```

在上面的例子中，`stripIndent()`方法会自动去除前导空格，生成一个不包含缩进的文本块。 文本块在许多场景中都非常有用，特别是在编写HTML、JSON、SQL或其他格式化字符串时。它提供了更清晰和易于维护的代码，提高了开发效率。 需要注意的是，文本块是在编译时进行处理的，并不会在运行时影响字符串对象的表示方式。因此，文本块并不改变字符串的基本性质和操作。

## 8.3 Hidden Classes（隐藏类）

隐藏类（Hidden Classes）是JDK 15引入的一项特性，它允许在运行时动态生成类，同时不会对程序的性能产生负面影响。隐藏类对于某些特定的应用场景非常有用，如动态代理、代码生成和Java虚拟机的实现。 隐藏类的主要目的是在不暴露类的字节码的情况下，允许生成和使用类。这对于一些涉及敏感逻辑或需要动态生成和加载类的应用程序非常有用。 隐藏类的生成和使用是通过Java虚拟机的`Lookup`对象和`MethodHandles` API进行的。`Lookup`对象提供了对类的私有成员访问的权限，并可以用于创建隐藏类。`MethodHandles` API提供了对方法句柄的操作和调用。 生成隐藏类的过程可以概括为以下几个步骤：

1. 获取`Lookup`对象：通过反射或其他方式获取`Lookup`对象，该对象具有足够的权限来创建隐藏类。
2. 定义类的结构：使用`Lookup`对象的`defineHiddenClass()`方法定义类的结构，包括类的名称、父类、接口等。
3. 加载和链接类：使用`Lookup`对象的`lookupClass()`方法加载和链接隐藏类。
4. 创建实例和调用方法：使用`MethodHandles` API和隐藏类的句柄来创建实例和调用方法。

隐藏类的主要优势在于，它们可以在运行时动态生成和加载，而无需事先编写和编译类的字节码。这对于某些场景，如实现动态代理、插件系统、代码生成和热部署等，提供了更大的灵活性和扩展性。 需要注意的是，隐藏类是一项高级特性，通常在特定的应用程序框架和库中使用，而不是普通的应用程序开发中。使用隐藏类需要谨慎，确保了解其工作原理和适用场景。

## 8.4 Records（记录类）

JDK 15对记录类进行了一些改进，主要包括：

1. 扩展性构造函数（Extended Constructor）：在JDK 15之前，记录类只有一个隐式的构造函数，它接受所有属性作为参数。在JDK 15中，记录类引入了扩展性构造函数，允许开发者自定义构造函数，同时保留隐式构造函数的功能。
2. @Override支持：在JDK 15之前，由于记录类的方法是由编译器自动生成的，因此无法使用@Override注解来标记覆盖父类方法。在JDK 15中，支持在记录类方法上使用@Override注解，以便更好地表示其覆盖关系。

## 8.5 Pattern Matching for instanceof（instanceof的模式匹配）

JDK 15引入了一项重要的特性，即"Pattern Matching for instanceof"（instanceof的模式匹配）。这个特性通过简化对类型的检查和转换来提高代码的可读性和简洁性。 在Java中，我们通常使用`instanceof`运算符来检查对象是否属于某个特定类型。在JDK 15之前，这通常需要将检查通过后的对象强制转换为目标类型，以便在后续的代码中使用。这样的代码模式非常普遍。 而在JDK 15中，引入了模式匹配的功能，可以将`instanceof`与类型转换结合起来，以更简洁的方式完成类型检查和转换。下面是一个使用模式匹配的示例：

```rust
rust
复制代码if (obj instanceof String str) {
    System.out.println(str.length());
}
```

在上面的例子中，我们使用`instanceof`运算符检查`obj`是否是`String`类型。如果是，那么我们可以在条件块内部直接使用`str`作为`String`类型的对象，并且编译器会自动将其视为`String`类型。这样，我们就可以直接调用`str`上的方法或访问其属性，而无需进行额外的强制类型转换。 这种模式匹配的语法可以帮助开发者更直接地处理类型检查和类型转换，并减少了样板代码的编写。它提供了更简洁和易读的方式来处理复杂的类型判断和转换逻辑。 需要注意的是，模式匹配的作用域仅限于条件块内部，超出该范围后，变量将不再被视为特定类型。

# 九 JDK16新特性

参考网站：[openjdk.org/projects/jd…](https://link.juejin.cn/?target=https%3A%2F%2Fopenjdk.org%2Fprojects%2Fjdk%2F16%2F)

```makefile
makefile
复制代码
338:
Vector API (Incubator)矢量 API (孵化器)
347:
Enable C++14 Language Features启用 C + + 14语言特性
357:
Migrate from Mercurial to Git从 Mercurial 迁移到 Git
369:
Migrate to GitHub迁移到 GitHub
376:
ZGC: Concurrent Thread-Stack ProcessingZGC: 并发线程堆栈处理
380:
Unix-Domain Socket ChannelsUnix 域套接字通道
386:
Alpine Linux Port高山 Linux 端口
387:
Elastic Metaspace弹性元空间
388:
Windows/AArch64 PortWindows/AArch64端口
389:
Foreign Linker API (Incubator)外部连接器 API (孵化器)
390:
Warnings for Value-Based Classes基于值的类的警告
392:
Packaging Tool包装工具
393:
Foreign-Memory Access API (Third Incubator)外部内存访问 API (第三孵化器)
394:
Pattern Matching for instanceof模式匹配
395:
Records记录
396:
Strongly Encapsulate JDK Internals by Default默认情况下强封装 JDK 内部
397:
Sealed Classes (Second Preview)密封类(第二次预览)
```

## 9.1 Foreign Function & Memory API（外部函数和内存 API）

Foreign Function & Memory API（外部函数和内存 API）是 JDK 16 中引入的一个特性，它提供了一种标准化的机制，用于与外部本机代码进行交互。该 API 旨在改善 Java 与本机代码集成的体验，并简化本机代码访问和操作。 Foreign Function & Memory API 的主要组成部分包括以下内容：

1. Foreign Linker API（外部链接器 API）：该 API 提供了一组接口和类，用于声明和操作与本机函数的链接。通过 Foreign Linker API，可以定义本机函数的签名、访问本机库中的函数，并在 Java 代码中调用本机函数。
2. Foreign Memory Access API（外部内存访问 API）：该 API 允许直接访问本机内存，以便在 Java 代码中高效地处理本机数据。通过 Foreign Memory Access API，可以分配本机内存、读写本机内存的值，并执行本机内存的释放操作。
3. Foreign Memory Segment API（外部内存段 API）：该 API 提供了对连续内存段的抽象，可以在 Java 代码中表示和操作本机内存块。它包括操作内存段的方法，如复制、填充、移动和比较等。

Foreign Function & Memory API 使得 Java 开发人员能够更加灵活地与本机代码进行交互，并且可以提高性能和降低与本机代码集成的复杂性。它在与本机库、操作系统接口、硬件加速等方面有广泛的应用场景。 请注意，使用 Foreign Function & Memory API 需要谨慎处理本机代码和内存的访问，以确保安全性和稳定性。在使用该 API 时，建议参考官方文档和示例，以便正确地使用和管理本机函数和内存。

## 9.2 Unix-Domain Socket Channel（Unix 域套接字通道）

Unix-Domain Socket Channel（Unix 域套接字通道）是 JDK 16 中引入的一个特性，它允许 Java 程序通过 Unix 域套接字进行本机进程间通信（IPC）。 Unix 域套接字是一种在同一台机器上的进程之间进行通信的机制，类似于网络套接字，但在内部使用文件系统路径作为通信端点而不是 IP 地址和端口号。 Unix-Domain Socket Channel 提供了一种与本机 Unix 域套接字进行交互的标准化方式，使开发人员能够使用 Java 程序进行进程间通信。使用 Unix-Domain Socket Channel，可以创建、绑定、连接和进行读写操作等。 以下是使用 Unix-Domain Socket Channel 的一些关键操作：

1. 创建 Unix-Domain Socket Channel：使用静态工厂方法 `UnixDomainSocketAddress.of(String path)` 创建一个 Unix 域套接字地址，并使用 `UnixDomainSocketChannel.open()` 创建一个 Unix-Domain Socket Channel。
2. 绑定和连接：通过调用 `bind(UnixDomainSocketAddress address)` 方法绑定 Unix-Domain Socket Channel 到指定的地址，或使用 `connect(UnixDomainSocketAddress address)` 方法连接到远程地址。
3. 读写操作：可以使用 `read(ByteBuffer dst)` 和 `write(ByteBuffer src)` 方法进行读取和写入操作，读写的数据将在本机套接字通道和对应的本机进程之间传输。
4. 关闭通道：通过调用 `close()` 方法关闭 Unix-Domain Socket Channel，释放相关的资源。

Unix-Domain Socket Channel 提供了一种在 Java 中进行本机进程间通信的高级机制，特别适用于需要高性能、低延迟和安全性的应用场景，如进程间协作、服务器和客户端通信等。 请注意，Unix-Domain Socket Channel 只能在支持 Unix 域套接字的操作系统上使用，如 Linux、Unix 和 macOS 等。

## 9.3 Vector API（向量 API）

Vector API（向量 API）是 JDK 16 中引入的一个特性，它提供了一组类和接口，用于在 Java 中执行数据并行计算和矢量化操作。通过使用向量 API，开发人员可以更高效地利用现代硬件的矢量化指令集执行并行计算。 以下是 Vector API 的一些关键特性和概念：

1. 向量化数据类型：向量 API 引入了一组新的数据类型，如 Vector128、Vector256 等，用于表示向量化数据。这些数据类型可以存储和操作多个数据元素，并利用硬件的矢量化指令集进行并行计算。
2. 向量操作方法：向量 API 提供了一组用于执行向量操作的方法，如加法、减法、乘法、除法、逐元素操作等。这些方法可以在向量数据类型上进行操作，并自动利用底层硬件的矢量化指令集进行并行计算。
3. 向量掩码：向量 API 引入了一种向量掩码的概念，用于选择性地应用向量操作。通过指定一个掩码，可以对向量数据的部分元素进行操作，而不影响其他元素。这样可以更灵活地处理不规则数据集和条件操作。
4. 并行迭代：向量 API 支持并行迭代操作，允许开发人员以向量化的方式处理数组和数据集。通过提供迭代器和并行流等机制，可以高效地在多个向量数据上执行操作。
5. 与现有 API 集成：向量 API 与现有的 Java API 集成良好，可以与 Stream API、并行流、并发集合等其他并行计算机制一起使用。这样可以在现有代码中无缝地加入向量化操作，提高性能和并行度。

向量 API 的目标是提供一种简单、直观的方式来利用硬件的矢量化指令集进行数据并行计算。它可以在各种场景中发挥作用，如科学计算、图像处理、机器学习等需要大量数据操作和并行计算的领域。

## 9.4 Alpine Linux 支持

在 JDK 16 中，引入了对 Alpine Linux 的官方支持。Alpine Linux 是一个轻量级的 Linux 发行版，主要用于容器化应用程序。通过在 JDK 16 中添加对 Alpine Linux 的支持，Java 开发人员可以更方便地在 Alpine Linux 上部署和运行他们的 Java 应用。 Alpine Linux 的特点是具有极小的镜像大小、简化的软件包管理和基于 musl libc 的轻量级 C 库。它专注于提供一个小巧、高度可定制和安全的基础操作系统环境。 在 JDK 16 中，通过添加对 musl libc 的支持，Java 程序可以直接在 Alpine Linux 上运行，而无需依赖传统的 glibc（GNU C Library）。这使得 Java 应用程序可以在更小的容器镜像中运行，并且可以更快地启动和部署。 Alpine Linux 支持对 Java 开发人员来说是一个好消息，尤其是在容器化和云原生应用程序的场景下。它提供了更轻量级的基础操作系统选择，并且与 JDK 16 一起使用，可以提供更高的灵活性和性能。 需要注意的是，尽管 JDK 16 提供了对 Alpine Linux 的官方支持，但仍然需要遵循最佳实践和适当的配置，以确保在 Alpine Linux 上的 Java 应用程序的正常运行。

# 十 JDK17 新特性

参考网站：[openjdk.org/projects/jd…](https://link.juejin.cn/?target=https%3A%2F%2Fopenjdk.org%2Fprojects%2Fjdk%2F17%2F)

```makefile
makefile
复制代码
306:
Restore Always-Strict Floating-Point Semantics恢复始终严格的浮点语义
356:
Enhanced Pseudo-Random Number Generators增强型伪随机数发生器
382:
New macOS Rendering Pipeline新的 macOS 渲染管道
391:
macOS/AArch64 PortMacOS/AArch64端口
398:
Deprecate the Applet API for Removal废弃 Applet API 以便删除
403:
Strongly Encapsulate JDK Internals强封装 JDK 内部结构
406:
Pattern Matching for switch (Preview)开关模式匹配(预览)
407:
Remove RMI Activation删除 RMI 激活
409:
Sealed Classes密封类
410:
Remove the Experimental AOT and JIT Compiler删除实验性 AOT 和 JIT 编译器
411:
Deprecate the Security Manager for Removal取消安全管理器以便删除
412:
Foreign Function & Memory API (Incubator)外部函数和内存 API (孵化器)
414:
Vector API (Second Incubator)矢量 API (第二个孵化器)
415:
Context-Specific Deserialization Filters特定于上下文的反序列化过滤器
JDK 17 will be a long-term support (LTS) release from most vendors. For a complete list of the JEPs integrated since the previous LTS release, JDK 11, please see here.
JDK 17将是来自大多数供应商的长期支持(LTS)版本。有关自上一个 LTS 版本 JDK 11以来集成的 JEP 的完整列表，请参阅这里。
```

## 10.1 Context-Specific Deserialization Filters（上下文特定的反序列化过滤器）

Context-Specific Deserialization Filters（上下文特定的反序列化过滤器）是 JDK 17 中引入的一个特性，用于增强 Java 应用程序对反序列化过程的控制和安全性。 在 Java 中，反序列化是将对象从字节序列还原为内存中的对象的过程。然而，反序列化操作可能存在安全风险，因为恶意序列化数据可以触发未经授权的代码执行或对象创建。 Context-Specific Deserialization Filters 允许开发人员定义特定于上下文的反序列化过滤器，以限制哪些类可以被反序列化。开发人员可以在特定的上下文中设置反序列化过滤器，例如在应用程序的安全管理器中、在 RMI 远程调用中、在对象输入流的特定实例上等。 通过定义上下文特定的反序列化过滤器，开发人员可以实现以下目标：

1. 类过滤：可以定义白名单或黑名单，只允许或禁止特定的类进行反序列化。这有助于限制潜在的安全风险和不受信任的类的反序列化。
2. 内容检查：可以对反序列化的对象进行更严格的验证和检查，以确保数据的完整性和合法性。
3. 安全策略实施：可以通过上下文特定的反序列化过滤器实施特定的安全策略，以满足应用程序的需求和要求。

通过使用 Context-Specific Deserialization Filters，开发人员可以在反序列化过程中更细粒度地控制和验证对象的创建和内容。这有助于提高应用程序的安全性，减少潜在的安全漏洞和攻击面。

## 10.2 Strong encapsulation of JDK internals（JDK 内部模块的强封装）

Strong encapsulation of JDK internals（JDK 内部模块的强封装）是 JDK 17 中引入的一个特性，旨在进一步限制和封装 JDK 内部模块的访问。 在 Java 9 中，引入了模块系统（Module System），通过将 JDK 分为一组模块来提供更严格的访问控制和模块化。然而，在较早版本的 JDK 中，某些 JDK 内部模块的访问级别仍然比较宽松，可能会导致潜在的不安全和不稳定的代码依赖关系。 JDK 17 引入了 Strong encapsulation 特性，通过进一步限制和封装 JDK 内部模块的访问，以提高系统的安全性和稳定性。具体来说，它包括以下方面：

1. JDK 内部 API 的强封装：许多 JDK 内部 API 在 JDK 17 中被标记为强封装，不再公开为公共 API。这些 API 的访问级别被限制在 JDK 内部，开发人员不应直接访问或依赖这些 API。
2. 模块的强封装：JDK 17 强化了一些 JDK 内部模块的封装，以防止外部模块直接访问这些内部模块。这增加了对 JDK 内部实现细节的保护，降低了对内部模块的直接依赖。

通过强封装 JDK 内部模块，可以减少对不稳定的和非正式的 API 的依赖，并提高应用程序的可靠性和可移植性。这也促使开发人员使用公共和稳定的 API，以确保应用程序在不同版本的 JDK 上的兼容性。 需要注意的是，在使用 JDK 17 时，如果应用程序依赖于 JDK 内部的 API 或模块，可能会受到这种强封装的影响。

## 10.3 Sealed JNI（密封 JNI）

Sealed JNI（密封 JNI）是 JDK 17 中引入的一个特性，旨在限制本机接口（JNI）的实现，以提高本机代码的安全性。 JNI 是 Java Native Interface 的缩写，它允许 Java 程序与本机代码进行交互。通过 JNI，Java 程序可以调用本机库中的函数，也可以从本机代码中调用 Java 方法。然而，JNI 也带来了一些安全风险，因为不受信任的本机代码可能会访问和修改 Java 程序的内部状态。 Sealed JNI 引入了密封 JNI 的概念，通过限制本机接口的实现来提高本机代码的安全性。具体来说，Sealed JNI 可以通过以下方式实现：

1. 密封本机接口库：开发人员可以将本机接口库标记为密封（sealed），以限制它的实现。密封本机接口库只能在特定的上下文中使用，例如指定的模块、类加载器或其他条件。
2. 安全策略的强制执行：通过密封 JNI，可以强制执行安全策略，只允许受信任的本机接口库进行调用。这可以减少潜在的恶意或不安全的本机代码对 Java 程序的影响。

Sealed JNI 的目标是提高本机代码的安全性，并减少与不受信任的本机代码相关的潜在风险。开发人员可以使用密封 JNI 来限制本机接口库的使用范围，并确保只有受信任的代码可以调用本机方法。

## 10.4 改进的垃圾收集器

在 JDK 17 中，进行了一些改进和优化，以提供更好的垃圾收集器性能和稳定性。以下是一些与垃圾收集器相关的改进：

1. ZGC：ZGC 是一种低延迟的垃圾收集器，旨在减少应用程序的停顿时间。在 JDK 17 中，ZGC 进行了一些改进，包括改进的内存分配、增强的并发处理和更好的性能。
2. Shenandoah GC：Shenandoah GC 是另一种低延迟的垃圾收集器，适用于大内存堆的应用程序。在 JDK 17 中，Shenandoah GC 进行了一些改进，包括更好的垃圾收集算法、并发处理和性能优化。
3. G1 GC 改进：G1（Garbage-First）是一种面向大堆和低停顿的垃圾收集器。在 JDK 17 中，对 G1 GC 进行了一些改进，包括性能优化、内存分配改进和更好的并发处理。
4. 垃圾收集器接口：JDK 17 引入了一组新的垃圾收集器接口，允许开发人员实现自定义的垃圾收集器。这样，开发人员可以根据自己的需求和场景开发定制的垃圾收集器。

这些改进和优化旨在提供更好的垃圾收集器性能、降低停顿时间，并提高大堆和低延迟应用程序的吞吐量。具体的改进和优化细节可以在 JDK 17 的发行说明和相关文档中找到。