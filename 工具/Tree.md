## 简介

- tree命令可以显示文件夹下的文件结构
- 可以获取开发项目目录结构、用于结构说明

## 安装

```shell
# Mac 安装 tree 命令，遇到 brew 的问题自行解决
brew install tree
```

## 使用

```shell
# 到指定文件夹下使用
# -d 表示递归
# -L 2 表示递归2层、2是可变参
tree -d -L 2 ./ 
```

## 案例

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1619432578992.png)

