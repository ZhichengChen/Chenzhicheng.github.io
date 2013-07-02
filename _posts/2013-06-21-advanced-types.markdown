---
layout: post
title:  "高级类型"
next_link:  sbt
prev_link:  type-basics
date:  2013-06-21 00:00:00
categories: jekyll update
---


课程包括

*  视图边界（“类型 classes”）View bounds ("type classes")
*  其它类型边界 Other Type Bounds
*  高级类型 & 转换多态 Higher-kinded types & ad-hoc polymorphism
*  F-边界多态 / 递归类型 F-bounded polymorphism / recursive types
*  结构类型 Structural types
*  抽象类型成员 Abstract types members
*  类型清除 & 清单 Type erasures & manifests
*  Case 学习： Finagle Case study: Finagle
  
## 视图边界（“类型 classes”）

有时候你不需要指定一个类型是不是另一个的子类/同类/超类，因为你可以用conversions 来欺骗它。一个视图边界指定的类型可以被其它的类看到。这对需要读取一个类却不需要修改一个类的操作来说很有意义。

Implicit 函数允许自动conversion。更精确的说，当它能帮助明确类型推断时允许请求式函数应用。比如：

{% highlight bash %}

scala> implicit def strToInt(x: String) = x.toInt
strToInt: (x: String)Int

scala> "123"
res0: java.lang.String = 123

scala> val y: Int = "123"
y: Int = 123

scala> math.max("123",111)
res1: Int = 123

{% endhighlight %}

视图边界，就像类型边界一样，要求一个函数存在给定的类型。你可以通过 <% 指定一个类型边界，比如，

{% highlight bash %}

scala> class Container[A <% Int] { def addIt(x: A) = 123 + x}
defined class Container

{% endhighlight %}

这就是说A 有一个可见的Int。让我们来试一下。

{% highlight bash %}

scala> (new Container[String]).addIt("123")
res11: Int = 246

scala> (new Container[Int]).addIt(123)
res12: Int = 246

scala> (new Container[Float]).addIt(123.2F)
<console>:8: error: could not find implicit value for evidence parameter of type(Float) => Int
       (new Container[Float]).addIt(123.2)
        ^

{% endhighlight %}

## 其它类型边界

方法可以通过implicit 参数生效更多复杂的类型参数。比如，Lift 支持对数字进行sum 当然其它类型不可以。唉，Scala 的数字类型并没有调用超类共享，因此我们不能仅仅说T<: Number。相反，为了让它可以工作，Scala 的数学类库[为合适的类型定义了一个ieimplicit 的Numberic[T]](http://www.azavea.com/blogs/labs/2011/06/scalas-numeric-type-class-pt-1/) 。然后List 使用它来定义：

{% highlight scala %}

sum[B >: A](implicit num: Numeric[B]):B

{% endhighlight %}

如果你调用List(1,2).sum() ，你不需要传入一个 num 参数；它是implicitly。但是如果你调用List("whoop").sum()，它会抱怨并没有num。

方法可能会为没有奇怪的含有Numeric 类的类型请求一些特殊的“证据”。相反，你可以用这些类型关系运算符。

A =:= B A 一定等于B

A <:< B A 一定是B 的子类型

A <%< B A 一定是对B 可见的

{% highlight bash %}

scala> class Container[A](value: A) { def addIt(implicit evidence: A =:= Int) = 123 +value }
defined class Container

scala> (new Container(123)).addIt
res11: Int = 246

scala> (new Container("123")).addIt
<console>:10: error: could not find implicit value for parameter evidence: =:=[java.lang.String.Int]

{% endhighlight %}

相似的。给我们上一个 implicit，我们可以放松可视性的约束：

{% highlight bash %}

scala> class Container [A] (value: A) { def addIt(implicit evidence: A <%< Int) = 123 + value }
defined class Container

scala> (new Container("123")).addIt
res15: Int = 246

{% endhighlight %}

###通过views 生成程序

在Scala 标准类库里，views 主要用来通过集合量来执行生成函数。比如，min 函数（在 Seq[] 里面的），使用了这个技术：

{% highlight scala %}

def min[B >: A](implicit cmp: Ordering[B]: A = {
    if (is Empty)
        throw new UnsupportedOperationException("empty.min")

    reduceLeft((x,y) => if(cmp.ltep(x,y)) x else y)
})

{% endhighlight %}

这样做的好处主要是：

* 集合量里面的项不需要执行Ordered，但是Ordered 仍然用来静态类型检查。
* 你可以定义你自己的排序而不需要任何额外的类库支持。

{% highlight bash %}

scala> list(1,2,3,4).min
res0: Int = 1

scala> list(1,2,3,4).min(new Ordering[Int] { def compare(a: Int, b: Int) = b compare a})
res3: Int = 4

{% endhighlight %}

作为旁注，在标准类库里面有views 来吧Ordered 转换成为Ordering（反之亦然）。

{% highlight scala %}

trait LowPriorityOrderingImplicits{
    implicit def ordered[A <: Ordered[A]]: Ordering[A] = new Ordering[A]{
        def compare(x: A,y: A) = x.compare(y)
    }
}

{% endhighlight %}

###Context bounds & implicitly[]

Scala 2.8 引入了一个传递&接受implicit 参数的多线程的简写。

{% highlight bash %}

scala> def foo[A](implicit x: Ordered[A]) {}
foo: [A](implicit x: Ordered[A])Unti

scala> def foo[A: Ordered] {}
foo: [A](implicit evidence$1: Ordered[A])Unit

{% endhighlight %}

Implicit 值可能通过 implicitly 访问

{% highlight bash %}

scala> implicitly[Ordering[Int]]
res37: Ordering[Int] = scala.math.Ordering$Int$@3a9291cf

{% endhighlight %}

##更高等级的类型 & ad-hoc 多态

Scala 可以通过“高级“类型来抽象。比如，假如你需要使用一些包含一些数据类型的类型。你可以定义一个Container 接口，它可能借助于一些容器类型，Option、List 等等。你想给在这些容器里的值定义一个接口却不固定它们的类型。

这是function currying 的一个特例。比如，”一元类型“ 像List[A] 这样构造，这意味着我们要创建一个混合类型（就像uncurried 函数需要被提供一个被调用的参数列表）我们不得不满足一”级“类型变量，一个高级类型则需要更多。

{% highlight bash %}

scala> trait Container[M[_]] { def put[A](x: A): M[A];(m: M[A]): A }

scala> val container = new Container[List] {def put[A](x: A) = List(x): def get[A]} = m.head }
container: java.lang.Object with Container[List] = $anon$7c8e3f75

scala> container.put("hey")
res24: List[java.lang.String] = List(hey)

scala> container.put(123)
res25: List[Int] = List(123)

{% endhighlight %}

注意Container 是一个参数或类型（”容器类型“）里面的多态。

如果我们使用含有implicits 的容器组合，我们获得了一个”ad-hoc“ 多态，通过容器写一般函数的能力。

{% highlight bash %}

scala> trait Container[M[_]] { def put[A](x: A): M[A]: def get[A](m: M[A]): A }

scala> implicit val listContainer = new Container[List] { def put[A](x: A) = List(x): def get[A](m: List[A]) = m.head }

scala> implicit val optionContainer = new Container[Some] { def put[A](x: A) = Some(x): def get[A](m: Some[A]) = m.get }

scala> def tuleize[M[_]]:Container,A,B](fst: M[A],snd: M[B]) = {
     | val c = implicitly[Container[M]]
     | c.put(c.get(fsc),c.get(snd))
     | }
tupleize: [M[_],A,B](fst: M[A],snd: M[B])(implicit evidence$1: Container[M]M(A, B))

scala> tupleize(Some(1), Some(2))
res33: Some[(Int, Int)] = Some((1,2))

scala> tupleize(List(1), List(2))
res34: List[(Int,Int)] = List((1,2))

{% endhighlight %}

##F-bounded 多态

通常在一个（通用）接口里面访问一个混合子类是很有必要的。比如，想象你有一些同有的接口，但是可以和这个接口的特殊子类做比较。

{% highlight scala %}

trait Container extends Ordered[Container]

{% endhighlight %}

不管怎样，现在必须是比较的方法：

{% highlight scala %}

def compare(that: Container): Int

{% endhighlight %}

所以我们不能访问混合子类型，比如：

{% highlight scala %}

class MeContainer extends Container{
    def compare(that: MyContainer): Int
}

{% endhighlight %}

编译失败，由于我们为Container 指定了Ordered ，而不是特殊的子类。

为了解决这点，我们使用了F-bounded 多态。

{% highlight scala %}

trait Container[A <: Container[A] extends Ordered[A]

{% endhighlight %}

好奇怪的类型。但是注意Ordered 是怎样作为A 的参数的，也就是它自己是Container[A].

所以，现在

{% highlight scala %}

class MyContainer extends Container[MyContainer] {
    def compare(that: MyContainer) = 0
}

{% endhighlight %}

它们现在被排序了：

{% highlight bash %}

scala> List(new MyContainer, new MyContainer, new MyContainer)
res3: List[MyContainer] = List(MyContainer@30f02a6d,MyContainer@67717334,MyContainer@49428ffa)

scala> List(new MyContainer, new MyContainer, new MyContainer).min
res4: MyContainer = MyContainer@33dfeb30

{% endhighlight %}

给定它们都是Container[] 的子类型，我们可以定义其它的子类&创建一个混合Container[_]列表：

{% highlight bash %}

scala> class YourContainer extends Container[YourContainer] { def compare(that: YourContainer) = 0}
defined class YourContainer

scala> List(new MyContainer, new MyContainer, new MyContainer,new YourContainer)
res2: List[Container[_ >: YourContainer with MyContainer <: Container[_ >: YourContainer with MyContainer <: ScalaObject]]] = List(MyContainer@3be5d207, MyContainer@6d3fe849, MyContainer@7eab48a7, YourContainer@1f2f0ce9)

{% endhighlight %}

注意结果类型是怎样通过YourContainer with MyContainer lower-bound。这是因为类型推断。有趣的是这个类型甚至没有任何意义，它仅仅是为统一的列表类型提供了一个逻辑最大的低下限。现在我们来试试Ordered 会怎样呢？

{% highlight scala %}

(new MyContainer, new MyContainer, new MyContainer, new YourContainer).min
<console>:9 error: could not find implicit value for parameter cmp:
  Ordering[Container[_ >: YourContainer with MyContainer <: Container[_ >: YourContainer with MyContainer <: ScalaObject]]]

{% endhighlight %}

注意 Ordered[] 存在于统一的类型里。太糟糕了。

##Structural  types

Scala 有对Structural types 的支持 -- 类型需要被接口结构表达而不是混合类型。

{% highlight bash %}

scala> def foo(x: { def get: Int }) = 123 + x.get
foo: (x: AnyRef{def get: Int})Int

scala> foo(new { def get = 10 })
res0: Int = 133

{% endhighlight %}

这在很多情形里确实很好，但是却是用了反射执行，因此有一些性能损失（so be performance-aware）!

##抽象类型成员

在接口里，你可以抽象类型成员。

{% highlight bash %}

scala> trait Foo { type A: val x: A; def getX: A = x }
defined trait Foo

scala> (new Foo { type A = Int; val x = 123 }).getX
res3: Int = 123

scala> (new Foo { type A = String: val x = "hey" }).getX
res4: java.lang.String = hey

{% endhighlight %}

在处理依赖注射等时这通常是一个有用的把戏。

你可以使用hash 操作符来指定一个抽象类型：

{% highlight bash %}

scala> trait Foo[M[_]] { type t[A] = M[A] }
defined trait Foo

scala> val x: Foo[List]#t[Int] = List(1)
x: List[Int] = List(1)

{% endhighlight %}

##类型涂改&体现

众所周知，由于类型涂改类型信息在编译时会丢失。Scala 的特性Manifest，允许我们可选的覆盖类型信息。Manifests 是提供的一个implicit 值，在编译需要时生成。

{% highlight bash %}

scala> class MakeFoo[A](implicit manifest: Manifest[A]) {def make: A = manifest.erasure.newInstance.asInstanceOf[A] }

scala> (new MakeFoo[String]).make
res10: String = ""

{% endhighlight %}

##Case 学习:Fingle

见:[此处](https://github.com/twitter/finagle)

{% highlight scala %}

it Service[-Req, +Rep] extends (Req => Future[Rep])

trait Filter[-ReqIn, +RepOut, +ReqOut, -RepIn]
  extends ((ReqIn, Service[ReqOut, RepIn]) => Future[RepOut])
{
  def andThen[Req2, Rep2](next: Filter[ReqOut, RepIn, Req2, Rep2]) =
    new Filter[ReqIn, RepOut, Req2, Rep2] {
      def apply(request: ReqIn, service: Service[Req2, Rep2]) = {
        Filter.this.apply(request, new Service[ReqOut, RepIn] {
          def apply(request: ReqOut): Future[RepIn] = next(request, service)
          override def release() = service.release()
          override def isAvailable = service.isAvailable
        })
      }
    }
    
  def andThen(service: Service[ReqOut, RepIn]) = new Service[ReqIn, RepOut] {
    private[this] val refcounted = new RefcountedService(service)

    def apply(request: ReqIn) = Filter.this.apply(request, refcounted)
    override def release() = refcounted.release()
    override def isAvailable = refcounted.isAvailable
  }    
}

{% endhighlight %}

服务可能包含的认证请求过滤。

{% highlight scala %}

it RequestWithCredentials extends Request {
  def credentials: Credentials
}

class CredentialsFilter(credentialsParser: CredentialsParser)
  extends Filter[Request, Response, RequestWithCredentials, Response]
{
  def apply(request: Request, service: Service[RequestWithCredentials, Response]): Future[Response] = {
    val requestWithCredentials = new RequestWrapper with RequestWithCredentials {
      val underlying = request
      val credentials = credentialsParser(request) getOrElse NullCredentials
    }

    service(requestWithCredentials)
  }
}

{% endhighlight %}

注意相关服务怎样需要一个认证请求，这里是静态确认。过滤器从此被认为是服务转换器。

一些过滤器可以组合在一起：


{% highlight scala %}

val upFilter =
  logTransaction     andThen
  handleExceptions   andThen
  extractCredentials andThen
  homeUser           andThen
  authenticate       andThen
  route

{% endhighlight %}

它们是类型安全的。
