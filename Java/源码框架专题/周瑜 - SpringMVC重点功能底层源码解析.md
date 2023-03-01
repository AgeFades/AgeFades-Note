[TOC]

# 周瑜 - SpringMVC重点功能底层源码解析

## 课程内容

1. 方法参数解析源码分析
2. 文件上传MultipartFile源码解析
3. 方法返回值解析源码分析
4. 视图解析核心源码分析
5. SpringMVC拦截器源码解析
6. @EnableWebMvc源码解析
7. WebApplicationInitializer使用方式
8. SpringMVC父子容器介绍与源码分析

## 流程图

[SpringMVC处理请求核心流程图](https://www.processon.com/view/link/63f4cf1176e6143857799c2a)

## 课堂疑问

1. 当使用 @RequestParam，且没有注册 StringToUserEditor 时，但 User 中提供了一个 String 类型参数的构造方法

```java
@RequestMapping(method = RequestMethod.GET, path = "/test")
@ResponseBody
public String test(@RequestParam("name") User user) {
    return user.getName();
}
```

```java
public class User {

    private String name;

    public User(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

- SpringMVC 在将 String 转成 User 对象时，

  1. 会先判断 User 有没有对应的 StringToUserEditor，有则利用其将String转成User对象
  2. 没有则找User类有没有String类型的构造方法，有则利用其构造创建出User对象

- 对应的源码方法为：TypeConverterDelegate#convertIfNecessary

![](https://note.youdao.com/yws/public/resource/10345a9ca9f3f8e7a8761c7db437b7ae/xmlnote/WEBRESOURCEbbb371e414f377c877a21988802bc625/8958)



2. 如果方法返回的是 byte[]

```java
@RequestMapping(method = RequestMethod.GET, path = "/test")
@ResponseBody
public byte[] test() {
    byte[] bytes = new byte[1024];
    return bytes;
}
```

- 这种情况会直接使用 ByteArrayHttpMessageConverter 来处理，会直接把 byte[] 写入响应中

```java
public class ByteArrayHttpMessageConverter extends AbstractHttpMessageConverter<byte[]> {

    /**
     * Create a new instance of the {@code ByteArrayHttpMessageConverter}.
     */
    public ByteArrayHttpMessageConverter() {
        super(MediaType.APPLICATION_OCTET_STREAM, MediaType.ALL);
    }


    @Override
    public boolean supports(Class<?> clazz) {
        return byte[].class == clazz;
    }

    // ...

    @Override
    protected void writeInternal(byte[] bytes, HttpOutputMessage outputMessage) throws IOException {
        StreamUtils.copy(bytes, outputMessage.getBody());
    }

}
```

## SpringMVC父子容器

- 可以在 web.xml 文件中这样定义:

```xml
<web-app>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/spring.xml</param-value>
    </context-param>

    <servlet>
        <servlet-name>app</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/spring-mvc.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>app</servlet-name>
        <url-pattern>/app/*</url-pattern>
    </servlet-mapping>

</web-app>
```

- 在该 web.xml 中，定义了一个 listener 和 servlet

### 父容器的创建

- ContextLoaderListener 用来创建父容器，执行流程为：
  1. Tomcat 启动，解析 web.xml
  2. 发现 ContextLoaderListener，Tomcat 执行该 listener 的 contextInitialized() 进行创建容器
  3. 从 ServletContext 中获取 contextClass 参数值，即所要创建的容器类型，可在 web.xml 中通过 <context-param> 来进行配置
  4. 如果未配置该参数，就会从 ContextLoader.properties 文件中读取 WebApplicationContext 配置项的值，SpringMVC默认提供了一个该文件，内容为：XmlWebApplicationContext
  5. 即要创建的Spring容器类型为 XmlWebApplicationContext
  6. 确认类型后，会用反射调用 **无参构造方法** 创建出一个该对象
  7. 继续从 ServletContext 中获取 contextConfigLocation 参数的值，也就是一个Spring配置文件的路径
  8. 将该路径设置给 Spring 容器，然后调用 refresh() 启动Spring容器，解析Spring配置文件，扫描生成bean对象...
  9. Spring容器创建完成后，会将XmlWebApplicationContext对象作为attribute设置到ServletContext中去，key为WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE
  10. 将Spring容器存在ServletContext，是为了给Servlet创建出的子容器来作为父容器的

### 子容器的创建

- Tomcat启动中，执行完ContextLoaderListener#contextInitialized()之后，就会创建 DispatcherServlet 了
- web.xml中定义DispatcherServlet时，load-on-startup为1，表示在Tomcat启动过程中就会把它创建并初始化出来，如果这个值不为1，则会在接收到请求后才进行创建和初始化，导致开始的请求速度比较慢



1. Tomcat 启动，解析 web.xml
2. 创建 DispatcherServlet 对象 & 调用其 init()
3. init() 中调用 initServletBean()、里面再调用 initWebApplicationContext()，即创建一个Spring容器(子容器)
4. 在initWebApplicationContext()执行中，会先从ServletContext拿出ContextLoadListener所创建的Spring容器，即父容器，记为 rootContext
5. 再读取contextClass参数值，可从servlet中的<init-param>标签来定义想要创建的Spring容器类型，默认为 XmlWebApplicationContext
6. 再创建一个Spring容器对象，即子容器
7. 将rootContext作为parent设置给子容器（父子关系的绑定）
8. 再读取contextConfigLocation参数值，得到所配置的Spring配置文件路径
9. 再调用Spring容器的refresh()方法，从而完成子容器的创建

### SpringMVC初始化

- 子容器创建完后，还会调用一个 DispatcherServlet 的 onRefresh()
- 该方法会从 Spring 容器中获取一些特殊类型的 bean 对象，并设置给 DispatcherServlet 对应的属性，比如 HandlerMapping、HandlerAdapter
- 流程为：
  1. 先从 Spring 容器获取 HandlerMapping 类型的 bean，如果不为空，就赋值给 DispatcherServlet 的 handlerMappings 属性
  2. 如果为空，则从 DispatcherServlet.properties 文件中读取配置，得到默认的HandlerMapping并赋值
- DispatcherServlet.properties 文件内容如下：

```properties
# Default implementation classes for DispatcherServlet's strategy interfaces.
# Used as fallback when no matching beans are found in the DispatcherServlet context.
# Not meant to be customized by application developers.

org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver

org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
org.springframework.web.servlet.function.support.RouterFunctionMapping

org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter,\
org.springframework.web.servlet.function.support.HandlerFunctionAdapter


org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver

org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
```

- 默认提供了3个HandlerMapping、4个HandlerAdapter，都会用于后续DsipatcherServlet处理请求



- 注意，这些类是为Spring容器创建出来的 bean，会触发 bean 的初始化逻辑
  - 比如 RequestMappingHandlerAdapter，创建时会执行 afterPropertiesSet()



#### RequestMappingHandlerAdapter初始化

- 简单理解作用为：收到请求时，调用请求对应方法
  - 需要解析方法参数、方法返回值
- 在其 afterPropertiesSet() 中，会做如下事：
  1. 从Spring容器找到加了 @ControllerAdvice  的 bean
     1. 解析出 bean 中加了 @ModelAttribute 的 Method 对象，并存于 modelAttributeAdviceCache 这个 map 中
     2. 解析出 bean 中加了 @InitBinder 的 Method 对象，并存于 initBinderAdviceCache 这个 map 中
     3. 如果 bean 实现了 RequestBodyAdvice 接口或 ResponseBodyAdivce 接口，就把该 bean 记于 requestResponseBodyAdvice 集合中
  2. 从Spring容器中获取用户定义的 HandlerMethodArgumentResolver（**用于解析方法参数**），以及 SpringMVC  默认提供的，整合为一个 HandlerMethodArgumentResolverComposite 对象
  3. 从Spring容器中获取用户定义的 HandlerMethodReturnValueHandler（**用于解析方法返回值**），以及SpringMVC默认提供的，整合为一个 HnadlerMethodReturnValuehandlerComposite 对象



#### RequestMappingHandlerMapping初始化

- 作用为：保存定义了哪些 @RequestMapping 方法及对应的访问路径
  - 而该bean的初始化就是去找到这些映射关系



1. 找出容器中定义的所有beanName
2. 根据beanName找出beanType
3. 判断beanType上是否有 @Controller 或 @RequestMapping，如有，则表示该bean为一个Handler
4. 如为Handler，则通过反射找其加了@RequestMapping的Method，并解析该注解上定义的参数信息，得到一个对应的 RequestMappingInfo 对象，然后结合 beanType 上 @RequestMapping 定义的 path，以及当前 Method 上 @RequestMapping 定义的 path，进行整合，得到该 Method对应的访问路径，并设置到 RequestMappingInfo 对象中
5. 把当前 Handler 的所有 RequestMappingInfo 解析创建完后，存到 MappingRegistry 对象中
6. 存到 MappingRegistry 过程中，会先把 Handler 生成一个 HandlerMethod 对象（其实就是表示一个方法）
7. 再获取 RequestMappingInfo 中的 path
8. 把 path 和 HandlerMethod 存在一个 Map 中，属性叫做 pathLookup
9. 这样在处理请求时，就可以通过请求路径找到 HandlerMethod，然后找到 Method，再执行了



### WebApplicationinitializer的方式

