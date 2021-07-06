## 简介

- 简化 Java 繁冗代码的插件

## 官方资料

[Lombok官方文档](https://projectlombok.org/features/all)

- 最终解释都以官方文档为主，下面的资料链接仅供参考

## 常用注解

[知乎 - Lombok常用注解、使用案例、原理解析](https://zhuanlan.zhihu.com/p/121018197)

- @Data
- @Getter | @Setter
- @AllArgsConstruct | @NoArgsConstruct | @RequiredArgsConstructor
- @ToString | @EqualsAndHashCode | @Sl4fj
- @Builder | @Accessors

## val 与 var

- val 与 var 都用于非显示对象类型的声明，如 var str = "用var声明一个字符串"
- val 与 var 的差异为：val声明的变量为 final

### 使用Lombok

```java
import java.util.ArrayList;
import java.util.HashMap;
import lombok.val;

public class ValExample {
  public String example() {
    val example = new ArrayList<String>();
    example.add("Hello, World!");
    val foo = example.get(0);
    return foo.toLowerCase();
  }
  
  public void example2() {
    val map = new HashMap<Integer, String>();
    map.put(0, "zero");
    map.put(5, "five");
    for (val entry : map.entrySet()) {
      System.out.printf("%d: %s\n", entry.getKey(), entry.getValue());
    }
  }
}
```

### 原生Java

```java
import java.util.ArrayList;
import java.util.HashMap;
import java.util.Map;

public class ValExample {
  public String example() {
    final ArrayList<String> example = new ArrayList<String>();
    example.add("Hello, World!");
    final String foo = example.get(0);
    return foo.toLowerCase();
  }
  
  public void example2() {
    final HashMap<Integer, String> map = new HashMap<Integer, String>();
    map.put(0, "zero");
    map.put(5, "five");
    for (final Map.Entry<Integer, String> entry : map.entrySet()) {
      System.out.printf("%d: %s\n", entry.getKey(), entry.getValue());
    }
  }
}
```

## @NonNull

- 标记在属性上，等于一段判空逻辑，为空则抛NPE

### 使用Lombok

```java
import lombok.NonNull;

public class NonNullExample extends Something {
  private String name;
  
  public NonNullExample(@NonNull Person person) {
    super("Hello");
    this.name = person.getName();
  }
}
```

### 原生Java

```java
import lombok.NonNull;

public class NonNullExample extends Something {
  private String name;
  
  public NonNullExample(@NonNull Person person) {
    super("Hello");
    if (person == null) {
      throw new NullPointerException("person is marked @NonNull but is null");
    }
    this.name = person.getName();
  }
}
```

## @Cleanup

- 标记在流对象上，让其自动关闭

### 使用Lombok

```java
import lombok.Cleanup;
import java.io.*;

public class CleanupExample {
  public static void main(String[] args) throws IOException {
    @Cleanup InputStream in = new FileInputStream(args[0]);
    @Cleanup OutputStream out = new FileOutputStream(args[1]);
    byte[] b = new byte[10000];
    while (true) {
      int r = in.read(b);
      if (r == -1) break;
      out.write(b, 0, r);
    }
  }
}
```

### 原生Java

```java
import java.io.*;

public class CleanupExample {
  public static void main(String[] args) throws IOException {
    InputStream in = new FileInputStream(args[0]);
    try {
      OutputStream out = new FileOutputStream(args[1]);
      try {
        byte[] b = new byte[10000];
        while (true) {
          int r = in.read(b);
          if (r == -1) break;
          out.write(b, 0, r);
        }
      } finally {
        if (out != null) {
          out.close();
        }
      }
    } finally {
      if (in != null) {
        in.close();
      }
    }
  }
}
```

## @SuperBuilder

[知乎 - 使用案例及原理简析](https://zhuanlan.zhihu.com/p/341630469)

[CSDN - 注解属性使用及注意事项](https://blog.csdn.net/qq_20021569/article/details/102471373)

- 增强 @Builder 注解，使其可以支持对父类属性构造赋值

