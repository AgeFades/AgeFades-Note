## 资料

[GitHub Redisson](https://github.com/redisson/redisson)

## 案例

### 引入依赖

- 此时最新版本为 3.15.4
- 这里直接使用 SpringBoot Starter、里面内置 SpringData Redis 依赖

```xml
<!-- redisson -->
<dependency>
  <groupId>org.redisson</groupId>
  <artifactId>redisson-spring-boot-starter</artifactId>
  <version>3.15.4</version>
</dependency>
```

### 配置

- 这里记录本机单例Redis最简配置，需扩展自行查阅官方文档
- 只演示分布式锁的使用，其他功能自行查阅官方文档

```yml
# 最简配置可以不配，因为SpringData Redis会默认为 localhost:6379
spring:
	redis:
		host: localhost
    port: 6379
```

### 使用

```java
@Slf4j
@RestController
@RequestMapping("test")
public class TestController {
  
    @Autowired
    RedissonClient redissonClient;

    // 模拟库存
    private static Integer TOTAL = 10;

    @GetMapping("lock")
    public String test() {
        // 第一次校验库存数,如果已经为0,直接抛出友好提示即可,无需尝试获取锁
        checkTotal();
        // 创建锁对象
        RLock lock = redissonClient.getLock("lock:decr");
        try {
            // 尝试获取锁
            // 3代表线程尝试获取锁最大等待时间为3秒，这3秒内该线程会不断自旋尝试获取锁
            // 2代表线程持锁成功后，完成业务需要的时间为2秒，如果超出2秒还未完成，redisson会自动释放锁
            // 第三个参数代表时间单位
            boolean tryLock = lock.tryLock(3, 2, TimeUnit.SECONDS);
            if (tryLock) {
                // 成功获取锁，第二次校验库存数（这里不校验基本就等于没上锁）
                checkTotal();
                // 获取到锁, 可以进行扣减
                --TOTAL;
                log.info("{} 扣减库存成功, 当前库存为: {}", Thread.currentThread().getName(), TOTAL);
                // 模拟业务操作耗时
                TimeUnit.NANOSECONDS.sleep(100);
                return "扣减库存成功, 剩余库存" + TOTAL;
            } else {
                return "尝试加锁失败，给用户友好提醒";
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
            return "加锁异常，需要开发排查";
        } finally {
            // 最后都需要判断是否为该线程持有锁，如果是，需要释放锁
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
      
    }
  
  	private void checkTotal() {
        Assert.isTrue(TOTAL > 0, "999", "当前商品已被抢购完~");
    }
}
```

### 结果

- 使用 Jmeter 模拟100线程并发

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1619597608969.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1619597652796.png)

