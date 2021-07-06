# Redis 问题集锦

## RedisTemplate setBit 的设计缺陷

### 官方文档连接

[Redis官方文档](https://redis.io/commands/setbit)

### 问题演示

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1606808197612.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1606808225263.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1606808258299.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1606808287212.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1606808303888.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1606808325589.png)

### 问题描述

- redis setbit 官方文档描述，返回值为 key 此偏移量处的原始值
  - 而 bitmap 中所有初始位都为 0，所以 setbit 第一次设置成功返回为 0
  - redisTemplate 接收到的值为 0，却使用 LongToBoolean 转为 Boolean，convert 方法则为 return result == 1
- 所以，结果为 setbit 实际成功，但 redisTemplate API 返回值为 false
  - 造成代码误判，举例如下
  - 如: if ( redisTemplate.setbit() ) { log.info("签到成功") }

