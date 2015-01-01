## 2. 语言规则 Language Rules

<img src="https://raw.githubusercontent.com/monifu/scala-best-practices/master/assets/scala-logo-256.png"  align="right" width="128" height="128" />

[英文版](../sections/2-language-rules)

### 2.1. 不要用""return"

java的"return"指令是有副作用的: 展开`栈`,把返回值给调用者。在一个充分使用副作用的语言里，这样合理。但scala是一个面向表达式的语言，scala里在控制限制使用副作用，使用`return`就不合适了。

更糟糕的事，`return`很可能跟你想象的不同。例如，在`PlayFramework`的`Controller`里，尝试这样做：

```scala
def action = Action { request =>
  if (someInvalidationOf(request))
    return BadRequest("bad")

 Ok("all ok")
}
```

在scala里，匿名嵌套函数里的`return`指令是通过抛出和捕捉一个`NonLocalReturnException`异常实现的。详见[Scala Language Specification, section 6.20](http://www.scala-lang.org/docu/files/ScalaReference.pdf).

另外，`return`是反结构化编程的，而函数可以实现多个退出点。如果需要`return`, 就像在一个有很多`if/else`分之的大的方法体中，多个`return`的出现让代码不易读，而且容易产生bug，遇到这种多`return`的情况就需要紧急修复重构代码。

*参考《Avoid multiple returns (exit points) in methods》*

```
译者：      
	1.	scala中可以省略return, scala会根据就近原则返回
	2.	多return退出点在运行期，有可能会降低性能，if/else导致
	3.	单一的return退出点是推荐的编程习惯
```

### 2.2. 用不变的数据结构

不变的数据结构事实上可以被比较和推导。易变的容易导致错误。最好不用可变的数据结构，除非你能防止它出错，但它很少能真正不出错。

例如：

```scala
trait Producer[T] {
 def fetchList: List[T]
}

// consumer side
someProducer.fetchList
```

问题	-	如果上面返回的`List`是可变对象，`Producer`接口会怎样？

一些可能的问题：
   
1. 如果这个`List`在另外一个线程上生成，就会有可见性和原子性的问题。你不知道会发生什么，除非看下生产者的实现  

2. 即使这个`List`是不变对象(也就是仍然可变，但一旦被赋值给Consumer就不能被修改)，你也不知道它是否可以被赋值给其他能修改它的其他Consumer, 所以不知如何处理它。

3. 即使对`List`的访问加同步，问题是 － 在哪个锁上同步？你能保证得到的锁顺序的正确性？锁是不能组合的。

现在明白了吧，一个公共API对外暴露了一个可变的数据结构是很麻烦的事情，会导致比手动管理内存更糟糕的事情。

```
译者：
	1. API尽可能用不变对象
	2.	未必所有的地方都适合用不变对象（尽信书不如无书）
```

### 2.3. 不要在循环或条件语句中修改var变量

大多java程序员转向scala时会做错，如：

```scala
var sum = 0
for (elem <- elements) {
  sum += elem.value
}
```

避免循环中修改循环外部的变量，推荐类似`foldLeft`的无副作用函数：

```scala
val sum = elements.foldLeft(0)((acc, e) => acc + e.value)
```

Or even better, know thy standard library and always prefer to
use the built-in functions - the more expressive you go, the less bugs
you'll have:

更好的做法，了解你自己的实际需要，用scala内建的函数。这样用的越多,错误越少。可以这样：

```scala
val sum = elements.map(_.value).sum
```

同样，不应使用条件分支更新部分返回结果，如：

```scala
def compute(x) = {
  var result = resultFrom(x)

  if(needToAddTwo) {
    result += 2
  }
  else {
    result += 1
  }

  result
}
```

推荐使用表达式和不变量。分支语句更直接返回不变量，这会让代码可读性会更好，减少犯错机会，这是好事情：

```scala
def computeResult(x) = {
  val r = resultFrom(x)
  if (needToAddTwo)
    r + 2
  else
    r + 1
}
```

要知道，就像前面讨论的`return`, 一旦条件判断分支太复杂，就是不雅代码的表现，那就需要重构。

### 2.4. 不应当定义没用到的`traits` SHOULD NOT define useless traits

There was this Java Best Practice that said "*program to an interface,
not to an implementation*", a best practice that has been cargo-cult-ed
to the point that people started defining completely useless
interfaces in their code. Generally speaking, that rule is healthy,
but it refers to the general engineering need of hiding implementation
details especially details of modifying state (encapsulation) and not
to slap interface declarations that often leak implementation details
anyway.

有个`面向接口编程，非面向实现`的最佳实践。人们非常崇拜这个规则，以至于为了满足这个规则，在编码中定义了无用的接口。通常来说，这个规则很好，但这个规则只是指的一般工程隐藏实现细节的需要，特别是修改内部状态而不影响那种无论如何都不能暴露实现细节的接口声明。

Defining traits is also a burden for readers of that code, because it
signals a need for polymorphism. Example:

定义的traits对阅读代码的人来说也是一个负担，因为traints表示这里需要多态。如：

```scala
trait PersonLike {
  def name: String
  def age: Int
}

case class Person(name: String, age: Int)
  extends PersonLike
```

Readers of this code might come to the conclusion that there are
instances in which overriding `PersonLike` is desirable. That couldn't
be further from the truth - `Person` is perfectly described by its
case class as a data-structure without behavior. In other words it
describes the shape of your data and if you need to override this
shape for some unknown reason, then this trait is badly defined
because it imposes the shape of your data and that's about the only
thing you can override. You can always come up with traits later, if
you're in need of polymorphism, after your needs evolve.



And if you're thinking that you may need to override the source of
this (as in to fetch the person's `name` from the DB on first access),
OMG don't do that!

Note that I'm not talking about algebraic data structures (i.e. sealed
traits that are signaling a closed set of choices - like `Option`).

Even in those cases in which you think the issue is clear-cut, it may
not be. Lets take this example:

```scala
trait DBService {
  def getAssets: Future[Seq[(AssetConfig, AssetPersistedState)]]

  def persistFlexValue(flex: FlexValue): Future[Unit]
}
```

This snippet is taken from real-wold code - we've got a `DBService`
that handles either queries or persistence in a database. Those two
methods are actually unrelated, so if you only need to fetch the
assets, why depend on things you don't need in components that require
DB interaction?

Lately my code looks a lot like this:

```scala
final class AssetsObservable
    (f: => Future[Seq[(AssetConfig, AssetPersistedState)]])
  extends Observable[AssetConfigEvent] {

  // ...
}

object AssetsObservable {
  // constructor
  def apply(db: DBService) = new AssetsObservable(db.fetchAssets)
}
```

See, I do not need to mock an entire `DBService` in order to test the
above.

### 2.5. case类中不要用var

case类是定义类的一个语法糖，case类中所有构造参数都是public和不可变的，并且它部分值是一致的，结构上恒等，一致的hashcode实现，并且编译器提供自动生成的apply/unapply函数。

如：

```scala
case class Sample(str: String, var number: Int)
```

上面的case类恰好打破了它的恒等性和hashcode操作。现在尝试用这个case类作为map的一个key

一般规则是，结构上的恒等性仅针对不变量，因为相等操作必须是确定的(不会因对象的历史而改变)。Case类针对的是严格的不变事物。如果想要可变量那就不要用case类。

Fogus在《The Joy of Clojure》中大致说过，或者Baker在他1993年的论文中说过：如果两个对象通过equal确定相同，并不能保证他们仅是瞬间相同。如果两个对象永远不相等，技术上来说他们决不会相等。

### 2.6. 不声明抽象的val/var成员变量

像下面这样在一个抽象类或者traints里声明抽象的var/val并不是好习惯

```scala
trait Foo {
 val value: String
}
```

推荐使用`def`声明抽象函数。

```scala
trait Foo {
 def value: String
}

// can then be overridden as anything, including val
class Bar(val value: String) extends Foo
```

强制限制这样做的原因是，一个`val`只能被`val`变量重写，`var`只能被`var`重写。在实现一个抽象成员时，唯一能自由选择的是`def`。`val`是在对象构造时被赋值的。例如：


```scala
trait Foo { val value: String }

trait Bar extends Foo { val uppercase = value.toUpperCase }

trait MyValue extends Foo { val value = "hello" }

// this triggers a NullPointerException
new Bar with MyValue

// this works
new MyValue with Bar
```

上面是一个jvm + scala在构造时关于执行命令顺序的典型例子。这是一个在scala理想化的做法和jvm的限制之间的陷阱。反转继承时的顺序，这是一个脆弱的容易出问题的解决办法。上面的例子不能通过使用`lazy val`覆盖`value`解决。一个更好的方式可以解决这个问题。

没有理由非要通过继承的方式初始化一个变量的值。`def`是通用的方式，所以`def`替代。

### 2.7. 不在验证用户输入或控制流中抛出异常

两个原因：

1.	 有多个退出点的结束程序不符合结构化编程原则，因此难以推断：随着栈循环解开会发生可怕的不可预知的副作用。
   
2. 异常不记录在函数的签名里 - Java试图用检查异常的概念修复这个在现实中人们容易忽略的严重问题。

Exceptions are useful for only one thing - signaling unexpected errors (bugs) up the stack, such that a supervisor can catch those errors and decide to do things, like log the errors, send notifications, restarting the guilty component, etc...

异常仅有个用处：在栈中递归向上标示出未捕获的错误，这样supervisor监督者可以捕捉并处理这些错误，发通知，重启错误的部分等.

合理引用[Functional Programming with Scala](http://www.manning.com/bjarnason/)第四章的内容。

### 2.8. 捕捉异常时不要捕捉Throwable

不要这样做：

```scala
try {
 something()
} catch {
 case ex: Throwable =>
   blaBla()
}
```

不捕捉`Throwable`的原因是一些极为重大异常应该直接让进程崩溃掉。例如jvm如果抛出内存溢出异常， 即使在catch中重新抛出异常也已经晚了。垃圾收集器可能会接管内存溢出的有僵死不可恢复状态的进程，并且冻结一切操作。这意味着外部的监督者(supervisitor)没有机会重启这个进程。


替代方式为:

```scala
import scala.util.control.NonFatal

try {
 something()
} catch {
 case NonFatal(ex) =>
   blaBla()
}
```

### 2.9. 不使用null

一定不要使用`null`. 推荐使用scala的`Option[T]`替代。为`null`的值会导致错误，因为编译器不能识别出可能为null的值。函数中可能为`null`的值并不会在定义中记录。所以避免这样做法：

You must avoid using `null`. Prefer Scala's `Option[T]` instead. Null
values are error prone, because the compiler cannot protect
you. Nullable values that happen in function definitions are not
documented in those definitions. So avoid doing this:

```scala
def hello(name: String) =
  if (name != null)
    println(s"Hello, $name")
  else
    println("Hello, anonymous")
```

第一步应该这样修改：

```scala
def hello(name: Option[String]) = {
  val n = name.getOrElse("anonymous")
  println(s"Hello, $n")
}
```

使用`Option[T]`时，编译器会强制你使用下面的方式处理可能为null的情况：

1. 通过提供缺升值，抛出异常等方式立刻处理
2. 或者向上传递`Option`结果给调用者

`Option`就像集合里的0或1元素，所以可以使用foreach遍历：

```scala
val name: Option[String] = ???

for (n <- name) {
  // executes only when the name is defined
  println(n)
}
```

组合多个options也很简单：

```scala
val name: Option[String] = ???
val age: Option[Int] = ???

for (n <- name; a <- age)
  println(s"Name: $n, age: $a")
```

既然`Option`可以被视为可迭代的对象，可以在集合中用`flatMap`避开`None`值：

```scala
val list = Seq(1,2,3,4,5,6)

list.flatMap(x => Some(x).filter(_ % 2 == 0))
// => 2,4,6
```

### 2.10. 不要用`Option.get`

可能习惯这样做：

```scala
val someValue: Option[Double] = ???

// ....
val result = someValue.get + 1
```

不要用`Option.get`, 因为当元素不存在时会报`NullPointerException`，这也不符合使用Option的初衷。

正确做法:

1. 用 `Option.getOrElse`
2. 用 `Option.fold`
3. 用模式匹配并明确处理`None`的分支情况
4. not taking the value out of its optional context

As an example for (4), not taking the value out of its context means
this:

```scala
val result = someValue.map(_ + 1)
```

### 2.11. 用Joda-Time 或者JSR-310代替Java的Date or Calendar

Java标注类库里的Date和Calend问题严重，因为：

1. 结果对象是可变量，时间相关的可变结果没有意义（你怎么看待到处使用`StringBuffer`还是`String`?）
2. 月份是从0开始的
3. `Date`没有记录时区信息，所以`Date`的值完全没用
4. 不能区分GMT和UTC
5. 年表示为2位数,而不是4

用 [Joda-Time](http://www.joda.org/joda-time/) - 如果切换到Java 8, 有个基于Joda-Time的新的标准实现
[JSR-310](http://www.threeten.org/) t

### 2.12. 不要用Any、AnyRef、isInstanceOf、asInstanceOf

如果没有充分的理由那就尽量避免使用`Any`、`AnyRef`或强制类型转换。scala是通过表达式类型系统传递值的语言，`Any`、`AnyRef`或强制类型转换的用法在表达式类型系统相当于一个洞，scala编译器不知道在这些地方如何帮助你。一般的，像下面的做法就不太好：

```scala
val json: Any = ???

if (json.isInstanceOf[String])
  doSomethingWithString(json.asInstanceOf[String])
else if (json.isInstanceOf[Map])
  doSomethingWithMap(json.asInstanceOf[Map])
else
  ???
```

Often we are using Any when doing deserialization. Instead of working with Any, think about the generic type you want and the set of sub-types you need, and come up with an Algebraic Data-Type:

在反序列化的时候会时常用`Any`。作为替代`Any`的方式，可以考虑你想要的范型和子类型，并且与数学相关的数据类型：

```scala
sealed trait JsValue

case class JsNumber(v: Double) extends JsValue
case class JsBool(v: Boolean) extends JsValue
case class JsString(v: String) extends JsValue
case class JsObject(map: Map[String, JsValue]) extends JsValue
case class JsArray(list: Seq[JsValue]) extends JsValue
case object JsNull extends JsValue
```

现在可以对JsValue做模式匹配，编译器会在缺失的有限的分支有帮助提示。
下面有缺失分支情况，scala编译器就会触发警告：

```scala
val json: JsValue = ???
json match {
  case JsString(v) => doSomethingWithString(v)
  case JsNumber(v) => doSomethingWithNumber(v)
  // ...
}
```

### 2.13. 必须序列化Unix时间戳或ISO 8601

Unix时间戳包含从`1970-01-01 00:00:00 UTC`以来的秒数或毫秒数, unix时间戳是一个适合跨平台的序列化格式。但它有缺点。ISO-8601是一个被大多数类库支持的序列化格式。

在MySQL等中存储没有时区的时间字段时固定使用UTC

### 2.14. 不要用魔数值	MUST NOT use magic values

尽管在其他语言里常使用`-1`表示特殊的输出，但在scala里有几个类型可以表达的更清晰。例如`Option`, `Either`, `Try`。
万一需要多个表示成功／失败的`boolean`，那就可以使用一个代数数据类型(algebraic data type)。

不要这样做:

```scala
val index = list.find(someTest).getOrElse(-1)
```

### 2.15. 不要用"var"作为共享状态

避免用"var"作为共享可变状态。如果这样做，那就最好对var变量做同步，结果就是var变的代码不易读。最好避免这样做。
万一真的需要可变的共享状态，可以用原子引用(atomic reference)并且在其中用不变量。 Also checkout
[Scala-STM](https://nbronson.github.io/scala-stm/).

正确做法：

```
class Something {
  private var cache = Map.empty[String, String]
}
```
如果不能避免变量，推荐这样：

```scala
import java.util.concurrent.atomic._

class Something {
  private val cache =
    new AtomicReference(Map.empty[String, String])
}
```
是的，原子引用意味着自旋锁，这会因为要求同步而增加开销。但这省却了以后许多头疼的事情。最好完全避免变化的对象。

### 2.16. Public的函数应该直接声明返回类型

推荐这样:

```scala
def someFunction(param1: T1, param2: T2): Result = {
  ???
}
```

不推荐这样写：

```scala
def someFunction(param1: T1, param2: T2) = {
  ???
}
```
是的，函数的类型推断非常强大和完备，但是对于公共(public)函数：

1. 不可以依靠IDE或者检查函数实现细节确定返回类型
2. Scala可能会推断出大多数特定的类型， 因为在scala里函数的返回类型是协变的，所以你事实上会得到一个上实际的丑陋的返回类型 

例如，看下面的返回类型是啥：

```scala
def sayHelloRunnable(name: String) = new Runnable {
  def sayIt() = println(s"Hello, $name")
  def run() = sayIt()
}
```

返回的是`Runnable`?
错， 是`Runnable{def sayIt(): Unit}`

这样的一个副作用会增加编译时间，因为`sayHelloRunnable`无论什么时候改变了实现，同时会改变函数签名，这样所有依赖这个函数的地方都需要重新编译。

### 2.17. 不应该把case类在其他类中定义

尽量不要在其他对象中定义内嵌case类，这样会让Java的序列化混乱。因为，当序列化一个靠近`this`指针的整个case类时，如果把他放到App对象中意味着对于每个case类的实例，你会序列化所有的对象。

case类具体的事情是:

1. 希望一个case类是immutable的
2. 希望一个case类容易被序列化

推荐扁平化的继承
