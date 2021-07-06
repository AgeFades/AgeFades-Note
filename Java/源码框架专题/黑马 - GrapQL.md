# 黑马 - GraphQL

## GraphQL 入门

### GraphQL 是什么

```shell
# GraphQL 是由 Facebook 创造的用于描述复杂数据模型的一种查询语言。
	# 这里的查询语言所指的不是常规意义上的类似 SQL 语句的查询语言
	# 而是一种用于前后端数据查询方式的规范
	
# 官网(中文)
	# https://graphql.cn/
	
# 规范地址
	# http://spec.graphql.cn/
```

### 分析 RESTful 存在的问题

```shell
# 举例如下

# 请求
GET http://127.0.0.1/user/1001
# 响应：
{
    id : 1001,
    name : "张三",
    age : 20,
    address : "北京市",
    ……
}

# 可能只需要 id 和 name 属性，那么就存在资源浪费
# 可能一次请求不满足需求，需要多次请求
```

### 进一步了解 GraphQL

```shell
# 按需索取数据，避免浪费
```

![UTOOLS1568620077279.png](https://i.loli.net/2019/09/16/vMRbpzc5stxrUAI.png)

```shell
# 一次查询多个数据
```

![UTOOLS1568620176759.png](https://i.loli.net/2019/09/16/mvAfyYKiX7odRZF.png)

```shell
# API 的演进无需划分版本
```

![UTOOLS1568620255164.png](https://i.loli.net/2019/09/16/gVaMb9Fw4RWY1Zl.png)

![UTOOLS1568620281691.png](https://i.loli.net/2019/09/16/SasoIPEC2iUBAnY.png)

### GraphQL 查询的规范

```shell
# GraphQL 定义了一套规范，用来描述语法定义，具体参考 http://graphql.cn/learn/queries/
```

#### 字段( Fields )

```shell
# 在 GraphQL 的查询中，请求结构中包含了所预期结果结构，这个就是字段。
	# 并且响应结构和请求结构基本一致。
```

![UTOOLS1568620463201.png](https://i.loli.net/2019/09/16/XwBjlAkvWaTdq3J.png)

#### 参数( Arguments )

```shell
# 查询数据时，离不开传递参数，在 GraphQL 的查询中，语法为 参数名:参数值
```

![UTOOLS1568620529434.png](https://i.loli.net/2019/09/16/WyqNsXrYTvzBFuL.png)

#### 别名( Aliases )

```shell
# 如果一次查询多个相同对象，但是值不同，这个时候就需要起别名了，否则无法通过 json 语法
```

![UTOOLS1568620675461.png](https://i.loli.net/2019/09/16/qjDvJ6Mc3RsKIAl.png)

####  片段( Fragments )

```shell
# 查询的属性如果相同，可以采用片段的方式进行简化定义
```

![UTOOLS1568620739507.png](https://i.loli.net/2019/09/16/k9DYKtOFhlvozIs.png)

### GraphQL 的 Schema 和类型规范

```shell
# Schema 是用于定义数据结构的，比如说，User 对象中有哪些属性，对象之间的关联关系等
```

#### Schema 定义结构

![UTOOLS1568620857934.png](https://i.loli.net/2019/09/16/XM7RLnucilY5gok.png)

#### 标量类型( Scalar Types )

```shell
# GraphQL 规范中，默认定义了 5 中类型
	# Int : 有符号 32 位整数
	# Float : 有符号双精度浮点值
	# String : UTF-8 字符序列
	# Boolean : true Or false
	# ID : ID 标量类型表示一个唯一标识符，通常用以重新获取对象或者作为缓存中的键
	
# 以上类型不能满足所有需求，所以各语言在实现中，都有扩充
	# 例如 graphql-java 实现中增加了 Long、Byte 等
```

#### 枚举类型

```shell
# 特殊的标量，限制在一个特殊的可选值集合内
```

![UTOOLS1568621073761.png](https://i.loli.net/2019/09/16/6jEMfBR9zgNi5DT.png)

#### 接口( interface )

```shell
# GraphQL 支持接口
	# 一个接口是一个抽象类型，它包含某些字段，而对象类型必须包含这些字段，才能算实现了这个接口
```

![UTOOLS1568621309353.png](https://i.loli.net/2019/09/16/TSCY48Dyvs91UBj.png)

## GraphQL 的 Java 实现

```shell
# 此处使用官方推荐的实现 graphql-java
	# 官网 : https://www.graphql-java.com/
	# github : https://github.com/graphql-java/graphql-java
```

### 开始使用

#### 导入依赖

```xml
<dependency>
    <groupId>com.graphql-java</groupId>
    <artifactId>graphql-java</artifactId>
    <version>11.0</version>
</dependency>
```

```xml
<!-- graphql-java 并没有发布到 Maven 中央仓库，需配置第三方仓库,setting.xml 配置 -->
<profile>
    <id>bintray</id>
    <repositories>
        <repository>
            <id>bintray</id>
            <url>http://dl.bintray.com/andimarek/graphql-java</url>
            <releases>
            	<enabled>true</enabled>
            </releases>
            <snapshots>
            	<enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>
    <pluginRepositories>
        <pluginRepository>
            <id>bintray</id>
            <url>http://dl.bintray.com/andimarek/graphql-java</url>
            <releases>
            	<enabled>true</enabled>
            </releases>
            <snapshots>
           	 	<enabled>false</enabled>
            </snapshots>
        </pluginRepository>
    </pluginRepositories>
</profile>
…………………………………………
    <activeProfiles>
  	 	 ………………
         <activeProfile>bintray</activeProfile>
    </activeProfiles>
```

```shell
# 具体使用感觉太麻烦了，不再记录。
```

