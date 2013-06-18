---
layout: post
title:  "模式匹配和复合函数"
next_link:  type-basics
prev_link:  collections
date:  2013-06-15 00:00:00
categories: jekyll update
---


课程包括

-  复合函数 Function Composition
  -  compose
  -  andThen
-  Curry vs 部分应用 Partial Application
-  部分函数 PartialFunctions
  -  range 和domain
  -  通过orElse 组合composition with orElse
-  case 语句是什么

##复合函数  Function Composition

让我们来看看这两个巧妙命名的函数：

{% highlight bash %}

scala> def f(s:String) = "f(" + s + ")"
f: (String)java.lang.String

scala> def f(s:String) = "g(" + s + ")"
g: (String)java.lang.String 

{% endhighlight %}

### compose

compose 产生了一个组合其它函数f(g(x)) 的新函数

{% highlight bash %}

scala> val fComposeG = f _ compose g _
fComposeG: (String) => java.lang.String = <function>

scala> fComposeG("yay")
res0: java.lang.String = f(g(yay))  

{% endhighlight %}

### andThen

andThen 和compose 很像，只不过首先调用然后在执行g(f(x))

{% highlight bash %}

scala> val fAndThenG = f _ andThen g _
fAndThenG: (String) => java.lang.String = <function>

scala> fAndThenG("yay")
res1: java.lang.String = g(f(yay))

{% endhighlight %}

##Currying 还是 部分应用

###case 语句

####好吧，只是介绍什么是case 语句

它是一个被叫做PartialFunction 的函数的子类。

####多个case 语句的集合是什么

它们是多个PartialFunction 组合在一起。

##理解PartialFunction

一般的函数任何定义类型的参数都起作用。换言之，如果函数定义为 (Int) => String ，那么会接受任何的Int 然后返回String。

一个PartialFunction 仅定义指定值的类型。一个Partial 函数 (Int) => String 不一定接受所有的Int。

isDefinedAt 是PartialFunction 的一个方法，它可以决定这个PartialFunction 是否接受指定的参数。

注意，PartialFunction 和我们之前讨论的部分应用函数无关。

也可以在这里Effective Scala 的 [PartialFunction](http://twitter.github.com/effectivescala/#Functional+programming-Partial+functions)里面找到更多的信息。 

{% highlight bash %}

scala> val one: PartialFunction[Int, String] = { case 1 => "one" }
one: PartialFunction[Int,String] = <function1>

scala> one.isDefinedAt(1)
res0: Boolean = true

scala> one.isDefinedAt(2)
res1: Boolean = false

{% endhighlight %}

你可以使用一个部分函数。

{% highlight bash %}

scala> one(1)
res2: String = one

{% endhighlight %}

PartialFunctions 可以通过其它东西组合起来，orElse，它能够反映出来PartialFunction 是不是通过提供的参定义。

{% highlight bash %}

scala> val two: PartialFunction[Int, String] = { case 2 => "two" }
two: PartialFunction[Int, String] = <function1>

scala> val three: PartialFunction[Int, String] = { case 3 => "three" }
three: PartialFunction[Int, String] = <function1>

scala> val wildcard: PartialFunction[Int, String] = { case _ => "something else" }
wildcard: PartialFunction[Int, String] = <function1>

scala> val partial = one orElse two orElse three orElse wildcard
partial: PartialFunction[Int, String] = <function1>

scala> partial(5)
res24: String = something else

scala> partial(3)
res25: String = three

scala> partial(2)
res26: String = two

scala> partial(1)
res27: String = one

scala> partial(0)
res28: String = something else

{% endhighlight %}

### case 之谜

上一节我们看到了一个奇怪的事情，我们看到了case 语句使用了函数的用法。

{% highlight bash %}

scala> case class PhoneExt(name: String, ext: Int)
defined class PhoneExt

scala> val extensions = List(PhoneExt("steve", 100), Phone("robey", 200))
extensions: List[PhoneExt] = List(PhoneExt(steve,100),PhoneExt(robey,200))

scala> extensions.filter { case PhoneExt(name, extension) => extension < 200 }
res0: List[PhoneExt] = List(PhoneExt(steve,100))

{% endhighlight %}

这是怎么回事？

filter 接受一个函数。在这里是一个判断函数(PhoneExt) => Boolean。

PartialFunction 是函数的子类型，所以filter 当然可以接受一个PartialFunction 啦！


