# Spring 面试整理

## 请你谈谈对 IOC 的理解

```shell
# Spring 的编程思想:
	# 源语容器，止于容器。
	
# 如果谈及 IOC 容器，你只知道 控制反转、依赖注入 这两个概念，那么你对 IOC 只能说一知半解。
```

### 控制反转

```shell
# 控制反转，应该是拆分为 控制 和 反转 两个词的。
```

####  控制

```shell
# 控制:
	# 这里是指 控制对象的创建的权力。
```

```java
class You {
  
  private Money money = new Money();
  private Handsome handsome = new Handsome();
  private GodJob godJob = new GodJob();
  
  BeautyGirl beautyGirl = new BeautyGirl(money, handsome, godJob);
  
  public void doHappyThings(BeautyGirl beautyGirl) {
    // TODO 和漂亮女孩做开心的事。
  }
  
  public static void main(String[] args) {
    You you = new You();
    you.doHappyThings(beautyGirl);
  }
  
}
```

```shell
# 从上述伪代码看来，You 这个对象做 doHappyThings 的过程是很繁琐的，
	# 需要我们程序员手动控制创建各种对象。
	# 这就是 对象的创建的权力。
```

```shell
# 利用 xml 和 注解 将对象导入 Spring
	# xml 中 <bean></bean>

	# @Bean

	# @Import

	# @CompentScanner + @Compent @Service @Repository @Controller....
	
# 这些被导入 Spring IOC 中的对象称之为: bean
```

#### 反转

```shell
# 反转:
	# 这里是指 控制对象的创建的权力被反转了。
	# 如上述伪代码中的对象，不再是由程序员手动创建，而是交给 Spring 容器工厂来创建。
	# Spring 容器工厂创建好了对象，我们只需要通过各种方式索取即可。
```

```java
class You {
  
  // 直接从 Spring 容器工厂索取需要的对象即可
  @Autowired
  private BeautyGirl beautyGirl;
  
  public void doHappyThings(BeautyGirl beautyGirl) {
    // TODO 和漂亮女孩做快乐的事情
  }
  
  public static void main(String[] args) {
    You you = new You();
    you.doHappyThings(beautyGirl);
  }
  
}
```

```shell
# 从上述两段伪代码就可以很清晰的分辨出，哪种方式更加友好。

# 直白的来说，Spring 容器工厂（IOC）将 业务代码 和 组件创建代码 彻底的分离开来了。
	# 程序员只需要专注业务代码，不需要关心所需组件是如何创建来的。
```

### 依赖注入

#### 依赖

```shell
# 依赖:
	# 对象之间的依赖关系。
	# 例如: 上述 You 这个对象，需要做 doHappyThings 就对 BeautyGirl 产生了依赖。
	# 直白的来说，就是 You 对象中需要使用到 BeautyGirl 这个对象。
```

#### 注入

```shell
# 注入:
	# 依赖对象里的属性注入。
	# 例如: 上述 You 这个对象，注入了 BeautyGirl 这个对象。
	# 而被注入的对象就是通过 Spring 容器工厂控制反转创建出来的对象。
```

### 结论

```shell
# 所以从上述结论中，得知:
	# 正是由于有了 控制反转（将对象的创建交给 IOC 容器管理），
	# 才能支持 依赖注入（向 IOC 容器中获取所需对象进行注入）。
	
# 而 依赖注入 这个功能，会引入一个容器的先天缺陷:
	# 循环依赖。
```

### 循环依赖演示

```java
class You {
  private BeautyGirl beautyGirl;
}

class BeautyGirl {
  private You you;
}
```

```shell
# 上述代码就完美诠释了 循环依赖 的概念。
	# A 依赖 B，B 依赖 A
```

### 如何解决循环依赖

```shell
# 二级缓存、三级缓存
	# 二级缓存可以保证拿到 普通对象 的原引用。

# Spring 循环依赖为啥要三级缓存？
	# 三级缓存可以保证拿到 代理对象 的原引用。
```

### 进阶回答

```shell
# Spring IOC 核心功能（控制反转、依赖注入）创建、管理我们的组件（Controller、Service、Dao...）

# 各种第三方开源框架就利用该功能，通过实现 Spring IOC 核心接口功能，将自己的组件也交给 IOC 管理
	# 例如: RedisTemplate、AmqpTemplate...
	# 极大的简化了程序员的工作，不再需要自己通过繁琐的组件对象创建，而只需要向 Spring IOC 容器索取即可。
	
# 举例 三方框架 通过实现 Spring 预留的扩展接口，将自己的组件交给 Spring IOC 容器管理:
	# MyBatis 利用 Spring 的扩展接口 BeanFacotryPostProcessor
	# Nacos 利用 ApplicationListener
	# Eureka 利用 SmartLifecycle
	# Ribbon 利用 SmartInitializingSingleton
	# Sentinel 利用 BeanPostProcessor
```