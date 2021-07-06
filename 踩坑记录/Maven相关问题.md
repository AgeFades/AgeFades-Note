## Maven下载不了源码

[参考链接](https://www.jianshu.com/p/a259e322794c)

### 操作案例

```shell
# 在项目根目录下执行如下命令，即重新下载依赖包
mvn dependency:resolve -Dclassifier=sources
```

