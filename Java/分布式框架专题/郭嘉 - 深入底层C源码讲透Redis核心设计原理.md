[TOC]

# 郭嘉 - 深入底层C源码讲透Redis核心设计原理

## 1. Redis K-V 底层设计原理

### Redis 基本特性

1. 非关系型的 **键值对** 数据库，可以根据键以O(1)的时间复杂度取出或插入关联值
2. Redis的数据是存在 **内存** 中的
3. 键值对中键的类型可以是 字符串、整型、浮点型等，且键是唯一的
4. 键值对中的值类型可以是 string、hash、list、set、sorted set 等
5. Redis 内战了复制、磁盘持久化、Lua脚本、事务、SSL、ACLs，客户端缓存，客户端代理等功能
6. 通过Redis哨兵和Redis Cluster模式提供高可用性

### Redis 应用场景

#### 缓存

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1658886521758.png)

#### 计数器

- 可以对 String 进行自增自减运算，从而实现计数器功能。Redis的读写性能非常高，很适合存储频繁读写的计数量。

#### 分布式ID生成

- 利用自增特性，一次请求一个大一点的步长，如 incr 2000，缓存在本地使用，用完再请求

#### 海量数据统计

- 用 bitmap 存储是否参与过某次活动，是否已读谋篇文章，用户是否为会员，日活统计

#### 会话缓存

-  可以用 Redis 来统一存储多台应用服务器的会话信息，当应用服务器不再存储用户的会话信息，也就不再具有状态。一个用户可以请求任意一个应用服务器，从而更易实现高可用性以及可伸缩性

#### 分布式队列/阻塞队列

- List是一个双向链表，可以通过 lpush/rpush 和 rpop/lpop 写入和读取消息。可以通过使用 brpop/blpop 来实现阻塞队列

#### 分布式锁实现

- 分布式场景下，无法使用基于单个进程的锁来对多个节点上的进程进行同步。可以使用 Redis 自带的 setnx 命令实现分布式锁

#### 热点数据存储

- 最新评论、最新文章列表等，可以使用 list 存储，ltrim 取出热点数据，删除老数据

#### 社交类需求

- Set 可以实现交集，从而实现共同好友等功能，Set通过求差集，可以进行好友推荐，文章推荐

#### 排行榜

- sorted set 可以实现有序性操作，从而实现排行榜等功能

#### 延迟队列

- 使用 sorted set，使用【当前时间戳 + 需要延迟的时长】做 score，消息内容作为元素，调用 zadd 来生产消息，消费者使用 zrangbyscore 获取当前时间之前的数据做轮询处理，消费完再删除任务 rem key memeber

### Redis 字符串的设计 SDS

- Redis 使用的字符串，不是C语言原生的字符串结构，而是自己设计的 SDS 类型
  - SDS：simple dynamic string

#### 1. 二进制安全的数据结构

- C语言中字符串举例：char data[] = "abc";

  - 会自动在末尾补上 \0，为 char data[] = "abc\0"，字符串的读取也就到 \0 为止
  - 而Redis接收各种语言客户端的数据流时，可能因各种原因，在字符串中间出现 \0，就会丢失后面的数据，如 "a\0bc"，就只会读到 a

- SDS 中结构大概为：

  - ```C
    sds:
    	// 内存空闲字节数（总长度 - 当前字节使用数）
    	free: 
    	// 当前字节使用数
    	len:
    	// 实际字节数据
    	char buf[] = ""
    ```

  - 接收数据时，会用 len 记录字符串长度，读取数据就按该数据长度来读取，从而避免丢失数据

#### 2. 提供了内存预分配机制，避免了频繁的内存分配

- C语言中，字符串在做类似 append、setbit 等函数操作时，都会重新开辟一个新的更大的内存空间，然后将原数据拷贝过去后，再进行添加的操作，就会产生频繁的内存分配
- 而SDS中，设计了 free 和 len 两个字段，当 free 空闲字节数 >= 新增字节数时，直接使用空闲字节空间，而不会产生新的内存分配
  - 当 free 空闲字节数 < 新增字节数时，扩容公式为（len + addlen）* 2，新的内存空间就会预留出部分空闲空间，以供下次新增数据使用
  - 注意，当字节总数为 1024 * 1024（1m）时，每次扩容只会添加 1m，而不是套用原先的公式

#### 3. 兼容C语言的函数库

- SDS也会在 buf 的末尾加上 \0，来兼容C语言的函数库

#### 4. SDS的不同版本数据结构

- redis 3.2 以前：

  - ```C
    struct sdshdr {
      int len;
      int free;
      char buf[];
    }
    ```

- redis 3.2 以后：

  - ```c
    typedef char *sds;
    
    // _attribute_ ((_packed_)) 用于内存对齐
    struct _attribute_ ((_packed_)) sdshdr5 {
      unsigned char flags;
      /* 3 lsb of type,and 5 msb of string length */
      char buf[];
    }
    
    struct _attribute_ ((_packed_)) sdshdr8 {
      // 当前使用内存长度
      uint8_t len;
      // 总的内存分配长度
      uint8_t alloc;
      unsigned char flags;
      char buf[];
    }
    
    struct _attribute_ ((_packed_)) sdshdr16 {
      uint16_t len;
      uint16_t alloc;
      unsigned char flags;
      char buf[];
    }
    
    struct _attribute_ ((_packed_)) sdshdr32 {
      uint32_t len;
      uint32_t alloc;
      unsigned char flags;
      char buf[];
    }
    
    struct _attribute_ ((_packed_)) sdshdr64 {
      uint64_t len;
      uint64_t alloc;
      unsigned char flags;
      char buf[];
    }
    ```

- 原因：int 是 4个字节，一个字节是 8个bit，4 * 8 = 32，32位，能表示的数据长度太大，一般来说，字符串长度都达不到那么大，而 string 又是 redis 中最常用使用最多的结构，那用 int 就会有大量的空间浪费

- 所以 3.2 以后，设计了多个 sds 对象，不同的后缀表示不同的位数区间，用于节省内存的使用

#### 5. flags 的含义

- flags一个字节，8个bit位，前三个bit为表示 sds 的类型，后5个bit 在 sdshdr5 中表示字符串长度，其余的为空闲未使用（5个bit位表示不了那么大的数据）
- 指向 buf 的指针，往前偏移一位就是 flags（内存对齐 成的连续内存空间）
  - 获取flags即可以获取其类型

<img src="https://agefades-note.oss-cn-beijing.aliyuncs.com/1658889854967.png" style="zoom:50%;" />

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1658889884574.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1658889901598.png)

### 键值数据库的设计

- RedisDB 主体数据结构

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1658976645735.png)

#### K-V

- key：即为上面讲的 string 类型
- value：可以为 string、hash、set、sorted set、list ...
- K-V 在 Java 中有 Map，在 Redis 中对应的即为 Dict

#### DB

- 数据库：海量数据的存储
- Redis使用的数据结构 及 随机访问的时间复杂度：
  1. 数组：O(1)
  2. 链表：O(N)
- 数组索引的计算方式：
  - hash(key)  =  自然数
    - 即利用 hash 函数来做散列
  - 此时得到的自然数是个非常大的值，比如（2132315312243），对内存消耗也会很大
    - Redis 会对该自然数做 取模 操作，从而减小内存空间的消耗
    - 举例：创建一个数组为 arr[4]
      - 则索引为：hash(key) -> 自然数[大] % 4 = [0, arr.size - 1]

#### Redis解决哈希冲突的设计

##### Hash函数的两个特点

1. 任意相同的输入，一定得到相同的输出
2. 不同的输入，有可能得到相同的输出（哈希冲突）
   - 举例：现有数组为 arr[4]、(k1,v1) (k2,v2) (k3,v3)
     - hash(k1) % 4 = 0
     - hash(k2) % 4 = 1
     - hash(k2) % 4 = 1
   - Redis 也是采用 数组加链表 + 头插法 来解决 Hash冲突
     - arr[0] -> (k1,v1, next -> null)
     - arr[1] -> (k3,v3, next -> k2) (k2, v2, next -> null)
   - 实际中，取模的运算都会被优化
     - 任意数 % 2^n -->  任意数 & (2^n - 1) 
       - 2的n次方，是为了哈希散列更加平均
       - 模运算变成与运算，是因为底层执行模运算时，可能要做循环的除法，而与运算只需要一次

#### Redis DB的数据结构

- Redis 有16个数据库（就是一个长度为16的数组 redisDb 对象）
  - 不要纠结为什么16个，可以在redis.conf文件里配置

```c
typedef struct redisDb {
  dict *dict;		// K-V 键值对数据集合
  dict *expires;	// Key过期时间维护集合
  dict *blocking_keys;	// 维护阻塞队列 Key和Client 的联系关系集合
  dict *ready_keys;			
  dict *watched_keys;		// 关于事务处理的
  int id;		// 数据库id，如 select 0 
  long long avg_ttl;
  unsigned long expires_cursor;
  list *defrag_later;
} redisDb;
```

#### Redis 中的Dict结构

- 类似于Java中的HashTable

```c
typedef struct dict {
  // 类型
  dictType *type;
  void *privdate;
  // hashtable 的缩写
  dictht ht[2];
  long rehashidx;
  unsigned long iterators;
} dict;
```

##### dictType

- 代表类型

```c
typedef struct dictType {
  // 哈希函数
  uint64_t (*hashFunction)(const void *key);
  void *(*keyDup)(void *privdata, const void *key);
  void *(*valDup)(void *privdata, const void *obj);
  // 哈希冲突的比对函数
  int (*keyCompare)(void *privdata, void *key);
  void (*keyDestructor)(void *privdata, void *key);
  void (*valDestructor)(void *privdata, void *Obj);
} dictType;
```

##### dictht

- 每个 Dict 都有两个 dictht，是为了实现渐进式的rehash

```c
typedef struct dictht {
  // 具体的K-V对象数组
  dictEntry **table;
  unsigned long size; // 容量 默认16，永远都是2的n次方
  unsigned long sizemask; // size - 1，用来做与运算的
  unsigned long used; // hashtable 元素个数
} dictht;
```

##### dictEntry

- 键值对具体封装成的对象

```c
typedef struct dictEntry {
  // key 指向一个 SDS 对象
  void *key;
  // 值的类型
  union {
    // 指向值的指针
    void *val;
    uint64_t u64;
    int64_t s64;
    double d;
  } v;
  // 用来解决哈希冲突时的链表 next 指针
  struct dictEntry *next;
} dictEntry;
```

##### redisObject

- K-V 中 V 的对象封装类型
- 读数据的优化，一个 redisObject 是占16个字节，一个缓存行是64个字节
  - 以 string 类型为例，set 一个长度为 44 的字符串，那么该 redisObject 和数据就会存在一个 cache line 中，是连续的内存空间，提高性能，类型就是 embstr
  - 超过 44 长度的，就是 raw

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1658978123464.png)

```c
typedef struct redisObject {
  // 类型, string hash list ... 用于约束客户端命令
  unsigned type:4;
  // 代表值底层的编码类型
  unsigned encoding:4;
  // 用于内存淘汰
  unsigned lru:LRU_BITS;
  // 使用引用计数法管理内存
  int refcount;
  // 具体指向真实数据的地址指针
  // 8个字节，64个bit，跟整型的空间一样
  // redis在set命令里会做判断，如果是整型，就直接存在这个字段上，而不是指针
  // 这样可以减少内存空间使用、和拿指针去内存中找实际值的IO开销
  void *ptr;
} robj;
```



## 2. Redis 渐进式rehash及动态扩容机制

- 扩容的原因：
  - 比如 arr[4]，现在有100w条数据进来，必然造成大量的哈希冲突，数组节点下的链表就会特别长，从而对数据的访问操作退化成 O(n)
  - 这时候就要进行数组扩容，并重新hash分配数据

- Redis DB 中 Dict 数组发生扩容时，内存容量是成倍增长的。
  - 举例：arr[4] 扩容后，即为 arr[8]
  - 如果直接将原先的数据，迁移到新的数组里，数据量特别大时，就会造成客户端访问卡顿（严重甚至业务宕机），比如一次性迁移 10个亿的数据
- Redis 的渐进式rehash设计：
  - 每个 Dict 中都有两个 dictht，一个是旧表，一个是新表
  - 每次访问，先访问旧表，旧表有该数据时，就会将旧表数据重新 hash 后迁移到 新表中
    - 而不是一次性把所有数据迁移过去
    - 如果没有对该桶的数据进行访问，也会有后台线程轮训搬运，直至搬空为止
  - 新的数据就会落在新表中，直至旧表清空

## 3. Redis核心编码结构精讲

### List

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1658989070513.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1658988350392.png)

- List是一个有序（按加入的时序排序）的数据结构，Redis采用 quicklist（双端链表）和 ziplist 作为 List 的底层实现
  - 可以通过设置每个 ziplist 的最大容量，quicklist 的数据压缩范围，提升数据存取效率
    - list-max-ziplist-size -2 ：单个ziplist节点最大能存储8kb，超出则进行分裂，将数据存储在新的ziplist节点中
    - list-compress-depth 1 ：0代表所有节点，都不进行压缩，1代表从头结点往后走一个，尾结点往前走一个 不用压缩，其他的全部压缩
  - 为什么不用单纯的链表？
    1. 胖指针占用内存过多、pre、next 各8字节，可能比元素本身占用的内存空间都大
    2. 产生大量内存碎片，各个节点用指针去建立关联关系，并不在连续的一片内存空间中

### Hash

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1658989337061.png)

- Hash 底层实现为一个字典（dict），也是 RedisDb 用来存储 K-V 的数据结构
  - 当数据量比较小，或单个元素比较小时，底层用 ziplist 存储
  - 数据大小和元素数量阈值可以通过如下参数设置：
    - hash-max-ziplist-entries 512 ：ziplist 元素个数超过512，将改为 hashtable 编码
    - hash-max-ziplist-value 64 ：单个元素大小超过64 byte时，将改为 hashtable 编码

### Set

- Set是无序、自动去重的集合数据类型，底层实现为一个 value 为 null 的字典（dict）
  - 当数据用整型表示时，Set集合将被编码为 intset 数据结构
- 以下两个条件任一满足时，Set将用 hashtable 存储数据
  1. 元素个数大于 set-max-intset-entries 512 ：intset 能存储的最大元素个数
  2. 元素无法用整型表示

#### intset

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1658989734712.png)

- 整数集合是Redis中用来存储有序、数据不重复的整型数据的结构
  - 有 int16_t、int32_t、int64_t
  - encoding：编码类型
  - length：元素个数
  - contents[]：元素存储

### ZSet

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1658989996251.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1658990070096.png)

- ZSet是有序、自动去重的数据类型，底层实现为 字典（dict）+ 跳表（skiplist）
  - 当数据较少时，用 ziplist 编码存储结构
  - zset-max-ziplist-entries 128 ：元素个数超过128，将用skiplist编码
  - zset-max-ziplist-value 64：单个元素大小超过64byte，将用skiplist编码

### GeoHash

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1658990293647.png)

- GeoHash 是一种地理位置编码方法，将地理位置编码为一串简短的字母和数字。
  - 是一种分层的空间数据结构，将空间细分为网格形状的桶
  - 这是所谓的z顺序曲线的众多应用之一，通常是空间填充曲线
- （Redis 的 Geo 偏差问题比较突出，具体细节就不再记录了）

## 4. 亿级用户日活统计BitMap实战及源码分析

- bit 二进制的表示方式，举例：01010101001
- 凡是只有两种状态的都可以用 bit 表示，举例：登陆，0 表示未登陆，1 表示登陆
  - 假设用户id是整型，如：1223213123123，就可以作为 bitmap 的索引位
  - 即: bitmap 中的 offset(偏移量)
  - 以当前登陆日期为 key，如：login:active:2022-07-28

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1658978763668.png)

- bitcount key [start end]，统计值为1的数量
  - start end 是字节数的范围，不是偏移量的范围
  - bitcount 用的是 [汉明重量] 的位运算方式高效统计的
- 一个 bitmap 最大是 512m内存，offset 最大是 2^32 - 1

### 统计用户连续登陆

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1658979315170.png)

- 按位与后，做一次bitcount

### 统计用户本周登陆过的

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1658979394123.png)

- 按位或后，做一次bitcount

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1658979675986.png)