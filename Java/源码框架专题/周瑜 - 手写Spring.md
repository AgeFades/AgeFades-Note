[TOC]

# 周瑜 - 手写Spring

## 目的

1. 了解 Spring 的底层源码启动过程
2. 了解 BeanDefinition、BeanPostProcessor 的概念
3. 了解 Spring 解析配置类 等底层源码工作流程
4. 了解 依赖注入、Aware回调 等底层源码工作流程
5. 了解 Spring AOP 的底层源码工作流程

## 项目结构

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637560179038.png)

## 代码

### Spring

#### Autowired

```java
package com.spring;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 标记自动注入注解
 *
 * @author DuChao
 * @date 2021/11/22 10:03 上午
 */
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD})
public @interface Autowired {
}
```

#### ScopeEnum

```java
package com.spring;

/**
 * bean作用域枚举
 *
 * @author DuChao
 * @date 2021/11/22 10:04 上午
 */
public enum ScopeEnum {

    /**
     * 单例
     */
    singleton,

    /**
     * 原型
     */
    prototype

}
```

#### Component

```java
package com.spring;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 标记对象为bean
 *
 * @author DuChao
 * @date 2021/11/22 10:13 上午
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Component {

    /**
     * bean name
     */
    String value() default "";

    /**
     * bean 作用域, 默认单例
     */
    ScopeEnum scope() default ScopeEnum.singleton;

}
```

#### CompnentScan

```java
package com.spring;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 根据路径,扫描bean
 *
 * @author DuChao
 * @date 2021/11/22 10:14 上午
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface ComponentScan {

    /**
     * 扫描路径
     */
    String value() default "";

}
```

#### BeanDefinition

```java
package com.spring;

/**
 * Bean定义
 *
 * @author DuChao
 * @date 2021/11/22 10:07 上午
 */
public class BeanDefinition {

    /**
     * bean类型
     */
    private Class beanClass;

    /**
     * bean作用域
     */
    private ScopeEnum scope;

    public Class getBeanClass() {
        return beanClass;
    }

    public void setBeanClass(Class beanClass) {
        this.beanClass = beanClass;
    }

    public ScopeEnum getScope() {
        return scope;
    }

    public void setScope(ScopeEnum scope) {
        this.scope = scope;
    }
}
```

#### BeanNameAware

```java
package com.spring;

/**
 * 设置bean name的扩展
 *
 * @author DuChao
 * @date 2021/11/22 10:10 上午
 */
public interface BeanNameAware {

    void setBeanName(String name);

}
```

#### BeanPostProcessor

```java
package com.spring;

/**
 * bean后置处理器
 *
 * @author DuChao
 * @date 2021/11/22 10:11 上午
 */
public interface BeanPostProcessor {

    /**
     * 初始化前调用
     */
    Object postProcessBeforeInitialization(String beanName, Object bean);

    /**
     * 初始化后调用
     */
    Object postProcessAfterInitialization(String beanName, Object bean);

}
```

#### InitializingBean

```java
package com.spring;

/**
 * 初始化bean接口
 *
 * @author DuChao
 * @date 2021/11/22 10:15 上午
 */
public interface InitializingBean {

    /**
     * bean创建属性赋值之后调用
     */
    void afterPropertiesSet();

}
```

#### ApplicationContext

```java
package com.spring;

import java.io.File;
import java.lang.reflect.Field;
import java.net.URL;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

/**
 * Spring IOC容器, 应用全局上下文
 *
 * @author DuChao
 * @date 2021/11/22 10:16 上午
 */
public class ApplicationContext {

    /**
     * bean定义缓存池
     * key: beanName
     * value: bean定义
     */
    private ConcurrentHashMap<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>();

    /**
     * 单例bean缓存池
     * key: beanName
     * value: bean实例
     */
    private ConcurrentHashMap<String, Object> singletonObjects = new ConcurrentHashMap<>();

    /**
     * bean后置处理器集合
     */
    private List<BeanPostProcessor> beanPostProcessorList = new ArrayList<>();

    /**
     * 构造方法,模拟Spring启动流程
     *
     * @param config : 应用配置类
     */
    public ApplicationContext(Class<?> config) {
        scan(config);

        instanceSingletonBean();
    }

    /**
     * 实例化(非懒加载)的单例bean
     * 1. 实例化
     * 2. 属性填充
     * 3. Aware回调
     * 4. 初始化
     * 5. 添加到单例池
     */
    private void instanceSingletonBean() {
        beanDefinitionMap.forEach((beanName, beanDefinition) -> {
            if (ScopeEnum.singleton.equals(beanDefinition.getScope())) {
                singletonObjects.put(beanName, doCreateBean(beanName, beanDefinition));
            }
        });
    }

    /**
     * 基于 BeanDefinition 创建bean
     */
    private Object doCreateBean(String beanName, BeanDefinition beanDefinition) {
        if (beanDefinition == null) {
            System.out.println(beanName + ", bean定义为空");
        }
        Class beanClass = beanDefinition.getBeanClass();
        try {
            // 实例化
            Object instance = beanClass.getDeclaredConstructor().newInstance();

            // 填充属性
            Field[] fields = beanClass.getFields();
            for (Field field : fields) {
                // 判断属性是否标记 @Autowired 注解
                if (field.isAnnotationPresent(Autowired.class)) {
                    // 创建或获取bean、并赋值到实例属性
                    Object bean = getBean(field.getName());
                    field.setAccessible(true);
                    field.set(instance, bean);
                }
            }

            // Aware回调
            if (instance instanceof BeanNameAware) {
                ((BeanNameAware) instance).setBeanName(beanName);
            }

            // 初始化
            if (instance instanceof InitializingBean) {
                ((InitializingBean) instance).afterPropertiesSet();
            }

            // bean后置处理器调用
            for (BeanPostProcessor beanPostProcessor : beanPostProcessorList) {
                instance = beanPostProcessor.postProcessAfterInitialization(beanName, instance);
            }

            return instance;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 通过 beanName 获取 bean
     */
    public Object getBean(String name) {
        // 先从单例池获取
        if (singletonObjects.containsKey(name)) {
            return singletonObjects.get(name);
        }
        // 没有则创建bean、再返回
        return doCreateBean(name, beanDefinitionMap.get(name));
    }

    /**
     * 1. 扫描配置路上注解@ComponentScan路径
     * 2. 获取BeanDefinition
     * 3. 放入 bean定义缓存池
     */
    private void scan(Class<?> config) {
        // 通过配置类上扫描路径,获取类集合
        ComponentScan componentScan = config.getAnnotation(ComponentScan.class);
        List<Class<?>> classes = getBeanClasses(componentScan.value());

        // 遍历类集合,转成BeanDefinition
        assert classes != null;
        for (Class<?> clazz : classes) {
            // 类上没有标记 @Component 注解的, 不作处理
            Component component = clazz.getAnnotation(Component.class);
            if (component == null) {
                continue;
            }

            // 转 beanDefinition、放入缓存池
            BeanDefinition beanDefinition = new BeanDefinition();
            beanDefinition.setBeanClass(clazz);
            beanDefinition.setScope(component.scope());
            beanDefinitionMap.put(component.value(), beanDefinition);

            // 判断是否实现了了 bean后置处理器
            if (BeanPostProcessor.class.isAssignableFrom(clazz)) {
                try {
                    // 加入 bean后置处理器集合
                    beanPostProcessorList.add((BeanPostProcessor) clazz.getDeclaredConstructor().newInstance());
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }

    /**
     * 通过包路径获取类集合
     */
    private List<Class<?>> getBeanClasses(String path) {
        // com.xx.xx 换成 com/xx/xx
        path = path.replace('.', '/');

        // 通过应用类加载器、获取路径资源
        ClassLoader classLoader = this.getClass().getClassLoader();
        URL resource = classLoader.getResource(path);
        assert resource != null;

        File file = new File(resource.getFile());
        // 非目录不处理
        if (!file.isDirectory()) {
            return null;
        }

        List<Class<?>> beanClasses = new ArrayList<>();

        return Arrays.stream(Objects.requireNonNull(file.listFiles()))
                // 获取文件绝对路径
                .map(File::getAbsolutePath)
                // 过滤,保留.class结尾的文件
                .filter(v -> v.endsWith(".class"))
                // 用类加载器加载类、最终收集成一个集合
                .map(v -> {
                    v = v.substring(v.indexOf("com"), v.indexOf(".class"))
                            .replace('/', '.');
                    try {
                        return classLoader.loadClass(v);
                    } catch (ClassNotFoundException e) {
                        e.printStackTrace();
                    }
                    return null;
                }).collect(Collectors.toList());
    }


}
```

### Business

#### OrderService

```java
package com.agefades.service;

public interface OrderService {

    void test();

}
```

#### UserServiceImpl

```java
package com.agefades.service.impl;

import com.spring.Component;

@Component("userService")
public class UserServiceImpl {
}
```

#### OrderServiceImpl

```java
package com.agefades.service.impl;

import com.agefades.service.OrderService;
import com.spring.Autowired;
import com.spring.Component;

@Component("orderService")
public class OrderServiceImpl implements OrderService {

    @Autowired
    private UserServiceImpl userService;

    @Override
    public void test() {
        System.out.println("test");
    }

}
```

#### BeanPostProcessor

```java
package com.agefades.service.impl;

import com.spring.BeanPostProcessor;
import com.spring.Component;

import java.lang.reflect.Proxy;

/**
 * 后置处理器
 *
 * @author DuChao
 * @date 2021/11/22 11:33 上午
 */
@Component("ageFadesBeanPostProcessor")
public class AgeFadesBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(String beanName, Object bean) {
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(String beanName, Object bean) {
        // 模拟切点
        if ("orderService".equals(beanName)) {
            System.out.println("调用[初始化之后的bean后置处理器]");
            // 返回代理类
            return Proxy.newProxyInstance(this.getClass().getClassLoader(), bean.getClass().getInterfaces(), (proxy, method, args) -> {
                System.out.println("[执行切面逻辑]");
                return method.invoke(bean, args);
            });
        }
        // 返回正常bean
        return bean;
    }

}
```

#### AppConfig

```java
package com.agefades;

import com.spring.ComponentScan;

/**
 * 应用配置类
 *
 * @author DuChao
 * @date 2021/11/22 11:24 上午
 */
@ComponentScan("com.agefades.service.impl")
public class AppConfig {
}
```

#### Test

```java
package com.agefades;

import com.agefades.service.OrderService;
import com.spring.ApplicationContext;

/**
 * 测试类
 *
 * @author DuChao
 * @date 2021/11/22 11:34 上午
 */
public class Test {

    public static void main(String[] args) {
        ApplicationContext context = new ApplicationContext(AppConfig.class);
        OrderService orderService = (OrderService) context.getBean("orderService");
        orderService.test();
    }

}
```

## 测试结果

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637560573398.png)

