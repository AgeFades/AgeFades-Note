# 徐庶 - Spring中的设计模式

## 简单工厂模式

### 实现方式

- 通过 Spring 中的 `BeanFactory` 实现
- 根据传入一个唯一标识来获得相应 Bean 对象

### 实质

- 由一个 `工厂类` 根据 `传入的参数`，`动态 `决定应该创建哪一个 `产品类`

### 实现原理

#### Bean容器的启动阶段

- 读取 bean 配置，将 bean 元素分别转换成一个 BeanDefinition 对象
- 通过 BeanDefiniitionRegistry 将这些 BeanDefinition 注册到 BeanFactory 中
  - 保存在 BeanFactory 中的一个 ConcurrentHashMap 中
- 注册完后，可以通过实现 BeanFactoryPostProcessor 在此处插入开发自己定义的代码
  - 例: `PropertyPlaceholderConfigurer`
  - 即一般在配置 数据库 的 dataSorce 时，使用到的占位符的值，就是它注入进去的

