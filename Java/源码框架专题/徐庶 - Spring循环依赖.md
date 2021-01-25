# 徐庶 - Spring循环依赖

[循环依赖脑图](https://www.processon.com/view/link/5f1fb2cf1e08533a628a7b4c)

## 简介

- 所谓循环依赖，即 A、B 之间的互相依赖
  - 例如：
    - UserService 属性依赖 DeptService
    - DeptService 属性依赖 UserService
  - 或者：
    - A 依赖 B
    - B 依赖 C
    - C 依赖 A

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1611284791534.png)

## 循环依赖代码演示

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class InstanceA {

    @Autowired
    private InstanceB instanceB;

    public InstanceA() {
        System.out.println("实例A实例化...");
    }

    public void say() {
        System.out.println("Hello World!");
    }

}
```

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class InstanceB {

    @Autowired
    private InstanceA instanceA;

    public InstanceB() {
        System.out.println("实例B实例化...");
    }

}
```

```java
import com.google.common.collect.Maps;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.beans.factory.support.RootBeanDefinition;

import java.lang.reflect.Field;
import java.util.Map;

/**
 * 模拟 Spring 启动类操作
 */
public class Main {

    /**
     * 1. Bean定义容器池
     */
    private static final Map<String, BeanDefinition> beanDefinitionMap = Maps.newConcurrentMap();

    /**
     * 1. 一级缓存
     */
    private static final Map<String, Object> singletonObjects = Maps.newConcurrentMap();

    public static void main(String[] args) throws Exception {
        // 1. 加载 BeanDefinition
        loadBeanDefinitions();

        // 1. 循环创建 Bean
        for (String beanName : beanDefinitionMap.keySet()) {
            getBean(beanName);
        }

        InstanceA instanceA = (InstanceA) singletonObjects.get("instanceA");
        instanceA.say();
    }

    /**
     * 1. 启动加载Bean定义 -> Bean定义容器池
     */
    public static void loadBeanDefinitions() {
        beanDefinitionMap.put("instanceA", new RootBeanDefinition(InstanceA.class));
        beanDefinitionMap.put("instanceB", new RootBeanDefinition(InstanceB.class));
    }

    public static Object getBean(String beanName) throws Exception {
        // 1. 实例化
        RootBeanDefinition beanDefinition = (RootBeanDefinition) beanDefinitionMap.get(beanName);
        Class<?> beanClass = beanDefinition.getBeanClass();
        Object instance = beanClass.getDeclaredConstructor().newInstance();

        // 1. 属性赋值
        Field[] fields = beanClass.getDeclaredFields();
        for (Field field : fields) {
            // 1. bean 中属性也是 bean
            if (field.getAnnotation(Autowired.class) != null) {
                // 1. 设置私有属性可操作
                field.setAccessible(true);

                // 1. 为属性赋值
                field.set(instance, getBean(field.getName()));
            }
        }

        // 1. 初始化 { 执行 bean 中可能定义的 init-method 之类的初始化钩子函数 }
        // 这里主要模拟循环依赖，就不演示了

        // 1. getBean 完毕，放入一级缓存
        singletonObjects.put(beanName, instance);
        return instance;
    }

}
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1611286301385.png)

## 一级缓存解决循环依赖思路

```java
/**
 * 模拟 Spring 启动类操作
 */
public class Main {

    /**
     * 1. Bean定义容器池
     */
    private static final Map<String, BeanDefinition> beanDefinitionMap = Maps.newConcurrentMap();

    /**
     * 1. 一级缓存
     */
    private static final Map<String, Object> singletonObjects = Maps.newConcurrentMap();

    public static void main(String[] args) throws Exception {
        // 1. 加载 BeanDefinition
        loadBeanDefinitions();

        // 1. 循环创建 Bean
        for (String beanName : beanDefinitionMap.keySet()) {
            getBean(beanName);
        }

        InstanceA instanceA = (InstanceA) singletonObjects.get("instanceA");
        instanceA.say();
    }

    /**
     * 1. 启动加载Bean定义 -> Bean定义容器池
     */
    public static void loadBeanDefinitions() {
        beanDefinitionMap.put("instanceA", new RootBeanDefinition(InstanceA.class));
        beanDefinitionMap.put("instanceB", new RootBeanDefinition(InstanceB.class));
    }

    public static Object getBean(String beanName) throws Exception {
        // 2. 先尝试从一级缓存获取
        Object singleton = getSingleton(beanName);
        if (singleton != null) {
            return singleton;
        }

        // 1. 实例化
        RootBeanDefinition beanDefinition = (RootBeanDefinition) beanDefinitionMap.get(beanName);
        Class<?> beanClass = beanDefinition.getBeanClass();
        Object instance = beanClass.getDeclaredConstructor().newInstance();

        // 2. 在实例化 bean 后放入 一级缓存
        singletonObjects.put(beanName, instance);

        // 1. 属性赋值
        Field[] fields = beanClass.getDeclaredFields();
        for (Field field : fields) {
            // 1. bean 中属性也是 bean
            if (field.getAnnotation(Autowired.class) != null) {
                // 1. 设置私有属性可操作
                field.setAccessible(true);

                // 1. 为属性赋值
                field.set(instance, getBean(field.getName()));
            }
        }

        // 1. 初始化 { 执行 bean 中可能定义的 init-method 之类的初始化钩子函数 }
        // 这里主要模拟循环依赖，就不演示了

        return instance;
    }

    /**
     * 2. 从一级缓存池中通过 beanName 获取 bean
     */
    public static Object getSingleton(String beanName) {
        return singletonObjects.get(beanName);
    }

}
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1611295686306.png)

### 缺陷

- 此时是单线程情况，一级缓存可以解决 循环依赖 问题
- 多线程时：
  - A 线程刚完成 实例化bean、放入一级缓存
  - B 线程调用 getBean，此时获取到的是一个不完整的bean(未经属性赋值、初始化方法调用)

## 二级缓存解决循环依赖思路

```java
/**
 * 模拟 Spring 启动类操作
 */
public class Main {

    /**
     * 1. Bean定义容器池
     */
    private static final Map<String, BeanDefinition> beanDefinitionMap = Maps.newConcurrentMap();

    /**
     * 1. 一级缓存
     */
    private static final Map<String, Object> singletonObjects = Maps.newConcurrentMap();

    /**
     * 3. 二级缓存
     *      用于分离 成熟bean 与 纯净bean
     *      避免读取到不完整的bean
     */
    private static final Map<String, Object> earlySingletonObjects = Maps.newConcurrentMap();

    public static void main(String[] args) throws Exception {
        // 1. 加载 BeanDefinition
        loadBeanDefinitions();

        // 1. 循环创建 Bean
        for (String beanName : beanDefinitionMap.keySet()) {
            getBean(beanName);
        }

        InstanceA instanceA = (InstanceA) singletonObjects.get("instanceA");
        instanceA.say();
    }

    /**
     * 1. 启动加载Bean定义 -> Bean定义容器池
     */
    public static void loadBeanDefinitions() {
        beanDefinitionMap.put("instanceA", new RootBeanDefinition(InstanceA.class));
        beanDefinitionMap.put("instanceB", new RootBeanDefinition(InstanceB.class));
    }

    public static Object getBean(String beanName) throws Exception {
        // 3. 先尝试从缓存池获取
        Object singleton = getSingleton(beanName);
        if (singleton != null) {
            return singleton;
        }

        // 1. 实例化
        RootBeanDefinition beanDefinition = (RootBeanDefinition) beanDefinitionMap.get(beanName);
        Class<?> beanClass = beanDefinition.getBeanClass();
        Object instance = beanClass.getDeclaredConstructor().newInstance();

        // 3. 在实例化 bean 后放入 二级缓存
        earlySingletonObjects.put(beanName, instance);

        // 1. 属性赋值
        Field[] fields = beanClass.getDeclaredFields();
        for (Field field : fields) {
            // 1. bean 中属性也是 bean
            if (field.getAnnotation(Autowired.class) != null) {
                // 1. 设置私有属性可操作
                field.setAccessible(true);

                // 1. 为属性赋值
                field.set(instance, getBean(field.getName()));
            }
        }

        // 1. 初始化 { 执行 bean 中可能定义的 init-method 之类的初始化钩子函数 }
        // 这里主要模拟循环依赖，就不演示了

        // 3. 在初始化 bean 后放入 一级缓存
        singletonObjects.put(beanName, instance);

        return instance;
    }

    /**
     * 3. 从各级缓存池中通过 beanName 获取 bean
     */
    public static Object getSingleton(String beanName) {
        Object instance = singletonObjects.get(beanName);
        return instance != null ? instance : earlySingletonObjects.get(beanName);
    }

}
```

### 二级缓存作用

- 避免获取不完整bean