# 基于交互的测试（Interaction Based Testing） #
基于交互的测试是一种早在2000年就在极限编程社区出现的一种设计和测试技术。它更关注于对象的行为而非状态，考察的是对象如何在声明之下通过方法调用的方式来与其合作者进行交互。

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

我们如何测试这个`Publisher`呢？采用基于状态的测试方法，我们可以证明这个发布者会将它的订阅者记录下来。尽管这样，但是人们更加感兴趣的问题是，subscriber是否收到了publisher发送的消息。为了解答这个问题，我们需要实现一个特殊的`Subscriber`，这个`Subscriber`会监听发布者和它的订阅者们之间的会话，实现的这种的对象经常被称作是“mock 对象”。

尽管我们肯定可以手工创建一个`Subscriber` 的mock实现，但是随着方法的数目以及交互的复杂性的增长，编写和维护这套代码可能会变得非常令人讨厌。这种情况下，mocking框架就应运而生了：它们提供了一种描述所声明的对象与协作方之间所期望的交互内容的方式，它们还可以生成那些协作方的mock实现来验证这些期望。

Java世界里并不缺乏这种流行而且成熟的mocking框架,举几个例子：[JMock](http://www.jmock.org),
[EasyMock](http://www.easymock.org), [Mockito](http://code.google.com/p/mockito)。尽管这里面每个框架可以在Spock中使用，但是我们还是决定开发我们自己的mocking框架，这个框架可以与Spock的规范式语言紧紧地地集成在一起。我们希望能够充分利用Groovy的功能来使得基于交互的测试代码更加易于编写、更加容易阅读、最终更加有趣，受此驱动我们决定开发这一mocking框架。我们希望在本章结束的时候，你认可我们已经实现了这些目标。

除非特别指明，Spock的mocking框架的所有特性在测试Java和Groovy代码的时候是同时有效的。
# 创建Mock对象 #

Mock对象是使用`MockingApi.Mock()`方法创建的【[1](#creating)】。下面让我们创建两个mock订阅者：

    def subscriber = Mock(Subscriber)
	subscriber2 = Mock(Subscriber
或者，下面这种Java样式的语法也是支持的，这种方式可能IDE支持的更好一些：

	Subscriber subscriber = Mock()
	Subscriber subscriber2 = Mock()

在这中情况下，mock对象的类型是从赋值操作的左边的变量类型来推断出来的。

 
> 注意
>>如果mock的类型已经通过赋值符左边来进行设置， 我们在右边忽略掉也是可以的（不是强制的）。


Mock对象照着字面意思实现（或者说，按照class的说法是，扩展 extend）它们所代表的类型。换句话说，在我们的这个例子中，`subscriber` is-a  `Subscriber`。因此，它可以传递要求该类型的给静态类型（Java）代码。

# Mock对象的默认行为 #
开始的时候，mock对象没有任何行为。调用它们的方法是允许的，但是除了返回该方法的返回类型的默认值（`false`, `0`, 或者 `null`）以外，这些调用不会造成没有任何影响，例外的情况是`Object.equals`, `Object.hashCode`和`Object.toString`方法，这些方法有如下的默认行为：mock对象只等于（equal）它自己，有唯一的hash code，toString返回的字符串只包含它所代表的类型的名称。默认行为是通过给这些方法打桩（stubbing）来覆盖的，我们会在[打桩](#stubbing)（Stubbing）一节中进行学习。

# [将Mock对象注入到规范中的代码](#injecting-mock-objects-into-code-under-specification)

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

我们现在已经准备来说明这两部分之间所期望的交互。

# [Mocking](#Mocking)

[1](#otherkindsofmockobjects)


[打桩](#stubbing)（Stubbing）

## 脚注 ##
<a name="creating">1</a> 对于其他创建mock对象的方式，请阅读 [其他类型的Mock对象](#otherkindsofmockobjects)（0.7新引入）