[TOC]

# Knife4j API接口文档

## 官网地址

[官方文档](https://doc.xiaominfo.com/)

## 使用案例

- 默认访问地址后缀 **doc.html**
  - 如：localhost:8080/doc.html

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1673840810295.png)

## 开发使用

- 跟 Swagger 一致，使用注解完成实时API文档，举例：
  - @ApiModel
  - @ApiModelProperty
  - ...
- 没有使用经验的自行参照上面的官方文档

## pom依赖

- 版本按自己需求来，推荐用最新版本

```xml
 <!-- Knife4j -->
<dependency>
  <groupId>com.github.xiaoymin</groupId>
  <artifactId>knife4j-spring-boot-starter</artifactId>
  <version>${knife4j.version}</version>
</dependency>
```

## 配置类

```java
package com.agefades.log.common.core.config;

import com.github.xiaoymin.knife4j.spring.annotations.EnableKnife4j;
import io.swagger.annotations.ApiOperation;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
import springfox.bean.validators.configuration.BeanValidatorPluginsConfiguration;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.service.ApiKey;
import springfox.documentation.service.AuthorizationScope;
import springfox.documentation.service.SecurityReference;
import springfox.documentation.service.SecurityScheme;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spi.service.contexts.SecurityContext;
import springfox.documentation.spring.web.plugins.Docket;

import java.time.LocalTime;
import java.util.Collections;
import java.util.List;
import java.util.function.Predicate;

/**
 * Swagger 配置类
 *
 * @author DuChao
 * @date 2021/1/12 4:06 下午
 */
@Configuration
@EnableKnife4j
@Import(BeanValidatorPluginsConfiguration.class)
public class Swagger3Config {

    @Value("${swagger.title:请设置配置}")
    private String title;

    @Value("${swagger.description:请设置配置}")
    private String description;

    @Value("${swagger.version:请设置配置}")
    private String version;

    @Bean
    public Docket webApiConfig() {
        return new Docket(DocumentationType.SWAGGER_2).
                useDefaultResponseMessages(false)
                .select()
                .apis(RequestHandlerSelectors.withMethodAnnotation(ApiOperation.class))
                .paths(Predicate.not(PathSelectors.regex("/error.*")))
                .build()
                .directModelSubstitute(LocalTime.class, String.class)
                .securitySchemes(securitySchemes())
                .securityContexts(securityContexts())
                .apiInfo(webApiInfo())
                ;
    }

    private List<SecurityScheme> securitySchemes() {
        return Collections.singletonList(new ApiKey("Authorization", "Authorization", "header"));
    }

    private List<SecurityContext> securityContexts() {
        return Collections.singletonList(SecurityContext.builder()
                        .securityReferences(defaultAuth())
                        .forPaths(PathSelectors.regex("^(?!auth).*$"))
                        .build());
    }

    List<SecurityReference> defaultAuth() {
        AuthorizationScope authorizationScope = new AuthorizationScope("global", "accessEverything");
        AuthorizationScope[] authorizationScopes = new AuthorizationScope[1];
        authorizationScopes[0] = authorizationScope;
        return Collections.singletonList(new SecurityReference("Authorization", authorizationScopes));
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

## Gateway聚合各服务文档

```java
package com.agefades.log.gateway.swagger;

import lombok.AllArgsConstructor;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.context.annotation.Primary;
import org.springframework.stereotype.Component;
import springfox.documentation.swagger.web.SwaggerResource;
import springfox.documentation.swagger.web.SwaggerResourcesProvider;

import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

/**
 * 获取具体服务Swagger Api Docs
 *
 * @author DuChao
 * @date 2021/12/6 2:11 下午
 */
@Component
@Primary
@AllArgsConstructor
public class GwSwaggerResourceProvider implements SwaggerResourcesProvider {

    /**
     * swagger2默认的url后缀
     */
    private static final String SWAGGER2URL = "/v2/api-docs";

    /**
     * 用于获取各服务名称
     */
    private DiscoveryClient discoveryClient;

    @Override
    public List<SwaggerResource> get() {
        List<String> services = discoveryClient.getServices();
        List<SwaggerResource> resources = new ArrayList<>();
        // 记录已经添加过的server，存在同一个应用注册了多个服务在nacos上
        Set<String> already = new HashSet<>();
        services.forEach(v -> {
            // 拼接url，样式为/serviceId/v2/api-info，当网关调用这个接口时，会自动通过负载均衡寻找对应的主机
            String url = "/" + v + SWAGGER2URL;
            if (!already.contains(url)) {
                already.add(url);
                SwaggerResource swaggerResource = new SwaggerResource();
                swaggerResource.setUrl(url);
                swaggerResource.setName(v);
                resources.add(swaggerResource);
            }
        });
        return resources;
    }
}
```

```java
package com.agefades.log.gateway.swagger;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import springfox.documentation.swagger.web.SecurityConfiguration;
import springfox.documentation.swagger.web.SecurityConfigurationBuilder;
import springfox.documentation.swagger.web.SwaggerResource;
import springfox.documentation.swagger.web.UiConfiguration;
import springfox.documentation.swagger.web.UiConfigurationBuilder;

import java.util.List;

/**
 * 网关Swagger文档统一端点
 *
 * @author DuChao
 * @date 2021/12/6 2:15 下午
 */
@RestController
@RequestMapping("/swagger-resources")
public class SwaggerResourceController {

    @Autowired
    private GwSwaggerResourceProvider swaggerResourceProvider;

    @RequestMapping(value = "/configuration/security")
    public ResponseEntity<SecurityConfiguration> securityConfiguration() {
        return new ResponseEntity<>(SecurityConfigurationBuilder.builder().build(), HttpStatus.OK);
    }

    @RequestMapping(value = "/configuration/ui")
    public ResponseEntity<UiConfiguration> uiConfiguration() {
        return new ResponseEntity<>(UiConfigurationBuilder.builder().build(), HttpStatus.OK);
    }

    @RequestMapping
    public ResponseEntity<List<SwaggerResource>> swaggerResources() {
        return new ResponseEntity<>(swaggerResourceProvider.get(), HttpStatus.OK);
    }
}
```

