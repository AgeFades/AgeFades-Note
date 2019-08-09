# 黄康 - Redis 实现分布式锁

## Redission 实现分布式锁

```xml
        <dependency>
            <groupId>org.redisson</groupId>
            <artifactId>redisson-spring-boot-starter</artifactId>
            <version>3.11.0</version>
        </dependency>
```

```yaml
# 集群版本配置
spring:
  redis:
    cluster:
      nodes: 39.108.158.33:16371,39.108.158.33:16372,39.108.158.33:16373,140.143.0.227:16371,140.143.0.227:16372,140.143.0.227:16373 #集群地址
    jedis:
      pool:
        min-idle: 4 #最小空闲连接数，默认0
        max-idle: 8 #最大空闲连接数，默认8
        max-wait: -1ms #连接池最大阻塞等待时间（使用负值表示没有限制），默认-1
        max-active: 10 #连接池最大连接数（使用负值表示没有限制）,默认8
    timeout: 5000ms #连接超时时间
```

``` yaml
# 单机配置
spring:
  redis:
    port: 20177 #端口号
    password: agefades #密码
    database: 12 #使用数据库
    timeout: 5000ms #超时时间
    host: 127.0.0.1 #redis地址
```

```java
@RestController
@RequestMapping("redission")
public class TestLockController {

    @Autowired
    private RedissonClient redisson;

    @GetMapping("lock")
    public String lock(@RequestParam String lock){
        //获取锁对象
        RLock lock1 = redisson.getLock(lock);
        //上锁，并且设置强制解锁时间，防止死锁
        lock1.lock(3,TimeUnit.SECONDS);
        System.out.println("上锁成功！");
        try {
            //睡眠一秒
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        //解锁
        lock1.unlock();
        System.out.println("解锁成功！");
        return "redission锁实现";
    }
}
```

## RedisTemplate

```java
// 我们需要获取一个redis数据如果他不存在我们则给他添加，但是只能有一个人去查询数据库并进行添加，也就是说只有一个人能拿到锁，那么拿不到锁的对象睡眠100毫秒后调用自身，此时已经添加直接走redis

public void get() throws InterruptedException {
//        从Redis读取数据，如果没有读取到，则缓存没有添加或者失效防止缓存击穿
        Object o = redisTemplate.opsForValue().get("123");
        if(o == null){
            System.out.println("尝试获取锁对象");
            Boolean lock = redisTemplate.opsForValue().setIfAbsent("lock", 321);
            if(lock){
                String name = "Agefades";
//                拿到锁，给Redis添加缓存
                redisTemplate.opsForValue().set("123",name);
                //给锁设定超时时间防止死锁
                redisTemplate.opsForValue().set("lock",name , 3L, TimeUnit.SECONDS);

            }else {
//                没有拿到锁则递归调用当前方法，此时先睡眠100毫秒防止数据未添加读取
                Thread.sleep(100);
            }
            System.out.println(lock ? "成功获取到锁" : "获取锁失败");
//            调用自生方法，此时数据已经添加到Redis中直接重新读取返回
            get();
        }else{
//            已经获取到数据直接返回查询到的数据
            System.out.println(o);
        }
    }


    @Test
    public void testGet() throws InterruptedException {
        for (int i = 0; i < 30; i++) {
            new Thread(() -> {
                try {
                    get();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }

```

