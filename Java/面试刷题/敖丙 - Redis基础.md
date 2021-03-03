#### 为什么使用 Redis？

- 因为 传统关系型数据库（如 MySQL）已经不能适用于所有场景
- 比如：
  - 秒杀的库存扣减
  - 应用的访问流量高峰
  - 登录token的分布式存储
  - ...
- 以上场景需要引入 `缓存中间件`
  - 市面上常用两种：
    - Redis
    - Memcached

#### Redis 和 Memcached 的区别

- Memcached 只支持 K-V  键值对、不支持持久化
- Redis 支持多种数据类型、支持持久化

#### Redis 数据结构类型

- String
- Hash
- List
- Set
- ZSet
- [HyperLogLog](https://zhuanlan.zhihu.com/p/58519480)
- [Geo](https://zhuanlan.zhihu.com/p/60011805)
- [Pub/Sub](https://zhuanlan.zhihu.com/p/136484218)
- [Redis Module](https://zhuanlan.zhihu.com/p/44685035) 
- [BloomFilter](https://zhuanlan.zhihu.com/p/149088311)

#### Redis 分布式锁

- 参考 Java -> 分布式框架专题 -> 诸葛 - Redis 下

#### Redis 1亿个key，找出10w个固定前缀开头的？

- scan命令，具体同样参考  Java -> 分布式框架专题 -> 诸葛 - Redis 

#### 线上使用 redis keys 命令有什么问题？

- 同上

#### Redis怎么做异步队列？

- 使用 List 数据结构，rpush 生产消息，blpop 消费消息（阻塞等待）