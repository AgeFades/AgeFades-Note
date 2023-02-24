[TOC]

# 周瑜 - SpringMVC启动与请求处理流程解析

## 课程内容

1. SpringMVC处理请求的基本流程分析
2. 四种Handler的作用与源码实现
3. 三种HandlerMapping的作用与源码实现
4. 四种HandlerAdapter的作用与源码实现
5. 方法参数解析器的作用及源码实现
6. 方法返回值处理器的作用及源码实现

[原理流程图](https://note.youdao.com/s/bK1Rd9zY)

## SpringMVC的初始化启动过程

- 在使用 SpringMVC 时，传统的方式是定义 web.xml，比如：

```xml
<web-app>

    <servlet>
        <servlet-name>app</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/spring.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>app</servlet-name>
        <url-pattern>/app/*</url-pattern>
    </servlet-mapping>

</web-app>
```

- 只要定义一个这样的 web.xml，然后启动 Tomcat，就可以正常使用 SpringMVC 了



- SpringMVC中，最为核心的就是 DispatcherServlet，在启动 Tomcat 的过程中：
  1. Tomcat 会先创建 DispatcherServlet 对象
  2. 然后调用 DispatcherServlet 对象的 init()



- 在 init() 方法中，会创建一个 Spring 容器，并添加一个 ContextRefreshListener 监听器
  - 该监听器会监听 ContextRefreshedEvent 事件（Spring容器启动完成后就会发布该事件）
  - 即 Spring容器启动完成后，就会执行 ContextRefreshListener 中的 onApplicationEvent()
  - 从而最终执行 DispatcherServlet 的 initStrategies()
  - 该方法里就会初始化更多内容...

```java
protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context);
    initLocaleResolver(context);
    initThemeResolver(context);

    initHandlerMappings(context);
    initHandlerAdapters(context);

    initHandlerExceptionResolvers(context);
    initRequestToViewNameTranslator(context);
    initViewResolvers(context);
    initFlashMapManager(context);
}
```

- 其中最为核心的就是 **HandlerMapping 和 HandlerAdapter**

## Handler

- 表示请求处理器，在 SpringMVC 中有四种 Handler

### 1. 实现了Controller接口的Bean对象

```java
@Component("/test")
public class ZhouyuBeanNameController implements Controller {

    @Override
    public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
        System.out.println("zhouyu");
        return new ModelAndView();
    }
}
```

### 2. 实现了HttpRequestHandler接口的Bean对象

```java
@Component("/test")
public class ZhouyuBeanNameController implements HttpRequestHandler {

    @Override
    public void handleRequest(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        System.out.println("zhouyu");
    }
}
```

### 3. 添加了@RequestMapping注解的方法

```java
@RequestMapping
@Component
public class ZhouyuController {

    @Autowired
    private ZhouyuService zhouyuService;

    @RequestMapping(method = RequestMethod.GET, path = "/test")
    @ResponseBody
    public String test(String username) {
        return "zhouyu";
    }

}
```

### 4. 一个HandlerFunction对象

```java
@ComponentScan("com.zhouyu")
@Configuration
public class AppConfig {

    @Bean
    public RouterFunction<ServerResponse> person() {
        return route()
                .GET("/app/person", request -> ServerResponse.status(HttpStatus.OK).body("Hello GET"))
                .POST("/app/person", request -> ServerResponse.status(HttpStatus.OK).body("Hello POST"))
                .build();
    }
    
}
```

## HandlerMapping

- 负责寻找Handler，并保存路径和Handler之间的映射关系



- 因为有不同类型的Handler，所以SpringMVC中会由不同的HandlerMapping来负责寻找Handler
  1. BeanNameUrlHandlerMapping：负责 Controller 接口和 HttpRequestHandler 接口
  2. RequestMappingHandlerMapping：负责@RequestMapping的方法
  3. RouterFunctionMapping：负责RouterFunction以及其中的HandlerFunction



### BeanNameUrlHandlerMapping

- BeanNameUrlHandlerMapping 的寻找过程：
  1. 找出 Spring 容器中所有的 beanName
  2. 判断 beanName 是不是以 "/" 开头
  3. 如果是，则把它当做一个Handler，并把beanName作为key，bean对象作为value 存入 **handlerMap** 



### RequestMappingHandlerMapping

- RequestMappingHandlerMapping的寻找过程：
  1. 找出 Spring 容器中所有 beanType
  2. 判断 beanType 是不是有 @Controller 注解，或是不是有 @RequestMapping 注解
  3. 判断成功则继续找 beanType 中加了 @RequestMapping 的 Method
  4. 解析 @RequestMapping 中的内容，比如 method、path，封装为一个 RequestMappingInfo 对象
  5. 最后把 RequestMappingInfo 对象作为 key，Method 对象封装为 HandlerMethod 对象后作为 value，存入 **registry（一个Map）** 中



### RouterFunctionMapping

- RouterFunctionMapping 寻找流程大差不差，相当于一个 path 对应一个 HandlerFunction



### AbstractHandlerMapping

- 各个 HandlerMapping 除开负责寻找 Handler 并记录映射关系之外，自然还需要根据请求路径找到对应的 Handler，在源码中，这三个 HandlerMapping 有一个共同的父类 AbstractHandlerMapping

![](https://note.youdao.com/yws/public/resource/45023907f226e47c11b2f64596124ff3/xmlnote/WEBRESOURCE01676546162998/8904)

- AbstractHandlerMapping 实现了 HandlerMapping 接口，并实现了 getHandler() 方法
  - 会负责调用子类的 getHandlerInternal() 找到请求对应的 Handler
  - 然后负责将 Handler 和应用中所配置的 HandlerInterceptor 整合成为一个 HandlerExecutionChain 对象
- 所以寻找 Handler 的源码实现在各个 HandlerMapping 子类中的 getHandlerInternal() 中
  - 根据请求路径找到 Handler 的过程并不复杂，因为路径和Handler的映射关系已经存在Map中了



### 如何确认使用哪一个HandlerMapping

- 难点在于：当 DispatcherServlet 接收到一个请求时，该用哪个 HandlerMapping 来寻找 Hnadler？

```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
        for (HandlerMapping mapping : this.handlerMappings) {
            HandlerExecutionChain handler = mapping.getHandler(request);
            if (handler != null) {
                return handler;
            }
        }
    }
    return null;
}
```

- 就是遍历，找到就返回，默认顺序为：

![](https://note.youdao.com/yws/public/resource/45023907f226e47c11b2f64596124ff3/xmlnote/WEBRESOURCE11676546162986/8905)

- 所以 BeanNameUrlHandlerMapping 的优先级最高

## HandlerAdapter

- 找到 Handler 之后，就该执行里面的逻辑了（就是找到接口，执行接口的逻辑）



- 由于有不同种类的Handler，所以执行方法是不一样的，再来总结一下 Handler 的类型：
  1. 实现了Controller接口或HttpRequestHandler接口的bean对象，执行的是其 handlerRequest()
  2. 添加了@RequestMapping的Method，执行就是当前加了注解的方法
  3. HandlerFunction 对象，执行的是 HandlerFunction 对象中的 handler()



- 所以这个代码逻辑可能会是这样...

```java
Object handler = mappedHandler.getHandler();
if (handler instanceof Controller) {
    ((Controller)handler).handleRequest(request, response);
} else if (handler instanceof HttpRequestHandler) {
    ((HttpRequestHandler)handler).handleRequest(request, response);
} else if (handler instanceof HandlerMethod) {
    ((HandlerMethod)handler).getMethod().invoke(...);
} else if (handler instanceof HandlerFunction) {
    ((HandlerFunction)handler).handle(...);
}
```



- 但是SpringMVC不是这么写的，而是采用的 **适配模式**，把不同种类的 Handler 适配成一个 HandlerAdapter，后续再执行 HandlerAdapter 的 handler()
- 针对不同的Handler，会有不同的适配器：
  1. HttpRequestHandlerAdapter
  2. SimpleControllerHandlerAdapter
  3. RequestMappingHandlerAdapter
  4. HandlerFunctionAdapter



- 适配逻辑为：

```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    if (this.handlerAdapters != null) {
        for (HandlerAdapter adapter : this.handlerAdapters) {
            if (adapter.supports(handler)) {
                return adapter;
            }
        }
    }
    throw new ServletException("No adapter for handler [" + handler +
                               "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}
```

- 传入handler，遍历上面4个Adapter，谁支持就返回谁，比如判断的代码依次为：

```java
public boolean supports(Object handler) {
    return (handler instanceof HttpRequestHandler);
}

public boolean supports(Object handler) {
    return (handler instanceof Controller);
}

public final boolean supports(Object handler) {
    return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
}

public boolean supports(Object handler) {
    return handler instanceof HandlerFunction;
}
```



- 根据 Handler 适配出了对应的 HandlerAdapter 后，就执行具体 HandlerAdapter 对象的 handler() 了

  - HttpRequestHandlerAdapter.handler()

    - ```java
      public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
                  throws Exception {
          ((HttpRequestHandler) handler).handleRequest(request, response);
          return null;
      }
      ```

  - SimpleControllerHandlerAdapter.handler()

    - ```java
      public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
                  throws Exception {
          return ((Controller) handler).handleRequest(request, response);
      }
      ```

  - HandlerFunctionAdapter.handler()

    - ```java
      HandlerFunction<?> handlerFunction = (HandlerFunction<?>) handler;
      serverResponse = handlerFunction.handle(serverRequest);
      ```
  
- 因为这三个接收的就是 Request 对象，不用 SpringMVC  做额外的解析，比较简单

- RequestMappingHandlerAdapter 比较复杂，它执行的是加了@RequestMapping 的方法

  - SpringMVC需要根据方法的定义去解析Request对象，从请求中获取出对应的数据然后传递给方法，并执行




### @RequestMapping 方法参数解析

- SpringMVC接收到请求，并找到对应的Method之后，就要执行该方法了
  - 在执行之前，需要根据方法定义的参数信息，从请求中获取出对应的数据，然后将数据传给方法并执行



- 一个 HttpServletRequest 通常有：
  1. request parameter：@RequestParam
  2. request attribute：@RequestAttribute
  3. request session：@SessionAttribute
  4. request header：@RequestHeader
  5. request body：@RequestBody



- SpringMVC解析方法参数，是通过 HandlerMethodArgumentResolver 来实现的

  1. RequestParamMethodArgumentResolver：负责处理@RequestParam
  2. RequestHeaderMethodArgumentResolver：负责处理@RequestHeader
  3. SessionAttributeMethodArgumentResolver：负责处理@SessionAttribute
  4. RequestAttributeMethodArgumentResolver：负责处理@RequestAttribute
  5. RequestResponseBodyMethodProcessor：负责处理@RequestBody
  6. 还有很多其他的...

- 而在判断某个参数该由哪个 HandlerMethodArgumentResolver 处理时，也比较粗暴
  - 即遍历所有的 HandlerMethodArgumentResolver，哪个支持就由哪个处理

```java
private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
    
    HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
    if (result == null) {
        for (HandlerMethodArgumentResolver resolver : this.argumentResolvers) {
            if (resolver.supportsParameter(parameter)) {
                result = resolver;
                this.argumentResolverCache.put(parameter, result);
                break;
            }
        }
    }
    return result;

}
```



- 示例：

```java
@RequestMapping(method = RequestMethod.GET, path = "/test")
@ResponseBody
public String test(@RequestParam @SessionAttribute String username) {
    System.out.println(username);
    return "zhouyu";
}
```

- 这里的 username 将对应 RequestParam 中的 username，而不是 session 的
  - 因为在源码中 RequestParamMethodArgumentResolver 更靠前



- 当然，HandlerMethodArgumentResolver 也会负责从 request 中获取对应的数据，对应的是 resolveArgument() 



- 比如：RequestParamMethodArgumentResolver

```java
protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
    HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);

    if (servletRequest != null) {
        Object mpArg = MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
        if (mpArg != MultipartResolutionDelegate.UNRESOLVABLE) {
            return mpArg;
        }
    }

    Object arg = null;
    MultipartRequest multipartRequest = request.getNativeRequest(MultipartRequest.class);
    if (multipartRequest != null) {
        List<MultipartFile> files = multipartRequest.getFiles(name);
        if (!files.isEmpty()) {
            arg = (files.size() == 1 ? files.get(0) : files);
        }
    }
    if (arg == null) {
        String[] paramValues = request.getParameterValues(name);
        if (paramValues != null) {
            arg = (paramValues.length == 1 ? paramValues[0] : paramValues);
        }
    }
    return arg;
}
```

- 核心是：

```java
if (arg == null) {
    String[] paramValues = request.getParameterValues(name);
    if (paramValues != null) {
        arg = (paramValues.length == 1 ? paramValues[0] : paramValues);
    }
}
```

- 同样的思路，可以找到方法中每个参数所要求的值，从而执行方法，得到方法的返回值



### @RequestMapping 方法返回值解析

- 方法返回值，也会分为不同的情况，比如该方法返回一个 String
  1. 加了@ResponseBody，表示直接将该String响应到浏览器
  2. 没有，则表示应该根据这个String找到对应的页面，把页面返回给浏览器



- SpringMVC中，会利用 HandlerMethodReturnValueHandler 来处理返回值
  1. RequestResponseBodyMethodProcessor：处理加了@ResponseBody注解的情况
  2. ViewNameMethodReturnValueHandler：处理没有加@ResponseBody注解并且返回值类型为String的情况
  3. ModelMethodProcessor：处理返回值是Model类型的情况
  4. 还有很多其他的...



- 这里只记 RequestResponseBodyMethodProcessor，因为目前使用最多的就是标注了 @ResponseBody 的



#### HttpMessageConver

- RequestResponseBodyMethodProcessor 会把方法返回的对象直接响应给浏览器，可能是 Map、业务对象等，该怎么把这些复杂对象响应给浏览器？
  - SpringMVC用**HttpMessageConvert**来处理，默认情况下，有4个HttpMessageConvert



1. BateArrayHttpMessageConverer：处理返回值为 **字节数组** 的情况，把字节数组返回给浏览器
2. StringHttpMessageConverter：处理返回值为 **字符串** 的情况，把字符串按指定的编码序列化后返回
3. SourceHttpMessageConverter：处理返回值为 **XML** 的情况，比如把 DOMSource 对象返回
4. AllEncompassingFormHttpMessageConverter：处理返回值为 **MultiValueMap对象** 的情况



- StringHttpMessageConverter 源码也比较简单

```java
protected void writeInternal(String str, HttpOutputMessage outputMessage) throws IOException {
    HttpHeaders headers = outputMessage.getHeaders();
    if (this.writeAcceptCharset && headers.get(HttpHeaders.ACCEPT_CHARSET) == null) {
        headers.setAcceptCharset(getAcceptedCharsets());
    }
    Charset charset = getContentTypeCharset(headers.getContentType());
    StreamUtils.copy(str, charset, outputMessage.getBody());
}
```

- 先看有没有设置 Content-Type，没有设置则去默认的（ISO-8859-1），所以默认情况中文会乱码
  - 可以通过如下方式解决

```java
@RequestMapping(method = RequestMethod.GET, path = "/test", produces = {"application/json;charset=UTF-8"})
@ResponseBody
public String test() {
    return "周瑜";
}
```

或者

```java
@ComponentScan("com.zhouyu")
@Configuration
@EnableWebMvc
public class AppConfig implements WebMvcConfigurer {

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        StringHttpMessageConverter messageConverter = new StringHttpMessageConverter();
        messageConverter.setDefaultCharset(StandardCharsets.UTF_8);
        converters.add(messageConverter);
    }
}
```



- 不过上述的Converter都不能处理Map或业务对象，需要单独配置一个Converter
  - 比如：**MappingJackson2HttpMessageConverter**
  - 这个Converter比较强大，能把String、Map、User对象 ... 都能转为JSON格式

```java
@ComponentScan("com.zhouyu")
@Configuration
@EnableWebMvc
public class AppConfig implements WebMvcConfigurer {

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        MappingJackson2HttpMessageConverter messageConverter = new MappingJackson2HttpMessageConverter();
        messageConverter.setDefaultCharset(StandardCharsets.UTF_8);
        converters.add(messageConverter);
    }
}
```

- 具体转化的逻辑就是 Jackson2 的转化逻辑了
