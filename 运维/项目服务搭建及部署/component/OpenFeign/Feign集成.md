## 依赖

```xml
<dependencies>

  <!-- openfeign 远程调用 -->
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
    <!-- 排除默认日志包，使用Log4j2 -->
    <exclusions>
      <exclusion>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-logging</artifactId>
      </exclusion>
    </exclusions>
  </dependency>

</dependencies>
```

## 注解

```java
// 启动类加该注解，basePackages是扫描 FeignClient，加载到 IOC 容器
@EnableFeignClients(basePackages = CommonConstant.BASE_PACKAGES)
public class SystemApplication {

    public static void main(String[] args) {
        SpringApplication.run(SystemApplication.class, args);
    }

}
```

## 使用

### 服务端

#### 端点

```java
@Api(tags = "测试服务间调用")
@RestController
public class TestFeignClient {

    @ApiOperation(value = "测试服务间调用")
    @PostMapping("test")
    public TestResp test(@RequestBody TestReq req){
        TestResp resp = new TestResp();
        resp.setIsOk(true);
        resp.setMsg("姓名: " + req.getName() + ",年龄: " + req.getAge());
        return resp;
    }

}
```

#### Feign客户端

```java
@FeignClient(value = CommonConstant.JDY_POINTS)
public interface TestFeignClient {

    @ApiOperation(value = "测试服务间调用")
    @PostMapping("test")
    TestResp test(@RequestBody TestReq req);

}
```

### 消费端

#### 依赖

```xml
<dependencies>
  <!-- 添加调用服务Client客户端依赖 -->
  <dependency>
    <groupId>com.botpy.jdy</groupId>
    <artifactId>jdy-points-client</artifactId>
  </dependency>
</dependencies>
```

#### 使用

```java
@RestController
@RequestMapping("/test")
@Api(tags = "测试API")
public class TestController {

    @Autowired
    TestFeignClient testFeignClient;

    @ApiOperation(value = "测试服务间调用")
    @PostMapping
    public TestResp test(@RequestBody TestReq req){
        return testFeignClient.test(req);
    }

}
```

