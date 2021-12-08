[TOC]

# 概述

​		SpringBoot整合ELK单指将SpringBoot日志推送至Logstash以及使用教程。

# 日志流程

​		项目输出Log日志  --> TCP推送Logstash  -->  Logstash接收日志处理  -->  输出到Elasticsearch中  -->  Kibana图形化展示

​		流程图如下：

<img src="https://blog-kang.oss-cn-beijing.aliyuncs.com/1619167113083.png" style="zoom:50%;" />

# Logstash配置

## TCP方式

```properties
input {
		# TCP输入，Json格式编码UTF-8
    tcp {
     port => 9400
      codec => json {
             charset => "UTF-8"
      }
    }
    stdin{}
}
filter {
}

output {
  elasticsearch {
    action => "index"
    # 这里是es的地址，多个es要写成数组的形式
    hosts  => "elasticsearch-server:9200"
    # 用于kibana过滤，可以填项目名称，按日期生成索引
    index  => "vosp-log-%{+YYYY-MM-dd}"
    # es用户名
    user => vosp
    # es密码
    password => vosp123
    # 超时时间
    timeout => 300
  }
}
```

# SpringBoot+Log4j2

## 引入依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <exclusions><!-- 去掉springboot默认配置 -->
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-logging</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency> <!-- 引入log4j2依赖 -->
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-log4j2</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
```

## 编写SpringBoot启动以及配置类

​		新建配置类,

```java

import org.slf4j.MDC;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.context.event.ApplicationEnvironmentPreparedEvent;
import org.springframework.boot.context.event.ApplicationFailedEvent;
import org.springframework.boot.context.event.ApplicationPreparedEvent;
import org.springframework.boot.context.event.ApplicationStartingEvent;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationEvent;
import org.springframework.context.event.ContextClosedEvent;
import org.springframework.context.event.GenericApplicationListener;
import org.springframework.core.Ordered;
import org.springframework.core.ResolvableType;
import org.springframework.core.env.ConfigurableEnvironment;
import org.springframework.core.env.MutablePropertySources;
import org.springframework.core.env.PropertySource;


/**
 * @Author HuangKang
 * @Date 2021/4/22 下午5:44
 * @Summarize Log4j2启动配置文件事件监听器
 * SpringApplication application = new SpringApplication(VospApiApplication.class);
 * Set<ApplicationListener<?>> ls = application.getListeners();
 * Log4j2PropertiesEventListener eventListener = new Log4j2PropertiesEventListener();
 * application.addListeners(eventListener);
 * application.run(args);
 * 启动类启动注册监听器
 */
public class Log4j2PropertiesEventListener implements GenericApplicationListener {

    public static final int DEFAULT_ORDER = Ordered.HIGHEST_PRECEDENCE + 10;

    private static Class<?>[] EVENT_TYPES = {ApplicationStartingEvent.class, ApplicationEnvironmentPreparedEvent.class,
            ApplicationPreparedEvent.class, ContextClosedEvent.class, ApplicationFailedEvent.class};

    private static Class<?>[] SOURCE_TYPES = {SpringApplication.class, ApplicationContext.class};

    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        if (event instanceof ApplicationEnvironmentPreparedEvent) {
            ConfigurableEnvironment envi = ((ApplicationEnvironmentPreparedEvent) event).getEnvironment();
            MutablePropertySources mps = envi.getPropertySources();
            // 获取配置文件配置源,注意版本不同SpringBoot可能配置源名称不一样，可以DEBUG查看名称
            PropertySource<?> ps = mps.get("configurationProperties");
            // 设置log4j2日志收集logstash地址
            if (ps != null && ps.containsProperty("log4j2.logstash.address")) {
                Object address = ps.getProperty("log4j2.logstash.address");
                if (address != null && !address.toString().trim().isEmpty()) {
                    MDC.put("log4j2.logstash.address", address.toString());
                }
            }
            // 设置log4j2日志收集logstash端口号
            if (ps != null && ps.containsProperty("log4j2.logstash.port")) {
                Object port = ps.getProperty("log4j2.logstash.port");
                if (port != null && !port.toString().trim().isEmpty()) {
                    MDC.put("log4j2.logstash.port", port.toString());
                }
            }
            // 设置log4j2日志收集logstash日志格式化表达式
            if (ps != null && ps.containsProperty("log4j2.logstash.pattern")) {
                Object pattern = ps.getProperty("log4j2.logstash.pattern");
                if (pattern != null && !pattern.toString().trim().isEmpty()) {
                    // 获取项目名
                    String appName = ps.getProperty("spring.application.name").toString();
                    // 获取项目名
                    String profile = ps.getProperty("spring.profiles.active").toString();
                    MDC.put("log4j2.logstash.pattern", pattern.toString().replace("${spring.application.name}", appName).replace("${spring.profiles.active}",profile));
                }
            }

        }
    }

    @Override
    public int getOrder() {
        return DEFAULT_ORDER;
    }

    @Override
    public boolean supportsEventType(ResolvableType resolvableType) {
        return isAssignableFrom(resolvableType.getRawClass(), EVENT_TYPES);
    }

    @Override
    public boolean supportsSourceType(Class<?> sourceType) {
        return isAssignableFrom(sourceType, SOURCE_TYPES);
    }

    private boolean isAssignableFrom(Class<?> type, Class<?>... supportedTypes) {
        if (type != null) {
            for (Class<?> supportedType : supportedTypes) {
                if (supportedType.isAssignableFrom(type)) {
                    return true;
                }
            }
        }
        return false;
    }
}

```

​		修改SpringBoot启动类，添加监听器

```java
    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(BootK8sApplication.class);
        // 添加Log4j2配置文件监听
        Log4j2PropertiesEventListener eventListener = new Log4j2PropertiesEventListener();
        application.addListeners(eventListener);
        application.run(args);
    }		
```

## 编写配置文件

​		在Yaml中写入如下参数

```properties
spring:
  application:
    # 项目名称启动时传入Log4j2作为app名称
    name: test-k8s

logging:
  # Log4j2配置文件xml地址
  config: classpath:log4j2-spring.xml
log4j2:
  logstash:
    # 自定义配置Logstash地址
    address: 124.71.9.101
    # 自定义配置Logstash端口号
    port: 9400
    # 自定义Json消息
    # {
        #	"app": "${spring.application.name}",  当前项目名称
        #	"class": "%c",                        类全限定名称
        #	"method": "%M",                       输出日志方法
        #	"traceId": "%X{traceId}",             traceId跟踪
        #	"level": "%level",                    日志级别
        #   "message": "[%thread] [%c:%L] --- %replace{%msg}{\"}{\\\"}\"  消息内容体，将"替换为转义，防止日志中输出Json导致解析异常
        #   "profile":"${spring.profiles.active}" 当前启动环境
    # }
    # Json写入至ELK中属性
    pattern: "{\"app\":\"${spring.application.name}\", \"profile\":\"${spring.profiles.active}\", \"class\":\"%c\",\"method\":\"%M\",\"traceId\":\"%X{traceId}\",\"level\": \"%level\", \"message\": \"[%thread] [%c:%L] --- %replace{%msg}{\\\"}{\\\\\\\"}\\\"}%n"
```

## 编写Log4j2-XML文件

```xml
<?xml version="1.0" encoding="UTF-8"?>

<configuration status="info" monitorInterval="30">

    <properties>
        <!--日志路径-->
        <property name="log.path">./logs</property>
        <!--日志打印格式-->
        <property name="pattern">%d{yyyy-MM-dd HH:mm:ss,SSS} [%-5level] [%X{traceId}] [%t] [%C{1.}] %msg%n</property>

        <!--SpringBoot读取的配置地址，端口以及日志格式-->
        <Property name="logstashAddress">${ctx:log4j2.logstash.address}</Property>
        <Property name="logstashPort">${ctx:log4j2.logstash.port}</Property>
        <Property name="logstashPattern">${ctx:log4j2.logstash.pattern}</Property>
    </properties>
    <!--先定义所有的appender-->
    <appenders>
        <!--这个输出控制台的配置-->
        <console name="console" target="SYSTEM_OUT">
            <!--输出日志的格式-->
            <PatternLayout>
                <!-- 控制台彩色日志格式 -->
                <Pattern>%style{%d{yyyy-MM-dd HH:mm:ss,SSS}}{bright,white} %highlight{%-5level} %style{[%X{traceId}]}{cyan} [%style{%t}{bright,blue}] [%style{%C{1.}}{bright,yellow}:%L] %msg%n%style{%throwable}{red}</Pattern>
                <!-- 控制台统一日志格式 -->
                <!--                <Pattern>${pattern}</Pattern>-->
            </PatternLayout>
            <ThresholdFilter level="info" onMatch="ACCEPT" onMismatch="DENY"/>
        </console>
        <RollingFile name="infoLog" immediateFlush="true" fileName="${sys:log.path}/info.log"
                     filePattern="${sys:log.path}/$${date:yyyy-MM}/info-%d{yyyy-MM-dd}-%i.log.zip">
            <Filters>
                <!--控制台只输出level及以上级别的信息（onMatch），其他的直接拒绝（onMismatch）-->
                <ThresholdFilter level="INFO" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
            <PatternLayout>
                <Pattern>${pattern}</Pattern>
            </PatternLayout>
            <Policies>
                <TimeBasedTriggeringPolicy interval="1"/>
                <SizeBasedTriggeringPolicy size="100 MB"/>
            </Policies>
        </RollingFile>

        <RollingFile name="warnLog" immediateFlush="true" fileName="${sys:log.path}/warn.log"
                     filePattern="${sys:log.path}/$${date:yyyy-MM}/warn-%d{yyyy-MM-dd}-%i.log.zip">
            <Filters>
                <!--控制台只输出level及以上级别的信息（onMatch），其他的直接拒绝（onMismatch）-->
                <ThresholdFilter level="warn" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
            <PatternLayout>
                <Pattern>${pattern}</Pattern>
            </PatternLayout>
            <Policies>
                <TimeBasedTriggeringPolicy interval="1"/>
                <SizeBasedTriggeringPolicy size="100 MB"/>
            </Policies>
        </RollingFile>

        <RollingFile name="errorLog" immediateFlush="true" fileName="${sys:log.path}/error.log"
                     filePattern="${sys:log.path}/$${date:yyyy-MM}/error-%d{yyyy-MM-dd}-%i.log.zip">
            <Filters>
                <!--控制台只输出level及以上级别的信息（onMatch），其他的直接拒绝（onMismatch）-->
                <ThresholdFilter level="ERROR" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
            <PatternLayout>
                <Pattern>${pattern}</Pattern>
            </PatternLayout>
            <Policies>
                <TimeBasedTriggeringPolicy interval="1"/>
                <SizeBasedTriggeringPolicy size="100 MB"/>
            </Policies>
        </RollingFile>
        <!-- Socket TCP方式推送日志到Logstash中，只推送INFO以及以上级别日志 -->
        <Socket>
            <name>logstash</name>
            <protocol>TCP</protocol>
            <port>${logstashPort}</port>
            <host>${logstashAddress}</host>
            <PatternLayout pattern="${logstashPattern}"/>
            <reconnectionDelay>5000</reconnectionDelay>
            <ThresholdFilter level="INFO" onMatch="ACCEPT" onMismatch="DENY"/>
        </Socket>
    </appenders>
    <!--然后定义logger，只有定义了logger并引入的appender，appender才会生效-->
    <loggers>
        <!--过滤掉无用的debug信息-->
        <logger name="org.springframework" level="INFO"/>
        <logger name="org.mybatis" level="INFO"/>
        <logger name="cn.binarywang" level="ERROR"/>
        <!--指定了logger就使用logger，没有指定就使用root-->
        <logger name="com.botpy.vosp.client" level="debug" additivity="false">
            <appender-ref ref="console"/>
            <appender-ref ref="infoLog"/>
            <appender-ref ref="warnLog"/>
            <appender-ref ref="errorLog"/>
        </logger>
        <logger name="logstashLog" level="info" additivity="false">
            <appender-ref ref="logstash"/>
        </logger>
        <root level="debug">
            <appender-ref ref="console"/>
            <appender-ref ref="infoLog"/>
            <appender-ref ref="warnLog"/>
            <appender-ref ref="errorLog"/>
            <appender-ref ref="logstash" />
        </root>
    </loggers>
</configuration>
```

## 新增traceId拦截器

​		新建拦截器

```java

import org.slf4j.MDC;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.UUID;

/**
 * @Author HuangKang
 * @Date 2021/4/23 下午2:23
 * @Summarize 请求TraceId拦截器
 */
@Component
public class RequestTraceIdInterceptor implements HandlerInterceptor {
    /**
     * 日志TRACE_ID名称常量
     */
    private static final String LOG_TRACE_ID_NAME = "traceId";

    /**
     *
     * @param request
     * @param response
     * @param handler
     * @return
     * @throws Exception
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        MDC.put(LOG_TRACE_ID_NAME, UUID.randomUUID().toString().replace("-",""));
        return true;
    }

}

```

​		新增Mvc配置拦截器

```java
package com.kang.test.k8s.config;

import com.kang.test.k8s.interceptor.RequestTraceIdInterceptor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

/**
 * @Author HuangKang
 * @Date 2021/4/23 下午4:00
 * @Summarize 自定义WebMvc配置
 */
@Configuration
public class CustomWebMvcConfig  implements WebMvcConfigurer {

    final RequestTraceIdInterceptor requestTraceIdInterceptor;

    @Autowired
    public CustomWebMvcConfig(RequestTraceIdInterceptor requestTraceIdInterceptor) {
        this.requestTraceIdInterceptor = requestTraceIdInterceptor;
    }

    /**
     * 添加拦截器
     * @param registry
     */
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 添加TraceId拦截器
        registry.addInterceptor(requestTraceIdInterceptor);
    }
}

```

​		使用方式，新建Controller测试功能

```java
    @RequestMapping("/")
    public Map index() throws UnknownHostException {
        Map<String, Object> map = new HashMap<>();
        InetAddress localHost = InetAddress.getLocalHost();
        String hostName = localHost.getHostName();
        String hostAddress = localHost.getHostAddress();
        map.put("主机名",hostName);
        map.put("主机地址",hostAddress);
        log.info("测试日志");
        return map;
    }
```

​		调用接口后打印如下日志

```
可以查看到INFO  [3d9e051ceafe4e59bce8aa6ac2972fda] 成功打印traceId
2021-04-23 16:21:13,929 INFO  [3d9e051ceafe4e59bce8aa6ac2972fda] [http-nio-8080-exec-1] [c.k.t.k.c.TestK8sController:37] 测试日志
```



# SpringBoot+Logback

## 引入依赖	

​		引入依赖正常即可

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>net.logstash.logback</groupId>
            <artifactId>logstash-logback-encoder</artifactId>
            <version>6.1</version>
        </dependency>
```

## 编写配置文件

```properties
spring:
  application:
    # 项目名称启动时传入，写入ELK通过app名称进行隔离
    name: test-k8s
  profiles:
  	# 启动环境，ELK隔离通过profile属性隔离dev以及prod
    active: log
server:
  port: 8080
logging:
	# 日志文件地址
  config: classpath:logback-spring.xml
  # 配置Logstash地址
  logstash:
    address: 124.71.9.101:9400
```

## 编写XML文件

​		新建XML文件，重点观察ELK Json属性，可以自定义拓展,新建logback-spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!--从配置文件读取配置路径，并且添加默认值-->
    <!--日志全局设置-->
    <springProperty scope="context" name="log.charset" source="log.charset" defaultValue="UTF-8"/>

    <springProperty scope="context" name="log.pattern" source="log.pattern"
                    defaultValue="%d{yyyy-MM-dd HH:mm:ss,SSS} [%-5level] [%X{traceId}] [%t] [%C{1.}] %msg%n"/>

    <!--INFO日志设置-->
    <springProperty scope="context" name="info.path" source="log.info.path" defaultValue="logs/info"/>
    <springProperty scope="context" name="info.history" source="log.info.history" defaultValue="10"/>
    <springProperty scope="context" name="info.maxsize" source="log.info.maxsize" defaultValue="1GB"/>
    <springProperty scope="context" name="info.filesize" source="log.info.filesize" defaultValue="10MB"/>
    <springProperty scope="context" name="info.pattern" source="log.info.pattern"
                    defaultValue="%date [%thread] %-5level [%logger{50}] %file:%line - %msg%n"/>

    <!--ERROR日志设置-->
    <springProperty scope="context" name="error.path" source="log.error.path" defaultValue="logs/error"/>
    <springProperty scope="context" name="error.history" source="log.error.history" defaultValue="10"/>
    <springProperty scope="context" name="error.maxsize" source="log.error.maxsize" defaultValue="1GB"/>
    <springProperty scope="context" name="error.filesize" source="log.error.filesize" defaultValue="10MB"/>
    <springProperty scope="context" name="error.pattern" source="log.error.pattern"
                    defaultValue="%date [%thread] %-5level [%logger{50}] %file:%line - %msg%n"/>

    <!--从SpringBoot配置文件读取项目名，环境，以及logstash地址-->
    <springProperty scope="context" name="springAppName" source="spring.application.name"/>
    <springProperty scope="context" name="springProfile" source="spring.profiles.active"/>
    <springProperty scope="context" name="logstashAddress" source="logging.logstash.address"/>

    <!-- 彩色日志 -->
    <!-- 彩色日志依赖的渲染类 -->
    <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter"/>
    <conversionRule conversionWord="wex"
                    converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter"/>
    <conversionRule conversionWord="wEx"
                    converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter"/>
    <!-- 定义彩色日志格式模板 -->

    <!--控制台日志打印格式-->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
        </filter>
        <encoder>
            <pattern>${log.pattern}</pattern>
            <charset>${log.charset}</charset>
        </encoder>
    </appender>

    <!--INFO日志打印-->
    <appender name="FILE_INFO" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!--如果只是想要 Info 级别的日志，只是过滤 info 还是会输出 Error 日志，因为 Error 的级别高， 所以我们使用下面的策略，可以避免输出 Error 的日志-->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <!--过滤 Error-->
            <level>ERROR</level>
            <!--匹配到就禁止-->
            <onMatch>DENY</onMatch>
            <!--没有匹配到就允许-->
            <onMismatch>ACCEPT</onMismatch>
        </filter>
        <!--日志名称，如果没有File 属性，那么只会使用FileNamePattern的文件路径规则如果同时有<File>和<FileNamePattern>，那么当天日志是<File>，明天会自动把今天的日志改名为今天的日期。即，<File> 的日志都是当天的。-->
        <!--<File>logs/info.spring-boot-demo-logback.log</File>-->
        <!--滚动策略，按照时间滚动 TimeBasedRollingPolicy-->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--文件路径,定义了日志的切分方式——把每一天的日志归档到一个文件中,以防止日志填满整个磁盘空间-->
            <FileNamePattern>${info.path}/info_%d{yyyy-MM-dd}.part_%i.log</FileNamePattern>
            <!--只保留最近10天的日志-->
            <maxHistory>${info.history}</maxHistory>
            <!--用来指定日志文件的上限大小，那么到了这个值，就会删除旧的日志-->
            <totalSizeCap>${info.maxsize}</totalSizeCap>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <!-- maxFileSize:这是活动文件的大小，默认值是10MB,本篇设置为1KB，只是为了演示 -->
                <maxFileSize>${info.filesize}</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
        <!--<triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">-->
        <!--<maxFileSize>1KB</maxFileSize>-->
        <!--</triggeringPolicy>-->
        <encoder>
            <pattern>${info.pattern}</pattern>
            <charset>${log.charset}</charset> <!-- 此处设置字符集 -->
        </encoder>
    </appender>
    <!--ERROR日志打印-->
    <appender name="FILE_ERROR" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!--如果只是想要 Error 级别的日志，那么需要过滤一下，默认是 info 级别的，ThresholdFilter-->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>
        <!--日志名称，如果没有File 属性，那么只会使用FileNamePattern的文件路径规则如果同时有<File>和<FileNamePattern>，那么当天日志是<File>，明天会自动把今天的日志改名为今天的日期。即，<File> 的日志都是当天的。-->
        <!--<File>logs/error.spring-boot-demo-logback.log</File>-->
        <!--滚动策略，按照时间滚动 TimeBasedRollingPolicy-->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--文件路径,定义了日志的切分方式——把每一天的日志归档到一个文件中,以防止日志填满整个磁盘空间-->
            <FileNamePattern>${error.path}/error_%d{yyyy-MM-dd}.part_%i.log</FileNamePattern>
            <!--只保留最近90天的日志-->
            <maxHistory>${error.history}</maxHistory>
            <!--用来指定日志文件的上限大小，那么到了这个值，就会删除旧的日志-->
            <totalSizeCap>${error.maxsize}</totalSizeCap>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <!-- maxFileSize:这是活动文件的大小，默认值是10MB,本篇设置为1KB，只是为了演示 -->
                <maxFileSize>${error.filesize}</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
        <encoder>
            <pattern>${error.pattern}</pattern>
            <charset>${log.charset}</charset> <!-- 此处设置字符集 -->
        </encoder>
    </appender>


    <appender name="logstash" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>${logstashAddress}</destination>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>INFO</level>
        </filter>
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp>
                    <timeZone>UTC</timeZone>
                </timestamp>
                <pattern>
                    <pattern>
                        {
                        <!--设置项目-->
                        "app": "${springAppName:-}",
                        <!--设置环境-->
                        "profile": "${springProfile:-}",
                        <!--设置等级-->
                        "level": "%level",
                        <!--设置traceId-->
                        "traceId": "%X{traceId}",
                        <!--设置类名-->
                        "class": "%c",
                        <!--设置方法名-->
                        "method": "%M",
                        <!--设置消息-->
                        "message": "[%thread] [%X{traceId}] [%logger{35}:%L] --- %msg"
                        }
                    </pattern>
                </pattern>
            </providers>
        </encoder>
    </appender>

    <root level="info">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE_INFO"/>
        <appender-ref ref="FILE_ERROR"/>
        <appender-ref ref="logstash"/>
    </root>

</configuration>
```

## 新增traceId拦截器

​		参考上方Log4j2一样

# Kibana使用

## 修改时间显示格式化

​		打开菜单  -->  打开Stack Management -->  找到Kibana  -->  选择高级设置  -->  修改日期格式

​		![](https://blog-kang.oss-cn-beijing.aliyuncs.com/1619172621747.png)

## 新建日志视图

​		打开菜单  -->  打开Stack Management -->  找到Kibana  -->  选择索引模式  -->  点击创建索引模式

![](https://blog-kang.oss-cn-beijing.aliyuncs.com/1619172816896.png)

​		点击创建编辑如下输入日志表达式，匹配格式，匹配日志索引，点击下一步

![](https://blog-kang.oss-cn-beijing.aliyuncs.com/1619172878418.png)

​		选择时间字段，TCP方式Logstash推送日志后会自动生成，选择日志即可，点击创建索引模式

![](https://blog-kang.oss-cn-beijing.aliyuncs.com/image-20210423181557486.png)

## 使用日志视图

​		打开菜单  -->  打开Discover -->  点击进入

![](https://blog-kang.oss-cn-beijing.aliyuncs.com/1619173031562.png)

​			打开菜单  -->  打开Discover -->  点击下拉框选择创建的索引匹配器，即可打开日志视图

![](https://blog-kang.oss-cn-beijing.aliyuncs.com/1619173136504.png)

## 日志筛选

​		点击添加筛选，选择运算符是 （等于），设置日志级别即可过滤

![](https://blog-kang.oss-cn-beijing.aliyuncs.com/1619173244068.png)

## 查看指定信息

​		点击Message，点击添加Message，就只查看Message的信息

​		![](https://blog-kang.oss-cn-beijing.aliyuncs.com/1619173334405.png)

​				点击点击后查看的结果如下

![](https://blog-kang.oss-cn-beijing.aliyuncs.com/1619173387005.png)

