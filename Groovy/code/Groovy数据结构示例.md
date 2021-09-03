```groovy
/**
 * Groovy数据结构语法示例
 *
 * @author DuChao* @date 2021/9/1 3:18 下午
 */
class DataStructure {

    static void main(String[] args) {
//        array()

//        list()

//        listCrud()

//        map()

//        mapOperate()

        range()
    }

    /**
     * 1、数组
     *
     在 Groovy 中使用 [ ] 表示的是一个 List 集合，如果要定义 Array 数组，我们就必须强制指定为一个数组的类型
     */
    static void array() {
        def array = ['张三', '李四', '王五'] as String[]
        println array.class
        println array
    }

    /**
     * 2、List
     * 2.1、列表集合定义
     *
     1、List 即列表集合，对应 Java 中的 List 接口，一般用 ArrayList 作为真正的实现类
     2、定义一个列表集合的方式有点像 Java 中定义数组一样
     3、集合元素可以接收任意的数据类型
     */
    static void list() {
        def list1 = [1, 2, 3]
        println list1.class
        println list1

        def list2 = [1, '张三', true]
        println list2.class
        println list2

        LinkedList list3 = [1, 2, 3]
        def list4 = [1, 2, 3] as LinkedList
        println list3.class
        println list4.class
    }

    /**
     * 4.2、列表集合增删改查
     */
    static void listCrud() {
        def list = [1, 2, 3]
        // 增加元素
        list.add 20
        list.leftShift 30
        list << 40
        println list

        // 删除元素(根据下标删除)
        list.remove(0)
        println list

        // 修改元素(根据下标修改)
        list[0] = 10
        println list

        // 查询元素
        list.find { println it }
    }

    /**
     * 3、Map
     * 3.1 定义
     *
     1、Map 表示键-值表，其底层对应 Java 中的 LinkedHashMap
     2、Map 变量由[:]定义，冒号左边是 key，右边是 Value。key 必须是字符串，value 可以是任何对象
     3、Map 的 key 可以用 '' 或 "" 或 ''' '''包起来，也可以不用引号包起来
     */
    static void map() {
        def map = [a: 1, 'b': true, 'c': "Groovy", '''d''': '''ddd''']
        println map
    }

    /**
     * 3.2 Map常用操作
     */
    static void mapOperate() {
        def map = [a: 1, 'b': true, 'c': "Groovy", '''d''': '''ddd''']

        /**
         * 元素访问
         * 1、map.key
         * 2、map[key]
         * 3、map.get(key)
         */
        println map.a
        println map['b']
        println map.get('c')

        /**
         * 元素的添加和修改
         * 如果当前 key 在 map 中不存在，则添加该元素，如果存在则修改该元素
         */
        map.put('key', "value")
        println map
        map.put('key', 'v')
        println map

        /**
         * Map遍历
         * 1、each：闭包一个参数即 entry,两个参数即 key value
         * 2、eachWithIndex：比 each 多个下标属性
         */
        map.each {println it}
        map.each {k,v -> println k + '\t' + v}
        map.eachWithIndex { k, v,i -> println k + '\t' + v + '\t' + i}
    }

    /**
     * 4、Range
     *
     1. Range表示范围,是List的一种扩展,由 begin 值 + 两个点 + end 值表示
        如果不想包含最后一个元素，则 begin 值 + 两个点 + < + end 表示
     2. 通过 aRange.from 与 aRange.to 来获对应的边界元素
     */
    static void range() {
        def range1 = 1..5
        range1.each { println it}
        println range1.from + "\t" + range1.to

        def range2 = 1..<5
        range2.each { println it}
    }

}
```

