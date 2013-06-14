---
layout: post
title:  "基础二"
next_link:  collections
prev_link:  basics
date:  2013-06-09 00:00:00 
categories: jekyll update
---


课程包括

*  应用 apply
*  伴生类 objects
*  函数就是对象 Functions are Objects
*  包 packages
*  模式匹配 pattern matching
*  条件类 case classes
*  try-catch-finally
  
##  应用 apply

当class 或objects 没有main 的时候，apply 方法是一个很好的语法糖。

{% highlight bash %}

scala> class Foo {}
defined class Foo

scala> object FooMaker {
    |    def apply() = new Foo
    |  }
defined module FooMaker

scala> val newFoo = FooMaker()
newFoo: Foo = Foo@5b83f762

{% endhighlight %}

或者

{% highlight bash %}

scala> class Bar {
    |    def apply() = 0
    |  }
defined class Bar

scala> val bar = new Bar
bar: Bar = Bar@47711479

scala> bar()
res8: Int = 0

{% endhighlight %}

这里好像是我们的实例调用了一个方法，到后面我们还会看到更多。

##  伴生类 objects

Objects 用来保持class 的一个单例。经常用来做工厂。


{% highlight scala %}


object Timer {
    var count = 0

    def currentCount(): Long = {
        count += 1
        count
    }
}

{% endhighlight %}

那么怎样使用呢？

{% highlight bash %}

scala> Timer.currentCount()
res0: Long = 1

{% endhighlight %}

Classes 和 Objects 可以有相同的名字。object 叫做‘伴生类’。我们通常把伴生类当作工厂。这里有一个琐碎的例子仅仅是免去总是用new 来创建一个实例的麻烦。

{% highlight scala %}

class Bar(foo: String)

object Bar{
    def apply(foo: String) = new Bar(foo)
}

{% endhighlight %}

##  函数就是对象 Functions are Objects

在Scala 里，我们经常谈论面向对象的函数式编程语言。这是什么意思呢？究竟什么是函数式编程语言呢？

函数是一组特质。特殊的，函数接受一个作为Function1 特质的参数。这个特质定义了我们之前学到的apply() 语法糖，允许你向调用函数那样调用对象。


{% highlight bash %}

scala> object addOne extends Function1[Int, Int]{
    |    def apply(m: Int):Int = m + 1
    |  }
defined module addOne

scala> addOne(1)
res2: Int = 2

{% endhighlight %}

这里有一个传入22的Function1。为什么是22？这是一个随机的魔法数字。在我看来只有用22当作参数传入函数才能工作。（搞毛？）

apply 的语法糖帮助object 和函数式编程的二元性统一。你可以像函数那样传递classes ，同时函数只是一个classes 的幕后的实例。

这是不是意味着每一次你在class 里面定义一个方法，你实际上就获得了一个Function* 的实例呢？不，classes 里的方法就是方法。方法是在 Function* 实例repl 单独定义的。

Classes 也可以扩展函数，这些函数可以通过() 调用。

{% highlight bash %}

scala> class AddOne extends Function1[Int, Int] {
    |    def apply(m: Int):Int = m + 1
    |  }
defined class AddOne

scala> val plusOne = new AddOne()
plusOne: AddOne = <function1>

scala> plusOne(1)
res0: Int = 2

{% endhighlight %}

extends Function1[Int, Int] 的一个比较好的简写是extends(Int => Int)

{% highlight scala %}

class AddOne extends (Int => Int){
    def apply(m: Int): Int = m + 1
}

{% endhighlight %}

##  包 packages

你可以把你的代码组织到包里面。


{% highlight scala %}

package com.twitter.example

{% endhighlight %}

在文件的最顶部会声明你这个包里文件的所有。

值和函数不能在class 或 object 里面定义。Object 是组织动态函数不错的工具。

{% highlight scala %}

package com.twitter.example

object colorHolder {
    val BLUE = "Blue"
    val RED = "Red"
}

{% endhighlight %}

现在你可以直接访问里面的成员了。

{% highlight scala %}

println("the color is: "+com.twitter.example.colorHolder.BLUE)

{% endhighlight %}

注意当定义这个对象的时候scala repl 的表述：

{% highlight scala %}

scala> object colorHolder{
     |   val Blue = "Blue"
     |   val Red = "Red"
     | }
defined module colorHolder

{% endhighlight %}

这或许给了你一点小暗示，Scala 设计者经常把objects 设计成为Scala 模组系统的一部分。

##  模式匹配 pattern matching

这可是Scala 里面最有用的部分之一。

匹配值：

{% highlight scala %}

val times = 1

times match {
    case 1 => "one"
    case 2 => "two"
    case _ => "some other number"
}

{% endhighlight %}

通过向导匹配：

{% highlight scala %}

times match {
    case i if i == 1 => "one"
    case i if i == 2 => "two"
    case _ => "some other number" 
}

{% endhighlight %}

注意我们是怎么获得变量i里面的值的。

最后一个case 的_ 是一个通配符；它确保我们可以处理任何陈述。否则当你传入一个不匹配的数时会抛出一个运行时异常。我们以后会讨论它。

也可以看看Effective Scala 的观点[when to use pattern matching](http://twitter.github.com/effectivescala/#Functional programming-Pattern matching)以及[pattern matching formatting](http://twitter.github.com/effectivescala/#Formatting-Pattern matching)。A Tour of Scala 也有描述[Pattern Matching](http://www.scala-lang.org/node/120)。

###匹配类型

你也可以用match 匹配处理不同类型的不同值。


{% highlight scala %}

def bigger(o: Any): Any = {
    o match {
        case i: Int if i < 0 => i-1
        case i: Int => i + 1
        case d: Double if d < 0.0 => d - 0.1
        case d: Double => d + 0.1
        case text: String => text + "s"
    }
}

{% endhighlight %}

###匹配class 成员

还记得我们之前的 calculator吧。

让我们来通过类型来给它们分类。


{% highlight scala %}

def calcType(calc: Calculator) = calc match {
    case calc.brand == "hp" && calc.model == "20B" => "financial"
    case calc.brand == "hp" && calc.model == "48G" => "scientific"
    case calc.brand == "hp" && calc.model == "30B" => "business"
    case _ => "unknow"
}

{% endhighlight %}

哇塞，从前处理这类问题太痛苦了。感谢Scala 给我们提供了这些特别的好工具。

##  条件类 case classes

case class 对于方便的存储和匹配class 里面的内容很有用。你可以不用new 就可以构建它们。


{% highlight bash %}

scala> case class Calculator (brand:String, model: String)
defined class Calculator

scala> val hp20b = Calculator("hp", "20b")
hp20b: Calculator = Calculator(hp,20b)

{% endhighlight %}

case classes 自动的基于构造参数生成equality 和比较好的toString 方法。

{% highlight bash %}

scala> val hp20b = Calculator("hp","20b")
hp20b: Calculator = Calculator(hp,20b)

scala> val hp20B = Calculator("hp","20b")
hp20B:Calculator = Calculator(hp,20b)

scala> hp20b == hp20B
res6: Boolean = true

{% endhighlight %}

case classses 有和正常类一样的方法。

###### 含有模式匹配的case classes

case classes 是用模式匹配定义的。让我们来简化一下我们之前的calculator 分类器。

{% highlight scala %}

val hp20b = Calculator("hp","20B")
val hp30b = Calculator("hp","30B")

def calcType(calc: Calculator) = calc match {
    case Calculator("hp","20B") => "financial"
    case Calculator("hp","48G") => "scientific"
    case Calculator("hp","30B") => "business"
    case Calculator(ourBrand,ourModel) => "Calculator: %s is of unknown type".format(ourBrand, ourModel)
}

{% endhighlight %}

最后一个匹配的另一种写法是。

{% highlight scala %}

case Calculator(_, _) => "Calculator of unknown type"

{% endhighlight %}

或者我们只是让它更简单的而不用指定它是一个Calculator。

{% highlight scala %}

case _=> "Calculator of unknown type"

{% endhighlight %}

或者我们可以使用其他名字重新绑定：

{% highlight scala %}

case c@Calculator(_, _) => "Calculator: %s of unknown type".format(c)

{% endhighlight %}

##  try-catch-finally

异常在Scala 里通常是try-catch-finally 或使用模式匹配。


{% highlight scala %}

try{
    remoteCalculatorService.add(1,2)
}catch{
    case e: ServeletIsDownException => log.error(e, "the remote calculator servieces is unavailable. should have kept your trusty HP.")
}finally{
    remoteCalculatorService.close()
}

{% endhighlight %}

try 也可以是面向对象的表达式

{% highlight scala %}

val result: Int = try {
    remoteCalculatorService.add(1,2)
}catch{
    case e: SeverIsDownException = > {
        log.error(e, "the remote calculator service is unavailable. should have kept trsty HP.")
        0
    }
}finally{
    remoteCalculatorService.close()
}

{% endhighlight %}

这并不是什么出色的编程风格，就像其他Scala 代码一样，仅仅是一个try-catch-finally 捕获表达式里的异常。

最后将要在异常已经被处理后调用，这并不是表达式的一部分。
