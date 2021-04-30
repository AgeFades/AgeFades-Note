[TOC]

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

### 动态代理问题

```java
/**
 * 模拟 Spring 启动类操作
 */
public class Main {

    /**
     * 1. Bean定义容器池
     */
    private static final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>();

    /**
     * 1. 一级缓存
     */
    private static final Map<String, Object> singletonObjects = new ConcurrentHashMap<>();

    /**
     * 3. 二级缓存
     *      用于分离 成熟bean 与 纯净bean
     *      避免读取到不完整的bean
     */
    private static final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>();
    
    public static void main(String[] args) throws Exception {
        // 1. 加载 BeanDefinition
        loadBeanDefinitions();

        // 4. 注册 Bean后置处理器

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

    /**
     * 4. 假设 beanA 被切点标记, (AspectJ 的切面表达式), 此时需要为 beanA 创建动态代理类
     */
    public static Object getBean(String beanName) throws Exception {
        // 2. 先尝试从缓存池获取
        Object singleton = getSingleton(beanName);
        if (singleton != null) {
            return singleton;
        }

        // 1. 实例化
        RootBeanDefinition beanDefinition = (RootBeanDefinition) beanDefinitionMap.get(beanName);
        Class<?> beanClass = beanDefinition.getBeanClass();
        Object instance = beanClass.getDeclaredConstructor().newInstance();

        // 4. 实例化之后，如果产生循环依赖，在这里创建 bean 的动态代理（正常是在 bean 初始化之后）
        // 解耦的方式，使用 BeanPostProcessor 后置处理器机制完成 bean 的动态代理创建
        instance = new JdkProxyBeanPostProcessor().getEarlyBeanReference(instance, beanName);

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

        // 4. beanA 的动态代理类, 正常是在 初始化 之后创建, 但发生 循环依赖 时不能在 初始化 后创建
        // 因为 初始化 之前，先做了 属性赋值, 在此处才创建 动态代理类 的话, beanB 中的 beanA 属性就不是动态代理类了

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

### 满足Spring设计

```java
/**
 * 模拟 Spring 启动类操作
 */
public class Main {

    /**
     * 1. Bean定义容器池
     */
    private static final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>();

    /**
     * 1. 一级缓存
     */
    private static final Map<String, Object> singletonObjects = new ConcurrentHashMap<>();

    /**
     * 3. 二级缓存
     *      用于分离 成熟bean 与 纯净bean
     *      避免读取到不完整的bean
     */
    private static final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>();

    public static void main(String[] args) throws Exception {
        // 1. 加载 BeanDefinition
        loadBeanDefinitions();

        // 4. 注册 Bean后置处理器

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

    /**
     * 4. 假设 beanA 被切点标记, (AspectJ 的切面表达式), 此时需要为 beanA 创建动态代理类
     */
    public static Object getBean(String beanName) throws Exception {
        // 2. 先尝试从缓存池获取
        Object singleton = getSingleton(beanName);
        if (singleton != null) {
            return singleton;
        }

        // 1. 实例化
        RootBeanDefinition beanDefinition = (RootBeanDefinition) beanDefinitionMap.get(beanName);
        Class<?> beanClass = beanDefinition.getBeanClass();
        Object instance = beanClass.getDeclaredConstructor().newInstance();

        // 4. 实例化之后，如果产生循环依赖，在这里创建 bean 的动态代理（正常是在 bean 初始化之后）
        // 解耦的方式，使用 BeanPostProcessor 后置处理器机制完成 bean 的动态代理创建
//        instance = new JdkProxyBeanPostProcessor().getEarlyBeanReference(instance, beanName);

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

        // 4. beanA 的动态代理类, 正常是在 初始化 之后创建, 但发生 循环依赖 时不能在 初始化 后创建
        // 因为 初始化 之前，先做了 属性赋值, 在此处才创建 动态代理类 的话, beanB 中的 beanA 属性就不是动态代理类了

        // 5. Spring 的设计, 正常Bean(没有循环依赖) 需要创建动态代理的, 仍然放在对象的 初始化 之后创建
        // 所以, 引入了 三级缓存 的概念, 用于判断当前 bean 是否处于 循环依赖 中

        // 3. 在初始化 bean 后放入 一级缓存
        singletonObjects.put(beanName, instance);

        return instance;
    }

    /**
     * 3. 从各级缓存池中通过 beanName 获取 bean
     */
    public static Object getSingleton(String beanName) {
        Object instance = null;

        // 5. 先从一级缓存中获取 bean、非空则返回
        instance = singletonObjects.get(beanName);
        if (instance != null) return instance;

        // 5. 再从二级缓存中获取 bean
        instance = earlySingletonObjects.get(beanName);
        if (instance != null) {
            // 5. 如果二级缓存中有 bean, 则表明当前处于 循环依赖 中
            // 可以在在这里创建动态代理, 完成 Spring 的设计
            instance = new JdkProxyBeanPostProcessor().getEarlyBeanReference(instance, beanName);
            earlySingletonObjects.put(beanName, instance);
        }

        return instance;
    }

}
```

## 三级缓存解决循环依赖代码

```java
/**
 * 模拟 Spring 启动类操作
 */
public class Main {

    /**
     * 1. Bean定义容器池
     */
    private static final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>();

    /**
     * 1. 一级缓存
     */
    private static final Map<String, Object> singletonObjects = new ConcurrentHashMap<>();

    /**
     * 3. 二级缓存
     *      用于分离 成熟bean 与 纯净bean
     *      避免读取到不完整的bean
     */
    private static final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>();

    /**
     * 6. 三级缓存
     */
    private static final Map<String, ObjectFactory<?>> singletonFactories = new ConcurrentHashMap<>();

    private static final Set<String> singletonsCurrentlyInCreation = new HashSet<>();

    public static void main(String[] args) throws Exception {
        // 1. 加载 BeanDefinition
        loadBeanDefinitions();

        // 4. 注册 Bean后置处理器

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

    /**
     * 4. 假设 beanA 被切点标记, (AspectJ 的切面表达式), 此时需要为 beanA 创建动态代理类
     */
    public static Object getBean(String beanName) throws Exception {
        // 2. 先尝试从缓存池获取
        Object singleton = getSingleton(beanName);
        if (singleton != null) {
            return singleton;
        }

        // 6. 标记为正在创建
        if (!singletonsCurrentlyInCreation.contains(beanName)) {
            singletonsCurrentlyInCreation.add(beanName);
        }

        // 6. createBean()...

        // 1. 实例化
        RootBeanDefinition beanDefinition = (RootBeanDefinition) beanDefinitionMap.get(beanName);
        Class<?> beanClass = beanDefinition.getBeanClass();
        Object instance = beanClass.getDeclaredConstructor().newInstance();

        // 6. 往三级缓存存入 创建bean动态代理 的钩子方法
        singletonFactories.put(beanName, () -> new JdkProxyBeanPostProcessor().getEarlyBeanReference(earlySingletonObjects.get(beanName), beanName));

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

        // 4. beanA 的动态代理类, 正常是在 初始化 之后创建, 但发生 循环依赖 时不能在 初始化 后创建
        // 因为 初始化 之前，先做了 属性赋值, 在此处才创建 动态代理类 的话, beanB 中的 beanA 属性就不是动态代理类了

        // 5. Spring 的设计, 正常Bean(没有循环依赖) 需要创建动态代理的, 仍然放在对象的 初始化 之后创建
        // 所以, 引入了 三级缓存 的概念, 用于判断当前 bean 是否处于 循环依赖 中

        // 6. 属性赋值之后，A还是原实例，要从二级缓存中拿到 proxy
        if (earlySingletonObjects.containsKey(beanName)) {
            instance = earlySingletonObjects.get(beanName);
        }

        // 3. 在初始化 bean 后放入 一级缓存
        singletonObjects.put(beanName, instance);

        // 6. 删除二级缓存、三级缓存、正在创建标识

        return instance;
    }

    /**
     * 3. 从各级缓存池中通过 beanName 获取 bean
     */
    public static Object getSingleton(String beanName) {
        Object instance;

        // 5. 先从一级缓存中获取 bean、非空则返回
        instance = singletonObjects.get(beanName);
        if (instance != null) return instance;

        // 6. 如果正在创建，则为循环依赖
        if (singletonsCurrentlyInCreation.contains(beanName)) {
            // 6. 二级缓存中已存在，则返回
            if (earlySingletonObjects.containsKey(beanName)) {
                return earlySingletonObjects.get(beanName);
            }

            // 6. 二级缓存中不存在，从三级缓存获取钩子方法创建代理对象
            ObjectFactory<?> objectFactory = singletonFactories.get(beanName);
            if (objectFactory != null) {
                // 6. 创建 & 放入二级缓存
                instance = objectFactory.getObject();
                earlySingletonObjects.put(beanName, instance);
            }

        }
        
        return instance;
    }

}
```

### 源码截图

```java
public class DefaultSingletonBeanRegistry
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1614222334113.png)

## 流程图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1614223307658.png)

## 为什么需要二级缓存

- 为了分离 `成熟Bean` 和 `纯净Bean` 的存放
- 防止多线程下，Bean还未创建完成时、读取到不完整的Bean(未属性赋值)

### 三级缓存下二级缓存的意义

- 二级缓存用于存储 三级缓存创建出来的早期Bean，避免 三级缓存 重复执行

## 为什么需要三级缓存

- Spring 中，getBean() 时，需创建AOP代理情况如下:
  - 正常Bean的代理对象，是在 初始化 之后
  - 循环依赖时，Bean的代理对象，是在 实例化 之后
- 上述情况，用 二级缓存 判断 也行，也可以满足 Spring 的设计
-  这里教程理解是:
  - Spring 为了解耦、扩展、性能、代码阅读性... 
  - 使用三级缓存做职责拆分、单一职责设计

## Spring不能解决构造器的循环依赖

- 在 Bean 实例化之前，三级缓存内没有任何 Bean 的信息
- 实例化时，Bean 的构造函数触发，属性循环依赖当前Bean
- 找不到Bean，如此反复，就抛出 循环依赖 异常

## Spring不能解决多例Bean的循环依赖

- 每次 getBean() 时，创建出来的都是一个新 Bean
- 就意味着，无法从 三级缓存 取出之前对象
- 如此反复，每次都是新的对象引用，就抛出 循环依赖 异常

## Spring的循环依赖解决可以关闭吗

- 可以，具体懒得记（还没碰到过这种需求场景）

## Spring读取不完整Bean的解决原理

- getSingleton() 和 createBean() 两个方法，对一级缓存、二级缓存进行加锁
  - 如: 两个线程同时 getBean(A)
  - 1线程先标记为正在创建、并存入三级缓存 Function，并准备进行实例化、属性赋值、初始化等...
  - 2线程进入，一级缓存未找到Bean，且该Bean正在创建，向下执行碰到锁，阻塞等待，直至1线程完成Bean的创建，2线程即可在 getBean() 从一级缓存取到完整的Bean

