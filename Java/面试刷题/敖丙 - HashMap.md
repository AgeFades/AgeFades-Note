#### HashMap 的底层数据结构

- 由 `数组` + `链表` + `红黑树` 组成的散射列表
- `数据结构` 里存储的是:
  - Key - Value 键值对
  - JDK7 叫做 `Entry`
  - JDK8 叫做 `Node`

![](https://tva1.sinaimg.cn/large/006tNbRwly1g9pchhbrp3j30ez02ngli.jpg)

#### HashMap 的存取原理

- `数据结构` 的 `Node` 里，本身所有位置都为 `NULL`
- 在 `put()` 插入元素时，会根据 `key` 的 `hash` 计算一个 index 下标值 进行填充

![](https://tva1.sinaimg.cn/large/006tNbRwly1g9pcqyo35ij30et03d0sq.jpg)

- `数组` 长度有限，当发生 `hash碰撞` 时，就形成了 `链表`
  - 如:  "张三"、"李四" 两个 key 的 hash 值一样，都落到了同一个数组下标

![](https://tva1.sinaimg.cn/large/006tNbRwly1g9pd6ckj3dj30eq06mmx8.jpg)

- `Node` 属性:
  - hash
  - key
  - value
  - next（下个节点）

![](https://tva1.sinaimg.cn/large/006tNbRwly1g9pcz67kemj30fv097759.jpg)

#### JDK7 和 JDK8 的区别

- JDK7 插入元素是 `头插法`
  - hash碰撞时，新来的值会位于表头
  - 考虑新来的值，被查找的可能性更大，位于表头可提升查询效率
- JDK8 插入元素是 `尾插法`
  - hash碰撞时，新来的值会位于表尾
  - 避免多线程下 resize 扩容、产生 `环形链表` 出现 `Infinite Loop`
- JDK8 数组长度 > 64、链表长度 > 8时，链表会变成 `红黑树`
  - 控制树的高度、提高查询性能

#### 为什么线程不安全

- 多线程操作HashMap，put()添加元素产生 resize 扩容时
  - JDK7使用 `头插法` 可能导致 `环形链表` 出现 `Infinite Loop`

![](https://tva1.sinaimg.cn/large/006tNbRwly1g9pkaxgg5ij305007odfr.jpg)

![](https://tva1.sinaimg.cn/large/006tNbRwly1g9phcpl968j309j054747.jpg)

![](https://tva1.sinaimg.cn/large/006tNbRwly1g9pkh8omjyj30al06pmx4.jpg)

- JDK8 使用 `尾插法`  避免了 `环形链表` 的产生
  - 但是 put()、 get() 都没有任何锁机制
  - 所以，多线程下，无法保证上一秒 put() 的值，下一秒 get() 的时候还是原值
    - 简单举例: A put()，未插入完成时，B put() 同 key 不同值完成，A put() 完成
    - 此时 B get() 的就是 A put() 的值

#### 有线程安全的类代替吗

- HashTable（不推荐）
  - 直接在方法上锁，并发度很低，只支持最多同时一个线程访问
- ConcurrentHashMap（推荐）

#### 默认初始化大小是多少？

- 16

![](https://tva1.sinaimg.cn/large/006tNbRwly1g9plqfeg1pj3174054jsb.jpg)

##### 为什么是这么多？

- 为了实现 `Hash均匀分布`

##### 为什么大小都是2的幂？

- 如果不是2的幂，Length - 1 的值是所有二进制位全为1
- 这种情况下，index 的结果就等同于 HashCode 后几位的值
- 就会增加 Hash碰撞、不能均匀分布

#### HashMap扩容

- 数组容量有限，在多次插入后，Node 数达到一定的数量，就会进行 resize 扩容

##### HashMap的扩容时机

- 主要看两个属性
  - `Capacity`：HashMap当前数组长度
  - `LoadFactor`: 负载因子，默认值 `0.75f`

![](https://tva1.sinaimg.cn/large/006tNbRwly1g9pdw39rwjj30xi056wf3.jpg)

- 比如:
  - HashMap 当前数组长度为 100，存第 76 个元素的时候
  - put() 首先会判断，76 > 100 * 0.75，所以就进行 扩容

##### HashMap的扩容方式

- 扩容步骤分为两步:
  - 创建一个新的 Entry 空数组，长度是原数组的 2倍
  - `ReHash`: 遍历原 Entry 数组，把所有的 Entry 重新 Hash 添加到新数组

##### 扩容时，为什么不直接从原数组复制到新数组？

- 因为数组长度扩大后，Hash 的规则也随之改变
- `Hash公式`:
  - index = HashCode（Key）& （Length - 1）
- 如:
  - Length = 8 时，Hash 得到 index = 2
  - Length = 16时，Hash 得到 index = 14

#### HashMap的主要参数都有哪些？

- 负载因子（默认0.75）
- 初始容量（默认16）

#### HashMap是怎么处理hash碰撞的？

- 数组节点转链表、链表转红黑树

#### Hash的计算规则？

- index = HashCode（Key）& （Length - 1）

#### 为什么重写 equals() 时，需要重写 hashCode()?

- Java中，所有的对象都是继承于 Object 类
  - Object 中两个方法 equals()、hashCode()，都是用于比较两个对象是否相等
  - equals() 用于比较两个对象的内存地址
    - 对于 `值对象`，== 比较的是值
    - 对于 `引用对象`，比较的是内存地址
- HashMap中，是通过对 key 的 hashCode() 计算 index
  - 当产生 Hash冲撞 时，就形成了链表
  - 比如：以下两个 key 在同个链表上
    - "你好" 的 index 为 2
    - "世界" 的 index 为 2
  - 此时，get() 时，就只能通过 equals() 获取目标对象
    - 所以，如果对 equals() 进行了重写，一定要对 hashCode() 重写
    - 用于保证 相同的对象返回相同的 hash，不同的对象返回不同的 hash