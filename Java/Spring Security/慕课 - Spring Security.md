# 慕课 - Spring Security

## 开发环境

​	SpringCloud + OAuth2 : 三方认证登录协议

```java
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-oauth2</artifactId>
		</dependency>
```

​	SpringSocial : 第三方登录依赖

```Java
		<dependency>
			<groupId>org.springframework.social</groupId>
			<artifactId>spring-social-config</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.social</groupId>
			<artifactId>spring-social-core</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.social</groupId>
			<artifactId>spring-social-security</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.social</groupId>
			<artifactId>spring-social-web</artifactId>
		</dependency>
```

## Restful 开发

### 	Restful 理解 

​		用 URL 描述资源

​		使用 HTTP 方法描述行为，使用 HTTP 状态码来表示不同结果

​		使用 json 交互数据

​		Restful 只是一种风格，并不是强制标准

### ​	常用注解

```java
@PageableDefault(page = 2,size = 10,sort = "id,asc") Pageable pageable
```

​		SpringDate 提供分页对象，及分页注解。

```java
@RequestMapping(value = "/user/{id::\\d+}")
```

​		映射路径中可以加入 正则表达式

### 	JsonView

​		使用接口来声明多个视图

```java
public interface UserSimpleView{};
```

​		在值的 get 方法上指定视图

```java
@JsonView(UserSimpleView.class)
```

​		在 Controller 中指定视图

```
@JsonView(UserSimpleView.class)
```

### 	@Valid 注解和BindingResult

​		验证请求参数的合法性并处理校验结果

​		在字段上添加校验注解

```java
@NotBlank
private String openId
```

​		在 Controller 方法参数中加入注解

```Java
@Valid ... , BindingResult errors
```

​		在 Controller 方法体中处理字段校验异常

```java
if (errors.hasErrors()){
		// TODO
    }
```

### 	常用校验注解

![UTOOLS1561090247701.png](https://i.loli.net/2019/06/21/5d0c58ca93c5b62766.png)

![UTOOLS1561090418964.png](https://i.loli.net/2019/06/21/5d0c5974655a010267.png)

### 	自定义校验注解

```Java
package com.beluga.web.validator;

import javax.validation.Constraint;
import javax.validation.Payload;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * @Author DuChao
 * @Date 2019-06-21 12:49
 * 自定义字段校验注解
 */
@Target({ElementType.METHOD, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = MyConstraintValidator.class)
public @interface MyConstraint {

    /**
     * 以下三个参数为 Validator 必须属性
     * message : 校验失败后返回消息
     * 以下两个不用具体了解
     */
    String message();

    Class<?>[] groups() default { };

    Class<? extends Payload>[] payload() default { };

}

```

```java
package com.beluga.web.validator;

import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;

/**
 * @Author DuChao
 * @Date 2019-06-21 12:53
 * 自定义校验逻辑类
 * ConstraintValidator : 校验接口
 * 第一个泛型 : 作用在某个注解上
 * 第二个泛型 : 校验的数据类型
 */
public class MyConstraintValidator implements ConstraintValidator<MyConstraint, Object> {

    /**
     * @Autowired
     * private XxxService xxsService
     * 此处可以注入 IOC 内服务
     * 父接口已经注入 IOC 容器，此类不用额外 @Componet 注解
     */

    /**
     * 初始化时逻辑
     * @param constraintAnnotation : 注解对象
     */
    @Override
    public void initialize(MyConstraint constraintAnnotation) {

    }

    /**
     * 校验逻辑
     * @param o : 校验的数据
     * @param constraintValidatorContext :
     * @return
     */
    @Override
    public boolean isValid(Object o, ConstraintValidatorContext constraintValidatorContext) {
        return true;
    }
}

```

### 	Restful Api 错误处理

#### 		Spring Boot 中默认的错误处理机制

​			BasicErrorController : Spring 异常处理源码类

![UTOOLS1561093564397.png](https://i.loli.net/2019/06/21/5d0c65c49d06719572.png)

​			浏览器响应 Html

​			客户端响应 Json

#### 		自定义异常处理

​			浏览器异常 :

​				resources 文件下 error 文件夹

​				404.html

​				500.html ...

​			Json 异常 :

```java
package com.beluga.web.exception;

import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.Map;

/**
 * @Author DuChao
 * @Date 2019-06-21 13:15
 * 自定义异常
 */
@Data
@NoArgsConstructor
public class MyException extends RuntimeException {

    private static final long serialVersionUID = -6112780192479692859L;

    /**
     * 抛出异常需要传递的数据
     */
    private Map<Object,Object> map;

    public MyException(Map<Object,Object> map){
        super("需要返回的异常信息...");
        this.map = map;
        map.put("message",this.getMessage());
    }

}

```

```java
package com.beluga.web.handle;

import com.beluga.util.web.Code;
import com.beluga.util.web.ResultVo;
import com.beluga.web.exception.MyException;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestControllerAdvice;

/**
 * @Author DuChao
 * @Date 2019-06-21 13:20
 * 自定义异常处理器
 */
@RestControllerAdvice
public class MyExceptionHandle {

    /**
     * 作用在某个异常上
     */
    @ExceptionHandler(MyException.class)
    public ResultVo handleUserNotExistException(MyException ex) {
        return ResultVo.result(ex.getMap(), Code.ERROR);
    }

}

```

### 	Restful Api 的拦截

#### ​		过滤器（Filter）

​			Filter 只能拿到请求数据和响应数据。

​			拿不到目标方法或参数。

```java
/**
 * 
 */
package com.imooc.web.filter;

import java.io.IOException;
import java.util.Date;

import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;

/**
 * @author zhailiang
 * @Component 可以加入 IOC 容器
 * 也可以用配置加入 IOC 容器
 */
//@Component	
public class TimeFilter implements Filter {

	/* (non-Javadoc)
	 * @see javax.servlet.Filter#destroy()
	 */
	@Override
	public void destroy() {
		System.out.println("time filter destroy");
	}

	/* (non-Javadoc)
	 * @see javax.servlet.Filter#doFilter(javax.servlet.ServletRequest, javax.servlet.ServletResponse, javax.servlet.FilterChain)
	 */
	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		System.out.println("time filter start");
		long start = new Date().getTime();
		chain.doFilter(request, response);
		System.out.println("time filter 耗时:"+ (new Date().getTime() - start));
		System.out.println("time filter finish");
	}

	/* (non-Javadoc)
	 * @see javax.servlet.Filter#init(javax.servlet.FilterConfig)
	 */
	@Override
	public void init(FilterConfig arg0) throws ServletException {
		System.out.println("time filter init");
	}

}

```

```java
/**
 * 
 */
package com.imooc.web.config;

import java.util.ArrayList;
import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;

import com.imooc.web.filter.TimeFilter;
import com.imooc.web.interceptor.TimeInterceptor;

/**
 * @author zhailiang
 *
 */
@Configuration
public class WebConfig extends WebMvcConfigurerAdapter {
	
	@SuppressWarnings("unused")
	@Autowired
	private TimeInterceptor timeInterceptor;
	
	@Override
	public void addInterceptors(InterceptorRegistry registry) {
//		registry.addInterceptor(timeInterceptor);
	}
	
//	@Bean
	public FilterRegistrationBean timeFilter() {
		
		FilterRegistrationBean registrationBean = new FilterRegistrationBean();
		
		TimeFilter timeFilter = new TimeFilter();
		registrationBean.setFilter(timeFilter);
		
		List<String> urls = new ArrayList<>();
		urls.add("/*");
		registrationBean.setUrlPatterns(urls);
		
		return registrationBean;
		
	}

}

```

#### ​		拦截器（Interceptor）

​			不再赘述，比 Filter 多一个参数，handle 可以拿到目标对象，拿不到参数

​			作用在源码中 DispacherServlet 

​			需要 @Component 并注册在配置类中，如上。

#### ​		切片（Aspect）

![UTOOLS1561095650657.png](https://i.loli.net/2019/06/21/5d0c6de50152452925.png)​		

```java
 		<!-- Aop 切面 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
```

```java
package com.beluga.web.aspect;

import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

import java.util.Date;

/**
 * @Author DuChao
 * @Date 2019-06-21 13:44
 * 自定义 Aop 切片类
 */
@Slf4j
@Aspect
@Component
public class MyAspect {

    /**
     * 在所有 Controller 方法作增强方法
     */
    @Around("execution(* com.beluga.*.controller.*.*(..))")
    public Object handleControllerMethod(ProceedingJoinPoint pjp) throws Throwable {

        // 目标类名
        String targetName = pjp.getTarget().getClass().getName();
        // 目标方法名
        String methodName = pjp.getSignature().getName();

        /**
         * 记录方法入参
         */
        Object[] args = pjp.getArgs();
        for (Object arg : args) {
            log.info("arg : {}",arg);
        }

        /**
         * 记录方法耗时
         */
        long start = new Date().getTime();

        Object object = pjp.proceed();

        log.info(targetName + ":" + methodName + ": 耗时 :"+ (new Date().getTime() - start));

        return object;
    }

}

```

### Restful 使用多线程提高性能

#### 	使用 Runnable 异步处理 Rest 服务

![UTOOLS1561104237930.png](https://i.loli.net/2019/06/21/5d0c8f6edb0da91633.png)

#### 	使用 DeferredResult 异步处理 Rest 服务

![UTOOLS1561104641149.png](https://i.loli.net/2019/06/21/5d0c910237bad40346.png)

## Spring Security

### 	核心功能

​		认证（你是谁）

​		授权（你能干什么）

​		攻击防护（防止伪造身份）



​	

​	