# 黑马 - 网站高可用 nginx + lua

## Lua 

### Lua 是什么

```shell
# lua 是一个小巧的脚本语言。

# lua 由标准 C 编写而成，几乎在所有操作系统和平台上都可以编译，运行。

# lua 没有提供强大的库，这是由它的定位决定的，所以不适合独立开发应用程序。

# lua 有一个同时进行的 JIT 项目，提供在特定平台上的即时编译功能。
```

### Lua 特性

```shell
# 支持 面向过程编程 & 函数式编程

# 自动内存管理:
	# 只提供了一种通用类型的表，用它可以实现数据，哈希表，集合，对象

# 语言内置模式匹配;
	# 闭包;
	# 函数也可以看做一个值;
	# 提供多线程（协同进程，并发操作系统所支持的线程）支持;
	
# 通过闭包和 table 可以很方便的支持面向对象编程所需要的一些关键机制:
	# 比如数据抽象，虚函数，继承和重载等
```

### Lua 应用场景

```shell
# 游戏开发

# 独立应用脚本

# Web 应用脚本

# 扩展和数据库插件如:
	# MySQL Proxy 和 MySQL WorkBench
	
# 安全系统，如入侵检测系统

# redis 中嵌套调用实现类似事务的功能

# web 容器中应用处理一些过滤，缓存等等的逻辑，例如 nginx
```

### Lua 的安装

#### Linux 版

```shell
yum install -y gcc

yum install libtermcap-devel ncurses-devel libevent-devel readline-devel

curl -R -O http://www.lua.org/ftp/lua-5.3.5.tar.gz

tar -zxf lua-5.3.5.tar.gz

cd lua-5.3.5

make linux test

make install

# 此时安装成功进入了 lua 控制台
Lua 5.1.4  Copyright (C) 1994-2008 Lua.org, PUC-Rio
>

# control + c 退出即可
```

#### Mac 版

```shell
curl -R -O http://www.lua.org/ftp/lua-5.3.5.tar.gz

tar zxf lua-5.3.5.tar.gz

cd lua-5.3.5

make macosx test

make install
```

### Lua Hello World

```shell
vim hello.lua	# 创建文件

# 文件内容:
print("Hello World!")

lua hello.lua	# 执行文件
```

### Lua 的基本语法

```shell
# lua 有交互式编程和脚本式编程

  # 交互式编程就是直接输入语法，就能执行

  # 脚本式编程需要编写脚本文件，然后再执行
 		# 一般采用脚本式编程。（例如: 编写一个 hello.lua 文件，输入文件内容，lua 命令执行文件）
```

#### 注释

```shell
# 单行注释语法
--

# 多行注释语法
--[[
	多行注释
	多行注释
--]]
```

#### 关键字

```shell
# 关键字就好比 java 中的 break、if、else 等等一样的效果

# lua 的关键字如下:
```

|          |       |       |        |
| -------- | ----- | ----- | ------ |
| and      | break | do    | else   |
| elseif   | end   | false | for    |
| function | if    | in    | local  |
| nil      | not   | or    | repeat |
| return   | then  | true  | until  |
| while    |       |       |        |

#### 定义变量

```shell
# 全局变量，默认情况下，定义一个变量都是全局变量

# 如果要使用局部变量，需要声明为 local，例如:
```

```lua
-- 全局变量赋值
a=1

-- 局部变量赋值
local b=2

-- 如果变量没有初始化，值为 nil，和 java 中的 null 不同
```

#### Lua 中的数据类型

```shell
# Lua 是动态类型语言，变量不要类型定义，只需要为变量赋值。

# 值可以存储在变量中，作为参数传递或结果返回。

# Lua 中的八个基本类型为:
```

| 数据类型 | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| nil      | 这个最简单，只有值nil属于该类，表示一个无效值（在条件表达式中相当于false）。 |
| boolean  | 包含两个值：false和true。                                    |
| number   | 表示双精度类型的实浮点数                                     |
| string   | 字符串由一对双引号或单引号来表示                             |
| function | 由 C 或 Lua 编写的函数                                       |
| userdata | 表示任意存储在变量中的C数据结构                              |
| thread   | 表示执行的独立线路，用于执行协同程序                         |
| table    | Lua 中的表（table）其实是一个"关联数组"（associative arrays），数组的索引可以是数字、字符串或表类型。在 Lua 里，table 的创建是通过"构造表达式"来完成，最简单构造表达式是{}，用来创建一个空表。 |

#### 流程控制

```lua
-- [ 0 是 true ]
if (0) then
  print("0 为 true")
else
  print("0 不为 true")
end
```

#### 函数

```shell
# lua 中也可以定义函数，类似 java 中的方法，例如:
```

```lua
--[[ 
	@param num1 : 第一个数字
	@param num2 : 第二个数组
	@return max : 两个参数中最大的数字
--]]
function max(num1, num2)
  if (num1 > num2) then
    result = num1;
  else
    result = num2;
  end
  return result;
end

-- 调用函数
print("最大值为: ", max(10, 4))
print("最大值为: ", max(5, 6))
```

#### require 函数

```shell
# require 用于引入其他的模块，类似 java 中的类要引用别的类的效果，语法如下:
require "<模块名>"
```

## OpenResty

### OpenResty 介绍

```shell
# openResty 是一个基于 nginx 的可升缩性的 Web 平台，由中国人章亦春发起。

# 强大的 Web 应用服务器，Web 开发人员可以使用 Lua 脚本语言调用 Nginx 支持的各种 C 以及 Lua 模块
	# 性能方面，openResty 可以快速构造出足以胜任 10K 乃至 100K 以上并发连接响应的超高性能 Web 应用程序
	
# 简单理解:
	# openResty 就相当于封装了 ngxin，并且集成了 lua 脚本
	# 开发人员只需要简单的调用其实现模块，就可以实现相关逻辑
	# 而不需要在 nginx 中自己编写 lua脚本，再进行调用了。
```

### OpenResty 安装

#### Linux 版

```shell
yum install yum-utils

yum-config-manager --add-repo https://openresty.org/package/centos/openresty.repo

yum install openresty

# 安装成功后的默认目录
/usr/local/openresty
```

#### 安装 nginx

```shell
# 安装步骤略

# 修改 nginx.conf
# 将用户修改为 root，目的是将来要使用 lua 脚本的时候，可以直接加载在 root 下的 lua 脚本
user root root;
```

