# 引入依赖

​		Swagger引入版本依赖（当时好像是因为版本问题后面排除mdels和注解然后另外依赖），如果直接引入也没问题可以直接引入即可不用排除

```xml
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>2.9.2</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>2.9.2</version>
            <exclusions>
                <exclusion>
                    <groupId>io.swagger</groupId>
                    <artifactId>swagger-annotations</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>io.swagger</groupId>
                    <artifactId>swagger-models</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>io.swagger</groupId>
            <artifactId>swagger-annotations</artifactId>
            <version>1.5.21</version>
        </dependency>
        <dependency>
            <groupId>io.swagger</groupId>
            <artifactId>swagger-models</artifactId>
            <version>1.5.21</version>
        </dependency>
```

# 编写属性配置

​		我们需要对Swagger等配置文件进行配置，我们使用Properties类统一的将这些配置管理起来。

```java
package com.test.boot.properties;

import lombok.Getter;
import lombok.Setter;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

/**
 * @Author BigKang
 * @Date 2020/11/9 10:10 上午
 * @Motto 仰天大笑撸码去, 我辈岂是蓬蒿人
 * @Summarize Swagger配置
 */
@Getter
@Setter
@Configuration
@ConfigurationProperties(prefix = "swagger")
public class SwaggerProperties {

    /**
     * 获取swagger配置title
     */
    private String title = "标题未设置";

    /**
     * 获取swagger配置description
     */
    private String description = "描述未设置";

    /**
     * 获取swagger配置version
     */
    private String version = "版本未设置";

    /**
     * 获取swagger配置作者
     */
    private String author = "BigKang";

    /**
     * 获取swagger配置文档包路径
     */
    private String docPackage = "com";

    /**
     * 需要排除掉的路径
     */
    private String excludePath = "/error*";
}

```

​		然后我们在配置文件中进行配置

​		yaml格式如下：

```properties
swagger:
  # Swagger标题
  title: Swagger标题
  # 描述
  description: 测试Swagger
  # 版本
  version: V2.0
  # 作者
  author: BigKang
  # 扫描包
  docPackage: com.test
  # 排除的路径
  excludePath: /error*
```

​		properties如下：

```properties
swagger.title=Swagger标题
swagger.description=测试Swagger
swagger.version=V2.0
swagger.author=BigKang
swagger.docPackage=com.test
swagger.excludePath=/error*
```



# 编写配置类

注意此处我们apis对应我们相对应的包下面才进行生成的文档，并且这里需要修改成自己的路径

### 单环境使用

```java

import com.github.xiaoymin.knife4j.spring.annotations.EnableKnife4j;
import com.google.common.base.Predicates;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
import springfox.bean.validators.configuration.BeanValidatorPluginsConfiguration;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger2.annotations.EnableSwagger2;

/**
 * @Author BigKang
 * @Date 2020/3/17 4:27 PM
 * @Summarize Swagger通用配置类
 */
@Configuration
@EnableSwagger2
@Slf4j
public class Swagger2Config {

    private final SwaggerProperties properties;

    @Autowired
    public Knife4jConfig(SwaggerProperties properties) {
        this.properties = properties;
    }

    @Bean
    public Docket webApiConfig() {
        return new Docket(DocumentationType.SWAGGER_2)
                // 调用apiInfo方法,创建一个ApiInfo实例,里面是展示在文档页面信息内容
                .apiInfo(webApiInfo())
                // 创建ApiSelectorBuilder对象
                .select()
                // 扫描的包
                .apis(RequestHandlerSelectors.basePackage(properties.getDocPackage()))
                // 过滤掉错误路径
                .paths(Predicates.not(PathSelectors.regex(properties.getExcludePath())))
                .build();
    }

    /**
     * 创建API——INFO信息
     * @return
     */
    private ApiInfo webApiInfo() {
        return new ApiInfoBuilder()
                .contact(properties.getAuthor())
                .title(properties.getTitle())
                .description(properties.getDescription())
                .version(properties.getVersion())
                .build();
    }

}

```

### 双环境区分

我们这里将api和admin分成两个文档进行区分

```java
@Configuration
@EnableSwagger2
public class Swagger2Config {

    @Bean
    public Docket webApiConfig() {
        return new Docket(DocumentationType.SWAGGER_2)
                .groupName("webApi")
                .apiInfo(webApiInfo())// 调用apiInfo方法,创建一个ApiInfo实例,里面是展示在文档页面信息内容
                .select()//创建ApiSelectorBuilder对象
                .apis(RequestHandlerSelectors.basePackage("com.kang.test"))//扫描的包
                .paths(Predicates.and(PathSelectors.regex("/admin/.*")))//过滤掉admin接口
                .paths(Predicates.not(PathSelectors.regex("/error.*")))//过滤掉错误路径
                .build();
    }
    @Bean
    public Docket adminApiConfig() {
        return new Docket(DocumentationType.SWAGGER_2)
                .groupName("adminApi")
                .apiInfo(adminApiInfo())// 调用apiInfo方法,创建一个ApiInfo实例,里面是展示在文档页面信息内容
                .select()//创建ApiSelectorBuilder对象
                .apis(RequestHandlerSelectors.basePackage("com.kang.test"))//扫描的包
                .paths(Predicates.and(PathSelectors.regex("/api/.*")))//过滤的接口
                .paths(Predicates.not(PathSelectors.regex("/error.*")))//过滤掉错误路径
                .build();
    }

    private ApiInfo adminApiInfo() {
        return new ApiInfoBuilder()
                .title("BigKang----V4.0后台管理接口文档")
                .description("BigKang最新4.0接口文档")
                .termsOfServiceUrl("http://bigkang.club")
                .version("4.0")
                .build();
    }

    private ApiInfo webApiInfo() {
        return new ApiInfoBuilder()
                .title("BigKang----V4.0API接口文档")
                .description("BigKang最新4.0接口文档")
                .termsOfServiceUrl("http://bigkang.club")
                .version("4.0")
                .build();
    }

}

```

# 测试

我们启动项目然后访问项目地址加上/swagger-ui.html即可访问

例如：

```http
www.localhost:8080/swagger-ui.html
```

即可看到如下页面

# Security整合

我们在Security的配置类中加入如下方法即可

```java
    /**
     * 配置Swagger-ui，避免无法使用文档
     * @param web
     * @throws Exception
     */
    @Override
    public void configure(WebSecurity web) throws Exception {
        web.ignoring(). antMatchers("/swagger-ui.html","/oauth/check_token")
                .antMatchers("/webjars/**")
                .antMatchers("/v2/**")
                .antMatchers("/swagger-resources/**");
    }
```

# Swagger常用注解

其实注解还有很多个，但是这里并没有展示全部，这几个注解在我们开发时会经常频繁的使用到

### 实体类注解

```java
@ApiModel(value="用户登录Vo对象",description="用户登录对象")
@Data
public class AuthLoginEntity {


    @ApiModelProperty(value="用户名",name="username",example="Bigkang")
    private String username;

    @ApiModelProperty(value="密码",name="password",example="Bigkang")
    private String password;
}
```

##### @ApiModel

他可以加在我们的实体类上，例如参数实体类，如上value为展示实体类的名，description为描述，如下图所示:

![](https://blog-kang.oss-cn-beijing.aliyuncs.com/UTOOLS1569568004143.png)

##### @ApiModelProperty

​		这个注解一般是加在我们的实体类的属性当中的

​		我们可以看到我们的实体的属性和我们的注解中一样，并且我们在使用swagger默认参数回填的时候也会自动加上比如我们example为Bigkang那么我们在使用文档时也会默认给我们回填为Bigkang，如下图所示：

​	![](https://blog-kang.oss-cn-beijing.aliyuncs.com/UTOOLS1569568252424.png)

### 控制器注解

```java
@RestController
@Api(tags = "登录控制器")
@RequestMapping("login")
public class LoginController {

    @PostMapping("/login")
    @ApiOperation("登录接口")
    public ResultVo login(@RequestBody AuthLoginEntity authLoginEntity){
        return ResultVo.result(Code.OK_CODE, Message.OK);
    }

}
```

##### @Api

我们通过@Api的tags属性表示这个控制器的名字，我们打开后如下图所示，我们的这个controller叫做登录控制器

![](https://blog-kang.oss-cn-beijing.aliyuncs.com/UTOOLS1569569286002.png)

##### @ApiOperation

这个注解我们加在方法上代表一个rest接口方法，如下图所示我们的登录方法我们展示为登录接口

![](https://blog-kang.oss-cn-beijing.aliyuncs.com/UTOOLS1569569436957.png)

我们可以看到



# 升级版Knife4j

Knife4j功能更加强大，同时页面更加美观

将swagger依赖替换为knife4j 官方文档地址：[点击进入](https://doc.xiaominfo.com/docs/quick-start)

## 引入依赖

```xml
       <dependency>
            <groupId>com.github.xiaoymin</groupId>
            <artifactId>knife4j-spring-boot-starter</artifactId>
            <version>2.0.2</version>
        </dependency>

```

​	

## 编写配置

```java
import com.github.xiaoymin.knife4j.spring.annotations.EnableKnife4j;
import com.google.common.base.Predicates;
import com.test.boot.properties.SwaggerProperties;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
import springfox.bean.validators.configuration.BeanValidatorPluginsConfiguration;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger2.annotations.EnableSwagger2;

/**
 * @Author BigKang
 * @Date 2020/3/17 4:27 PM
 * @Summarize Swagger通用配置类
 */
@Configuration
@EnableSwagger2
@EnableKnife4j
@Import(BeanValidatorPluginsConfiguration.class)
public class Knife4jConfig {

    private final SwaggerProperties properties;

    @Autowired
    public Knife4jConfig(SwaggerProperties properties) {
        this.properties = properties;
    }

    @Bean
    public Docket webApiConfig() {
        return new Docket(DocumentationType.SWAGGER_2)
                // 调用apiInfo方法,创建一个ApiInfo实例,里面是展示在文档页面信息内容
                .apiInfo(webApiInfo())
                // 创建ApiSelectorBuilder对象
                .select()
                // 扫描的包
                .apis(RequestHandlerSelectors.basePackage(properties.getDocPackage()))
                // 过滤掉错误路径
                .paths(Predicates.not(PathSelectors.regex(properties.getExcludePath())))
                .build();
    }

    /**
     * 创建API——INFO信息
     * @return
     */
    private ApiInfo webApiInfo() {
        return new ApiInfoBuilder()
                .contact(properties.getAuthor())
                .title(properties.getTitle())
                .description(properties.getDescription())
                .version(properties.getVersion())
                .build();
    }


}


```



# WebFlux整合Swagger2

引入依赖，由于目前Swagger2  3.0并未正式上线，所以我们引入依赖并且配置仓库地址

```xml
    <dependencies>
				<dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>3.0.0-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>3.0.0-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-spring-webflux</artifactId>
            <version>3.0.0-SNAPSHOT</version>
        </dependency>
    </dependencies>
    <repositories>
        <repository>
            <id>spring-snapshots</id>
            <url>http://repo.spring.io/snapshot</url>
            <snapshots>
                <enabled>true</enabled>
            </snapshots>
        </repository>
        <repository>
            <id>swagger-snapshots</id>
            <url>http://oss.jfrog.org/artifactory/oss-snapshot-local</url>
            <snapshots>
                <enabled>true</enabled>
            </snapshots>
        </repository>
        <repository>
            <id>spring-milestones</id>
            <url>http://repo.spring.io/milestone</url>
        </repository>
    </repositories>
```

然后我们这里配置WebFlux的Swagger2

此处注解改为@EnableSwagger2WebFlux

```java
@Configuration
@EnableSwagger2WebFlux
@Slf4j
public class SwaggerConfig {

    //获取swagger配置title
    @Value("${swagger.title:请设置配置}")
    private String title;
    //获取swagger配置description
    @Value("${swagger.description:请设置配置}")
    private String description;
    //获取swagger配置version
    @Value("${swagger.version:请设置配置}")
    private String version;

    @Bean
    public Docket webApiConfig() {
//
//        ParameterBuilder parameterBuilder = new ParameterBuilder();
        List<Parameter> parameters = new ArrayList<Parameter>();
//        parameterBuilder.name("hk") // 参数名
//                .description("token") // 描述
//                .modelRef(new ModelRef("string")) // 类型
//                .parameterType("cookie") // //参数类型支持header, cookie, body, query etc
//                .required(false).build();
//        parameters.add(parameterBuilder.build());


        return new Docket(DocumentationType.SWAGGER_2)
                .globalOperationParameters(parameters) // 添加全局参数（头信息或者Cookie）
                .apiInfo(webApiInfo())// 调用apiInfo方法,创建一个ApiInfo实例,里面是展示在文档页面信息内容
                .select()//创建ApiSelectorBuilder对象
                .apis(RequestHandlerSelectors.basePackage("com"))//扫描的包
                .paths(PathSelectors.any())//过滤掉错误路径
                .build();
    }

    private ApiInfo webApiInfo() {
        return new ApiInfoBuilder()
                .title(title)
                .description(description)
                .version(version)
                .build();
    }

}
```

# 工具类以及通用配置



```java

import com.github.xiaoymin.knife4j.spring.annotations.EnableKnife4j;
import com.google.common.base.Predicates;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
import springfox.bean.validators.configuration.BeanValidatorPluginsConfiguration;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger2.annotations.EnableSwagger2;

/**
 * @Author BigKang
 * @Date 2020/3/17 4:27 PM
 * @Summarize Swagger通用配置类
 */
@Configuration
@EnableSwagger2
@EnableKnife4j
@Slf4j
@Import(BeanValidatorPluginsConfiguration.class)
public class Swagger2Config {

    /**
     * 获取swagger配置title
     */
    @Value("${swagger.title:请设置配置}")
    private String title;

    /**
     * 获取swagger配置description
     */
    @Value("${swagger.description:请设置配置}")
    private String description;

    /**
     * 获取swagger配置version
     */
    @Value("${swagger.version:请设置配置}")
    private String version;

    /**
     * 获取swagger配置作者
     */
    @Value("${swagger.author:BigKang}")
    private String author;

    /**
     * 获取swagger配置文档包路径
     */
    @Value("${swagger.author:org.kang.cloud}")
    private String docPackage;

    @Bean
    public Docket webApiConfig() {
        return new Docket(DocumentationType.SWAGGER_2)
                // 调用apiInfo方法,创建一个ApiInfo实例,里面是展示在文档页面信息内容
                .apiInfo(webApiInfo())
                // 创建ApiSelectorBuilder对象
                .select()
                // 扫描的包
                .apis(RequestHandlerSelectors.basePackage(docPackage))
                // 过滤掉错误路径
                .paths(Predicates.not(PathSelectors.regex("/error.*")))
                .build();
    }

    /**
     * 创建API——INFO信息
     * @return
     */
    private ApiInfo webApiInfo() {
        return new ApiInfoBuilder()
                .contact(author)
                .title(title)
                .description(description)
                .version(version)
                .build();
    }


}
```

