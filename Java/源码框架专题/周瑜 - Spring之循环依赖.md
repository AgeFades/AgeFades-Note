[TOC]

# 周瑜 - Spring之循环依赖

## 循环依赖问题演示

![](https://cdn.nlark.com/yuque/0/2020/png/365147/1592471211638-86636131-146d-46c3-8775-421ef3322cc3.png)

## 三级缓存

1. 一级缓存：`singletonObjects`
   - 存放已经经历了 完整生命周期的 bean对象
2. 二级缓存：`earlySingletonObjects`
   - 存放早期 bean对象
   - 即未完成整个生命周期的 bean对象
3. 三级缓存：`singletonFactories`
   - 存放 `ObjectFactory` 对象工厂
   - 表示用来创建早期 bean对象 的工厂

## 二级缓存解决循环依赖图解

![](https://cdn.nlark.com/yuque/0/2020/png/365147/1592471597769-3e23cc26-2b1d-4742-8c74-cea46327ada7.png)

