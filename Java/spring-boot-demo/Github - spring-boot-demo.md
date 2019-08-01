# Github - spring-boot-demo

## spring-boot-async

```yaml
spring:
	task:
		execution:
			pool:
				max-size: 16 # 最大线程数
				core-size: 16 # 核心线程数
				keep-alive: 10s # 存活时间
				queue-capacity: 100 # 队列大小
				allow-core-thread-timeout: true # 是否允许核心线程超时
			thread-name-prefix: async-task- # 线程名称前缀
```

```java
@EnableAsync // SpringBoot 启动类上添加此注解
```

```java
@Async // 在异步方法上加此注解
public Future<Boolean> asyncTask() throws Exception {
    // TODO 异步任务处理逻辑
    return new AsyncResult<>(Boolean.TRUE);
}
```

## spring-boot-ehcache

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>

<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

```java
@EnableCaching // 启动类上加此注解
```

```yaml
spring:
	cache:
		type: ehcache
		ehcache:
			config: classpath:ehcache.xml
```

```xml
<!-- ehcache配置 -->
<ehcache
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="http://ehcache.org/ehcache.xsd"
        updateCheck="false">
    <!--缓存路径，用户目录下的base_ehcache目录-->
    <diskStore path="user.home/base_ehcache"/>

    <defaultCache
            maxElementsInMemory="20000"
            eternal="false"
            timeToIdleSeconds="120"
            timeToLiveSeconds="120"
            overflowToDisk="true"
            maxElementsOnDisk="10000000"
            diskPersistent="false"
            diskExpiryThreadIntervalSeconds="120"
            memoryStoreEvictionPolicy="LRU"/>

    <!--
    缓存文件名：user，同样的可以配置多个缓存
    maxElementsInMemory：内存中最多存储
    eternal：外部存储
    overflowToDisk：超出缓存到磁盘
    diskPersistent：磁盘持久化
    timeToLiveSeconds：缓存时间
    diskExpiryThreadIntervalSeconds：磁盘过期时间
    -->
    <cache name="user"
           maxElementsInMemory="20000"
           eternal="true"
           overflowToDisk="true"
           diskPersistent="false"
           timeToLiveSeconds="0"
           diskExpiryThreadIntervalSeconds="120"/>

</ehcache>
```

```java
@Cacheable(value = "user",key = "#id") // 在需要缓存的方法上加此注解
```

## spring-boot-redis

```xml
		<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>

        <!-- 对象池，使用redis时必须引入 -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>

        <!-- 引入 jackson 对象json转换 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-json</artifactId>
        </dependency>
```

```yaml
spring:
	redis:
		host: localhost # redis Ip
		timeout: 10000ms # 超时时间
		lettuce:
			pool:
				max-active: 8 # 连接池最大连接数，使用负值表示没有限制，默认 8
				max-wait: -1ms # 连接池最大阻塞等待时间（使用负值表示没有限制） 默认 -1
				max-idle: 8 # 连接池中的最大空闲连接 默认 8
				min-idle: 0 # 连接池中的最小空闲连接 默认 0
```

```java
@Configuration
@AutoConfigureAfter(RedisAutoConfiguration.class)
@EnableCaching
public class RedisConfig {

    /**
     * 默认情况下的模板只能支持RedisTemplate<String, String>，也就是只能存入字符串，因此支持序列化
     */
    @Bean
    public RedisTemplate<String, Serializable> 
        		redisCacheTemplate(LettuceConnectionFactory redisConnectionFactory) {
        
        RedisTemplate<String, Serializable> template = new RedisTemplate<>();
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setConnectionFactory(redisConnectionFactory);
        return template;
        
    }
}
```

```java
@CachePut(value="user",key="#user.id") // 数据入库入redis
@Cacheable(value="user",key="#id") // 查询启用缓存
@CacheEvict(value="user",key="#id") // 删除数据、删除缓存
```

## 