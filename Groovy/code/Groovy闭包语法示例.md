```groovy
/**
 * Groovy闭包语法示例
 *
 * 1. 闭包定义
 * Groovy 中的闭包是一个开放的、匿名的代码块，它可以接受参数、返回值并将值赋给变量
 * 通俗的讲，闭包可以作为方法的参数和返回值，也可以作为一个变量而存在，闭包本质上就是一段代码块

 * 2. 闭包声明
 * 2.1、闭包基本的语法结构：外面一对大括号，接着是申明参数，参数类型可省略，在是一个 -> 箭头号，最后就是闭包体里面的内容
     { params ->
        //do something
     }

 * 2.2、闭包也可以不定义参数，如果闭包没定义参数的话，则隐含有一个参数，这个参数名字叫 it
     {
        //do something
     }
 *
 * @author DuChao
 * @date 2021/8/25 5:51 下午
 */
class Closure {

    static void main(String[] args) {
        call()
    }

    /**
     * 3. 闭包调用
     *
     * 1、闭包可以通过 .call 方法来调用
     * 2、闭包可以直接用括号+参数来调用
     */
    static void call() {
        def closure = { println("无参闭包") }
        def closure1 = { arg1, arg2 -> { arg1 + arg2 } }

        closure()
        println(closure1.call(1, 2))
    }

    /**
     * 4. 闭包进阶
     * 4.1、闭包中的关键变量
     *
     每个闭包中都含有 this、owner 和 delegate 这三个内置对象
     getThisObject() 方法 和 thisObject 属性等同于 this
     getOwner() 方法 等同于 owner
     getDelegate() 方法 等同于 delegate
     *
     this 永远指向定义该闭包最近的类对象，就近原则，定义闭包时，哪个类离的最近就指向哪个
     owner 永远指向定义该闭包的类对象或者闭包对象(不是指向当前闭包)，顾名思义，闭包只能定义在类中或者闭包中
     delegate 和 owner 是一样的，我们在闭包的源码中可以看到，owner 会把自己的值赋给 delegate，但同时 delegate 也可以赋其他值
     *
     注意：在我们使用 this , owner , 和 delegate 的时候， this 和 owner 默认是只读的，我们外部修改不了它，这点在源码中也有体现，但是可以对 delegate 进行操作
     */
    static class OuterClosure {

        static def outerClosure = {
            // outer: Closure$OuterClosure$__clinit__closure1@7a52f2a2
            println("outer: " + outerClosure)

            // outer this: class Closure$OuterClosure
            println("outer this: " + this)

            // outer owner: class Closure$OuterClosure
            println("outer owner: " + owner)

            // outer delegate: class Closure$OuterClosure
            println("outer delegate: " + delegate)

            def innerClosure = {
                // inner this: class Closure$OuterClosure
                println("inner this: " + this)

                // inner owner: Closure$OuterClosure$__clinit__closure1@7a52f2a2
                println("inner owner: " + owner)

                // inner delegate: Closure$OuterClosure$__clinit__closure1@7a52f2a2
                println("inner delegate: " + delegate)

            }

            // inner: Closure$OuterClosure$__clinit__closure1$_closure2@22635ba0
            println("inner: " + innerClosure)
            innerClosure()
        }

        static void main(String[] args) {
            outerClosure.call()
        }

    }

    /**
     * 4.2 闭包委托策略
     *
     * 修改闭包的 delegate 实操
     */
    static class DelegateStrategy {

        static class People {
            def age
        }

        static void main(String[] args) {
            def tom = new People(age: 18)
            def jerry = new People(age: 20)

            def closure = { println(delegate.age) }
            /**
             * 如果不加下面这句代码(改变 delegate 指向), 会报如下异常
             * Exception in thread "main" groovy.lang.MissingPropertyException: No such property: age for class: Closure
             *
             * 原因:
             *  delegate 默认同 owner, 执行定义该闭包的类对象或闭包对象,
             *  此处即指向 DelegateStrategy, 该类没有 age 属性, 所以报错
             *
             * 解决:
             *  改变 delegate 的委托对象指针到 tom 或 jerry 对象上即可
             */
            closure.delegate = jerry
            closure.call();
        }

    }

    /**
     * 4.3 深入闭包委托策略
     *
     1、OWNER_FIRST：默认策略,首先从 owner 上寻找属性或方法,找不到则在 delegate 上寻找
     2、DELEGATE_FIRST：先找 delegate, 找不到再找 owner
     3、OWNER_ONLY：只找 owner
     4、DELEGATE_ONLY：只找 delegate
     5、TO_SELF：开发自定义策略,必须自定义实现一个Closure类,一般用不上
     */
    static class ClosureDepth {
        def str1 = "Hello "

        def outerClosure = {
            println str1

            def str2 = "World"

            def innerClosure = {
                // 这里能访问到最外层的 ClosureDepth 中的属性str1，是因为默认的解析委托策略
                // 上级没有该属性、则继续往上级找
                println str1
                println str2
            }
            innerClosure.call()
        }

        static void main(String[] args) {
            def closureDepth = new ClosureDepth()
            closureDepth.outerClosure.call()
        }
    }

    /**
     * 4.3 测试修改闭包默认委托策略为 DELEGATE_FIRST
     */
    static class DelegateFirst {
        static class People1 {
            def name = '我是People1'

            static def action(){
                println '吃饭'
            }

            def closure = {
                println name
                action()
            }
        }

        static class People2 {
            def name = '我是People2'

            static def action(){
                println '睡觉'
            }
        }

        static void main(String[] args) {
            def people1 = new People1()
            def people2 = new People2()
            people1.closure.resolveStrategy = groovy.lang.Closure.DELEGATE_FIRST
            people1.closure.delegate = people2
            people1.closure.call()
        }
    }

}
```

