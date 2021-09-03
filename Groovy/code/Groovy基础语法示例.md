```groovy
/**
 * Groovy基础语法示例
 *
 * @author DuChao
 * @date 2021/8/25 5:51 下午
 */
class BasicGrammar {

    static void main(String[] args) {
//        dynamicType()
//
//        referenceType()

//        stringChanges()

//        println(supportPowerOperator(2, 4))

//        println(simpleTernaryExpression(""))

//        switchChanges("张三")

//        judgeNull()

        tryCatchChanges(1)
    }

    /**
     * 1. 动态类型示例
     * Groovy 中，我们可以这样定义，在变量赋值后，Groovy 编译器会推断出变量的实际类型
     * 不用写 ; 号， 现在比较新的语言都不用写，如 Kotlin
     */
    static void dynamicType() {
        def name = "张三 "
        def age = 10
        println(name + age)
    }

    /**
     * 2. 引用类型示例
     * Groovy中没有基本数据类型，全是引用类型，即使定义了基础类型，也会被转换成对应的包装类
     */
    static void referenceType() {
        byte mByte = 1
        short mShort = 2
        int mInt = 3
        long mLong = 4
        float mFloat = 5
        double mDouble = 6
        char mChar = 'a'
        boolean mBoolean = true

        println(mByte.class)
        println(mShort.class)
        println(mInt.class)
        println(mLong.class)
        println(mFloat.class)
        println(mDouble.class)
        println(mChar.class)
        println(mBoolean.class)
    }

    /**
     * 3. 方法变化
     1、使用 def 关键字定义一个方法，方法不需要指定返回值类型，参数类型，方法体内的最后一行会自动作为返回值，而不需要return关键字
     2、方法调用可以不写 () ，最好还是加上 () 的好，不然可读性不好
     3、定义方法时，如果参数没有返回值类型，我们可以省略 def，使用 void 即可
     4、实际上不管有没有返回值，Groovy 中返回的都是 Object 类型
     5、类的构造方法，避免添加 def 关键字
     */
    static class MethodChanges {

        static void main(String[] args) {
            println(sum(1, 2))
            println(sum("张三", "李四"))
        }

        /**
         * 还可以写成这样，但是可读性不好 def sum = sum 1,2
         */
        static def sum(a, b) {
            a + b
        }

        /**
         * 类的构造方法，避免添加 def 关键字
         */
        MethodChanges() {

        }

    }

    /**
     * 4. 字符串变化
     在 Groovy 中有三种常用的字符串定义方式，如下所示：
     这里先解释一下可扩展字符串的含义，可扩展字符串就是字符串里面可以引用变量，表达式等等
     1 、单引号 '' 定义的字符串为不可扩展字符串
     2 、双引号 "" 定义的字符串为可扩展字符串，可扩展字符串里面可以使用 ${} 引用变量值，当 {} 里面只有一个变量，非表达式时，{}也可以去掉
     3 、三引号 ''' ''' 定义的字符串为输出带格式的不可扩展字符串
     */
    static void stringChanges() {
        def name = "张三"
        def age = 10

        // 定义一个不可扩展字符串，和我们在Java中使用差不多
        def str1 = 'hello ' + name

        // 定义可扩展字符串，字符串里面可以引用变量值，当 {} 里面只有一个变量时，{}也可以去掉
        def str2 = "hello $name ${name + age}"

        // 定义带输出格式的不可扩展字符串 使用 \ 字符来分行
        def str3 = '''
            \
            hello
            name
        '''

        // str1类型: class java.lang.String
        println('str1类型: ' + str1.class)
        println('str1输出: ' + str1)

        // str2类型: class org.codehaus.groovy.runtime.GStringImpl
        println('str2类型: ' + str2.class)
        println('str2输出: ' + str2)

        // str3类型: class java.lang.String
        println('str3类型: ' + str3.class)
        println('str3输出: ' + str3)

        // 编码的过程中，不需要特别关注 String 和 GString 的区别，编译器会帮助我们自动转换类型
        String str4 = str2
        println('str4类型: ' + str4.class)
        println('str4输出: ' + str4)
    }

    /**
     * 5. 不用写 get 和 set 方法
     1、在我们创建属性的时候，Groovy会帮我们自动创建 get 和 set 方法
     2、当我们只定义了一个属性的 get 方法，而没有定义这个属性，默认这个属性只读
     3、我们在使⽤对象 object.field 来获取值或者使用 object.field = value 来赋值的时候，实际上会自动转而调⽤ object.getField() 和 object.setField(value) 方法，如果我们不想调用这个特殊的 get 方法时则可以使用 .@ 直接域访问操作符访问属性本身
     下面用 People1、People2、People3 来模拟1，2，3这三种情况
     */

    /**
     * 情况1：在我们创建属性的时候，Groovy会帮我们自动创建 get 和 set 方法
     */
    static class People1 {
        def name
        def age

        static void main(String[] args) {
            def people = new People1()
            people.name = '张三'
            people.age = 10
            println("姓名: $people.name, 年龄: $people.age")
        }
    }

    /**
     * 情况2 当我们定义了一个属性的 get 方法，而没有定义这个属性，默认这个属性只读
     */
    static class People2 {
        def name
        def getAge() {
            10
        }

        static void main(String[] args) {
            def people = new People2()
            people.name = '张三'
            /**
             * Exception in thread "main" groovy.lang.ReadOnlyPropertyException: Cannot set readonly property: age for class: Demo$People2
             * 大致意思就是不能修改一个只读的属性
             */
            people.age = 10
            println("姓名: $people.name, 年龄: $people.age")
        }
    }

    /**
     * 情况3: 如果我们不想调用这个特殊的 get 方法时则可以使用 .@ 直接域访问操作符访问属性本身
     */
    static class People3 {
        def name
        def age

        def getName() {
            "My name is $name"
        }

        static void main(String[] args) {
            def people = new People3()
            people.name = '张三'
            people.age = 10

            /**
             * 使用 people.name 则会去调用这个属性的get方法，而 people.@name 则会访问这个属性本身
             */
            println(people.@name)
            println("姓名: $people.name, 年龄: $people.age")
        }
    }

    /**
     * 6. Class 是一等公民，所有的 Class 类型可以省略 .class
     */
    static class ClassChanges {

        static def testClass(myClass) {}

        static void main(String[] args) {
            testClass(ClassChanges)
            testClass(ClassChanges.class)
            println(ClassChanges)
            println(ClassChanges.class)
        }

    }

    /**
     * 7. == 和 equals
     在 Groovy 中，== 就相当于 Java 的 equals，如果需要比较两个对象是否是同一个，需要使用 .is()
     */
    static class CompareChanges {
        def name

        static void main(String[] args) {
            def obj1 = new CompareChanges(name: '张三')
            def obj2 = new CompareChanges(name: '张三')
            // false
            println(obj1.is(obj2))
            // true
            println(obj1.name == obj2.name)
        }
    }

    /**
     * 8. 使用 assert 来设置断言，当断言的条件为 false 时，程序将会抛出异常
     */
    static class AssertChanges {

        static void main(String[] args) {
            /**
             Exception in thread "main" Assertion failed:

             assert 2 ** 4 == 15
             |    |
             16   false
             */
            assert 2 ** 4 == 15
        }

    }

    /**
     * 9. 支持 ** 次方运算符(幂运算)
     */
    static def supportPowerOperator(base, index) {
        base ** index
    }

    /**
     * 10. 简洁的三元表达式
     在Groovy中，我们可以这样写，?: 操作符表示如果左边结果不为空则取左边的值，否则取右边的值
     */
    static def simpleTernaryExpression(str) {
        str ?: "空白字符串"
    }

    /**
     * 11. 简洁的非空判断
     在Groovy中，我们可以这样写 ?. 操作符表示如?果当前调用对象为空就不执行了
     */
    static class NotNullJudge {
        People1 people

        static void main(String[] args) {
            def test = new NotNullJudge(people: new People1())
            if (test?.people) println("对象连续判断皆不为空,执行打印")
            if (test?.people?.name) println("对象连续判断中name为空,不执行打印")
        }
    }

    /**
     * 12. 强大的 Switch
     在 Groovy 中，switch 方法变得更加灵活，强大，可以同时支持更多的参数类型，比在 Java 中增强了很多
     */
    static void switchChanges(obj) {
        switch (obj) {
            case [1, 2, "张三"]:
                println("匹配成功!")
                break
            default:
                println("进入默认分支")
                break
        }
    }

    /**
     * 13、判断是否为 null 和 非运算符
     在 Groovy 中，所有类型都能转成布尔值，比如 null 就相当于0或者相当于false，其他则相当于true
     */
    static void judgeNull() {
        def name = "张三"

        // 在 Groovy 中，可以这么用，如果name为 null 或 0 则返回 false，否则返回true
        if (name) {
            println("name不为空")
        }

        // 直接断言非空
        assert name
        assert null
    }

    /**
     * 14. 可以使用 Number 类去替代 float、double 等类型，省去考虑精度的麻烦
     */

    /**
     * 15. 默认是 public 权限
     默认情况下，Groovy 的 class 和 方法都是 public 权限，所以我们可以省略 public 关键字，除非我们想使用 private 修饰符
     */

    /**
     * 16. 使用命名的参数初始化和默认的构造器
     Groovy中，我们在创建一个对象实例的时候，可以直接在构造方法中通过 key value 的形式给属性赋值，而不需要去写构造方法
     */
    static class DefaultConstructor {

        static void main(String[] args) {
            // 我们可以通过以下几种方式去实例化一个对象，注意我们People类里面没有写任何一个构造方法哦
            def people1 = new People1(age: 10, name: '张三')
            def people2 = new People1(age: 10)
            def people3 = new People1(name: '张三')
            def people4 = new People1()
        }

    }

    /**
     * 17. 使用 with 函数操作同一个对象的多个属性和方法
     */
    static class With {
        static class People4 {
            def name
            def age

            static void running() {
                println("跑步")
            }
        }

        static void main(String[] args) {
            def people = new People4()

            //调用 with 函数 闭包参数即为people 如果闭包不写参数，默认会有一个 it 参数
            people.with {
                name = '张三'
                age = 10
                println("$name $age")
                running()
            }
        }
    }

    /**
     * 18. 异常捕获
     如果你实在不想关心 try 块里抛出何种异常，你可以简单的捕获所有异常，并且可以省略异常类型：
     */
    static void tryCatchChanges(obj) {
        try {
            obj / 0
        } catch (ignored) {
            // 并不包括Throwable
            println("异常! $ignored.message")
        }
    }

}
```

