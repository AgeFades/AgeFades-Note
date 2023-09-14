[TOC]

# Redis Lua实现限流(防表单重复提交)

## 限流注解

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1641525503914.png)

```java
import org.springblade.core.log.enums.RateLimiterTypeEnum;
import org.springframework.core.annotation.AliasFor;
import org.springframework.core.annotation.AnnotationUtils;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import java.util.concurrent.TimeUnit;

/**
 * 限流注解, {@link AliasFor} 必须通过 {@link AnnotationUtils} 才会生效
 * 默认: 通过请求参数限流，3秒钟内 最大请求1次
 *
 * @author DuChao
 * @date 2020/10/16 2:45 下午
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RateLimiter {

    long DEFAULT_MAX_REQUEST = 1;

    /**
     * 限流类型 {@link RateLimiterTypeEnum}，默认通过请求参数限流
     */
    RateLimiterTypeEnum type() default RateLimiterTypeEnum.PARAM;

    /**
     * max 最大请求数, 默认1
     */
    @AliasFor("value") long max() default DEFAULT_MAX_REQUEST;

    /**
     * max 最大请求数, 默认1
     */
    @AliasFor("max") long value() default DEFAULT_MAX_REQUEST;

    /**
     * 超时时长，默认 3 秒
     */
    long timeout() default 3;

    /**
     * 超时时间单位，默认 秒
     */
    TimeUnit timeUnit() default TimeUnit.SECONDS;
}

```

## 限流类型枚举

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1641525553457.png)

```java
/**
 * 限流类型枚举
 *
 * @author DuChao
 * @date 2020/10/16 3:00 下午
 */
public enum RateLimiterTypeEnum {

    /**
     * 通过IP限流
     */
    IP,

    /**
     * 通过请求参数限流
     */
    PARAM

}
```

## 限流脚本

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1641525574827.png)

```lua
-- 下标从1开始
local key = KEYS[1]
local now = tonumber(ARGV[1])
local ttl = tonumber(ARGV[2])
local expired = tonumber(ARGV[3])
-- 最大访问量
local max = tonumber(ARGV[4])

-- 清除过期的数据
-- 移除指定分数区间内的所有元素，expired 即已经过期的 score
-- 根据当前时间毫秒数 - 超时毫秒数，得到过期时间 expired
redis.call('zremrangebyscore', key, 0, expired)

-- 获取 zset 中的当前元素个数
local current = tonumber(redis.call('zcard', key))
local next = current + 1

if next > max then
    -- 达到限流大小 返回 0
    return 0;
else
    -- 往 zset 中添加一个值、得分均为当前时间戳的元素，[value, score]
    redis.call("zadd", key, now, now)
    -- 每次访问均重新设置 zset 的过期时间，单位毫秒
    redis.call("pexpire", key, ttl)
    return next
end
```

## 限流Bean注册

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.ClassPathResource;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.data.redis.core.script.RedisScript;
import org.springframework.scripting.support.ResourceScriptSource;

/**
 * Redis限流配置类
 *
 * @author DuChao
 * @date 2022/1/7 11:20 AM
 */
@Configuration
public class RateLimitConfig {

	/**
	 * 限流脚本
	 */
	@Bean
	public RedisScript<Long> limitRedisScript() {
		DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
		redisScript.setScriptSource(new ResourceScriptSource(new ClassPathResource("scripts.redis/limit.lua")));
		redisScript.setResultType(Long.class);
		return redisScript;
	}

}
```

## 限流切面

```java
/**
 * 限流注解切面（通过Redis Lua脚本实现限流）
 *
 * @author DuChao
 * @date 2020/10/16 2:47 下午
 */
@Slf4j
@Aspect
@Component
@RequiredArgsConstructor
public class RateLimiterAspect {

    private final static String SEPARATOR = ":";
    private final static String REDIS_LIMIT_KEY_PREFIX = "limit";
    private final StringRedisTemplate stringRedisTemplate;
    private final RedisScript<Long> limitRedisScript;

    @Pointcut("@annotation(org.springblade.core.log.annotation.RateLimiter)")
    private void rateLimit() {
    }

    @Around("rateLimit()")
    public Object pointcut(ProceedingJoinPoint point) throws Throwable {
        MethodSignature signature = (MethodSignature) point.getSignature();
        Method method = signature.getMethod();

        // 通过 AnnotationUtils.findAnnotation 获取 RateLimiter 注解
        RateLimiter rateLimiter = AnnotationUtils.findAnnotation(method, RateLimiter.class);
        if (rateLimiter != null) {
            // limit:类名:方法名
            String key = REDIS_LIMIT_KEY_PREFIX
                    + SEPARATOR
                    + StrUtil.subAfter(method.getDeclaringClass().getName(), ".", true)
                    + SEPARATOR
                    + method.getName();

            if (RateLimiterTypeEnum.IP.equals(rateLimiter.type())) {
                // limit:类名:方法名:ip
                key = key + SEPARATOR + IpUtil.getIpAddr();
            } else {
                // limit:类名:方法名:参数哈希Code
                key = key + SEPARATOR + Objects.hash(AspectUtil.getParams(point));
            }

            // 执行限流方法，判断是否应该限流
            Assert.isFalse(shouldLimited(key, rateLimiter.max(), rateLimiter.timeout(), rateLimiter.timeUnit()), "操作过于频繁，请稍后再试");
        }

        return point.proceed();
    }

    private boolean shouldLimited(String key, long max, long timeout, TimeUnit timeUnit) {
        // 统一使用单位毫秒
        long ttl = timeUnit.toMillis(timeout);
        // 当前时间毫秒数
        long now = Instant.now().toEpochMilli();
        long expired = now - ttl;
        // 注意这里必须转为 String，否则会报错 java.lang.Long cannot be cast to java.lang.String
        Long executeTimes = stringRedisTemplate.execute(limitRedisScript, Collections.singletonList(key), now + "", ttl + "", expired + "", max + "");
        if (executeTimes != null) {
            if (executeTimes == 0) {
                log.warn("【{}】在单位时间 {} 秒内已达到访问上限，当前接口上限 {}", key, ttl / 1000, max);
                return true;
            } else {
                log.info("【{}】在单位时间内 {} 秒内访问 {} 次", key, ttl / 1000, executeTimes);
                return false;
            }
        }
        return false;
    }

}
```

## 使用案例 - 防表单重复提交

### 注解使用

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1641525787139.png)

### 使用效果

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1641525827823.png)

### 日志证明

#### 日志时间

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1641525945935.png)

#### 日志输出

##### 关键信息

1. Redis 中的 key
2. 限流的时间（这里是3秒）
3. 限流的次数（这里是1次）

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1641525990362.png)

##### 总结

- 这里是根据 `请求参数` 的 `hashcode` 作为 key，通过 lua 在 redis 中暂存的 zset 结构
- 根据注解参数（默认通过 请求参数 限流、3s内参数相同只会被接口响应一次）
- 可在注解上自行修改参数选型，决定限流规则：
  1. 根据 请求ip 限流 或 根据 请求参数 限流（默认）
  2. 限流时间（默认3）、限流时间单位（默认秒）
  3. 限流次数（默认1）

