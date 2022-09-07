[TOC]

# 技术分享 - SpringCloud灰度发布方案

## 参考资料

[掘金 - 聊聊 Spring Cloud 全链路灰度发布！](https://juejin.cn/post/7088147514733363237)

注意，本人参照上文操作，未能达到预期效果，可阅读了解基本定义、方案思路

## 0. 灰度标记上下文工具类

```java
package com.agefades.log.common.core.util;

import com.agefades.log.common.core.constants.CommonConstant;
import lombok.extern.slf4j.Slf4j;

/**
 * 当前线程灰度标记上下文工具类
 *
 * @author DuChao
 * @date 2021/12/6 5:13 下午
 */
@Slf4j
public class GrayContextUtil extends ThreadLocalUtil {

    /**
     * 往当前线程ThreadLocal中存储灰度标记
     */
    public static void setGrayTag(Boolean tag) {
        set(CommonConstant.GRAY, tag);
    }

    /**
     * 从当前线程ThreadLocal中取出灰度标记，取不到默认为 false
     */
    public static Boolean getGrayTag() {
        return get(CommonConstant.GRAY, Boolean.class, Boolean.FALSE);
    }

}
```

## 1. 网关过滤器

- 注意，文章中是 implement GlobalFilter，本人使用时，代码实际运行中，Ribbon的自定义负载均衡策略代码，会比其先执行，导致拿不到事先预存在本地线程ThreadLocal里的信息
- 这里改用成 extends ServerWebExchangeContextFilter
- 用于获取请求头信息、逻辑处理、请求头传递、存放于当前线程ThreadLocal

```java
package com.agefades.log.gateway.filter;

import cn.hutool.core.convert.Convert;
import com.agefades.log.common.core.constants.CommonConstant;
import com.agefades.log.common.core.util.GrayContextUtil;
import org.springframework.http.HttpHeaders;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.reactive.ServerWebExchangeContextFilter;
import org.springframework.web.server.ServerWebExchange;
import org.springframework.web.server.WebFilterChain;
import reactor.core.publisher.Mono;

/**
 * 灰度标记过滤器类, ServerWebExchangeContextFilter 才能在Ribbon自定义负载策略类前执行
 *
 * @author DuChao
 * @date 2022/8/3 14:31
 */
@Component
public class GlobalGrayFilter extends ServerWebExchangeContextFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        HttpHeaders headers = exchange.getRequest().getHeaders();
        Boolean grayTag = Boolean.FALSE;
        if (headers.containsKey(CommonConstant.GRAY)){
            grayTag = Convert.toBool(headers.getFirst(CommonConstant.GRAY));
        }
        // 将灰度标记放入当前线程上下文
        GrayContextUtil.setGrayTag(grayTag);
        // 将灰度标记放入请求头中
        ServerHttpRequest tokenRequest = exchange.getRequest().mutate()
                .header(CommonConstant.GRAY, Convert.toStr(grayTag))
                .build();
        ServerWebExchange build = exchange.mutate().request(tokenRequest).build();
        return chain.filter(build);
    }

}
```

## 2. Ribbon自定义负载选择策略

- 注意：这个类一定不能加@Configuration

```java
package com.agefades.log.common.core.config;

import cn.hutool.core.collection.CollUtil;
import com.agefades.log.common.core.constants.CommonConstant;
import com.agefades.log.common.core.util.GrayContextUtil;
import com.alibaba.cloud.nacos.ribbon.ExtendBalancer;
import com.alibaba.cloud.nacos.ribbon.NacosServer;
import com.alibaba.nacos.api.naming.pojo.Instance;
import com.netflix.client.config.IClientConfig;
import com.netflix.loadbalancer.AbstractLoadBalancerRule;
import com.netflix.loadbalancer.Server;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Configuration;

import java.util.ArrayList;
import java.util.List;

/**
 * Ribbon + Nacos 灰度切流逻辑实现类
 *
 * @author DuChao
 * @date 2022/8/3 10:08
 */
@Slf4j
public class GrayRule extends AbstractLoadBalancerRule {

    @Override
    public Server choose(Object key) {
        try {
            // 获取所有可用服务
            List<Server> serverList = this.getLoadBalancer().getReachableServers();
            if (CollUtil.isEmpty(serverList)) {
                log.error("当前注册中心没有可用服务");
                return null;
            }
            // 拆分为灰度服务与正常服务
            List<Instance> grayInstances = new ArrayList<>();
            List<Instance> normalInstances = new ArrayList<>();
            for (Server server : serverList) {
                NacosServer nacosServer = (NacosServer) server;
                String gray = nacosServer.getMetadata().get(CommonConstant.GRAY);
                Instance instance = nacosServer.getInstance();
                if (gray != null) {
                    grayInstances.add(instance);
                } else {
                    normalInstances.add(instance);
                }
            }

            Boolean grayTag = GrayContextUtil.getGrayTag();

            // 当灰度服务不为空 且 请求带灰度标记
            if (grayTag && CollUtil.isNotEmpty(grayInstances)) {
                log.info("当前使用灰度服务");
                return new NacosServer(ExtendBalancer.getHostByRandomWeight2(grayInstances));
            }
            // 其余走正常服务
            return new NacosServer(ExtendBalancer.getHostByRandomWeight2(normalInstances));
        } finally {
            GrayContextUtil.reset();
        }
    }

    @Override
    public void initWithNiwsConfig(IClientConfig iClientConfig) {

    }

}
```

- 灰度规则配置类

```java
package com.agefades.log.common.core.config;

import org.springframework.context.annotation.Bean;

public class GrayRuleConfig {

    @Bean
    public GrayRule grayRule(){
        return new GrayRule();
    }

}
```

- 在每个启动类上加配置注解（不管是网关还是业务服务） @RibbonClients(defaultConfiguration = GrayRuleConfig.class)

```java
package com.agefades.log.gateway;

import com.agefades.log.common.core.config.GrayRuleConfig;
import com.agefades.log.common.core.constants.CommonConstant;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.ribbon.RibbonClients;
import org.springframework.context.annotation.ComponentScan;

@ComponentScan(basePackages = CommonConstant.BASE_PACKAGES)
@RibbonClients(defaultConfiguration = GrayRuleConfig.class)
@SpringBootApplication
public class GatewayApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }

}
```

## 3. Feign请求头信息传递

```java
package com.agefades.log.common.log.config;

import cn.hutool.extra.servlet.ServletUtil;
import feign.RequestInterceptor;
import feign.RequestTemplate;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import javax.servlet.http.HttpServletRequest;
import java.util.Map;

/**
 * Feign请求头传值拦截器
 *
 * @author DuChao
 * @date 2021/12/6 6:38 下午
 */
@Configuration
public class ShareFeignInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (attributes != null) {
            HttpServletRequest request = attributes.getRequest();
            for (Map.Entry<String, String> entry : ServletUtil.getHeaderMap(request).entrySet()) {
                // 设置请求头到新的Request中
                template.header(entry.getKey(), entry.getValue());
            }
        }
    }

}
```

## 4. 公用信息透传过滤器

- 本文仅需关注灰度发布标记的透传即可
- 如有线程上下文透传的需求，参考 [此文](https://github.com/AgeFades/AgeFades-Note/blob/master/Java/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E4%B8%93%E9%A2%98/%E6%8A%80%E6%9C%AF%E5%88%86%E4%BA%AB%20-%20%E5%BE%AE%E6%9C%8D%E5%8A%A1%E6%97%A5%E5%BF%97%E4%BD%93%E7%B3%BB.md)

```java
package com.agefades.log.common.log.config;

import cn.hutool.core.convert.Convert;
import cn.hutool.core.util.StrUtil;
import cn.hutool.json.JSONUtil;
import com.agefades.log.common.core.constants.CommonConstant;
import com.agefades.log.common.core.util.GrayContextUtil;
import com.agefades.log.common.core.util.LogUtil;
import com.agefades.log.common.core.util.dto.SysUserDTO;
import com.agefades.log.common.core.util.UserInfoContextUtil;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.web.filter.OncePerRequestFilter;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.net.URLDecoder;
import java.nio.charset.StandardCharsets;

/**
 * 日志TraceId、用户信息、灰度标记...等信息透传 过滤器
 *
 * @author DuChao
 * @date 2020/9/10 3:13 下午
 */
@Slf4j
@Order(1)
@Configuration
public class LogFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        // 从请求头取traceId，取到值即为其他服务调用，属于traceId的传递
        String traceId = request.getHeader(CommonConstant.TRACE_ID);
        // setTraceId() 方法中，如果参数为空，则会自己生成一个uuid当做traceId，即第一个进入的服务自行生成traceId
        LogUtil.setTraceId(traceId);

        // 灰度标记
        String gray = request.getHeader(CommonConstant.GRAY);
        GrayContextUtil.setGrayTag(Convert.toBool(gray));

        // 处理网关或其他服务传递的用户信息
        String userInfoStr = request.getHeader(CommonConstant.HEADER_SYS_USER);
        if (StrUtil.isNotBlank(userInfoStr)) {
            String decode = URLDecoder.decode(userInfoStr, StandardCharsets.UTF_8);
            // 放入业务系统上下文,方便业务使用
            SysUserDTO sysUserDTO = JSONUtil.toBean(decode, SysUserDTO.class);
            UserInfoContextUtil.setSysUserDTO(sysUserDTO);

            // 用户id放入日志线程上下文
            LogUtil.setUserId(sysUserDTO.getId());
        }

        try {
            filterChain.doFilter(request, response);
        } finally {
            LogUtil.clearAll();
            UserInfoContextUtil.reset();
            GrayContextUtil.reset();
        }
    }

}
```

## 5. 灰度服务启动参数打标

- 设置实例注册到Nacos上的元数据配置参数
- 实际场景中：在Jenkins里设置一套新视图，给该视图下的任务加上灰度参数，CICD的pipline中，加上该Java启动参数即可

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1659595591264.png)

```apl
--spring.cloud.nacos.discovery.metadata.gray=true
```

## 6. 一键切换实例灰度标记

[Nacos 官方OpenAPI](https://nacos.io/zh-cn/docs/open-api.html)

实际使用时，用Nacos的服务类获取所有服务实例，轮询操作写逻辑更新实例的元数据信息即可

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1659595719541.png)

```java
@GetMapping("update/metadata")
@ApiOperation(value = "测试修改元数据")
public Result<Void> testUpdateMetadata() {
  Map<String, Object> form = new HashMap<>();
  form.put("namespaceId", "public");
  form.put("serviceName", "log-order");
  Map<String, Object> metadata = new HashMap<>();
  metadata.put("test", "agefades123");
  metadata.put("gg", "mm123");
  form.put("metadata", JSONUtil.toJsonStr(metadata));
  HttpUtil.doPut(HttpEnum.NACOS, "http://localhost:8848/nacos/v1/ns/instance/metadata/batch?" + URLUtil.buildQuery(form, null));
  return Result.success();
}
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1659595759809.png)

## 简单演示效果

### 正常生产服务

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1659596274815.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1659596324340.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1659596375755.png)

### 灰度发布服务

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1659596414607.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1659596442846.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1659596509555.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1659596544865.png)