# 徐庶 - Spring AOP

## 画图

[AOP核心概念图](https://www.processon.com/view/link/5ecca5ebe0b34d5f262eae3a)

## 简介

- `AOP` 指在原有代码基础上，做一些增强功能处理，如:
  - 方法执行前
  - 方法返回后
  - 方法抛出异常后
  - ...
- `AOP` 的实现并不是像 Vue 在生命周期里提供钩子方法，
  - 靠的是实现 `代理`，实际运行的实例其实是 `生成的代理类的实例 `

## 核心概念

### 切面

- `Aspect`：指关注点模块化，横切多个对象，如:
  - LogAspect
  - Spring事务管理
  - ...

### 连接点

- `Join point`：代表增强的方法，如:
  - 如 UserController.add() 被 LogAspect 横切，add() 就被称为连接点

## 通知

- `Advice`：在切面某个连接点上执行的增强动作，有多种类型的通知:
  - @Before
  - @After
  - @Around
  - ...

### 目标对象

- `Target`：指要被增强的对象，如:
  - UserController 被 LogAspcet 横切，UserController 就是目标对象

### 切点

- `Pointcut`：匹配连接点的断言
  - 通知 和 切点表达式 相关联，并在满足这个切点的连接点上运行
  - 如:  @Pointcut("@annotation(org.springframework.web.bind.annotation.GetMapping)")

### 顾问

- `Advisor`：是 `Pointcut` 以及 `Advice` 的一个结合包装类，Spring内部封装的类，应用无需关心

### 织入

- `Weaving`：将 `Advice` 切入 `Pointcut` 的过程

### 引入

- `Introductions`：将其他接口和实现，动态引入进 targetClass 中
  - 如: User 有 add()、edit() 方法，引入 SuperUser，即拥有 work() 方法
  - 就是对目标类补充方法的意思，一般没什么用，Spring内部会有使用

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1618886902033.png)

## 代码案例

```java
@Slf4j
@Aspect
@Component
public class LogAspect {

    private final static int MAX_STR_LENGTH = 500;

    /**
     * 切入点
     */
    @Pointcut("@annotation(org.springframework.web.bind.annotation.GetMapping) ||" +
            "@annotation(org.springframework.web.bind.annotation.PostMapping) ||" +
            "@annotation(org.springframework.web.bind.annotation.PutMapping) ||" +
            "@annotation(org.springframework.web.bind.annotation.DeleteMapping) ||" +
            "@annotation(com.agefades.single.common.annotation.DoLog)")
    public void log() {
    }

    /**
     * 在所有 Controller 方法作增强方法
     */
    @Around("log()")
    public Object handleControllerMethod(ProceedingJoinPoint pjp) throws Throwable {
        // 调用目标方法，方法计时，单位毫秒
        TimeInterval timer = DateUtil.timer();
        Object object = pjp.proceed();
        doLog(pjp, null, object, timer.interval());
        return object;
    }

    @AfterThrowing(pointcut = "log()", throwing = "e")
    public void logThrowing(JoinPoint joinPoint, Throwable e) {
        doLog((ProceedingJoinPoint) joinPoint, e.getMessage(), null, null);
    }

    private void doLog(ProceedingJoinPoint pjp, String message, Object result, Long time) {
        // 获取类名、方法名
        String targetName = pjp.getSignature().getDeclaringTypeName();
        String className = targetName.substring(targetName.lastIndexOf(".") + 1);
        String methodName = pjp.getSignature().getName();

        log.info("【请求参数】:" + className + ":" + methodName + ": " + AspectUtil.getParams(pjp));
        if (StrUtil.isNotBlank(message)) {
            log.info("【异常信息】:" + className + ":" + methodName + ": " + message);
        } else {
            String resultStr = JSONUtil.toJsonStr(result);
            if (StrUtil.isNotBlank(resultStr) && resultStr.length() > MAX_STR_LENGTH) {
                resultStr = resultStr.substring(0, MAX_STR_LENGTH) + "......";
            }
            log.info("【响应结果】:" + className + ":" + methodName + ": " + resultStr);
            log.info("【方法耗时】:" + className + ":" + methodName + ": 耗时: " + time + "毫秒");
        }
    }

}
```

## Spring AOP 和 AspectJ 异同

### Spring AOP

- 基于 `动态代理` 实现
  - 实现接口时，使用 JDK动态代理实现（默认）
  - 基于继承时，使用 CGLIB 动态代理实现
- 依赖于 IOC 容器，只能作用于 Spring IOC 容器中的 Bean
- 容器启动时，需要生成代理实例，在方法调用上，会增加 栈的深度，性能不如 AspectJ
- Spring AOP 已经基本满足普通应用开发 对切面 的使用需求

### AspectJ

- 属于 `静态织入`，通过修改代码实现，织入时机有:
  - `Compile-time weaving`：编译期织入
  - `Post-compile weaving`：编译后织入
  - `Load-time weaving`：类加载织入
- 功能比 Spring AOP 更加强大，性能更好，但使用更加复杂（所以我选Spring AOP）

## Spring AOP使用

- 目前 Spring AOP 有三种配置方式

  - Spring1.2 `基于接口配置`
    - 课程有代码案例展示，但考虑到现在基本不用，了解即可，故不作记录
  - Spring2.0 `schema-based配置` 
    - 即 XML 配置，<aop></aop>
  - Spring2.0 `基于注解配置`
    - 即现在通用的 @Aspect、@Before...

- Spring AOP 使用了 `责任链模式` 来对 Advice 和 Interceptor 进行调用
- `Advisor`：对 `Advice` 和 `Interceptor` 的包装
  - Advisor 决定拦截哪些方法
  - Advice 对拦截方法进行增强处理
- Advisor 的类型:
  - `NameMatchMethodPointcutAdvisor`：根据名称匹配
  - `RegexpMethodPointcutAdvisor`：根据正则匹配
- 上述横切作用范围太受限，每次还是对单个 Bean 做代理，这里引入 AutoPorxy
  - `BeanNameAutoProxyCreator`：根据Bean名称创建代理
  - `DefaultAdvisorAutoProxyCreator`：自动匹配创建代理,一般搭配 `RegexMethodPointcutAdvisor` 配置

## Spring AOP源码

### 使用现象

1. 切面类上加 @Aspect 注解
2. 定义一个 Pointcut 方法
3. 定义系列增强方法

### 分析思路

1. 找到所有的切面类
2. 解析出所有的 Advice 并保存
3. 创建一个动态代理类
4. 调用目标类方法时，找到其代理类，并依次调用 Advice 对目标方法进行增强

![](https://note.youdao.com/yws/public/resource/30678c0adbb76ce345399b05ea785959/xmlnote/C38E8FA12B1B4B2886053D70201B491E/6492)

### 一、切面类的解析

1. 通过 `@EnableAspectJAutoProxy` 开启 AOP 切面

   1. ![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1618889792637.png)

2. `@Import` 引入 `AspectJAutoProxyRegistrar`

   1. ![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1618890010411.png)

3. `AspectJAutoProxyRegistrar` 实现了 `ImportBeanDefinitionRegistrar`

   1. 可以通过 `registerBeanDefinitions()` 给 IOC 容器注入 BeanDefinition
   2. 类似的操作都是通过 Spring 提供的各种拓展点完成的（Aware、BeanPostProcessor...） 
   3. ![](https://note.youdao.com/yws/public/resource/30678c0adbb76ce345399b05ea785959/xmlnote/B04D31F8D87546ADAC9D59704BDD608A/6477)

   

