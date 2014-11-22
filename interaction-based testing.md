# 基于交互（Interaction）的测试  #
   基于Interaction的测试是一种早在2000年就已经在极限编程社区出现的设计和测试技术。它更关注于对象的行为(behavior)而非状态(state)，它考察的是对象如何在规格说明书（specification）之下通过方法调用的方式来与其协作者进行交互。

比如，假设我们有一个`Publisher`发布者，它会向其“Subscriber”订阅者们发送消息：

    class Publisher {
    List<Subscriber> subscribers
    void send(String message)
    }
    
    interface Subscriber {
    void receive(String message)
    }
    
    class PublisherSpec extends Specification {
    Publisher publisher = new Publisher()
    }

我们该如何测试这个`Publisher`呢？如果我们采用基于状态的测试方法，我们能够证明这个发布者会对它的订阅者进行跟踪记录。尽管这样，但是人们更加关心的问题是，subscriber是否收到了publisher发送的消息。为了回答这个问题，我们需要实现一个特殊的`Subscriber`，这个`Subscriber`能够监听发布者和它的订阅者们之间的会话，我们所实现的这种的对象通常被称作是mock 对象。

尽管我们肯定可以手工创建一个`Subscriber`的mock实现，但是随着方法的数目以及交互的复杂性的不断增长，编写和维护这套代码可能会变得非常令人不爽。这种情况下，mocking框架就应运而生了：它们可以用于描述规范说明书中的对象与协作方之间所期望的交互内容（expected interaction），它们还可以生成那些协作方的mock实现来用于验证这些期望。

Java世界里并不缺乏这种流行而且成熟的mocking框架,比如有：[JMock](http://www.jmock.org),
[EasyMock](http://www.easymock.org), [Mockito](http://code.google.com/p/mockito)。尽管这些框架可以在Spock中正常使用，但是我们还是决定开发我们自己的mocking框架，这个框架可以与Spock的规范式语言(specification language)紧紧地地集成在一起。我们希望能够充分利用Groovy的能力来使得基于Interaction的测试代码更加易于编写、更加容易阅读、以及更加有趣，受此驱动我们决定开发这一mocking框架。我们希望在本章结束的时候，你能认可我们已经实现了这些目标。

除非特别指明，不管是测试Java还是测试Groovy代码，Spock的mocking框架的所有特性都是同时有效的。



>Mock实现是如何生成的？
>>就像大多数的Java mocking框架一样，Spock采用JDK动态代理（用于mocking 接口）和CGLIB代理（用于mocking 类 class）来在运行时生成mock实现。相较于基于Grooy元编程的实现，它的优点就是它同样可以用于测试java代码。

#创建Mock对象#

Mock对象是使用`MockingApi.Mock()`方法创建的【[1](#creating)】。下面让我们创建两个mock订阅者：

    def subscriber = Mock(Subscriber)
	subscriber2 = Mock(Subscriber

或者，下面这种Java样式的语法也是支持的，可能这种方式IDE支持的更好一些：

	Subscriber subscriber = Mock()
	Subscriber subscriber2 = Mock()

在这种情况下，mock对象的类型是从赋值操作的等号左边的变量类型来推断出来的。

 
> 注意
>>如果你已经通过等号左边的类型声明来设置mock的类型，那么你就可以在右边忽略掉类型的（不是强制的）。


Mock对象会像代码所描述的那样准确地实现（或者，按照class的说法是，扩展 extend）它们所代表的类型。换句话说，在我们的这个例子中，`subscriber` is-a  `Subscriber`。因此，可以将它传递给希望使用该类型的静态类型（Java）代码。

# Mock对象的默认行为 #
最开始的时候，mock对象并不具备任何行为。调用它们的方法是允许的，但是除了返回该方法的返回类型的默认值（`false`, `0`, 或者 `null`）以外，这些调用不会起到任何真正的作用，例外的情况是`Object.equals`, `Object.hashCode`和`Object.toString`方法，这些方法有如下的默认行为：mock对象只等于（equal）它自己，有唯一的hash code，toString返回的字符串只包含它所代表的类型的名称。方法调用的默认行为是可以通过给这些方法打桩（stubbing）来覆盖的，我们会在[打桩](#stubbing)（Stubbing）一节中进行学习。

#<a name =injecting-mock-objects-into-code-under-specification> 将Mock对象注入到规范说明书的代码中</a>#

在创建了发布者和订阅者以后，我们需要让前者知道后者的存在。

	class PublisherSpec extends Specification {
    Publisher publisher = new Publisher()
    Subscriber subscriber = Mock()
    Subscriber subscriber2 = Mock()

    def setup() {
        publisher.subscribers << subscriber // << is a Groovy shorthand for List.add()
        publisher.subscribers << subscriber2
    	}
	}

我们现在已经准备好来描述这两者之间所期望的Interaction了。

# <a name=Mocking>  Mocking  </a>#
Mocking是一种行为，这种行为描述了规格说明书中的对象与其协作者之间的（（强制的））Interaction， 例子如下：
	
	def "should send messages to all subscribers"() {
	    when:
	    publisher.send("hello")
	
	    then:
	    1 * subscriber.receive("hello")
	    1 * subscriber2.receive("hello")
	}
		
让我们大声读出来 ：“当发布者发送了一条'hello'消息时（When the publisher sends a ‘hello’），两个订阅者都应该能够准确地接收到这条消息，并且次数是只有一次”。
在这个方法运行起来以后，当执行`when`代码段的时候，就会出现一些对mock对象的调用，对这些mock对象的所有调用都会与`then:`代码段中所描述的Interaction进行匹配。如果其中一个Interaction不满足，一个`InteractionNotSatisfiedError`或者其子类的异常对象就会被抛出来。这一验证过程会自动执行，不需要任何额外的代码。

# <a name="Interactions">交互（Interactions）</a> #
现在让我们近距离观察一下`then:`代码段。它包含两条Interaction，每一条Interaction又包含四部分：基数（cardinality），目标约束（target constraint）, 方法约束（method constraint），和一个参数约束（argument constraint）：
	
	1 * subscriber.receive("hello")
	|   |          |       |
	|   |          |       argument constraint
	|   |          method constraint
	|   target constraint
	cardinality

>Interaction 仅仅是一个普通的函数调用么？ 
>>并不完全是。尽管Interaction看起来和普通的方法调用很相似，但是它仅仅是一种表述方式，表示出希望发生哪个方法调用。理解Interaction的一种很好的方式就是将它当作一个正则表达式，所有对mock对象的调用都必须和它进行匹配。视情况而定，交互记录可能匹配零次、一次或者多次调用。

# <a name="cardinality">基数（cardinality）</a> #
一条Interaction的基数描述的是希望一个方法被调用的次数。它可以是一个固定的数值也可以是一个范围：
	
	1 * subscriber.receive("hello")      // 只调用1次
	0 * subscriber.receive("hello")      // 调用0次
	(1..3) * subscriber.receive("hello") // 调用1到3次（包括）
	(1.._) * subscriber.receive("hello") // 至少要调用1次
	(_..3) * subscriber.receive("hello") // 最多调用3次
	_ * subscriber.receive("hello")      // 调用任意次，包括0次
	                                     // (很少需要; 参见 'Strict Mocking')

# <a name="target-constraint">目标约束（target constraint）</a> #
一条Interaction中的目标约束描述的是希望由哪个mock对象来接收方法调用：
	
	1 * subscriber.receive("hello") //	对 'subscriber'对象的调用
	1 * _.receive("hello")          // 对任何 mock对象的调用


# <a name="method-constraint">方法约束（method constraint）</a> #
Interaction中的方法约束描述的是所希望被调用的方法:
	
	1 * subscriber.receive("hello") // 名为 'receive'的方法
	1 * subscriber./r.*e/("hello")  //符合所给定的正则表达式的方法 
	                                // (本处: 以'r'开头并以'e'结尾的方法)
当期望调用getter方法时，可以使用Groovy的属性语法来代替方法语法:

	1 * subscriber.status //等同于: 1 * subscriber.getStatus()
当期望调用一个setter方法时，只能使用方法语法：

	1 * subscriber.setStatus("ok") // 不可以使用: 1 * subscriber.status = "ok"

# <a name="argument-constraint">参数约束（argument constraint）</a> #
Interaction中的参数约束描述的是所期望的方法参数：
	
	1 * subscriber.receive("hello")     // 一个等于 "hello"字符串的参数
	1 * subscriber.receive(!"hello")    //一个不等于 "hello"字符串的参数
	1 * subscriber.receive()            // 空参数列表 (并不与本例相匹配)
	1 * subscriber.receive(_)           //任何单个参数 (包括 null)
	1 * subscriber.receive(*_)          // 任何参数列表 (包括空的参数列表)
	1 * subscriber.receive(!null)       // 任何非null参数
	1 * subscriber.receive(_ as String) //任何非null的 String类型的参数
	1 * subscriber.receive({ it.size() > 3 }) // 一个符合给定的断言的参数
	                                          // (本处: message 的长度要大于3)

对于那些拥有多个参数的方法，参数约束也同样是有效的：

	1 * process.invoke("ls", "-a", _, !null, { ["abcdefghiklmnopqrstuwx1"].contains(it) })

当处理可变参数方法时，可变参数语法同样可以在对应的Interaction中进行使用：
	
	interface VarArgSubscriber {
	    void receive(String... messages)
	}
	
	...
	
	subscriber.receive("hello", "goodbye")
 
> Spock进阶：Groovy可变参数
> >Groovy 允许任何最后一个参数是array类型的方法采用可变参数的形式进行调用。因此，可变参数语法同样可以用于和这些方法相匹配的交互记录中。

#<a name="matching-any-method-call">匹配“任意”方法调用</a>  #
有时候，它在匹配某种意义上的“任意”的时候非常有用。
	
	1 * subscriber._(*_)     // subscriber的具有任意参数列表的任意方法
	1 * subscriber._         // 效果和上面例子相同，是上面例子的一种缩写快捷方式并且是优先采纳的方式
	1 * _._                  // 对任意mock对象的任意方法的调用
	1 * _                    // 上面例子的缩写快捷方式并且优先采用。
>注意
>> 尽管`(_.._) * _._(*_) >> _`是一个合法的Interaction声明，但是它既不是一种好的风格也没有特别的用处。


#<a name="strict-mocking">Strict Mocking</a> #

现在，什么时候匹配任意方法调用是有用的呢？一种好的例子就是strict mocking，它是一种mocking形式，在这种情况下，除了那些显示声明的交互记录，任何其他的交互记录都是不允许的。

	when:
	publisher.publish("hello")
	
	then:
	1 * subscriber.receive("hello") // 要求调用一次subscriber的'receive'方法`
	_ * auditing._                   // 允许任何'auditing'对象的interaction
	0 * _                           // 不允许任何其他的interaction

    
`0 *`只有在作为`then:`代码段或者方法中的最后一条记录时才有意义。注意`_ *`
的使用（任意数目的调用），它允许auditing元素的任何interaction。

>注意
>>`_ *`只有在strict mocking的上下文环境中才有意义。特别是，当对某个调用打桩（stubbing）时，这是没有任何必要的。比如`_ * auditing.record(_) >> "ok"`可以（并且应该！）简化为`auditing.record(_) >> "ok"`。



#<a name="where-to-declare-interactions">何处声明interaction</a>#

到目前而言，我们将所有的interaction都声明在 `then:`代码段中。这通常会使得规格说明书读起来更加自然。但是，我们也允许将interaction放在`when:`代码段之前任何它们应该满意的地方。尤其是，这意味着可以在`setup`方法中定义interaction。interaction也可以在同一规格说明书（specification）类的任何"helper"实例方法中进行定义。

当出现对mock对象的调用时，它会按照interaction中说声明的顺序来与interaction进行匹配。如果调用与多个interaction相匹配，那么声明最早的那条没有达到调用限制要求的interaction就获胜了。这条规则的一个例外就是：在`then:`代码段中声明的interaction会先于任何其他的interaction而进行匹配。这就使得我们可以用`then:`代码段中声明的interaction来覆盖 `setup`代码段中声明的interaction。


>Spock进阶：interaction是如何被识别的？
>> 换句话说，是什么使得一个表达式成为了一个interaction，而非一个普通的方法调用呢？Spock使用一个简单的语法规则来识别interaction：如果表达式处于声明区域，并且是一个乘法(`*`)或者右移（`>>`,`>>>`）操作，那么它就被认为是一个interaction，进而相应地被进行解析。这种表达式在声明区中几乎没有任何价值，所以改变它的含义是可行的。注意这些操作是如何对应于声明基数（用于mocking）或者响应生成器（用于stubbing）时的语法。这两种情况总有种情况是存在的，单独的`foo.bar()`从来不会被当作是interaction。

# 在Mock创建时声明interaction #

如果mock有一系列的不会改变的“基本的”interaction，我们就可以在创建mock的时候就声明：

	def subscriber = Mock(Subscriber) {
	    1 * receive("hello")
	    1 * receive("goodbye")
	}

这一特性对于stubbing是十分有吸引力的，尤其是当其与专门的Stubs一起使用时。
注意这些interaction没有（并且不能有）目标约束，通过上下文，我们能够很清楚的了解它们属于哪个mock对象。

interaction也可以在使用mock初始化实例的时候进行声明：

	class MySpec extends Specification {
	    Subscriber subscriber = Mock {
	        1 * receive("hello")
	        1 * receive("goodbye")
	    }
	}
将具有相同目标的interaction进行分组
可以在`Specification.with`代码段中对共用同一个目标的interaction进行分组。类似于 在Mocking创建时声明interaction，这样就不需要重复说明目标约束了。

	with(subscriber) {
	    1 * receive("hello")
	    1 * receive("goodbye")
	}

`with`代码段同样可以用于将同一目标的 condition进行分组。

将交互和条件进行混合

`then:`代码段可以同时包含interaction和条件(Conditions)。尽管没有严格要求，习惯上，我们先声明interaction然后再声明条件：

	when:
	publisher.send("hello")
	
	then:
	1 * subscriber.receive("hello")
	publisher.messageCount == 1

让我们大声读出来“当发布者发送一条'hello'消息后，订阅方就应该准确地接受到这条消息，接受的次数是1次，而且发布者的消息数目应该是1“。


显式interaction代码段

从内部而言，Spock必须在所期望的interaction出现之前就拥有了关于这些interaction的所有全部信息。那么我们怎么又可能将interaction定义在 `then:`代码段中呢？答案就是，在这种情况下，Spock会立刻将`then:`代码段中所声明的interaction移到`when:`代码段之前。在大多数情况下，这是可行的。但是有时候，这回导致一些问题：

	when:
	publisher.send("hello")
	
	then:
	def message = "hello"
	1 * subscriber.receive(message)

在这里，我们为所期待的参数引入了一个变量（同样地，我们可以为基数引入一个变量）。然而，Spock并不足够智能来识别内部的interaction是与变量声明相关联的。因此，它只会将interaction移走，这就会导致在运行的时候抛出`MissingPropertyException `异常提示找不到message变量。

解决该问题的一种办法就是（至少）将变量定义移动到`when:`代码段的前面（数据驱动测试的爱好者们可能会将变量移动到`where`代码段中）。在我们的例子中，这样做的另一个好处就是我们可以在发送消息的时候使用同样的变量。

另一种解决方案就是显式地说明那个变量声明和interaction是属于一起的。

	when:
	publisher.send("hello")
	
	then:
	interaction {
	    def message = "hello"
	    1 * subscriber.receive(message)
	}

因为`MockingApi.interaction`代码段总是作为一个整体一起移动的，所以这段代码现在就可以像预期的那样正常运行了。

##interaction的作用域##

在`then:`代码段中声明的interaction的作用域的作用域在`when:`代码段之前：
	
	when:
	publisher.send("message1")
	
	then:
	1 * subscriber.receive("message1")
	
	when:
	publisher.send("message2")
	
	then:
	1 * subscriber.receive("message2")

这保证在执行第一个 `when:`代码段的时候，`subscriber`会接收到`message1`,
在执行第二个 `when:`代码段的时候，`subscriber`会接收到`message2`。

在`then:`代码段之外声明的interaction从它们的声明开始到该方法的结尾一直都是活动状态。

interaction的作用域一直都是特定的方法。所以，不能在静态方法 `setupSpec`方法或者`cleanupSpec`方法中进行声明。同样mock对象不应该保存到静态字段或者`@Shared`字段中。

##验证interaction##
对于基于mock的测试，失败的原因有两种：interaction能够匹配的方法调用要多于所允许次数，或者它所匹配的方法调用的次数要少于所要求的。第一种情况会在调用发生以后立刻被侦测出来，然后抛出`TooManyInvocationsError`异常：
	
	Too many invocations for:
	
	2 * subscriber.receive(_) (3 invocations)

为了更加便于诊断出那么多调用的原因，Spock会显示出所有匹配这个有问题的interaction的调用：
	
	Matching invocations (ordered by last occurrence):
	
	2 * subscriber.receive("hello")   <-- this triggered the error
	1 * subscriber.receive("goodbye")

按照这些输出内容所示，有一个`receive("hello")`调用触发了`TooManyInvocationsError`异常。请注意：由于对`subscriber.receive("hello")`的两次调用是很难区分的，它们被合并到输出结果的同一行中，所以第一个`receive("hello")`有可能在`receive("goodbye")`之前已经出现了。

第二种情况(调用次数比所要求的次数要少)只有在执行完一次`when`代码段以后才能被检测出来。（到那时，其他的调用还是可能继续执行的）它会抛出`TooFewInvocationsError`异常：

	Too few invocations for:
	
	1 * subscriber.receive("hello") (0 invocations)

注意：到底有没有被调用过这个方法，有没有使用过不同的参数来调用这同一个方法，有没有不同的mock对象用过这同一个方法、或者是否调用过不同的方法来代替当前这个方法，这都不重要。不管哪种情况，`TooFewInvocationsError `异常都会抛出来。

为了便于诊断到底发生了什么事情而使得有些调用漏掉了，Spock会展示出和所有interaction都不匹配的所有调用，并按照它们与有问题的interaction的相似度来进行排序。特别是，那些只和interaction的参数不匹配的调用记录会展示在前面：

	Unmatched invocations (ordered by similarity):
	
	1 * subscriber.receive("goodbye")
	1 * subscriber2.receive("hello")

调用顺序

Often, the exact method invocation order isn’t relevant and may change over time. To avoid over-specification, Spock defaults to allowing any invocation order, provided that the specified interactions are eventually satisfied:
经常，准确的方法调用顺序是不相关的并且可能会随着时间变化。为了避免过度规范，倘若最终能够满足所规定的interaction，Spock默认是允许任意的调用顺序的：

	then:
	2 * subscriber.receive("hello")
	1 * subscriber.receive("goodbye")

在这里，任意的调用序列`"hello"` `"hello"` `"goodbye"`, `"hello"` `"goodbye"` `"hello"`和 `"goodbye"` `"hello"` `"hello"`都满足上面指定的interaction。

在某些调用顺序很重要的例子中，你可以采用将interaction分割到多个 `then:`代码段中的方式来影响顺序：

	then:
	2 * subscriber.receive("hello")

	then:
	1 * subscriber.receive("goodbye")

现在，Spock将会首先验证两个`"hello"`消息是否早于`"goodbye"`而被接收到。换句话说，调用的顺序是按照各个`then:`代码段之间的顺序来要求的，而非按照`then:`代码段内部的顺序来要求。
> 注意
>> 使用`and`单词切分`then:`代码段并不会对顺序有任何影响，因为`and` 仅仅起到的文档记录展示的作用，并不承载任何语义概念。

# 对类进行mock#

除了接口以外，Spock同样支持对类的mocking。对类进行mocking的作用类似于对接口进行mocking；唯一的额外要求就是将`cglib-nodep-2.2`或者更高版本的cglib-nodep 以及`objenesis-1.2`或更高版本的objenesis添加到class path 中。如果class path中没有这两个功能库，Spock将会非常委婉地对此进行提示让你知道。 

# 打桩（Stubbing） #
Stubbing是一种让协作方以某种方式来响应方法调用的行为。当对某个方法打桩时，你并不关心这个方法将来是否会被调用以及将会被调用多少次；你只是想要不管什么时候调用该方法，它都会返回某些值，或者触发某些意外情况。

为了便于后续的例子，我们现在对`Subscriber`的 `receive`方法进行修改，让它返回一个状态码来表明subscriber是否能够处理这条消息：

	interface Subscriber {
	    String receive(String message)
	}

现在，让我们来使得`receive`方法在每次调用的时候都返回 "ok"：

	subscriber.receive(_) >> "ok"

大声读出来：“不管subscriber什么时候收到一条消息，它都会用‘ok’来进行响应”。

Compared to a mocked interaction, a stubbed interaction has no cardinality on the left end, but adds a response generator on the right end:

与被mock的接口相比，被打桩的接口在左端没有基数，但是在右端增加了一个响应生成器（response generator）：

	subscriber.receive(_) >> "ok"
	|          |       |     |
	|          |       |     响应生成器（response generator）
	|          |       参数约束（argument constraint）
	|          方法约束（method constraint）
	目标约束（target constraint）


一个被打桩的接口可以在老地方进行声明：在`then:`代码段内部，或者任何`when:`代码段之前的地方。（参见“何处声明接口”一节）。如果某个mock对象只是用于打桩，那么在创建mock的时候或者在`setup:`代码段进行声明是比较常见的。

#返回固定值#

我们已经了解使用右移操作符（`>>`）来返回一个固定值的用法了：

	subscriber.receive(_) >> "ok"


为了针对不同的调用返回不同的值，我们可以使用多个interaction：

	subscriber.receive("message1") >> "ok"
	subscriber.receive("message2") >> "fail"

This will return `"ok"` whenever `"message1"` is received, and `"fail"` whenever `"message2"` is received. There is no limit as to which values can be returned, provided they are compatible with the method’s declared return type.
这样，不论什么时候收到`"message1"`，它都会返回`"ok"`，而不管什么时候收到 `"message2"` ，它都会返回`"fail"`。只要返回的具体值和这个方法所声明的返回类型相兼容，是没有其他限制的。

#返回一系列的值#

为了给一系列连续的调用返回不同的值，我们需要使用 三个右尖括号的操作符 (`>>>`) ：

	subscriber.receive(_) >>> ["ok", "error", "error", "ok"]


这样，第一次调用的时候就会返回`"ok"`, 第二次和第三次调用就会返回`"error"`, 剩下的所有调用都会返回`"ok"`。右边的数据必须是Groovy知道如何迭代的值；在本例中，我们使用了一个简单的list。

#计算返回值#

为了根据方法的参数来计算返回值，我们可以使用右移操作符(`>>`) 和闭包一起来实现。如果闭包声明的是一个无类型参数，它传递的就是该方法的参数列表：

	subscriber.receive(_) >> { args -> args[0].size() > 3 ? "ok" : "fail" }

在本例中，如果这条message多余3个字符，方法就会返回`"ok"`，否则就返回`"fail"`。
>译者注： 
    类似情况的多参数例子：
	subscriber.receive( _, _, _) >> { args -> args[0].size() > 3 && args[1].size>0 &&args[2].size>0 ? "ok" : "fail" }
 


在多数情况下，如果能够直接访问这个方法的参数，这无疑是更加方便的。如果这个闭包声明了多个参数或者声明了一个有类型的参数，方法参数会一个个地映射到闭包的参数中[4]：

	subscriber.receive(_) >> { String message -> message.size() > 3 ? "ok" : "fail" }
 
这个响应生成器的作用和前面一个是一样的，但是可以说这个更加具有可读性。
>译者注：
类似情况的多参数例子：
subscriber.receive(_, _, _) >> { String message, String title, String from -> message.size() > 3 && title.size>0 &&from.size>0 ? "ok" : "fail" }
）
 
如果你发现你需要了解某个方法调用除了参数之外的更多信息，你可以看看`org.spockframework.mock.IMockInvocation`。这个接口所声明的所有方法都可以在闭包内进行使用，而不需要加前缀（用Groovy的术语来说，闭包是委托给一个`IMockInvocation`实例的）。

#PerPerformingforming Side Effects#
#实现意外效果（Side Effects）#

有时候你可能想要完成一些其他事情而非仅仅计算一个返回值。一个常见的例子就是抛出一个异常。这时候还是需要闭包来出手相救：

	subscriber.receive(_) >> { throw new InternalError("ouch") }


当然，闭包可以包含更多的代码，比如一个`println`语句。它会在每次新来的调用和该交互相匹配的时候进行执行。
# Chaining Method Responses #
#方法响应链#

方法的响应可以串联起来形成一条链：

	subscriber.receive(_) >>> ["ok", "fail", "ok"] >> { throw new InternalError() } >> "ok"

这样，在前三次调用时会分别返回` "ok", "fail", "ok"`，而在第四次调用的时候会抛出`InternalError` 异常，后面的所有调用都会返回  ok。

#Mocking和Stubbing的共用#


Mocking和Stubbing并肩出现：

	1 * subscriber.receive("message1") >> "ok"
	1 * subscriber.receive("message2") >> "fail"

当mocking和stubbing 同一个方法调用时，它们必须是在同一个交互中。尤其是在下面这个Mockito风格的代码中，将stubbing和mocking切分成两句单独的代码，这是不能运行的。

	setup:
	subscriber.receive("message1") >> "ok"
	
	when:
	publisher.send("message1")
	
	then:
	1 * subscriber.receive("message1")

正如我在“何处声明交互”一节中说解释的那样，`receive`调用会首先和`then:`代码段中的交互进行匹配。因为这条交互记录没有声明一个响应，那么就会返回这个方法的返回类型的默认值。（这是Spock对于mocking的宽容的一面）。因此 `setup:`代码段中的交互永远没有机会去匹配。

>注意
>>对同一个方法调用的Mocking和Stubbing必须是在同一个交互中。


#其他类型的Mock对象(New in 0.7)#

到现在为止，我们已经使用MockingApi.Mock方法创建了mock对象了。除了该方法外，`MockingApi`还提供了一些其他的工厂方法用于创建更多特定类型的mock对象。

# Stubs #

stub是使用` MockingApi.Stub`工厂方法创建的：

	def subscriber = Stub(Subscriber)

不管在哪里，mock都可以用于stubbing和mocking，而stub只能用于stubbing。将协作者限定为一个stub能够便于向该specification的读者传达出它的作用。


>注意
>>如果某个stub调用与某个强制的交互记录（比如1 * `foo.bar()`）相匹配，那么这时，就会抛出一个`InvalidSpecException`异常。

就像mock一样，stub也允许未定义声明的调用存在。 然而在这种情况下stub返回的值考虑更加全面。

- 对于原始类型，会返回原始类型的默认值。
- 对于非原始的数值型返回值（比如`BigDecimal`）,会返回0。
- 对于非数值型返回值，会返回一个“empty”或者“dummy”的对象。这意味着可能是一个空字符串、空的collection、由默认构造函数生成的一个对象 或者另一个能返回默认值的stub。参见 `org.spockframework.mock.EmptyOrDummyResponse`类来获取更多内容。


一个stub通常具备一组固定的交互记录，这使得在mock创建期声明这些交互记录特别吸引人：

	def subscriber = Stub(Subscriber) {
	    receive("message1") >> "ok"
	    receive("message2") >> "fail"
	}
# Spies #
 
在使用这一功能之前请再考虑一下。修改该specification之下的代码设计方案可能是更好的选择。
 
spy是由`MockingApi.Spy`工厂方法创建的： 

	def subscriber = Spy(SubscriberImpl, constructorArgs: ["Fred"])

spy总是基于某个真实的对象创建的。因此，你必须提供一个class类型，而非接口interface类型，同时还需要提供该类型的构造函数所有的参数。如果没有提供构造参数，就会使用该类型的默认构造函数。

对于spy的方法调用都自动被委托给真实的对象。同样地，从这个真实对象返回的值会通过这个spy传递给调用者。
 在创建了一个spy以后，你可以通过spy来监听调用方和真正的对象之间的会话：

	1 * subscriber.receive(_)
 除了确保这个`receive`只被调用一次以外，publisher同spy所依赖的`SubscriberImpl`实例之间的会话依然没有改变。 
 
当对某个spy的方法进行打桩stubbing时，那个真正的方法就不再被调用了：

	subscriber.receive(_) >> "ok"
 
`receive`方法现在会仅仅返回`"ok"`，而不再调用`SubscriberImpl.receive`。

 
有时候，我们既希望执行一些代码，又希望委托给真实的代码：

	subscriber.receive(_) >> { String message -> callRealMethod(); message.size() > 3 ? "ok" : "fail" }
 
在此处，我们使用 `callRealMethod()`来将方法调用委托给真实的对象。请注意，我们不需要传递 `message`参数；它会被自动处理掉。`callRealMethod()`会返回真实的调用结果，但是在本例中，我们选择返回我们自己的结果。如果我们想要给真正的方法传递一个不同的mesage，我们可以使用`callRealMethodWithArgs("changed message")`。

# Partial Mocks #
 
在使用这一功能之前请再考虑一下。修改该specification之下的代码设计方案可能是更好的选择。
 
Spy同样可以作为部分mock（partial mock）进行使用：

	// this is now the object under specification, not a collaborator
	def persister = Spy(MessagePersister) {
	  // stub a call on the same object
	  isPersistable(_) >> true
	}

	when:
	persister.receive("msg")
	
	then:
	// demand a call on the same object
	1 * persister.persist("msg")
Groovy Mocks (New in 0.7)
So far, all the mocking features we have seen work the same no matter if the calling code is written in Java or Groovy. By leveraging Groovy’s dynamic capabilities, Groovy mocks offer some additional features specifically for testing Groovy code. They are created with the `MockingApi.GroovyMock()`, `MockingApi.GroovyStub()`, and `MockingApi.GroovySpy()` factory methods.
到现在为止。

>When Should Groovy Mocks be Favored over Regular Mocks?
>>Groovy mocks should be used when the code under specification is written in Groovy and some of the unique Groovy mock features are needed. When called from Java code, Groovy mocks will behave like regular mocks. Note that it isn’t necessary to use a Groovy mock merely because the code under specification and/or mocked type is written in Groovy. Unless you have a concrete reason to use a Groovy mock, prefer a regular mock.

# Mocking 动态方法#

所有的Groovy mock都实现了GroovyObject接口，只要它们是真正生命的方法，它们就支持对动态方法进行mock和stubbing：

	def subscriber = GroovyMock(Subscriber)
	
	1 * subscriber.someDynamicMethod("hello")
 

#对某一类型的所有实例进行Mocking#
 
在使用这一功能之前请再考虑一下。修改该specification之下的代码设计方案可能是更好的选择。

通常，Groovy mock需要像普通的mock那样被注入到specification下的代码里。可是，当一个Groovy mock作为一个全局变量被创建出来以后，它会在这个功能方法生命周期里自动地替换掉所mock的类型的所有真实的实例【3】：

	
	def publisher = new Publisher()
	publisher << new RealSubscriber() << new RealSubscriber()
	
	def anySubscriber = GroovyMock(RealSubscriber, global: true)
	
	when:
	publisher.publish("message")
	
	then:
	2 * anySubscriber.receive("message")

在此处，我们创建了一个拥有两个真实的subscriber实例的publisher。然后我们创建了一个相同subscriber类型的global mock。这就使得所有对真实的subscriber所发起的方法调用都被重新转换到mock对象上。这个mock对象实例都并没有被传给publisher对象；它仅仅是用于描述这个交互。

>Note
>>A global mock can only be created for a class type. It effectively replaces all instances of that type for the duration of the feature method.
Since global mocks have a somewhat, well, global effect, it’s often convenient to use them together with `GroovySpy`. This leads to the real code getting executed unless an interaction matches, allowing you to selectively listen in on objects and change their behavior just where needed.

>注意
>>全局的mock只能用于为class类型创建。它非常有效地在功能方法的整个时间段里取代所有该类型的实例。
因为global mock

>How Are Global Groovy Mocks Implemented?
>>Global Groovy mocks get their super powers from Groovy meta-programming. To be more precise, every globally mocked type is assigned a custom meta class for the duration of the feature method. Since a global Groovy mock is still based on a CGLIB proxy, it will retain its general mocking capabilities (but not its super powers) when called from Java code.


 
#Mocking 构造器#
 
在使用这一功能之前请再考虑一下。修改该specification之下的代码设计方案可能是更好的选择。
 
Global mocks支持对构造器进行mocking：
	
	def anySubscriber = GroovySpy(RealSubscriber, global: true)
	
	1 * new RealSubscriber("Fred")
 
因为我们是在使用spy，所以从构造函数调用所返回的对象是没有被修改的。为了修改所构造出来的对象，我们可以对构造器进行打桩：

	new RealSubscriber("Fred") >> new RealSubscriber("Barney")

 
现在，不管什么时候，如果有代码试图构造一个名字为Fred的subscriber，我们都将改为构造一个名字为Barney的subscriber。
 
#Mocking静态方法#
 
在使用这一功能之前请再考虑一下。修改该specification之下的代码设计方案可能是更好的选择。

 
全局mock支持对静态方法的mocking和stubbing：

	def anySubscriber = GroovySpy(RealSubscriber, global: true)
	
	1 * RealSubscriber.someStaticMethod("hello") >> 42
 
这对于动态静态方法（dynamic static methods）同样有效。
 
当一个全局mock仅仅是用于mocking一个构造器和静态方法时，并不真的需要一个mock实例。在这种情况下，可以这么写：

	GroovySpy(RealSubscriber, global: true)

 
#高级功能（v0.7新增）#
 
大多数时候你并不需要用到这些特定，但是如果你愿意的话，你会很乐于了解它们。
 
照单点菜式的Mock
 
到现在为止，`Mock()`、`Stub()`和`Spy()`这些工厂方法都不过是按照某些固定的配置信息来创建mock对象的方式。如果你想要对mock的配置拥有更加细化的控制，可以去了解一下`org.spockframework.mock.IMockConfiguration`接口。这个接口的所有属性都可以作为命名参数传递给`Mock()`方法。比如：

	def person = Mock(name: "Fred", type: Person, defaultResponse: ZeroOrNullResponse, verified: false)

在这里，我们创建了一个mock对象，它的默认返回值和那些通过`Mock()`创建出来的对象的返回值是一样的，但是调用不会进行验证（如同`Stub()`）。我们本来也可以提供我们自定义的`org.spockframework.mock.IDefaultResponse`的，我们可以用它来代替`ZeroOrNullResponse`以响应那些没有定义声明的方法调用。
 
#检测Mock对象#
 
为了确定某个特定的对象是否是一个Spock mock对象，我们可以使用`org.spockframework.mock.MockDetector`：


	def detector = new MockDetector()
	def list1 = []
	def list2 = Mock(List)
	
	expect:
	!detector.isMock(list1)
	detector.isMock(list2)
	
 
这个检测器同样还可以用于获取某个mock对象的其他更多信息：
	
	def mock = detector.asMock(list2)
	
	expect:
	mock.name == "list2"
	mock.type == List
	mock.nature == MockNature.MOCK

 

## 脚注 ##
<a name="creating">1</a> 对于其他创建mock对象的方式，请阅读 [其他类型的Mock对象](#otherkindsofmockobjects)（0.7新引入）