[TOC]

[掘金 - 原文链接](https://juejin.cn/post/6986191903888769032)

# Gradle 系列 （二）、Gradle 技术探索

## 前言

很高兴遇见你~

这又是一个新的系列，关于 Gradle 学习，我所理解的流程如下图：

![Gradle_learning](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f65d4a418c9f4c209d6eb3d26a452fba~tplv-k3u1fbpfcp-watermark.awebp)

在本系列的上一篇文章中，我们对 Gradle 的一些基础概念及 Groovy 语法进行了讲解，完成了第一个环节。还没有看过上一篇文章的朋友，建议先去阅读 [Gradle 系列 （一）、Gradle相关概念理解，Groovy基础](https://juejin.cn/post/6939662617224937503)。

今天我们主要介绍环节二：熟悉 Gradle 常用 API，了解 Settings，Project，Task 等等。

[Github Demo 地址](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fsweetying520%2FGradleDemo) , 大家可以结合 demo 一起看，效果杠杠滴🍺

下面就正式进入 Gradle 的学习

## 一、Gradle 构建流程

### 1）、Gradle 构建阶段

Gradle 构建流程主要分为三个阶段：

1、初始化阶段

2、配置阶段

3、执行阶段

#### 1、初始化阶段

Gradle 初始化阶段主要就是执行 settings.gradle 脚本，构建 Project 对象

我们使用 AndroidStudio 新建一个 Android 项目的时候会自动生成 settings.gradle 文件，内容如下：

```groovy
rootProject.name = "GradleDemo"
include ':app'
复制代码
```

1、指定项目根 Project 的名称

2、使用 include 导入 app 工程

实际上 settings.gradle 对应一个 Settings 对象，include 就是 Settings 对象下的一个方法，它的作用就是引用哪些工程需要加入构建。然后 Gradle 会为每个带有 build.gradle 脚本文件的工程构建一个与之对应的 Project 对象

##### 1、include 扩展

我们可以使用 include + project 方法引用任何位置下的工程，如下：

```groovy
include ':test'
project(':test').projectDir = file('当前工程的绝对路径')
复制代码
```

通常我会使用这种方式引用自己写的库进行调试，非常的方便

但有的时候会遇到同时引入了 AAR 和源码的情况，我们可以使用 include + project，结合一些其他的配置，来实现 AAR 和源码的快速切换，具体步骤如下：

```groovy
//步骤一：在 settings.gradle 中引入源码工程
include ':test'
project(':test').projectDir = file('当前工程的绝对路径')

//步骤二：在根 build.gradle 下进行如下配置
allprojects {
    configurations.all {
        resolutionStrategy {
            dependencySubstitution {
                substitute module("com.dream:test") with project(':test')
            }
        }
    }
}
复制代码
```

##### 2、Settings

关于 Settings 的所有属性和方法，如下图：

![image-20210707215114352](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/353d7348729c4e50bd3ebb0ed107c623~tplv-k3u1fbpfcp-watermark.awebp)

结合官网提供的文档 [传送门](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.gradle.org%2Fcurrent%2Fjavadoc%2Forg%2Fgradle%2Fapi%2Finitialization%2FSettings.html) 去查看，效果杠杠的😄

#### 2、配置阶段

Gradle 配置阶段主要就是解析 Project 对象(build.gradle 脚本文件)，构建 Task 有向无环图

配置阶段会执行的代码：**除 Task 的 Action 中编写的代码都会被执行**，不懂 Action 的继续往下看，后面会讲到。如：

1、build.gradle 中的各种语句

2、Task 配置段语句

配置阶段完成后，整个工程的 Task 依赖关系都确定了，我们可以通过 Gradle  对象的 getTaskGraph 方法访问 Task ,对应的类为 TaskExecutionGraph ，关于 TaskExecutionGraph API 文档 [传送门](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.gradle.org%2Fcurrent%2Fjavadoc%2Forg%2Fgradle%2Fapi%2Fexecution%2FTaskExecutionGraph.html)

**注意：** 执行任何 Gradle 命令，在初始化阶段和配置阶段的代码都会被执行

#### 3、执行阶段

Gradle 执行阶段主要就是执行 Task 及其依赖的 Task

### 2）、Gradle 生命周期 Hook 点

引用 joe_H 一张完整的 Gradle 生命周期图，如下：

![image-20210704212912830](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/403a1114a8ad4d8bb725224a70dedc5a~tplv-k3u1fbpfcp-watermark.awebp)

上图对 Gradle 生命周期总结的很到位，我们解析一波：

**注意**：Gradle 执行脚本文件的时候会生成对应的实例，主要有如下三种对象：

> 1、Gradle 对象：在项目初始化时构建，全局单例存在，只有这一个对象
>
> 2、Project 对象：每一个 build.gradle 都会转换成一个 Project 对象
>
> 3、Settings 对象：Seetings.gradle 会转变成一个 Seetings 对象

1、Gradle 在各个阶段都提供了生命周期回调，在添加监听器的时候需要注意：**监听器要在生命周期回调之前添加，否则会导致有些回调收不到**

2、Gradle 初始化阶段

- 在 settings.gradle 执行完后，会回调 Gradle 对象的 settingsEvaluated 方法
- 在构建所有工程 build.gradle 对应的 Project 对象后，也就是初始化阶段完毕，会回调 Gradle 对象的 projectsLoaded 方法

3、Gradle 配置阶段：

- Gradle 会循环执行每个工程的 build.gradle 脚本文件
- 在执行当前工程 build.gradle 前，会回调 Gradle 对象的 beforeProject 方法和当前 Project 对象的 beforeEvaluate 方法
- 在执行当前工程 build.gradle 后，会回调 Gradle 对象的 afterProject 方法和当前 Project 对象的 afterEvaluate 方法
- 在所有工程的 build.gradle 执行完毕后，会回调 Gradle 对象的 projectsEvaluated 方法
- 在构建 Task 依赖有向无环图后，也就是配置阶段完毕，会回调 TaskExecutionGraph 对象的 whenReady 方法

**注意**： Gradle 对象的 beforeProject，afterProject 方法和 Project 对象的 beforeEvaluate ，afterEvaluate 方法回调时机是一致的，区别在于：

> 1、Gradle 对象的 beforeProject，afterProject 方法针对项目下的所有工程，即每个工程的 build.gradle 执行前后都会收到这两个方法的回调
>
> 2、 Project 对象的 beforeEvaluate ，afterEvaluate 方法针对当前工程，即当前工程的 build.gradle 执行前后会收到这两个方法的回调

4、执行阶段：

- Gradle 会循环执行 Task 及其依赖的 Task
- 在当前 Task 执行之前，会回调 TaskExecutionGraph 对象的 beforeTask 方法
- 在当前 Task 执行之后，会回调 TaskExecutionGraph 对象的 afterTask 方法

5、当所有的 Task 执行完毕后，会回调 Gradle 对象的 buildFinish 方法

了解了 Gradle 生命周期后，我们就可以根据自己的需求添加 Hook。例如：我们可以打印 Gradle 构建过程中，各个阶段及各个 Task 的耗时

### 3）、打印 Gradle 构建各个阶段及各个任务的耗时

在 settings.gradle 添加如下代码：

```groovy
//初始化阶段开始时间
long beginOfSetting = System.currentTimeMillis()
//配置阶段开始时间
def beginOfConfig
//配置阶段是否开始了，只执行一次
def configHasBegin = false
//存放每个 build.gradle 执行之前的时间
def beginOfProjectConfig = new HashMap()
//执行阶段开始时间
def beginOfTaskExecute
//初始化阶段执行完毕
gradle.projectsLoaded {
    println "初始化总耗时 ${System.currentTimeMillis() - beginOfSetting} ms"
}

//build.gradle 执行前
gradle.beforeProject {Project project ->
    if(!configHasBegin){
        configHasBegin = true
        beginOfConfig = System.currentTimeMillis()
    }
    beginOfProjectConfig.put(project,System.currentTimeMillis())
}

//build.gradle 执行后
gradle.afterProject {Project project ->
    def begin = beginOfProjectConfig.get(project)
    println "配置阶段，$project 耗时：${System.currentTimeMillis() - begin} ms"
}

//配置阶段完毕
gradle.taskGraph.whenReady {
    println "配置阶段总耗时：${System.currentTimeMillis() - beginOfConfig} ms"
    beginOfTaskExecute = System.currentTimeMillis()
}

//执行阶段
gradle.taskGraph.beforeTask {Task task ->
    task.doFirst {
        task.ext.beginOfTask = System.currentTimeMillis()
    }

    task.doLast {
        println "执行阶段，$task 耗时：${System.currentTimeMillis() - task.ext.beginOfTask} ms"
    }
}

//执行阶段完毕
gradle.buildFinished {
    println "执行阶段总耗时：${System.currentTimeMillis() - beginOfTaskExecute}"
}

//执行 Gradle 命令
./gradlew clean

//打印结果如下：
初始化总耗时 140 ms

> Configure project :
配置阶段，root project 'GradleDemo' 耗时：1181 ms

> Configure project :app
配置阶段，project ':app' 耗时：1122 ms
配置阶段总耗时：2735 ms

> Task :clean
执行阶段，task ':clean' 耗时：0 ms

> Task :app:clean
执行阶段，task ':app:clean' 耗时：1 ms
执行阶段总耗时：325
复制代码
```

了解了 Gradle 的三个阶段及生命周期，接下来我们就学习 Gradle 的一些核心 API

## 二、Project 介绍

对于一个 Android 项目，build.gradle 脚本文件是我们经常操作的文件之一，而每个 build.gradle 就对应了一个 Project 对象，因此学习好 Project 对应的 API 能帮助我们更好的去操作 build.gradle 脚本文件, 同时也能看懂大佬们所写的一些配置语句。

![image-20210718164127432](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e9bc9ab568af4fb2af66bcb398926898~tplv-k3u1fbpfcp-watermark.awebp)

首先看一眼我的项目结构，后续就是基于它来做演示

**注意**：

1、下面所演示的 API 都是一些常用的 API，对 API 使用有疑问的可以去查询官方文档

2、API 的演示如果没做特殊说明，则是在 app 的 build.gradle 文件下操作的

### 1）、Project API

[Project API 文档](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.gradle.org%2Fcurrent%2Fjavadoc%2Forg%2Fgradle%2Fapi%2FProject.html)，我们主要介绍一些常用的 API

#### 1、getRootProject 方法

获取根 Project 对象

```groovy
println getRootProject()

//打印结果
> Configure project :app
root project 'GradleDemo'
复制代码
```

#### 2、getRootDir 方法

获取根目录文件夹路径

```groovy
println getRootDir()

//打印结果
> Configure project :app
/Users/zhouying/learning/GradleDemo
复制代码
```

#### 3、getBuildDir 方法

获取当前 Project 的 build 文件夹路径

```groovy
println getBuildDir()

//打印结果
> Configure project :app
/Users/zhouying/learning/GradleDemo/app/build
复制代码
```

#### 4、getParent 方法

获取当前父 Project 对象

```groovy
println getParent()

//打印结果
> Configure project :app
root project 'GradleDemo'
复制代码
```

#### 5、getAllprojects 方法

获取当前 Project 及其子 Project 对象，返回值是一个 Set 集合

```groovy
//当前在根工程的 build.gradle 文件下
println getAllprojects()

//打印结果
> Configure project :
[root project 'GradleDemo', project ':app']
复制代码
```

我们还可以使用其闭包的形式

```groovy
allprojects {
    println it
}

//打印结果
> Configure project :
root project 'GradleDemo'
project ':app'

//我们通常会使用闭包的语法在根 build.gradle 下进行相关的配置，如下：
allprojects {
    repositories {
        google()
        mavenCentral()
    }
}
复制代码
```

**注意**：根 Project 与其子 Project 组成了一个树形结构，但这颗树的高度也仅仅被限定为了两层

#### 6、getSubprojects 方法

获取当前 Project 下的所有子 Project 对象，返回值是一个 Set 集合

```groovy
//当前在根工程的 build.gradle 文件下
println getSubprojects()

//打印结果
> Configure project :
[project ':app']
复制代码
```

同样我们也可以使用其闭包的形式

```groovy
subprojects {
    println it
}

//打印结果
> Configure project :
project ':app'
复制代码
```

#### 7、apply 系列方法

引用插件

```groovy
//引用第三方插件
apply plugin: 'com.android.application'

//引用脚本文件插件
apply from: 'config.gradle'
复制代码
```

#### 8、configurations 闭包

编写 Project 一些相关的配置，如全局移除某个依赖

```groovy
configurations {
    all*.exclude group: '组名', module: '模块名'
}
复制代码
```

#### 9、project 系列方法

指定工程实例，然后在闭包中对其进行相关的配置

```groovy
project("app") {
    apply plugin: 'com.android.application'
}
复制代码
```

### 2）、扩展属性

扩展属性作用：方便我们全局的一个使用。类似 Java 中，在工具类里面定义静态方法

#### 1、扩展属性定义

我们可以通过以下两种方式来定义扩展属性：

1、通过 **ext** 关键字定义扩展属性

2、在 **gradle.properties** 下定义扩展属性

##### 1、通过 ext 关键字定义扩展属性

通过 ext 定义扩展属性的语法有两种：

```groovy
//当前在根 build.gradle 下
//方式1：ext.属性名
ext.test = 'erdai666'

//方式2：ext 后面接上一个闭包
ext{
  test1 = 'erdai777'
}
复制代码
```

##### 2、在 gradle.properties 下定义扩展属性

通过 gradle.properties 定义扩展属性，直接使用 **key=value** 的形式即可：

```groovy
test2=erdai888
复制代码
```

#### 2、扩展属性调用

1、ext 定义的扩展属性调用的时候可以去掉 ext 前缀直接调用

2、ext 定义的扩展属性也可以通过 **当前定义扩展属性的 Project 对象.ext.属性名** 进行调用

3、gradle.properties 定义的扩展属性直接通过属性名调用即可

下面我们在 app 的 build.gradle 下进行演示：

```groovy
/**
 * 下面这种写法之所以能这么写
 * 1、ext 定义的扩展属性调用的时候可以去掉 ext 前缀直接调用
 * 2、子 Project 能拿到根 Project 中的属性和方法
 */
println test
println test1
println test2

//2、ext 定义的扩展属性也可以通过 当前定义扩展属性的 Project 对象.ext.属性名 调用
println rootProject.ext.test
println rootProject.ext.test1
println test2

//上述两种方式打印结果均为
> Configure project :app
erdai666
erdai777
erdai888
复制代码
```

**注意：** 子 Project 和根 Project 存在继承关系，因此根 Project 中定义的属性和方法子 Project 能获取到

#### 3、扩展属性应用

通常我们会使用扩展属性来优化 build.gradle 脚本文件，例如我们以优化 app 下的 build.gradle 为例：

首先看一眼优化之前 app 的 build.gradle 长啥样：

```groovy
apply plugin: 'com.android.application'

android {
    compileSdkVersion 30

    defaultConfig {
        applicationId "com.dream.gradledemo"
        minSdkVersion 19
        targetSdkVersion 30
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}

dependencies {
    implementation 'androidx.appcompat:appcompat:1.3.0'
    implementation 'com.google.android.material:material:1.4.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.0.4'
    testImplementation 'junit:junit:4.13.2'
    androidTestImplementation 'androidx.test.ext:junit:1.1.3'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.4.0'
}
复制代码
```

下面我们就来进行改造

**步骤1：** 在根目录下创建一个脚本文件 config.gradle ，用来存放扩展属性

```groovy
ext{

    androidConfig = [
            compileSdkVersion : 30,
            applicationId : 'com.dream.gradledemo',
            minSdkVersion : 19,
            targetSdkVersion : 30,
            versionCode : 1,
            versionName : '1.0'
    ]


    implementationLib = [
            appcompat : 'androidx.appcompat:appcompat:1.3.0',
            material  : 'com.google.android.material:material:1.4.0',
            constraintlayout : 'androidx.constraintlayout:constraintlayout:2.0.4'
    ]

    testImplementationLib = [
            junit : 'junit:junit:4.13.2'
    ]


    androidTestImplementationLib = [
            junit : 'androidx.test.ext:junit:1.1.3',
            'espresso-core' : 'androidx.test.espresso:espresso-core:3.4.0'
    ]
}
复制代码
```

**步骤2：** 在根 build.gradle 对 config.gradle 进行引用

```groovy
apply from: 'config.gradle'
复制代码
```

**注意：** 在根 build.gradle 进行引用的好处就是所有的子 build.gradle 都能够获取到这些扩展属性

**步骤3:** 在 app 的 build.gradle 里面进行扩展属性的调用

```groovy
apply plugin: 'com.android.application'

android {
    compileSdkVersion androidConfig.compileSdkVersion

    defaultConfig {
        applicationId androidConfig.applicationId
        minSdkVersion androidConfig.minSdkVersion
        targetSdkVersion androidConfig.targetSdkVersion
        versionCode androidConfig.versionCode
        versionName androidConfig.versionName

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}

def implementationLibMap = implementationLib
def testImplementationLibMap = testImplementationLib
def androidTestImplementationLibMap = androidTestImplementationLib

dependencies {
    implementationLibMap.each{k,v ->
        implementation v
    }

    testImplementationLibMap.each{k,v ->
        testImplementation v
    }

    androidTestImplementationLibMap.each{k,v ->
        androidTestImplementation v
    }
}
复制代码
```

### 3）、文件操作 API

#### 1、file/files 系列文件定位

Project 对象提供的 file/files 系列方法主要用来定位一个或者多个文件，值的注意的是：**它们接收的参数是一个相对路径，从当前 project 工程开始查找**，而我们通过 new File 的方式需要传入一个绝对路径，下面通过代码演示感受一下他们的区别：

```groovy
//============================== 1、file 方法应用============================
//通过 file 方法传入一个相对路径，返回值是一个 file 对象
println file('../config.gradle').text

//通过 new File 方式传入一个绝对路径
def file = new File('/Users/zhouying/learning/GradleDemo/config.gradle')
println file.text

//上述两者打印结果相同，如下截图

//============================== 2、files 方法应用============================
//通过 files 方法传入多个相对路径，返回值是一个 ConfigurableFileCollection 即文件集合
files('../config.gradle','../build.gradle').each {
    println it.name
}

//打印结果
> Configure project :app
config.gradle
build.gradle
复制代码
```

![image-20210717121715148](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/152c811c766440cd857b72a8b8c1542c~tplv-k3u1fbpfcp-watermark.awebp)

#### 2、copy 文件拷贝

> 1、Project 对象提供了 copy 方法，它使得我们拷贝一个文件或文件夹变得十分简单
>
> 2、copy 方法能够接收一个闭包，闭包的参数 CopySpec ，CopySpec 提供了很多文件操作的 API，具体可以查看文档 [传送门](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.gradle.org%2Fcurrent%2Fjavadoc%2Forg%2Fgradle%2Fapi%2Ffile%2FCopySpec.html)

下面会使用到 CopySpec 的 from 和 into 方法

**注意：** from 和 into 接收的参数是 Object 类型的，因此我们可以传入一个路径或文件

1、文件拷贝

例如我们实现：**将根目录下的 config.gradle 文件拷贝拷贝到 app 目录下。** 如下：

```groovy
//1、传入路径
copy {
    from getRootDir().path + "/config.gradle"
    into getProjectDir().path
}

//2、传入文件
copy {
    from file('../config.gradle')
    into getProjectDir()
}

//最终结果是这两种方式都能拷贝成功
复制代码
```

2、文件夹拷贝

例如我们实现：**将根目录下的 gradle 文件夹下的所有文件和文件夹拷贝到 app 目录下的 gradle 文件夹**

```groovy
copy {
    from file('../gradle/')
    into getProjectDir().path + "/gradle/"
}

//最终结果拷贝成功
复制代码
```

此时如果 app 目录下没有 gradle 文件夹，那么 copy 方法会给我们自动创建，非常的方便

#### 3、fileTree 文件树映射

Project 对象提供了 fileTree 方法，方便我们将一个目录转换为文件树，然后对文件树进行相关的逻辑处理，它接收的参数和 file/files 类似，也是一个相对路径

例如我们实现：**遍历根目录下的 gradle 文件夹，并打印文件及文件夹的名称**

```groovy
fileTree('../gradle/'){ FileTree fileTree ->
    fileTree.visit { FileTreeElement fileTreeElement ->
        println fileTreeElement.name
    }
}

//打印结果
> Configure project :app
wrapper
gradle-wrapper.jar
gradle-wrapper.properties
复制代码
```

我们通常会在 app 的 build.gradle 下看到这么一个配置语句：

```groovy
implementation fileTree(include: ['*.jar'], dir: 'libs')
复制代码
```

他实际上是调用了 fileTree 接收 Map 参数的重载方法：

```groovy
ConfigurableFileTree fileTree(Map<String, ?> var1);
复制代码
```

这句配置语句的意思就是：**引入当前 project 目录下的 libs 文件夹下的所有 jar 包**

### 4）、buildscript 解读

我们通常在新建一个 Android 项目的时候可以看到根 build.gradle 有这么一段配置：

```groovy
buildscript {
    //插件仓库地址
    repositories {
        google()
        mavenCentral()
    }
  
    //插件依赖
    dependencies {
        classpath "com.android.tools.build:gradle:4.2.1"
    }
}
复制代码
```

它的作用是：**引入 Gradle 构建过程中的一些插件**

实际上上面这段代码的完整写法如下：

```groovy
buildscript { ScriptHandler scriptHandler ->
    scriptHandler.repositories { RepositoryHandler repositoryHandler ->
        repositoryHandler.google()
        repositoryHandler.mavenCentral()
    }
  
    scriptHandler.dependencies { DependencyHandler dependencyHandler ->
        dependencyHandler.classpath "com.android.tools.build:gradle:4.2.1"
    }
}
复制代码
```

你是否会有这么一个疑问：为啥这些参数都能够去掉，简化成上面那样？🤔️

要明白上面这个问题，首先我们得对闭包有一定的了解：

1、首先闭包中有 owenr this delegate 三个对象，这三个对象拥有的属性和方法我们都可以调用，并且无需写出来

2、这三个对象调用的先后顺序取决于闭包的委托策略，一般我们会对 delegate 进行操作并修改它的委托策略

实际上，Gradle 对上面的这些闭包的 delegate 修改为了传入闭包的参数，并把委托策略设置为了 DELEGATE_FIRST ，因此我们调用的时候才能把这些参数给去掉，感兴趣的可以点击 buildscript 进去看下源码，这里就不对源码进行分析了

### 5）、exec 外部命令执行

Project 对象提供了 exec 方法，方便我们执行外部的命令

我们可以在 linux 下通过如下命令去移动一个文件夹：

```groovy
mv -f 源文件路径 目标文件路径
复制代码
```

现在我们在 Gradle 下去进行这一操作

例如我们实现：**使用外部命令，将我们存放的 apk 目录移动到项目的根目录** ，如下：

```groovy
task taskMove() {
    doLast {
        // 在 gradle 的执行阶段去执行
        def sourcePath = buildDir.path + "/outputs/apk"
        def destinationPath = getRootDir().path
        def command = "mv -f $sourcePath $destinationPath"
        exec {
            try {
                executable "bash"
                args "-c", command
                println "The command execute is success"
            } catch (GradleException e) {
                e.printStackTrace()
                println "The command execute is failed"
            }
        }
    }
}
复制代码
```

## 三、Task 介绍

Task 中文翻译即任务，它是 Gradle 中的一个接口，代表了要执行的任务，不同的插件可以添加不同的 Task，每一个 Task 都要和 Project关联。众所周知，线程是 cpu 执行的最小单元。同理，Task 是 Gradle 执行的最小单元，Gradle 将一个个 Task 串联起来，完成一个具体的构建任务

### 1）、doFirst、doLast 介绍

首先我们要搞懂 Action 这个概念，Action 本质上是一个执行动作，它只有在我们执行当前 Task 时才会被执行，Gradle 执行阶段本质上就是在执行每个 Task 中的一系列 Action

doFirst，doLast 是 Task 给我们提供的两个 Action

**doFirst** 表示：Task 执行最开始时被调用的 Action

**doLast** 表示： task 执行完时被调用的 Action

值的注意的是：**doFirst 和 doLast 可被多次添加执行** ，如下：

```groovy
task erdai{
    println 'task start...'

    doFirst {
        println 'doFirst1'
    }

    doLast {
        println 'doLast1'
    }

    doLast {
        println 'doLast2'
    }

    println 'task end...'
}

//执行当前 task
./gradlew erdai

//打印结果如下
> Configure project :app
task start...
task end...

> Task :app:erdai
doFirst1
doLast1
doLast2
复制代码
```

从上述打印结果我们可以发现

1、`println 'task start...'`，` println 'task end...'`这两句的代码在 Gradle 配置阶段就被执行了

2、doFirst，doLast 中的代码是在 Gradle 执行阶段，执行 erdai 这个 task 时被执行的

因此也验证了一开始我说的那个结论： **Gradle 配置阶段，除 Task 的 Action 中编写的代码都会被执行**

### 2）、Task 属性介绍

| 属性            | 描述                                        | 默认值       |
| --------------- | ------------------------------------------- | ------------ |
| name            | task 名字                                   | 无，必须指定 |
| type            | Task 的父类                                 | DefaultTask  |
| action          | 当 Task 执行的时候，需要执行的闭包或 Action | null         |
| overwrite       | 替换一个已存在的 Task                       | false        |
| dependsOn       | 该 Task 所依赖的 Task 集合                  | []           |
| group           | 该 task 所属分组                            | null         |
| description     | 该 Task 的描述信息                          | null         |
| constructorArgs | 传递到 Task Class 构造器中的参数            | null         |

### 3）、Task 类型介绍

一般我们创建的 Task 默认是继承 DefaultTask，我们可以通过 type 属性让他继承其他的类，也可以通过 extends 关键字直接指定，Gradle 自带的有 Copy、Delete 等等，如下：

```groovy
// 1、继承 Delete 这个类，删除根目录下的 build 文件
task deleteTask(type: Delete) {
    delete rootProject.buildDir
}

//通过 extends 关键字指定
class DeleteTask extends Delete{

}
DeleteTask deleteTask = tasks.create("deleteTask",DeleteTask)
deleteTask.delete(rootProject.buildDir)


// 2、继承 Copy 这个类
task copyTask(type: Copy) {
    //...
}

//通过 extends 关键字指定
class CopyTask extends Copy{
    //...
}
复制代码
```

### 4）、TaskContainer 介绍

TaskContainer 你可以理解为一个 Task 容器，Project 对象就是通过 TaskContainer 来管理 Task，因此我们可以通过 TaskContainer，对 Task 进行相关的操作，一些常用的 API 如下：

```groovy
//查找task
findByPath(path: String): Task 
getByPath(path: String): Task
getByName(name: String): Task

//创建task
create(name: String): Task
create(name: String, configure: Closure): Task 
create(name: String, type: Class): Task
create(options: Map<String, ?>): Task
create(options: Map<String, ?>, configure: Closure): Task

//当 task 被加入到 TaskContainer 时的监听
whenTaskAdded(action: Closure)
复制代码
```

### 5）、Task 定义及配置

因为 Task 和 Project 是相互关联的，Project 中提供了一系列创建 Task 的方法，下面介绍一些常用的：

```groovy
//1、创建一个名为 task1 的 Task
task task1

//2、创建一个名为 task2 的 Task，并通过闭包进行相应的配置
task task2{
    //指定 task 的分组
    group 'erdai666'
  
    doFirst{
    
    }
}

//3、创建一个名为 task3 的 Task，该 Task 继承自 Copy 这个 Task，依赖 task2
task task3(type: Copy){
    dependsOn "task2"
    doLast{
    
    }
}

//4、创建一个名为 task4 的 Task 并指定了分组和描述
task task4(group: "erdai666", description: "task4") {
    doFirst {
        
    }
    
    doLast {
        
    }
}

//5、通过 Project 对象的 TaskContainer 创建名为 task5 的 Task
tasks.create("task5"){

}

//6、通过 Project 对象的 TaskContainer 创建名为 task6 的 Task
//相对于 5 ，只是调用了不同的重载方法而已
tasks.create(name: "task6"){

}
复制代码
```

### 6）、Task 执行实战

通常我们会使用 doFirst 与 doLast 在 Task 执行期间进行相关操作，下面我们就来实现 **build 任务执行期间耗时**：

```groovy
// Task 执行实战：计算 build 执行期间的耗时
def startBuildTime, endBuildTime
// 1、在 Gradle 配置阶段完成之后进行操作，
// 以此保证要执行的 task 配置完毕
this.afterEvaluate { Project project ->
    // 2、找到当前 project 下第一个执行的 task，即 preBuild task
    def preBuildTask = project.tasks.getByName("preBuild")
    preBuildTask.doFirst {
        // 3、获取第一个 task 开始执行时刻的时间戳
        startBuildTime = System.currentTimeMillis()
    }
    // 4、找到当前 project 下最后一个执行的 task，即 build task
    def buildTask = project.tasks.getByName("build")
    buildTask.doLast {
        // 5、获取最后一个 task 执行完成前一瞬间的时间戳
        endBuildTime = System.currentTimeMillis()
        // 6、输出 build 执行期间的耗时
        println "Current project execute time is ${endBuildTime - startBuildTime}"
    }
}

//执行 build 任务
./gradlew build

//打印结果
Current project execute time is 21052
复制代码
```

### 7）、指定 Task 执行顺序

在 Gradle 中，有三种方式可以指定 Task 执行顺序：

1、dependsOn 强依赖方式

2、通过 Task 输入输出

3、通过 API 指定执行顺序

#### 1、dependsOn 强依赖方式

dependsOn 强依赖方式可细分为**静态依赖**和**动态依赖**

- 静态依赖：在创建 Task 的时候，直接通过 dependsOn 指定需要依赖的 Task
- 动态依赖：在创建 Task 的时候，不知道需要依赖哪些 Task，需通过 dependsOn 动态依赖符合条件的 Task

```groovy
//=================================静态依赖=============================
task taskA{
    doLast {
        println 'taskA'
    }
}

task taskB{
    doLast {
        println 'taskB'
    }
}

task taskC(dependsOn: taskA){//多依赖方式 dependsOn:[taskA,taskB]
    doLast {
        println 'taskC'
    }
}

//执行 taskC
./gradlew taskC

//打印结果
> Task :app:taskA
taskA

> Task :app:taskC
taskC
复制代码
```

上述代码，当我们执行 taskC 的时候，因为依赖了 taskA，因此 taskA 会先执行，在执行 taskC

**注意**：当一个 Task 依赖多个 Task 的时候，被依赖的 Task 之间如果没有依赖关系，那么它们的执行顺序是随机的，并无影响，如下：

```groovy
task taskC(dependsOn:[taskA,taskB]){
    doLast {
        println 'taskC'
    }
}
复制代码
```

taskA 和 taskB 的执行顺序是随机的

```groovy
//=================================动态依赖=============================
// Task 动态依赖方式
task lib1 {
    doLast{
        println 'lib1'
    }
}
task lib2 {
    doLast{
        println 'lib2'
    }
}
task lib3 {
    doLast{
        println 'lib3'
    }
}

// 动态指定taskX依赖所有以lib开头的task
task taskDynamic{
    // 动态指定依赖
    dependsOn tasks.findAll{ Task task ->
        return task.name.startsWith('lib')
    }

    doLast {
        println 'taskDynamic'
    }
}

//执行 taskDynamic
./gradlew taskDynamic

//打印结果
> Task :app:lib1
lib1

> Task :app:lib2
lib2

> Task :app:lib3
lib3

> Task :app:taskDynamic
taskDynamic
复制代码
```

#### 2、通过 Task 输入输出指定执行顺序

当一个参数，作为 TaskA 的输入参数，同时又作为 TaskB 的输出参数，那么 TaskA 执行的时候先要执行 TaskB。即输出的 Task 先于输入的 Task 执行

但是我在实际测试过程中发现：**输入的 Task 会先执行，然后在执行输出的 Task**，如下：

```groovy
ext {
    testFile = file("${projectDir.path}/test.txt")
    if(testFile != null || !testFile.exists()){
        testFile.createNewFile()
    }
}

//输出 Task
task outputTask {
    outputs.file testFile
    doLast {
        outputs.getFiles().singleFile.withWriter { writer ->
            writer.append("erdai666")
        }
        println "outputTask 执行结束"
    }
}

//输入 Task
task inputTask {
    inputs.file testFile
    doLast {
        println "读取文件内容：${inputs.files.singleFile.text}"
        println "inputTask 执行结束"
    }
}

//测试 Task
task testTask(dependsOn: [outputTask, inputTask]) {
    doLast {
        println "testTask1 执行结束"
    }
}

//执行 testTask
./gradlew testTask

//理论上会先执行 outputTask，在执行 inputTask，最后执行 testTask
//但实际打印结果
> Task :app:inputTask
读取文件内容：
inputTask 执行结束

> Task :app:outputTask
outputTask 执行结束

> Task :app:testTask
testTask1 执行结束
复制代码
```

最终我对 inputTask 指定了具体依赖才达到了预期效果：

```groovy
task inputTask(dependsOn: outputTask) {
    inputs.file testFile
    doLast {
        println "读取文件内容：${inputs.files.singleFile.text}"
        println "inputTask 执行结束"
    }
}

//修改之后的打印结果
> Task :app:outputTask
outputTask 执行结束

> Task :app:inputTask
读取文件内容：erdai666
inputTask 执行结束

> Task :app:testTask
testTask1 执行结束
复制代码
```

#### 3、通过 API 指定执行顺序

可以指定 Task 执行顺序的 API 有：

**mustRunAfter**：指定必须在哪个 Task 执行完成之后执行

**shouldRunAfter**：跟 mustRunAfter 类似，区别在于不强制，不常用

**finalizeBy**：在当前 Task 执行完成之后，指定执行的 Task

下面我们通过代码来演示一下：

```groovy
//======================================= mustRunAfter ===========================
task taskX{
    doLast {
        println 'taskX'
    }
}

task taskY{
    mustRunAfter taskX
    doLast {
        println 'taskY'
    }
}

task taskXY(dependsOn: [taskX,taskY]){
    doLast {
        println 'taskXY'
    }
}

//执行 taskXY
./gradlew taskXY

//打印结果
> Task :app:taskX
taskX

> Task :app:taskY
taskY

> Task :app:taskXY
taskXY

//======================================= finalizeBy ===========================
task taskI{
    doLast {
        println 'taskI'
    }
}

task taskJ{
    finalizedBy taskI
    doLast {
        println 'taskJ'
    }
}


task taskIJ(dependsOn: [taskI,taskJ]){
    doLast {
        println 'taskIJ'
    }
}

//执行 taskIJ
./gradlew taskIJ

//打印结果
> Task :app:taskJ
taskJ

> Task :app:taskI
taskI

> Task :app:taskIJ
taskIJ
复制代码
```

## 四、自定义 Task 挂接到 Android 应用构建流程

### 1）、Task 依赖关系插件介绍

我们可以引入如下插件来查看 Task 的一个依赖关系：

```groovy
//1、在根 build.gradle 添加如下代码
buildscript {
    repositories {
      	//...
        maven{
           url "https://plugins.gradle.org/m2/"
        }
    }
    dependencies {
      	//..
        classpath "gradle.plugin.com.dorongold.plugins:task-tree:1.5"
    }
}


// 2、在 app 的 build.gradle 中应用插件
apply plugin: com.dorongold.gradle.tasktree.TaskTreePlugin

/**
 * 3、在命令行中执行：./gradlew <任务名> taskTree --no-repeat 命令即可查看
 * 这里以执行 build 这个 task 为例
 */
./gradlew build taskTree --no-repeat
复制代码
```

经过上面 3 步，我们看下依赖关系图，仅截取部分：

![image-20210718114010544](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/86c516a5a77b459da8395072715338d8~tplv-k3u1fbpfcp-watermark.awebp)

### 2）、自定义 Task 挂接到 Android 构建流程

我们知道，Gradle 在执行阶段就是执行 Task 及其依赖的 Task，就比如上面截图的 build Task 的关系依赖图，它会按照这个依赖图有条不紊的去执行。

那么如果我想把自己自定义的 Task 挂接到这个构建流程，该怎么做呢？

##### 1、通过 dependsOn 指定

**注意：** 单独使用 dependsOn ，必须让构建流程中的 Task 依赖我们自定义的 Task，否则我们的 Task 不会生效

如下代码演示一下：

```groovy
task myCustomTask{
    doLast {
        println 'This is myCustomTask'
    }
}

afterEvaluate {
    //1、找到需要的构建流程 Task
    def mergeDebugResources = tasks.findByName("mergeDebugResources")
    //2、通过 dependsOn 指定
    mergeDebugResources.dependsOn(myCustomTask)
  
    //如果换成下面这种写法则自定义 Task 不会生效
    //myCustomTask.dependsOn(mergeDebugResources)
}
复制代码
```

接下来我们验证一下

首先看一眼 Task 依赖关系图：

![image-20210718130023237](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/abf5fd86aaed4ecd818e9df7b0aa8ba1~tplv-k3u1fbpfcp-watermark.awebp)

我们自定义的 Task 挂接到了 mergeDebugResources 上

执行下 build 这个 Task，可以发现我们的 Task 被执行了：

![image-20210718130303391](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4d4a4b36ece04058ba2430def0685a72~tplv-k3u1fbpfcp-watermark.awebp)

##### 2、通过 finalizedBy 指定

在某个 Task 执行完成后，指定需要执行的 Task

```groovy
task myCustomTask{
    doLast {
        println 'This is myCustomTask'
    }
}

afterEvaluate {
    def mergeDebugResources = tasks.findByName("mergeDebugResources")
    //将 myCustomTask 挂接在 mergeDebugResources 后面执行
    mergeDebugResources.finalizedBy(myCustomTask)
}
复制代码
```

##### 3、通过 mustRunAfter 配合 dependsOn 指定

在两个 Task 之间，插入自定义的 Task

```groovy
task myCustomTask{
    doLast {
        println 'This is myCustomTask'
    }
}

afterEvaluate {
    //在 mergeDebugResources 和 processDebugResources 之间插入 myCustomTask
    def processDebugResources = tasks.findByName("processDebugResources")
    def mergeDebugResources = tasks.findByName("mergeDebugResources")
    myCustomTask.mustRunAfter(mergeDebugResources)
    processDebugResources.dependsOn(myCustomTask)
}
复制代码
```

上述 Task 依赖变化过程：

processDebugResources -> mergeDebugResources ===> processDebugResources -> myCustomTask -> mergeDebugResources

## 五、Gradle 相关命令介绍

### 1）、查看项目所有的 Project 对象

```groovy
./gradlew project
复制代码
```

### 2）、查看 module 下所有的 task

```groovy
./gradlew $moduleName:tasks

//演示
//查看 app 下的所有 Task
./gradlew app:tasks

//查看根 Project 的所有 Task
./gradlew tasks
复制代码
```

### 3）、执行一个 Task

```groovy
./gradlew $taskName

//执行 build Task
./gradlew build
复制代码
```

### 4）、查看 module 下的第三方库依赖关系

```groovy
./gradlew $moduleName:dependencies

//查看 app 下的第三方库依赖关系
./gradlew app:dependencies
复制代码
```

## 六、总结

本篇文章讲的一些重点内容：

1、Gradle 三个阶段及生命周期 Hook 点

2、Project 对象常用 API 介绍，扩展属性的应用与实战

3、Task 常用配置介绍，其中通过 Task 输入输出指定执行顺序遇到了坑：会先执行输入的 Task。最终还是通过使用 dependsOn 指定具体依赖才达到预期效果

4、自定义 Task 挂接到 Android 应用构建流程的三种方式：

> 1、单独使用 dependsOn （注意必须使用构建流程中的 Task 依赖我们自定义的 Task）
>
> 2、使用 finalizedBy
>
> 3、mustRunAfter 配合 dependsOn

5、Gradle 一些常用的命令介绍

好了，本篇文章到这里就结束了，希望能给你带来帮助 🤝

**感谢你阅读这篇文章**

### 下篇预告

下篇文章我会讲如何自定义第三方插件，敬请期待吧😄

### 参考和推荐

[补齐Android技能树 - 玩转Gradle](https://juejin.cn/post/6950643579643494431#heading-14)

[Gradle学习系列（二）：Gradle核心探索](https://juejin.cn/post/6937208620337610766/#heading-31)

[深度探索 Gradle 自动化构建技术（三、Gradle 核心解密）](https://juejin.cn/post/6844904122492125198#heading-26)

[从Gradle生命周期到自定义Task挂接到Build构建流程全解](https://juejin.cn/post/6982379643311489032#heading-15)

[7个你应该知道的Gradle实用技巧](https://juejin.cn/post/6947675376835362846#heading-1)

> 全文到此，原创不易，欢迎点赞，收藏，评论和转发，你的认可是我创作的动力

> 欢迎关注我的 **公 众 号**，微信搜索 **sweetying** ，文章更新可第一时间收到