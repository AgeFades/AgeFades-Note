[TOC]

# 杨过 - Atomic & Unsafe

## 原子操作

- 不可被中断的一个或一系列操作

### 相关术语

| 术语名称     | 英文                   | 解释                                                         |
| ------------ | ---------------------- | ------------------------------------------------------------ |
| 缓存行       | Cache line             | 缓存的最小操作单位                                           |
| 比较并交换   | Compare and Swap       | 旧值 oldValue、新值 newValue<br>先用旧值与内存中的值比较，相等则替换成新值，否则失败 |
| CPU流水线    | CPU pipeline           | 在CPU中由5~6个不同功能的电路单元 组成 一条指令处理流水线<br>然后将一条x86指令分成5~6步后，再由这些电路单元分别执行<br>即实现，在一个CPU时钟周期完成一条指令，提高CPU运算速度 |
| 内存顺序冲突 | Memory order violation | 一般由 假共享 引起，当出现 内存顺序冲突 时，CPU必须清空流水线<br>假共享: 多核CPU同时修改同一个缓存行的不同部分，引起其中一个CPU的操作无效 |

### CPU如何实现原子操作

- 32位IA-32 CPU 使用 `基于对缓存加锁或总线加锁` 的方式来实现多处理器之间的原子操作

#### CPU自动保证基本内存操作的原子性

- CPU保证从系统内存中 `读取或写入一个字节` 是原子的
  - 即: 当一个CPU读取一个字节时，其他CPU不能访问该字节的内存地址
- 对于复杂的内存操作，CPU不能自动保证其原子性，例如:
  - 跨总线宽度
  - 跨多个缓存行
  - 跨页表的访问
- 但是CPU提供 `总线锁定` 和 `缓存锁定` 两种机制来保证 `复杂内存操作` 的原子性

#### 使用总线锁保证原子性

##### 问题

- 如果多个CPU同时对`共享变量`进行读改写（i++就是经典的读改写操作）
- 那么该 `共享变量` 的读改写操作就不是原子的
- 举例: i=1，进行两次 i++，期望结果是 i=3，但可能是 i=2

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1605853155767.png)

- 原因可能是，多个CPU同时从`各自的缓存`中读取变量i
- 分别进行加1操作，然后分别写入 `系统内存` 中

##### 思路

- 如果想保证读改写 `共享变量` 的操作是原子的
- 就必须保证CPU1读改写 `共享变量` 时，CPU2不能操作缓存了该 `共享变量` 内存地址的缓存

##### 实现

- CPU提供一个 `LOCK#` 信号
- 当CPU1在总线上输出此信号时，其他CPU的请求将被阻塞住，CPU1即可独占使用该 `共享内存`

#### 使用缓存锁定保证原子性

##### 问题

- 一般来说，在同一时刻，只需保证对 `某个内存地址的操作 ` 是原子的即可
- 而 `总线锁定` 把 CPU和内存之间的通信全锁住了，其他CPU都不能操作任何总线上的数据
- 所以，在某些场合下，使用 `缓存锁定` 代替 `总线锁定` 来进行优化

##### 思路

- 利用缓存一致性 MESI 保证多个CPU无法在同一时间对 `共享变量` 进行操作

##### 缺陷

- 当操作的数据不能被缓存在CPU内部 或 操作的数据跨多个缓存行
- 此时， `缓存锁定` 解决不了， CPU会调用 `总线锁定`

## Java中如何实现原子操作

1. 锁
2. 循环CAS
   1. CAS 利用 cmpxchg 指令实现
   2. 基本思路: 循环 `CAS` 直至成功

## Atomic

- Atomic 包中共12个类
- 四种原子更新方式
  - `原子更新基本类型`
  - `原子更新数组`
  - `原子更新引用`
  - `原子更新字段`
- Atomic 包的类，基本都是使用 Unsafe 实现的包装类

### 基本类

- `AtomicInteger`
- `AtomicLong`
- `AtomicBoolean`

#### 常用方法

```java
/**
 * 原子相加
 */
public final int addAndGet(int delta) {
  return U.getAndAddInt(this, VALUE, delta) + delta;
}


/**
 * 判断 AtomicInteger 中的 value 是否等于 expectedValue
 *
 * 是则将其设值为 newValue，并返回 true
 * 否则返回 false
 */
public final boolean compareAndSet(int expectedValue, int newValue) {
  return U.compareAndSetInt(this, VALUE, expectedValue, newValue);
}


/**
 * 原子自增
 *
 * 返回自增前的值
 */
public final int getAndIncrement() {
  return U.getAndAddInt(this, VALUE, 1);
}


/**
 * 最终会设值为 newValue
 *
 * 使用 lasySet 设值后，可能导致其他线程在之后一小段时间内、还是读到 oldValue
 */
public final void lazySet(int newValue) {
  U.putIntRelease(this, VALUE, newValue);
}


/**
 * 原子赋值为 newValue，返回 oldValue
 */
public final int getAndSet(int newValue) {
  return U.getAndSetInt(this, VALUE, newValue);
}
```

#### 注意

- `Unsafe` 只提供了三种 CAS 方法
  - compareAndSwapObject
  - compareAndSwapInt
  - compareAndSwapLong
- AtomicBoolean 中先将 Boolean 准换为整型，再使用 compareAndSwapInt 进行 CAS
- char、float、double 也可以用类似 AtomicBoolean 的思路实现

### 引用类型

- `AtomicReference`
- `AtomicReference` 的ABA实例
- `AtomicStampedReference`
- `AtomicMarkableReference`

### 数组类型

- `AtomicIntegerArray`
- `AtomicLongArray`
- `AtomicReferenceArray`

### 属性原子修改器

- `AtomicIntegerFieldUpdater`
- `AtomicLongFieldUpdater`
- `AtomicReferenceFieldUpdater`



