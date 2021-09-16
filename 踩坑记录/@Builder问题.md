[参考链接](https://blog.csdn.net/weixin_42332985/article/details/108498550)

## 问题

- 使用Lombok的`@Builder`注解后，**编译后实体有全参数的构造方法，那么编译器就不会再默认提供一个默认的无参构造。** 所以问题可能就出现在这个无参构造方法缺失上，**mybatis在将查询结果映射成实体时是通过无参构造获得一个实例，然后调用setter方法将值存入相应字段的**。

## 解决

- 添加Lombok注解
  - `@NoArgsConstructor`和 `@AllArgsConstructor`

