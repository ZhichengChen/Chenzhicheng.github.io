---
layout: post
title:  "类型 & 多态基础"
next_link:  advanced-types
prev_link:  pattern-matching-and-functional-composition
date:  2013-06-19 00:00:00
categories: jekyll update
---

课程包括

*  什么是静态类型 What are static types?
*  Scala 里面的类型 Types in Scala
*  参量的多态 Parametric Polymorphism
*  类型推论：Hindley-Milner 还是 类型推论 Type inference: Hindley vs. type inference
*  变化量 Variance
*  限定 Bounds
*  定量 Quantification
  
##什么是静态类型，为什么说他们很有用？

Pierce 说过：“静态类型就是一个语句方法，它根据它们计算出来的的种类分类程序语句，来自动检查指定的缺少的错误行为。”


类型允许你表示函数 domain & codomains。比如，从数学角度来看，我们经常看到：

{% highlight bash %}

f: R -> N

{% endhighlight %}

它告诉我们函数“f” 是从是实数集合到自然书集合的一个映射。

理论上的，这就是具体的类型。类型系统给我们一些更有力的方式来表达这些集合。

给定了这些注释，编辑器现在可以静态的（在编译时间）确定程序是健全的。这就是说，如果值（在运行时）没有遵从程序强加的约束编译就会失败。

通俗的讲，类型检查仅仅保证不健全的程序不会被编译。但不能保证没一个健全的程序都呢给你被编译。

随着类型系统的表达式增加，我们可以写出更稳定的代码，因为它允许我们的程序甚至在运行之前（当然类型自己以bug 的模来计算）保证不变性。

注意在编译时所有的类型信息都被移除了。它不在需要了。这个叫做消磁（erasure）。

##Scala 里面的类型

Scala 的强类型系统允许非常丰富的表达式。一些主要的特性有：

* 参数的多态 parametric polymorphism 粗略的讲，通用的编程
* （本地）类型推断 (local)type inference 粗略的讲，为什么要写 val i: Int = 12 :Int 呢
* 存在的定量 existential quantification 粗略的讲，为一些未命名的类型定义一些东西
* 视图 view 我们下节课将会学习，粗略的讲，一个类型到另一个类型的值的可塑性

##参数的多态

多态用来写没有丰富的静态类型组合的一般性代码（为不同的类型值）。

比如，没有参数多态，一个普通的list 数据应该看起来这样（这看起来确实很像Java 的泛型）：

{% highlight bash %}

scala> 2 :: 1 :: "bar" :: "foo" :: Nil
res5: List[Any] = List(2, 1, bar, foo)

{% endhighlight %}

现在我们不能回复个体成员的任何信息。

{% highlight bash %}

scala> res5.head
res6: Any = 2

{% endhighlight %}

所以我们的应用会进入到一系列的投掷中（asInstanceOf\[\]）我们也可能缺少类型安全（因为它们都是动态的）。

多态解决了指定类型变量的问题。

{% highlight bash %}

scala> def drop1[A](l: List[A]) = l.tail
drop: [A](l: List[A])List[A]

scala> drop1(List(1,2,3))
res1: List[Int] = List(2, 3)

{% endhighlight %}

###Scala 有一级多态

粗略的讲，这意味着有一些你喜欢在Scala 里面表述的类型概念对于编辑器理解来说却太一般了。假设你有一些函数

{% highlight bash %}

def toList[A](a: A) = List(a)

{% endhighlight %}
你通常希望是这样的：

{% highlight bash %}

def foo[A, B](f: A => List[A],b: B) =f(b)

{% endhighlight %}

这是不会被编译通过的，因为所有的类型变量不的不固定在调用的位置。甚至如果你固定为type B。

{% highlight bash %}

def foo[A](f: A => List[A], i: Int) = f(i)

{% endhighlight %}

....你会得到一个类型不匹配

##类型推断

一个传统的反对静态类型的声音是它话费了太多的语句。Scala 通过类型推断减轻了这点。

在函数式编程里面的一个经典的类型推断的方法被Hindley-Milner 提出来，他首先服役于ML。

Scala 的类型推断系统工作起来不太一样，但是它的要义很简单：约束推断，尝试去统一类型。

在Scala 里举例如下：

{% highlight bash %}

scala> { x => x }
<console>:7 error: missing parameter type
       { x => x }

{% endhighlight %}

在OCaml 里，你可以：

{% highlight bash %}

# fun x -> x;;
- : 'a -> 'a = <fun>

{% endhighlight %}

在Scala 里所有的推断都是局部的。Scala 一次只考虑一个表达式。比如：

{% highlight bash %}

scala> def id[T](x: T) = x
id: [T](x: T)T

scala> val x = id(322)
x: Int = 322

scala> val x = id("hey")
x: java.lang.String = hey

scala> val x = id(Array(1,2,3,4))
x: Array[Int] = Array(1, 2, 3, 4)

{% endhighlight %}

类型现在保存了，Scala 编译器为我们推断出来了参数类型。注意我们怎样做不一定明确返回指定类型。

##变化

Scala 的类型系统不得不和多态来解释class 层级 。Class 层级允许子类型关系的表达式。当混合了OO  和多态时的一个中心的讨论问题是： 如果T' 是T 的一个子类，Cotainer\[T'\] 是Container\[T\] 的子类吗？变化调用允许你的表达式class 层级和部署类型的下列关系。

<table class="table table-bordered">
    <tr>
        <th>&nbsp;</th>
        <th>Meaning</th>
        <th>Scala notation</th>
    </tr>
    <tr>
        <td>covariant</td>
        <td>C\[T'\] 是C\[T\] 的子类</td>
        <td>\[+T\]</td>
    </tr>
    <tr>
        <td>contravariant</td>
        <td>C\[T\] 是C\[T'\] 的子类</td>
        <td>\[-T\]</td>
    </tr>
    <tr>
        <td>invariant</td>
        <td>C\[T\] 和C\[T'\] 没关系</td>
        <td>\[T\]</td>
    </tr>
</table>



子类型的关系意味着：给定类型T，如果T' 是一个子类型，你能替代他吗？


{% highlight bash %}

scala> class Covariant[+A]
defined calss Covariant

scala> val cv: Covariant[AnyRef] = new Covariant[String]
cv: Covariant[AnyRef] = Covariant@4035acf6

scala> val cv: Covariant[String] = new Covariant[AnyRef]
<console>:6: error: type mismatch;
 found   : Covariant[AnyRef]
 required: Covariant[String]
       val cv: Covariant[String] = new Covariant[AnyRef]

{% endhighlight %}

{% highlight bash %}

scala> class Contravariant[-A]
defined class Contravariant

scala> val cv: Contravariant[String] = new Contravariant[AnyRef]
cv: Contravariant[AnyRef] = Contravariant@49fa7ba

scala> val fail: Contravariant[AnyRef] = new Contravariant[String]
<console>:6: error: type mismatch;
 found   : Contravariant[String]
 required: Contravariant[AnyRef]
       val fail: Contravariant[AnyRef]  = new Contravariant[String]
                                     ^
{% endhighlight %}

Contravariance 看起来有点奇怪，什么时候用它？有点让人吃惊！

{% highlight bash %}

trait Function1 [-T1, +R] extends AnyRef
 
{% endhighlight %}

如果你从视图替代物的观点来考虑，它很重要。让我们首先来简单的定义一个calss 层级：

{% highlight bash %}

scala> class Animal { val sound = "rustle" }
defined class Animal

scala> class Bird extends Animal { override val sound = "call" }
defined class Bird

scala> class Chicken extend Bird { override val sound = "cluck" }
defined class Chicken

{% endhighlight %}

假设你需要一个接受Bird 参数的函数：

{% highlight bash %}

scala> val getTweet: (Bird => String) = // TODO

{% endhighlight %}

标准的animal 库有一个做你想做的函数，不同的是它接受一个Animal 参数。在大多数情形，如果你说，“我需要一个～～我有一个子类～～”，你很好，但是函数参数是contravariant。如果你需要一个接受Bird 的函数，你就会有一个接受Chicken 的函数，函数会选择“Duck”。但是函数也接受Animal ：

{% highlight bash %}

scala> val getTweet: (Bird => String) = ((a: Animal) => a.sound )
getTweet: Bird => String = <function1>

{% endhighlight %}

一个返回covariant 的类型值的函数。如果你需要一个返回Bird 的函数，但是有一个返回Chicken 的函数，非常好：

{% highlight bash %}

scala> val hatch: (() => Bird) =(() => new Chicken )
hatch: () => Bird = <function0>

{% endhighlight %}

##界限

Scala 允许你使用bounds 来限制多态变量。bounds 表达子类型的关系。

{% highlight bash %}

scala> def cacophony[T](things: Seq[T]) = things map (_.sound)
<console>:7:error:value sound is not a member of type parameter T
       def cacophony[T](things: Seq[T]) = things map (_.sound)
                                                        ^

scala> def biophony[T <: Animal](things: Seq[T]) = things map (_.sound)
biophony: [T <: Animal](things: Seq[T])Seq[java.lang.String]

scala> biophony(Seq(new Chicken, new Bird))
res5: Seq[java.lang.String] = List(cluck,call)

{% endhighlight %}

低等级的bounds 也被支持，它们和contravariance 和clever covariance迟早会有用。List\[+T\] 是covariant; 一个Birds 列表是一个Animals 列表。List 定义了一个操作::(elem T)返回一个含有elem 的新List。新Lift 有相同的原始类型。

{% highlight bash %}

scala> val flock = List(new Bird, new Bird)
flock: List[Bird] = List(Bird@7e1ec70e, Bird@169ea8d2)

scala> new Chicken :: flock
res53: List[Bird] = List(Chicken@56fbda05, Bird@7e1ec70e,Bird@169ea8d2)

{% endhighlight %}

List 还定义了::\[B >: T\](x: B)，它返回List\[B\]。注意B >: T。指定了类型B 是T的超类。这让当考虑一个Animal 到List\[Bird\] 时我们使用正确的东西：

{% highlight bash %}

scala> new Animal :: flock
res59: List[Animal] = List[Animal@11f8d3a8,Bird@7e1ec70e,Bird@169ea8d2]

{% endhighlight %}

注意返回的类型是Animal。

##定量

有些时候你不关心类型变量的名字，比如

{% highlight bash %}

scala> def count[A](l: List[A]) = l.size
count: [A](List[A])Int

{% endhighlight %}

作为替代你可以使用wildcards:

{% highlight bash %}

scala> def count(l: List[_]) = l.size
count: (List[_])Int

{% endhighlight %}

也可以简写为：

{% highlight bash %}

scala> def count(l: List[T forSome { type T }]) = l.size
count: (List[T forSome { type T} ])Int

{% endhighlight %}

注意，它也可以耍花招：

{% highlight bash %}

scala> def drop(l: List[_]) = l.tail
drop1: (List[_])List[Any]

{% endhighlight %}

我们突然失去了类型信息！来看看继续发生什么，恢复到笨手笨脚的语法：

{% highlight bash %}

scala> def drop1(l: List[T forSome { type T }]) = l.tail
drop1:(List[T forSome { type T }])List[T forSome { type T}]

{% endhighlight %}

我们没有看到关于T 的任何东西，因为类型不允许。

你也可能应用bounds 到wildcard 类型变量上：

{% highlight bash %}

scala> def hashcode(l: Seq[_ <: AnyRef]) = l map (_.hashCode)
hashcodes: (Seq[_ <: AnyRef])Seq[Int]

scala> hashcodes(Seq(1,2,3))
<console>:7: error: type mismatch;
 found   : Int(1)
 required: ANyRef
 Note: primitive types are not implicitle converted to AnyRef,
 You and safely force boxing by casting x.asInstanceOf[AnyRef],
        hashcodes(Seq(1,2,3))
                      ^

scala> hashcodes(Seq("one","two","three"))
res1: Seq[Int] = List(110182, 115276, 110339486)

{% endhighlight %}

也可以看看[Existential types in Scala by D. R. Maclver](http://www.drmaciver.com/2008/03/existential-types-in-scala/)


