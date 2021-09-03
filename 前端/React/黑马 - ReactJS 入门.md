[TOC]



# 黑马 - ReactJS 入门

## ES6 新特性

```shell
# 现在使用主流的前端框架中，如 ReactJS、Vue.js、Angular.js，都会使用到 ES6 的新特性。
```

### 了解 ES6

```shell
# ES6,是 ECMAScript 6 的简称，是 JavaScript 语言的下一代标准，2015年6月正式发布。

# 它的目标是使 JavaScript 语言可以用于编写复杂的大型应用程序，成为企业级开发语言。
```

#### 什么是 ECMAScript

```shell
# 前端的发展历程:

# web1.0 时代:
	# 最初的网页以 HTML 为主，是纯静态的网页，网页是只读的，信息流只能从服务端到客户端单向流通，开发人员也只关心页面的样式和内容。
	
# web2.0 时代:
	# 1995 年，网景工程师 Brendan Eich 花了10天时间设计了 JavaScript 语言。
	# 1996 年，微软发布了 JSCript，其实是 JavaScript 的逆向工程实现。
	# 1997 年，为了统一各种不同 script 脚本语言，ECMA<欧洲计算机制造商协会> 以 JavaScript 为基础，制定了 ECMAScript 标准规范。
	
# 所以，ECMAScript 是浏览器脚本语言的规范，而各种js语言，如 JavaScript 则是规范的具体实现。
```

### let 和 const 命令

```shell
# var,之前我们写 js 定义变量的时候，只有一个关键字: var
	# var 有一个问题，就是定义的变量有时会莫名其妙的成为全局变量。
```

```javascript
for(var i = 0; i < 5; i++){
    console.log("循环内 : " + i);
}
    console.log("循环外 : " + i);
```

![UTOOLS1572919758314.png](https://i.loli.net/2019/11/05/Yxbh2rEneITmtPq.png)

```shell
# 可以看出，在循环外部也可以获取到变量 i 的值，显然变量 i 的作用域范围太大了，在做复杂页面时，会带来很大的问题。
```

#### let

```shell
# let 所声明的变量，只在 let 命令所在的代码块内有效

# 把刚才的 var 改成 let 试试
```

#### const

```shell
# const 声明的变量是常量，不能被修改，类似于 Java 中 final 关键字
```

```javascript
const a = 1;
console.log("a = " + a);
// 重新赋值
a = 2;
console.log("a = " + a);
```

![UTOOLS1572920003813.png](https://i.loli.net/2019/11/05/wOR471daqGfVrKi.png)

### 字符串扩展

```shell
# ES6 中，为字符串扩展了几个新的 API:

# includes()
	# 返回布尔值，表示是否找到了参数字符串
	
# startsWith()
	# 返回布尔值，表示参数字符串是否在源字符串的头部
	
# endsWith()
	# 返回布尔值，表示参数字符串是否在源字符串的尾部
```

#### 字符串模板

```javascript
// ES6 中提供了 ` 来作为字符串模板标记
// 在两个 ` 之间的部分都会被作为字符串的值，可以任意换行
<script>
    let str = `
	快乐的
	一只
	小青蛙
	`;
	console.log(str);
</script>
```

### 解构表达式

```shell
# 解构:
	# ES6 中允许按照一定模式从数组和对象中提取值，然后对变量进行赋值，这被称为解构
```

#### 数组解构

```javascript
// 比如有一个数组
let arr = [1,2,3];
// 之前想获取其中的值，只能通过角标，ES6 可以这样
const [x,y,z] = arr;
console.log(x,y,z);

// 只匹配一个参数
const [a] = arr;
console.log(a);
```

#### 对象解构

```javascript
// 比如有个 person 对象
const person = {
    name:"AgeFades",
    age:21,
    language:['java','sql']
}

// 我们可以这么做
const {name,age,language} = person;
// 打印
console.log(name);
console.log(age);
console.log(language);

// 如果需要用其他变量接收，需要额外指定别名
// {name:n} : name 是 person 中的属性名，冒号后面的 n 是解构后要赋值给的变量
const {name:n} = person;
console.log(n);
```

### 函数优化

```shell
# ES6 中，对函数的操作做了优化，使得我们在操作函数时更加的便捷。
```

#### 函数参数默认值

```javascript
// 在 ES6 以前，我们无法给一个函数参数默认设置值，只能采用变通写法
function add(a,b){
    // 判断 b 是否为空，为空默认为1
    b = b || 1;
    return a + b;
}
console.log(add(10));

// 现在可以这么写
function add(a, b = 1){
    return a + b;
}
console.log(add(10));
```

#### 箭头函数

```javascript
// ES6 中定义函数的简写方式

// 一个参数时
var print = function(obj){
    console.log(obj);
}
// 简写为
var print = obj => console.log(obj);

// 多个参数时
var sum = function(a, b){
    return a + b;
}
// 简写为
var sum = (a,b) => a + b;

// 没有参数时，通过() 进行占位，代表参数部分
let sayHello = () => console.log("hello");
sayHello();

// 代码不止一行时，用 {} 包含方法体
let sayHello = () => {
    console.log("Hello");
    console.log("World");
}
sayHello();
```

#### 对象的函数属性简写

```javascript
// 比如一个 Person 对象，里面有 eat 方法
let person = {
    name:"AgeFades",
    // 以前
    eat: function(food){
        console.log(this.name + "在吃" + food);
    },
    // 箭头函数版，这里拿不到 this
    eat2: food => console.log(person.name + "在吃" + food),
    // 简写版
    eat3(food){
        console.log(this.name + "在吃" + food);
    }
}
```

#### 箭头函数结合解构表达式

```javascript
// 比如有一个函数
const person = {
    name:"AgeFades",
    age:21
}
function hello(person) {
    console.log("hello," + person.name)
}

// 如果用箭头函数和解构表达式
var hello = ({name}) => console.log("hello, " + name);
hello(person);
```

### map 和 reduce

```shell
# ES6 中，数组新增了 map 和 reduce 方法
```

#### map

```shell
# map() :
	# 接收一个函数，将原数组中的所有元素用这个函数处理后放入新数组返回
```

```javascript
let arr = ['1','2','3'];
console.log(arr);

// 转成 int 数组
let newArr = arr.map(s => parseInt(s));
console.log(newArr);
```

#### reduce

```shell
# reduce() :
	# 接收一个函数<必须>和一个初始值<可选>，该函数接收两个参数

# 函数的第一个参数是上一次 reduce 处理的结果

# 函数的第二个参数是数组中要处理的下一个元素

# reduce() 会从左到右依次把数据中的元素用 reduce 处理，并把处理的结果作为下次 reduce 的第一个参数。如果是第一次，会把前两个元素作为计算参数，或者把用户指定的初始值作为起始参数
```

```javascript
// 举例
const arr = [1,2,3,4];
// 没有初始值
arr.reduce((a,b) => a + b);	// 10
arr.reduce((a,b) => a * b); // 24

// 指定初始值
arr.reduce((a,b) => a * b,-1); // -24
```

### 扩展运算符

```javascript
// 扩展运算符(spread) 是三个点(...)，将一个数组转为用逗号分隔的参数序列。
console.log(...[1,2,3]); // 1 2 3
console.log(1,...[2,3,4],5); // 1 2 3 4 5

let add = (x,y) => x + y;
var num = [1,2];
console.log(add(...num)); // 3

// 数组合并
let arr = [...[1,2,3],...[4,5,6]];
console.log(arr); // [1,2,3,4,5,6]

// 与解构表达式结合
const [first,...rest] = [1,2,3,4,5];
console.log(first,rest); // 1 [2,3,4,5]

// 将字符串转成数组
console.log([...'hello'])	// ["h","e","l","l","o"]
```

### Promise

```shell
# Promise，简单说就是一个容器，里面保存着某个未来才会结束的事件<通常是一个异步操作的结果>。

# 从语法上来说，Promise 是一个对象，从它可以获取异步操作的消息。

# Promise 提供统一的API，各种异步操作都可以用同样的方法进行处理。

# 我们可以通过 Promise 的构造函数来创建 Promise 对象，并在内部封装一个异步执行的结果。
```

```javascript
// 语法
const promise = new Promise(function(resolve,reject) {
    // ...执行异步操作
    
    if(/* 异步操作成功 */){
    	resolve(value); // 调用 resolve，代表Promise 将返回成功的结果
	} else {
        reject(error); // 调用 reject，代表Promise 会返回失败结果                    
    }
});

// 这样，在 promise 中就封装了一段异步执行的结果。
// 如果我们想要等待异步执行完成，做一些事情，可以通过 promise 的 then 方法来实现
promise.then(function(value)){
   // 异步执行成功后的回调          
});

// 如果想要处理 promise 异步执行失败的事件，还可以跟上 catch
promise.then(function(value)){
   // 异步执行成功后的回调          
}).catch(function(error)){
	// 异步执行失败后的回调
}

// 示例
const p = new Promise(function(resolve,reject){
    // 这里用定时任务模拟异步
    setTimeout(() => {
        const num = Math.random();
        // 随机返回成功或失败
        if(num < 0.5){
            resolve("成功! num : " + num)
        } else{
			reject("出错了! num: " + num)
		}
    },300)
})

// 调用 promise
p.then(function(msg){
    console.log(msg);
}).catch(function(msg){
    console.log(msg);
})
```

### set 和 map

```shell
# ES6 提供了 Set 和 Map 的数据结构
```

#### Set

```javascript
// Set，本质与数组类似，不同在于Set 中只能保存不同元素，如果元素相同会被忽略。

// 构造函数，可以接收一个数组或空
let set = new Set();
set.add(1);	// [1]
// 接收数组
let set2 = new Set([2,3,4,5,5]); // 得到[2,3,4,5]

// 方法:
set.add(1); // 添加
set.clear(); // 清空
set.delete(2); // 删除指定元素
set.has(2); // 判断是否存在
set.forEach(function(){}); // 遍历元素
set.size; // 元素个数，是属性，不是方法。
```

#### Map

```javascript
// map,本质是与Object 类似的结构。不同在于，Object 强制规定key 只能是字符串，而 map 结构的 key 可以是任意对象。即:
	// Object 是 <string,object>
	// Map 是 <object,object>
	
// 构造函数
// map 接收一个数组，数组中的元素是键值对数组、Set、Map 等
const map = new Map([
    ['key1','value1'],
    ['key2','value2']
])

// 方法
map.set(key,value); // 添加
map.clear(); // 清空
map.delete(key); // 删除指定元素
map.has(key); // 判断是否存在
map.forEach(function(key,value){}); // 遍历元素
map.size; // 元素个数，是属性，不是方法
map.values(); // 获取 value 的迭代器
map.keys(); // 获取 key 的迭代器
map.entries(); // 获取 entry 的迭代器

// 用法
for (let key of map.keys()) {
    console.log(key);
} 
// 或者
console.log(...map.values()); // 通过扩展运算符进行展开
```

### class<类> 的基本语法

```javascript
// JavaScript 语言的传统方法是通过构造方法定义并生成新对象
// ES6 中引入了 class 的概念，通过 class 关键字自定义类

// 基本用法
class User{
    constructor(name, age = 20){	// 构造方法
        this.name = name;	// 添加属性并且赋值
        this.age = age;
    }
    
    sayHello(){	// 定义方法
        return "hello";
    }
    
    static isAdult(age){	// 静态方法
        if(age >= 18){
            return "成年人";
        }
        return "未成年";
    }
}    

let user = new User("AgeFades",21);
```

#### 类的继承

```javascript
class Programmer extends User{
    constructor(){
        // 如果父类中的构造方法有参数，那么子类必须通过 super 调用父类的构造方法
        super("AgeFades",21); 
        // 设置子类中的属性，位置必须处于 super 下面
        this.address = "北京"; 
    }
}

let programmer = new Programmer();
```

### Generator 函数

```javascript
// Generator 函数是 ES6 提供的一种异步编程解决方案，语法行为与传统函数完全不同

// Generator 函数有两个特征
	// 一是 function 命令与函数名之间有一个星号
	// 二是 函数体内部使用 yield 语句定义不同的内部状态

// 用法
function* hello(){
    yield "hello";
    yield "world";
    return "done";
}

let h = hello();
console.log(h.next());	// {value: "hello", done: false}
console.log(h.next());	// {value: "world", done: false}
console.log(h.next());	// {value: "done", done: true}
console.log(h.next());	// {value: undefined, done: true}

// 可以看到，通过 hello() 返回的 h 对象，每调用一次 next() 方法返回一个对象，该对象包含了 value 值和 done 状态，直到遇到 return 关键字或者函数执行完毕，此时返回状态为 true，表示执行结束
```

#### for ...of 循环

```javascript
// 通过 for ...of 可以循环遍历Generator 函数返回的迭代器
// 函数接上一个 hello 使用
let h = hello();

for (let obj of h){
    console.log(obj);	
}
// 输出:
// hello
// world
```

### 修饰器（Decorator）

```javascript
// 修饰器<Decorator> 是一个函数，用来修改类的行为。

@T // 通过@ 符号进行引用该方法，类似 Java 中的注解
class User {
    constructor(name, age = 20){
        this.name = name;
        this.age = age;
    }
}

function T(target){ // 定义一个普通的方法
   console.log(target);	// target 对象为修饰的目标对象，这里是 User 对象
   target.country = "中国"; // 为 User 类添加一个静态属性 country
}

console.log(user.country); // 打印出 country 属性值

// 运行报错，因为ES6 中并没有支持该用法，在 ES2017 中才有，所以需要 Babel 转码运行
```

### 转码器

```shell
# Babel 是一个广为使用的 ES6 转码器，可以将 ES6 代码转为 ES5 代码，从而在浏览器或其他环境执行。

# Google 的 Traceur 转码器也可以将 ES6 代码转为 ES5 的代码
```

#### 了解 UmiJS

```shell
# 官网 : https://umijs.org/zh/

# UmiJS 读音 : (乌米)

# 特点:
	# 插件化
		# umi 的整个生命周期都是插件化的，甚至其内部实现都是由大量插件组成，比如 pwa、按需加载、一键切换 preact、一键兼容 ie9 等等，都是由插件实现。
		
	# 开箱即用
		# 只需一个 umi 依赖就可启动开发，无需安装 react、preact、webpack、react-router、babel、jest 等等..
		
	# 约定式路由
		# 类 next.js 的约定式路由，无需再维护一份冗余的路由配置，支持权限、动态路由、嵌套路由等等..
```

#### 部署安装

```shell
# 首先需要安装 Node.js

# 安装 yarn，其中 tyarn 使用的是 npm.taobao.org 的源，速度要快一些
npm i yarn tyarn -g # -g 是指全局安装

# 安装 umi
tyarn global add umi
```

#### 快速入门

```shell
# 通过初始化命令生成 package.json 文件，它是 Node.js 约定的用来存放项目的信息和配置等信息的文件。
tyarn init -y

# 通过 umi 命令创建 index.js 文件
umi g page index	# 在 pages 下创建好了index.js 和index.css 文件

# 通过命令行启动 umi 的后台服务，用于本地开发
umi dev

# 通过 http://localhost:8000/ 查看效果
# 此处访问的是 umi 的后台服务，不是 idea 提供的服务
```

#### 目录结构

```shell
# umi 中，约定的目录结构如下:
```

![UTOOLS1572944638166.png](https://i.loli.net/2019/11/05/FOs941mejq8PKaT.png)

### 模块化

#### 什么是模块化

```shell
# 模块化就是把代码进行拆分，方便重复利用。

# 模块功能主要是由两个命令构成:
	# export : 用于规定模块的对外接口
	# import : 用于导入其他模块提供的功能
```

#### export

```javascript
// 比如定义一个 js 文件 Util.js，里面有一个 Util 类
class Util {
    static sum = (a,b) => a + b;
}

// 导出该类
export default Util;
```

#### import

```javascript
// 使用 export 命令定义了模块的对外接口以后，其他 JS 文件就可以通过 import 命令加载这个模块

import Util from './Util'

// 使用 Util 中的 sum 方法
console.log(Util.sum(1, 2));
```

## ReactJS 入门

### 前端开发的演变

```shell
# 到目前为止，前端的开发经历了四个阶段，目前处于第四个阶段。

# 阶段一 : 静态页面阶段
	# 第一个阶段中，前端页面都是静态的，所有的前端代码和前端数据都是后端生成的。前端只是纯粹的展示功能，js 脚本的作用只是增加一些特殊效果，比如那时很流行用脚本控制页面上飞来飞去的广告。
	
	# 那时的网站开发，采用的是后端 MVC 模式。
		# Model<模型> : 提供/保存数据
		# Controller<控制层> : 数据处理，实现业务逻辑
		# View<视图层> : 展示数据，提供用户界面
		
	# 前端只是后端 MVC 的V
	
# 阶段二 : ajax 阶段
	# 2004 年，ajax 技术诞生，改变了前端开发。Gmail 和 Google 地图这样革命性的产品出现，似的开发者发现，前端的作用不仅仅是展示页面，还可以管理数据并与用户互动。
	
	# 从这个阶段开始，前端脚本开始变得复杂，不再仅仅是一些玩具性的功能。
	
# 阶段三 : 前端 MVC 阶段
	# 2010年，第一个前端 MVC 框架 Backbone.js 诞生，它基本上是把 MVC 模式搬到了前端，但是只有 M<读写数据> 和 V<展示数据>，没有 C<处理数据>
	
	# 有些框架提出了 MVVM 模式，用 View Model 代替 Controller。Model 拿到数据以后，View Model 将数据处理成视图层<View> 需要的格式，在视图层展示出来。
	
# 阶段四 : SPA 阶段
	# 前端可以做到读写数据、切换视图、用户交互，这意味着，网页其实是一个应用程序，而不是信息的纯展示。这种单张网页的应用程序称为 SPA< single-page-application >
	
	# 2010 年后，前端工程师从开发页面<切模板>，逐渐变成了开发"前端应用"< 跑在浏览器里面的应用程序 >
	
	# 目前，最流行的 Vue、Angular、React 等等，都属于 SPA 开发框架。
```

### ReactJS 简介

```shell
# 官网 : https://reactjs.org/

# 一个用于构建用户界面的 JavaScript 框架，是 Facebook 开发的一款 JS框架

# ReactJS 把复杂的页面，拆分成一个个的组件，将这些组件一个一个的拼装起来，就会呈现多样的页面。

# ReactJS 可以用于 MVC 架构，也可以用于 MVVM 架构，或者别的架构

# ReactJS 圈内的一些框架简介:
	# Flux
		# Facebook 用户建立客户端Web 应用的前端架构，通过利用一个单向的数据流补充了 React 的组合视图组件，这更是一种模式而非框架。
		
	# Redux
		# Redux 是 JavaScript 状态容器，提供可预测化的状态管理。Redux 可以让React 组件状态共享变得简单。
		
	# Ant Design of React
		# 阿里开源的基于 React 的企业级后台产品，其中集成了多种框架，包含了上面提到的 Flux、Redux
		# Ant Design 提供了丰富的组件，包括 : 按钮、表单、表格、布局、分页、树组件、日历等
```

### 搭建环境

#### 创建项目

```shell
# 创建空工程

# 初始化工程
tyarn init -y

# 添加 umi 的依赖
tyarn add umi --dev
```

#### 编写 HelloWorld 程序

```shell
# 在工程的根目录创建 config 目录，在 config 目录下创建 config.js 文件

# 在 UmiJS 的约定中，config/config.js 将作为UmiJS 的全局配置文件

# 在 config.js 文件中编写:
export default {};

# 在 umi 中，约定存放页面代码的文件夹是在 src/pages

# 创建 HelloWorld.js 页面文件，编写:
export default () => {
    return <div> Hello World! </div>;
}

# 在 js 文件中写 html 代码，是 react 自创的写法，叫 JSX

# 启动服务
umi dev

# 访问 localhost:8000/HelloWorld

# umi 中，可以使用约定式的路由，在 pages 下面的 JS 文件都会按照文件名映射到一个路由
```

#### 添加 umi-plugin-react 插件

```shell
# umi-plugin-react 插件是 umi 官方基于 react 封装的插件，包含了 13 个常用的进阶功能。

# 具体可查看 : https://umijs.org/zh/plugin/umi-plugin-react.html

# 添加插件
tyarn add umi-plugin-react --dev

# 在 config.js 文件中引入该插件
export default {
    plugins: [
        ['umi-plugin-react',{
            
        }]
    ]
}
```

#### 构建和部署

```shell
# 现在我们写的 js,必须通过 umi 先转码后才能正常的执行

# 通过 umi 进行转码生成文件
umi build
```

### React 快速入门

#### JSX 语法

```shell
# JSX : 在 js 文件中插入 Html 片段，是 React 自创的一种语法

# JSX 会被 Babel 等转码工具进行转码，得到正常的 js 代码再执行

# 使用 JSX 语法，需要注意两点
	# 所有的 Html 标签必须是闭合的
	
	# 在 JSX 语法中，只能有一个根标签，不能有多个
	
# 在 JSX 语法中，如果想要在 Html 标签中插入 js 脚本，需要通过{}
```

#### 组件

```shell
# 组件是 React 中最重要也是最核心的概念，一个网页，可以被拆分成一个个的组件

# 在 React 中，这样定义一个组件:
```

```javascript
import React from 'react'; // 第一步，导入 React

// 第二步，编写类并继承 React.Component
class HelloWorld extends React.Component {
    
    render(){	// 第三步，重写 render() 方法，用于渲染页面
        return <div> Hello World <div>	// JSX 语法
    }
    
}
    
export default HelloWorld; // 第四步，导出该类
```

##### 导入自定义组件

```javascript
// 创建 Show.js，用于测试导入组件
import React from 'react'
import HelloWorld from './HelloWorld' // 导入 HelloWorld 组件

class Show extends React.Component{
    
    render(){
        return <HelloWorld/>; // 使用 HelloWorld
    }
    
}

export default Show;
```

##### 组件参数

```javascript
// 组件是可以传递参数的，有两种方式传递，分别是属性和标签包裹的内容传递
import React from 'react'
import HelloWolrd from './HelloWorld'

class Show extends React.Component{
    render(){
        return <HelloWorld name="AgeFades"> China <HelloWorld>;
    }
}

export default Show;

// name="AgeFades" 是属性传递
// China 就是标签包裹的内容传递

// 在 HelloWorld.js 组件中如何接收参数呢？
// 属性 : this.props.name 接收
// 标签内容 : this.props.children 接收

import React from 'react';

class HelloWorld extends React.Component {
   
    render(){
        return <div> 
            Hello World! 
            name={this.props.name},
            address={this.props.children} 
        <div>
    }
    
}

export default HelloWorld;
```

##### 组件状态

```shell
# 每个组件都有一个状态，其保存在 this.state 中，当状态值发生变化时，React 框架会自动调用 render() 方法，重新渲染页面。

# 其中，要注意两点:
	# this.state 值的设置要在构造参数中完成。
	# 要修改 this.state 的值，需要调用 this.setState() 完成，不能直接对 this.state 进行修改。
```

```javascript
// 案例: 通过点击按钮，不断的更新 this.state，从而反应到页面中
import React from 'react'

class List extends React.Component {
    
    constructor(props){	// 构造参数中必须要 props 参数
        super(props);	// 调用父类的构造方法
        this.state = {	// 初始化 this.state
            dataList : [1,2,3],
            maxNum : 3
        };
    }
    
    render(){
        return (
        	<div>
            	<ul>
           			{	// 遍历值
                        this.state.dataList.map((value,index) => {
            				return <li key={index}> {value} </li>
        				})
            		}
    			</ul>
				<button
                    onClick = { () => {	// 为按钮添加点击事件
						let maxNum = this.state.maxNum + 1;
                        let list = [...this.state.dataList, maxNum];
                        this.setState({	// 更新状态值
                            dataList : list,
                            maxNum : maxNum
                        });
                    }}>
                        添加
				</button>
			</div>
        );
    }
    
}

export default List;
```

##### 生命周期

```shell
# 组件的运行过程中，存在不同的阶段。React 为这些阶段提供了钩子方法，允许开发者自定义每个阶段自动执行的函数。这些方法统称为生命周期方法< lifecycle methods >
```

![UTOOLS1573008721967.png](https://i.loli.net/2019/11/06/J2ZIfjAO6vbs5l1.png)

```javascript
// 生命周期示例
import React from 'react';

class LifeCycle extends React.Component {
    
    constructor(props) {
        super(props);
        console.log("constructor()");
    }
    
    componentDidMount() {
        // 组件挂载后调用
        console.log("componentDidMount()");
    }
    
    componentWillUnmount() {
        // 在组件从 DOM 中移除之前立刻被调用
        console.log("componentWillUnmount()");
    }
    
    componentDidUpdate() {
        // 在组件完成更新后立即调用，在初始化时不会被调用
        console.log("componentDidUpdate()");
    }
    
    shouldComponentUpdate(nextProps, nextState){
        // 每当 this.props 或 this.state 有变化，在 render 方法执行之前，就会调用这个方法
        // 该方法返回一个布尔值，表示是否应该继续执行 render 方法，即如果返回 false，UI 就不会更新，默认返回 true。
        // 组件挂载时，render 方法的第一次执行，不会调用这个方法
        console.log("shouldComponentUpdate()");
    }
    
    render() {
        return (
        	<div>
            	<h1> React Life Cycle! </h1>
            </div>
        );
    }
}

export default LifeCycle;
```

#### Model

##### 分层

```shell
# 服务端代码的层次结构，由 Controller、Service、Data Access 三层组成服务端系统:
	# Controller 层负责与用户直接打交道，渲染页面、提供接口等，侧重于展示型逻辑。
	
	# Service 层负责处理业务逻辑，供 Controller 层调用。
	
	# Data Access 层顾名思义，负责与数据源对接，进行纯粹的数据读写，供 Service 层调用
	
# 前端代码的结构，同样需要进行必要的分层:
	# Page 层负责与用户直接打交道: 渲染页面、接受用户的操作输入，侧重于展示型交互性逻辑。
	
	# Model 负责处理业务逻辑，为 Page 做数据、状态的读写、变换、暂存等。
	
	# Service 负责与 HTTP 接口对接，进行纯粹的数据读写。
```

##### 使用 DVA 进行数据分层管理

```shell
# DVA 是基于 redux、redux-saga 和 react-router 的轻量级前端框架。
	# 官网: https://dvajs.com/
```

```javascript
// 引入 dva，由于 umi 对 dva 进行了整合，所以导入非常简单
// 在 config.js 文件中进行配置
export default {
    plugins: [
        ['umi-plugin-react', {
            dva: true	// 开启 dva 功能
        }]
    ]
}

// 在 umi 中，约定在 src/models 文件夹中定义 model
// 在该目录下创建 ListData.js
export default {
    namespace: 'list',
    state: {
        data: [1,2,3],
        maxNum: 3
    }
}

// 对 List.js 进行改造
import React from 'react';
import { connect } from 'dva';

const namespace = 'list';

const mapStateToProps = (state) => {
    const listData = state[namespace].data;
    return {
        listData
    };
};

@connect(mapStateToProps)
class List extends React.Component {
    render() {
        return (
        	<div>
            	<ul>
                    {
                        this.props.listData.map( (value,index) => {
            				return <li key={index}> {value} </li>
        				})
                    }
    			</ul>
				<button
                    onClick={ () => {

                    }}>
                    添加
				</button>
			</div>
        );
    }
}

export default List;
```

```shell
# 流程说明:
1. umi 框架启动，会自动读取 models 目录下的 model 文件，即 ListData.js 中的数据

2. @connect 修饰符的第一个参数，接收一个方法，该方法必须返回 {},将接收到 model 数据

3. 在全局的数据中，会有很多，所以需要通过 namespace 进行区分，所以通过 state[namespace] 进行获取数据

4. 拿到 model 数据中的 data，也就是[1,2,3] 数据，进行包裹 {} 后返回

5. 返回的数据，将被封装到 this.props 中，所以通过 this.props.listData 即可获取到 model 中的数据
```

```shell
# 刚刚只是将数据展示出来，如果点击按钮，需要修改 state 的值，怎么操作？
```

```javascript
// 在 model 中新增 reducers 方法，用于更新 state 中的数据
export default {
	namespace: 'list',
    state: {
        data: [1,2,3],
        maxNum: 3
    },
    reducers : {
        addNewDate(state){	// state 是更新前的对象
            let maxNum = state.maxNum + 1;
            let list = [...state.data, maxNum];
            return {	// 返回更新后的 state 对象
                data : list,
                maxNum : maxNum
            }
        } 
    
    }
}

// 修改 List.js 新增点击事件
import React from 'react';
import { connect } from 'dva';

const namespace = 'list';

const mapStateToProps = (state) => {
    const listData = state[namespace].data;
    const maxNum = state[namespace].maxNum;
    return {
		listData, maxNum
    };
};

const mapDispatchToProps = (dispatch) => {	// 定义方法，dispatch 是内置函数
    return {	// 返回对象将绑定到 this.props 对象中
        addNewData : () => {
            dispatch({	// 通过调用 dispatch() 方法，调用 model 中reducers 方法
                // 指定方法，格式 : namespace/方法名
                type: namespace + "/addNewData"
            });
        }
        
    }
    
}

// mapDispatchToProps 函数，将方法映射到 props 中
@connect(mapStateToProps, mapDispatchToProps)
class List extends React.Component {
    
    render(){
		return (
        	<div>
				<ul>
                    {
                        this.props.listData.map((value, index) => {
            				return <li key={index}> {value} </li>
        				})
                    }
    			</ul>
                <button
                    onClick={ () => {this.props.addNewData()}}>
                        添加
                </button>
			</div>
        );
    }
    
}

export default List;
```

##### 在 model 中请求数据

```shell
# 前面数据都是写死在 model 中的，实际开发中，更多的是需要异步加载数据，在 model 中如何异步加载数据？

# 在 src 下创建 util 目录，并创建 request.js 文件:
```

```javascript
// import fetch from 'dva/fetch';

function checkStatus(response) {
    if (response.status >= 200 && response.status < 300) {
        return response;
    }
    
    const error = new Error(response.statusText);
    error.response = response;
    throw error;
}

/**
 * Requests a URL, returning a promise
 * url: The URL we want to request
 * options: the options we want to pass to "fetch"
 * An object containing either "data" or "err"
 */
export default async function request(url,options) {
	const response = await fetch(url,options);
    checkStatus(response);
    return await response.json();
}
```

```javascript
// 然后，在 model 中新增请求方法
import request from '../util/request';

export default {
    namespace: 'list',
    state: {
        data: [],
        maxNum: 0
    },
    reducers: {
        // result 就是拿到的结果数据
        addNewData(state, result){
            // 判断 result 中的 data 是否存在，如果存在，说明是初始数据，直接返回
            if (result.data) {	
                return result.data;
            }
            let maxNum = state.maxNum + 1;
            let list = [...state.data, maxNum];
            
            return {	// 更新状态值
                data: list,
                maxNum: maxNum
            }
        }
    },
    
    effects: { // 新增 effects 配置，用于异步加载数据
        *initData(params, sagaEffects) { // 定义异步方法
            const {call, put} = sagaEffects; // 获取到 call、put 方法
            const url = "/ds/list"; // 定义请求的 url
            let data = yield call(request, url); // 执行请求
            yield put({	// 调用 reducers 中的方法
                type: "addNewData", // 指定方法名
                data : data // 传递 ajax 回来的数据
            });
        }
    }
}
```

```javascript
// 改造页面逻辑
import React from 'react';
import { connect } from 'dva';

const namespace = 'list';

const mapStateToProps = (state) => {
    const listData = state[namespace].data;
    const maxNum = state[namespace].maxNum;
    return {
      listData, maxNum  
    };
};

const mapDispatchToProps = (dispatch) => {
    return {
        addNewData : () => {
            dispatch({
                type: namespace + "/addNewData"
            });
        },
        
        initData : () => {	// 新增初始化方法的定义
            dispatch({
                type: namespace + "/initData"
            });
        }
    }
}

@connect(mapStateToProps, mapDispatchToProps)
class List extends React.Component {
    
    componentDidMount() {
        this.props.initData();	// 组件加载完后进行初始化操作
    }
    
    render() {
        return (
        	<div>
            	<ul>
                    {
                        this.props.listData.map((value, index) => {
            				return <li key={index}> {value} </li>
        				})
                    }
    			</ul>
				<button
					onClick={ () => {this.props.addNewData()}}>
                        添加
				</button>
			<div>
        );
    }
}

export default List;
```

```shell
# 测试结果报错，请求返回数据为 html 代码
```

#### Mock 数据

```shell
# umi 中支持对请求的模拟

# 在项目根目录下创建 mock 目录，然后创建 MockListData.js 文件:
```

```javascript
export default {
    'get /ds/list': function(req, res) {	// 模拟请求返回数据
        res.json({
           data: [1,2,3,4],
           maxNum: 4
        });
    }
}
```

