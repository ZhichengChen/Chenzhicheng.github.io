---
layout: post
title:  "集合量"
next_link:  pattern-matching-and-functional-composition
prev_link:  basics-continued
date:  2013-06-10 00:00:00
categories: jekyll update
---


课程包括

*  基本数据结构 Basic Data Structures
   *  列表 List  
   *  集合 Sets
   *  元组 Tuple
   *  映射 Maps
*  函数式选择器 Functional Combinators
   *  映射 Maps
   *  foreach
   *  过滤器 filter
   *  zip 
   *  partition
   *  find
   *  drop 和 dropWhile
   *  foldRight 和 foldLeft
   *  flatten
   *  flatMap
   *  生成函数式选择器 Generalized functional combinators
   *  映射吗 Map?
  
##  基本数据结构 Basic Data Structures

Scala 提供一些很棒的集合量。

也可以看看 Effective Scala 里面的观点[collections](http://twitter.github.com/effectivescala/#Collections)

###  列表 List  

{% highlight bash %}

scala> val numbers = List(1,2,3,4)
numbers: List[Int] = List(1,2,3,4)

{% endhighlight %}

###  集合 Sets

集合里面没有重复项

{% highlight bash %}

scala> Set(1, 1, 2)
res0:scala.collection.immutable.Set[Int] = Set(1, 2)

{% endhighlight %}

###  元组 Tuple

元组不通过定义类的形式简单的把逻辑集合量组合在一起。

{% highlight bash %}

scala> val hostPort = ("localhost", 80)
hostPort: (String, Int) = (localhost, 80)

{% endhighlight %}

和case classes 不一样，它们没有命名访问函数，它们同通过基于1而不是0开始的位置命名来访问。

{% highlight bash %}

scala> hostPort._1
res0: String = localhost

scala> hostPort._2
res1: Int = 80

{% endhighlight %}

元组可以很好的适应模式匹配。

{% highlight scala %}

hostPort match {
    case ("localhost", port) => ...
    case (host, port) => ...
}

{% endhighlight %}

元组有一个特殊的调味剂，简单的通过两个值: -> 来定义

{% highlight bash %}

scala> 1 -> 2
res0: (Int, Int) = (1,2)

{% endhighlight %}

在Effective Scala 里面有更多关于[描述绑定](http://twitter.github.com/effectivescala/#Functional programming-Destructuring bindings)的（解包元组）内容。

###  映射 Maps

它可以存储基本类型数据。

{% highlight scala %}

Map(1 -> 2)
Map("foo" -> "bar")

{% endhighlight %}

这个语法看起来很特殊，不过我们之前讨论元组的时候就曾经用过 -> 来创建元组。

Map() 也可以使用我们之前在第一课介绍的可变参数语法：Map(1 -> "one", 2 -> "two")，可以展开成为Map((1, "one"),(2,"two")),其中第一个元素是key，第二个是对应映射的值。

映射可以包含映射甚至是做为值的函数。

{% highlight scala %}

Map(1 -> Map("foo" -> "bar"))

{% endhighlight %}

{% highlight scala %}

Map("timesTwo" -> { timesTwo(_) })

{% endhighlight %}

###选项 Option

选项是一个可能包含或可能不包含一些东西的容器。

选项的最基本的接口看起来这样：


{% highlight scala %}

trait Option[T] { def isDefined: Boolean def get: T def getOrElse(t: T): T }

{% endhighlight %}

选项自己是通用类，它可以包含两个子类：Some[T] 或者 None。

让我们来看看使用选项的例子：

Map.get 使用Option 来返回它的类型。选项告诉你方法可能并不返回你想要的东西。

{% highlight bash %}

scala> val numbers = Map(1 -> "one", 2 -> "two")
numbers: scala.collection.immutable.Map[Int,String] = Map((1,one),(2,two))

scala> numbers.get(2)
res0: Option[java.lang.String] = Some(two)

scala> numbers.get(3)
res1: Option[java.lang.String] = None

{% endhighlight %}

现在我们的数据看起来被Option 捕获了。我们怎样使用它呢。

出于本能的可能有一个isDefined 方法可做为附加条件的。

//我们想要给一个数乘以2,否则就返回0
val result = if (res1.isDefined) { res1.get * 2 } else { 0 }

我们建议你使用getOrElse 或者模式匹配来处理结果。

getOrElse 让你方便的定义默认值。

val = result = res1.getOrElse(0) * 2

选项可以很自然的应用于模式匹配：

val result = res1 match { case Some(n) => n * 2 case None => 0 }

这里有Effective Scala 里面关于[选项](http://twitter.github.com/effectivescala/#Functional+programming-Options)的观点。 

##  函数式选择器 Functional Combinators

List(1, 2, 3) map squared 给列表应用squared 函数，返回一个新的list，可能是List（1, 4, 9）。我们把这个操作叫做类map 选择器。（如果你想要更好的定义，你应该会喜欢Stackoverflow 上面的[Explanation of combinators](http://stackoverflow.com/questions/7533837/explanation-of-combinators-for-the-working-man)）。它们大多数时候应用于基本数据结构。

###  映射 map

通过把没一个函数放到list 里面来求值的函数，返回一个含有相同元素数的list。

{% highlight bash %}

scala> numbers.map((i: Int) => i * 2)
res0: List[Int] = List(2, 4, 6, 8)

{% endhighlight %}

或者部分传入求值函数。

scala> def timesTwo(i: Int):Int = i * 2
timesTwo: (i: Int)Int

scala> numbers.map(timesTwo _)
res0: List[Int] = List(2, 4, 6, 8)

###  foreach

foreach 和map 类似，不过它不返回值，foreach 仅适用于它的副作用。

{% highlight bash %}

scala> numbers.foreach((i: Int) => i * 2)

{% endhighlight %}

不返回值。

你可以试着把返回的值存储起来不过它会是Unit 类型(就像，void)

{% highlight bash %}

scala> val doubled = numbers.foreach((i: Int) => i * 2)
doubled: Unit = ()

{% endhighlight %}

###  过滤器 filter

根据你传入函数计算为false 的元素删除。返回Boolean 的函数经常被称为谓词函数（predicate functions）。

{% highlight bash %}

scala> number.filter((i: Int) => i % 2 == 0)
res0: List[Int] = List(2,4)

{% endhighlight %}

{% highlight bash %}

scala> def isEven(i: Int): Boolean = i % 2 == 0
isEven: (i: Int)Boolean

scala> numbers.filter(isEven _)
res2: List[Int] = List(2, 4)

{% endhighlight %}

###  zip 

zip 把两个lists 组合成为一个list。


{% highlight bash %}

scala> List(1, 2, 3).zip(List("a","b","c"))
res0: List[(Int, String)] = List((1,a), (2,b), (3,c))

{% endhighlight %}

###  partition

partition 基于一个谓词函数分割list。

{% highlight bash %}

scala> val numbers = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
scala> numbers.partitin(_ %2 == 0)
res0: (List[Int], List(2, 4, 6, 8, 10),List(1, 3, 5, 7, 9))

{% endhighlight %}

###  find

find 返回集合里面匹配谓词函数的第一个元素。

{% highlight bash %}

scala> numbers.find((i: Int) => i > 5)
res0: Option[Int] = Some(6)

{% endhighlight %}

###  drop 和 dropWhile

drap 删除第i 个元素

{% highlight bash %}

scala> numbers.drop(5)
res0: List[Int] = List(6, 7, 8, 9, 10)

{% endhighlight %}

dropWhile 删除匹配谓词函数的第一个元素。比如，如果dropWhile 从list numbers 里面删除奇数元素，1 会被删除（而不是被2 隔离了的3）。

{% highlight bash %}

scala> numbers.dropWhile(_ %2 != 0)
res0: List[Int] = List(2, 3, 4, 5, 6, 7, 8, 9, 10)

{% endhighlight %}

###  foldRight 和 foldLeft

{% highlight bash %}

scala> numbers.foldLeft(0)((m: Int,n: Int) => m + n)
res0: Int = 55

{% endhighlight %}

0 是开始值，（记住numbers 是一个List[Int]）,m 是一个累加器。

这样看起来更形象：

{% highlight bash %}

scala> numbers.foldLeft(0) { (m: Int, n: Int) => println("m: "+ m + " n: " +n); m + n }
m: 0 n: 1
m: 1 n: 2
m: 3 n: 3
m: 6 n: 4
m: 10 n: 5
m: 15 n: 6
m: 21 n: 7
m: 28 n: 8
m: 36 n: 8
m: 45 n: 9
res0: Int = 55

{% endhighlight %}

foldRight 看起来和foldLeft 一样，只不过它是逆向运行的

{% highlight bash %}

scala> numbers.foldRight(0) { (m: Int, n: Int) => println(" m: " + m + " n: " + n); m + n}
m: 10 n: 0
m: 9 n:10
m: 8 n: 19
m: 7 n: 27
m: 6 n: 34
m: 5 n: 40
m: 4 n: 45
m: 3 n: 49
m: 2 n: 52
m: 1 n: 54
res0: Int = 55

{% endhighlight %}

###  flatten

flatten 折叠一个级别的嵌套结构。

{% highlight bash %}

scala> List(List(1, 2),List(3, 4)).flatten
res0: List[Int] = List(1, 2, 3, 4)

{% endhighlight %}

###  flatMap

flatMap 是一个人把mapping 和flatting 联接起来的一个很常用的联接符。flatMap 接受一个嵌套函数列表然后把他们组合起来。

{% highlight bash %}

scala> val nestedNumbers = List(List(1, 2), List(3, 4))
nestedNumbers: List[List[Int]] = List(List(1, 2), List(3, 4))

scala> nestedNumbers.flatMap(x => x.map(_ * 2))
res0: List[Int] = List(2, 4, 6, 8)

{% endhighlight %}

可以考虑把它作为mapping 和flatterning 的简写：

{% highlight bash %}

scala> nestedNumbers.map((x: List[Int]) => x.map(_ * 2)).faltten
res1: List[Int] = List(2, 4, 6, 8)

{% endhighlight %}

这个例子调用map 然后又调用flatten 做为这两个函数很自然的的一个连接。

Effective Scala 里面有关于[flatMap](http://twitter.github.com/effectivescala/#Functional+programming-`flatMap`) 的信息。

###  生成函数式选择器 Generalized functional combinators

现在我们学习了集合量的混合函数。

我们接下来的要做的是写出我们自己的函数式选择器。

有趣的是，下面展示的每一个函数式选择器都被写在fold 的顶部。让我们看看例子。

{% highlight scala %}

def ourMap(numbers: List[Int], fn: Int => Int): List[Int] = {
    numbers.foldRight(List[Int]()) { x: Int, xs: List[Int]) =>
        fn(x) :: xs
    }
}

scala> ourMap(numbers, timesTwo(_))
res0: List[Int] = List(2, 4, 6, 8, 10, 12, 14, 16, 18, 20)

{% endhighlight %}

为什么是List[Int] 呢？Scala 还没有聪明到明白你想要一个空的Ints 的 list然后在来累积它。

所有的函数式选择器都是用Maps 来做例子。Map 可以被当成是list 的搭档，所以你写的函数在一组键值对上起作用。

{% highlight bash %}

scala> val extensions = Map("steve" -> 100, "job" -> 201)
extensions: scala.collection.immutable.Map[String, Int] = Map((steve,100),(bob,101),(joe,201))

{% endhighlight %}

现在来以电话扩展低于200 为入口过滤。

{% highlight bash %}

scala> extensions.filter((namePhone: (String, Int)) => namePhone._2 < 200)
res0: scala.collection.immutable.Map[String,Int] = Map((steve,100), (bob,101))

{% endhighlight %}

因为它返回一个元组，你可以通过位置访问函数来取出键和值，很讨厌。

幸运的是，我么也可以用模式匹配来更好的取出键和值。

{% highlight bash %}

scala> extensions.filter({case (name, extension) => extension < 200})
res0: scala.collection.immutable.Map[String,Int] = Map((steve,100), (bob,101))

{% endhighlight %}

为什么这样也行呢？为什么可以传入部分模式匹配呢？

下节课再说～

