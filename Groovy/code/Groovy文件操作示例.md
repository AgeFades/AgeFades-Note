```groovy
import groovy.json.JsonSlurper
import groovy.xml.MarkupBuilder
import groovy.xml.XmlSlurper

/**
 * 文件处理
 *
 * @author DuChao* @date 2021/9/2 10:33 上午
 */
class FileProcess {

    static void main(String[] args) {
//        io()

//        xmlAnalyze()

//        xmlGenerate()

        jsonAnalyze()
    }

    /**
     * 1. IO
     */
    static void io() {

        // 读文件,该路径表示文件在 src 同级目录
        def file = new File('test.txt')

//        // 遍历文件行
//        file.eachLine { println it }
//
//        // 获取输入流遍历文件行
//        file.withInputStream { it -> it.eachLine { println it } }
//        file.withReader { it -> it.readLines().each { println it } }
//
//        // 获取输出流写文件(以下两种方式会覆盖源文件)
//        file.withOutputStream { it.write("测试写文件".getBytes()) }
//        file.eachLine { println it }
//        file.withWriter { it.write("测试覆盖写文件") }
//        file.eachLine { println it }

        // 通过文件输入输出流实现文件拷贝
        def targetFile = new File('testCopy1.txt')
//        targetFile.withOutputStream { output ->
//            file.withInputStream {
//                input -> output << input
//            }
//        }

        targetFile.withWriter { writer ->
            file.withReader { reader ->
                {
                    reader.eachLine { line ->
                        {
                            writer.write line + "\r\n"
                        }
                    }
                }
            }
        }

    }

    /**
     * 2. XML文件操作
     * 2.1、解析XML文件
     */
    static void xmlAnalyze() {
        def xml = '''
            <response>
                <value>
                    <books id="1" classification="android">
                        <book available="14" id="2">
                           <title>第一行代码</title>
                           <author id="2">郭霖</author>
                       </book>
                       <book available="13" id="3">
                           <title>Android开发艺术探索</title>
                           <author id="3">任玉刚</author>
                       </book>
                   </books>i z
               </value>
            </response>
        '''

        def xmlSlurper = new XmlSlurper()
        def response = xmlSlurper.parseText(xml)
        response.value.books.each { books ->
            books.book.each {
                println it.title
                println it.author
            }
        }

        //2、深度遍历 XML 数据
        def str1 = response.depthFirst().findAll { book ->
            return book.author == '郭霖'
        }
        println str1


        //3、广度遍历 XML 数据
        def str2 = response.value.books.children().findAll { node ->
            node.name() == 'book' && node.@id == '2'
        }.collect { node ->
            "$node.title $node.author"
        }
        println str2
    }

    /**
     * 2. XML文件操作
     * 2.2、生成XML文件(数据写死)
     */
    static void xmlGenerate() {
        def sw = new StringWriter()
        def xmlBuilder = new MarkupBuilder(sw)
        xmlBuilder.response {
            value {
                books(id: '1', classification: 'android') {
                    book(available: '14', id: '2') {
                        title('第一行代码')
                        author(id: '2', '郭霖')
                    }
                    book(available: '13', id: '3') {
                        title('Android开发艺术探索')
                        author(id: '3', '任玉刚')
                    }
                }
            }
        }
        println sw
    }

    /**
     * 2. XML文件操作
     * 2.2、创建XML对应数据模型
     */
    static class xmlGenerateModel {
        static class Response {
            def value = new Value()

            static class Value {

                def books = new Books(id: '1', classification: 'android')

                static class Books {
                    def id
                    def classification
                    def book = [new Book(available: '14', id: '2', title: '第一行代码', authorId: 2, author: '郭霖'),
                                new Book(available: '13', id: '3', title: 'Android开发艺术探索', authorId: 3, author: '任玉刚')]

                    static class Book {
                        def available
                        def id
                        def title
                        def authorId
                        def author
                    }
                }
            }
        }

        static void main(String[] args) {
            def sw = new StringWriter()
            def xmlBuilder = new MarkupBuilder(sw)
            def response = new Response()

            xmlBuilder.response {
                value {
                    books(id: response.value.books.id, classification: response.value.books.classification) {
                        response.value.books.book.each {
                            def book1 = it
                            book(available: it.available, id: it.id) {
                                title(book1.title)
                                author(authorId: book1.authorId, book1.author)
                            }
                        }
                    }
                }
            }
            println sw
        }
    }

    /**
     * 3. Json解析
     */
    static void jsonAnalyze() {
        def connect = new URL('https://www.wanandroid.com/banner/json').openConnection()
        connect.connect()
        def jsonSlurper = new JsonSlurper()
        println jsonSlurper.parseText(connect.content.text)
    }

}

```

