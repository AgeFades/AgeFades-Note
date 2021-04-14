# 徐庶 - SpringBoot自动装配

## 流程图

[SpringBoot自动装配流程图](https://www.processon.com/view/link/5fc0abf67d9c082f447ce49b)

## @SpringBootApplication

- 标记该类为主配置类
- 运行该类 main() 启动整个 SpringBoot 应用

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1617953147950.png)

### 注解说明

#### @Target

- 描述注解的使用范围
  - `CONSTRUCTOR`：描述构造器
  - `FIELD`：描述字段
  - `LOCAL_VARIABLE`：描述局部变量
  - `METHOD`：描述方法
  - `PACKAGE`：描述包
  - `PARAMETER`：描述参数
  - `TYPE`：描述类、接口、枚举

#### @Retention

- 标记注解会被保留到哪个阶段
  - `RUNTIME`：被 JVM 保留，在运行时能被 JVM 或 其他反射机制读取和使用
  - `CLASS`：编译时保留，class文件中存在，JVM中忽略
  - `SOURCE`：源码保留，编译忽略

#### @Documented

- 生成 Java Doc

#### @Inherited

- 标记会被自动继承（子类继承父类所有属性）

#### @SpringBootConfiguration

- 标记为 SpringBoot 配置类

#### @EnableAutoconfiguration

- 标记开启自动装配

#### @ComponentScan

- 扫描包（扫描指定路径Bean，加载至IOC）
- 未指定时，自动扫描 `当前配置类所有同级及子级` 的包路径
  - `TypeExcludeFilter`：自定义排除 SpringBoot 提供的扩展类
  - `AutoConfigurationExcludeFilter`：过滤会自动装配的配置类，避免重复

