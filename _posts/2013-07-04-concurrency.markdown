---
layout: post
title:  "Scala 里面的并发"
next_link:  java
prev_link:  specs
date:  2013-07-04 00:00:00
categories: jekyll update
---


课程包括

*  Runnable/Callable
*  线程 Threads
*  Executors/ExecutorService
*  Futures
*  线程安全问题 Thread Safety Problem
*  例子：搜索引擎 Example：Search Engine
*  解决方案 Solutions

##Runnable/Callabel

Runnable 有一个不返回值的方法。

{% highlight scala %}

trait Runnable {
  def run(): Unit
}

{% endhighlight %}

Callable 和Runnable 很像，有一点区别是它返回一个值。
  
{% highlight scala %}

trait Callable[V] {
  def call(): V
}

{% endhighlight %}

##线程 Threads

Scala 的并发构建在Java 并发模型的基础上。

在Sun 的JVM 上面，通过一个IO-heavy workload，我们可以在一个机器上运行上万个线程。

一个线程接受一个Runnable。你可以调用线程上的start 方法来运行Runnable。

{% highlight bash %}

scala> val hello = new Thread(new Runnable {
  def run() {
    println("hello world")
  }
})
hello: java.lang.Thread = Thread[Thread-3,5,main]

scala> hello.start
hello world

{% endhighlight %}

当你看到一个实现了Runnable 的类时，你就知道了某人将要在某个地方运行一个线程了。

###单线程 Something single-threaded

这里有一个能运行却有问题的代码片段。

{% highlight scala %}

import java.net.{Socket, ServerSocket}
import java.util.concurrent.{Executors, ExecutorService}
import java.util.Date

class NetworkService(port: Int, poolSize: Int) extends Runnable {
  val serverSocket = new ServerSocket(port)

  def run() {
    while (true) {
      // This will block until a connection comes in.
      val socket = serverSocket.accept()
      (new Handler(socket)).run()
    }
  }
}

class Handler(socket: Socket) extends Runnable {
  def message = (Thread.currentThread.getName() + "\n").getBytes

  def run() {
    socket.getOutputStream.write(message)
    socket.getOutputStream.close()
  }
}

(new NetworkService(2020, 2)).run

{% endhighlight %}

每一个请求都会返回一个含有当前线程名字的响应，它通常是main。

含有main 代码的缺陷是一次只能有一个请求被响应。

你可以把所有的请求放在一个线程里。简单的修改如下把

{% highlight scala %}

(new Handler(socket)).run()

{% endhighlight %}

改成

{% highlight scala %}

(new Thread(new Handler(socket))).start()

{% endhighlight %}

可是，如果你想要重用线程或者是要使用线程其它的方法该怎么办呢？

##Executors

随着Java 5 的发布，注定需要一个更抽象的线程接口。

你可以在Executors object 上面使用静态方法获得一个ExecutorService。这些方法允许你用大量的方法如线程池来配置一个ExecutorService。

这是我们用老的代码写的允许线程请求的网络服务。

{% highlight scala %}

import java.net.{Socket, ServerSocket}
import java.util.concurrent.{Executors, ExecutorService}
import java.util.Date

class NetworkService(port: Int, poolSize: Int) extends Runnable {
  val serverSocket = new ServerSocket(port)
  val pool: ExecutorService = Executors.newFixedThreadPool(poolSize)

  def run() {
    try {
      while (true) {
        // This will block until a connection comes in.
        val socket = serverSocket.accept()
        pool.execute(new Handler(socket))
      }
    } finally {
      pool.shutdown()
    }
  }
}

class Handler(socket: Socket) extends Runnable {
  def message = (Thread.currentThread.getName() + "\n").getBytes

  def run() {
    socket.getOutputStream.write(message)
    socket.getOutputStream.close()
  }
}

(new NetworkService(2020, 2)).run

{% endhighlight %}

这是一个用来显示内部线程复用的一个重复连接。

{% highlight bash %}

$ nc localhost 2020
pool-1-thread-1

$ nc localhost 2020
pool-1-thread-2

$ nc localhost 2020
pool-1-thread-1

$ nc localhost 2020
pool-1-thread-2

{% endhighlight %}

##Futures

Future 代表一个异步计算，你可以在你需要结果的时候把你的计算包裹在Future里，然后简单在上面的调用get() 方法，Executor 会返回一个Future。如果你使用Finagle RPC 系统，你使用Future 实例来保存一个并没有需要访问的结果。

一个FutureTask 是一个Runnable ，它被设计成被一个Executor 运行。

{% highlight scala %}

val future = new FutureTask[String](new Callable[String]() {
  def call(): String = {
    searcher.search(target);
}})
executor.execute(future)

{% endhighlight %}

现在我需要一个结果以便在完成之前阻止它。

{% highlight scala %}

val blockingResult = future.get()

{% endhighlight %}

见[Scala School 的Finagle](/finagle.html) 页上有大量的使用Future 的例子，包括一些来组合它们的方式。Effective Scala 里面也有关于[Futures](http://twitter.github.com/effectivescala/#Twitter%27s%20standard%20libraries-Futures) 的描述。

##线程安全问题

{% highlight scala %}

class Person(var name: String) {
  def set(changedName: String) {
    name = changedName
  }
}

{% endhighlight %}

这个程序在多线程的环境下是不安全的。如果两个线程同时引用相同的Person 实例并且调用set，你不会知道它们都调用完之后会是什么样的名字。

在Java 的内存模型里，每个处理器都允许在L1 或者L2 缓存里面建立值的缓存，因此两个运行在不同处理器里面的线程可以都有它们的数据试图。

让我们来讨论一下强制把线程保持在一致的数据视图的工具。

###线程池

####同步

Mutexes 提供语义所有权。当你进入到一个mutex 时，你就拥有了它。在JVM 里面使用mutex 的最普遍的方法就是通过一些东西里面的同步。在这里，我们在我们的Person 同步。

在JVM里面，你可以同步任何非空的实例。

{% highlight scala %}

class Person(var name: String) {
  def set(changedName: String) {
    this.synchronized {
      name = changedName
    }
  }
}

{% endhighlight %}

####vloatile

随着Java 5对内存模型的改变，volatile 和 同步除了volatile，null被允许外基本上相同。同步允许更细粒度的锁，volatile 在每次访问时同步。

{% highlight scala %}

class Person(@volatile var name: String) {
  def set(changedName: String) {
    name = changedName
  }
}

{% endhighlight %}

###AtomicReference

还是在Java 5,增加了一个低级同步原语的整个raft。其中的一个就是AtomicReference 类。

{% highlight scala %}

import java.util.concurrent.atomic.AtomicReference

class Person(val name: AtomicReference[String]) {
  def set(changedName: String) {
    name.set(changedName)
  }
}

{% endhighlight %}

####这花费什么事情吗？

当你不得不通过方法dispatch 来访问值时，@AtomicReference 是这两个选择里面代价最大的。

volatile 和 同步是构建在Java 内建监视器顶部的。如果没有冲突的话监视器的消耗是非常小的。同时同步允许你更细粒度的控制你的同步，这样的话冲突会比较少，所以同步倾向于更昂贵的选择。

当你进入到同步点时，访问volatile 引用，或者不同的AtomicReferences，Java 强制处理器刷新缓存，然后提供一个一致的数据视图。

如果我这里有错误请纠正我。这是一个复杂的东西，我确定一定会有大量的研讨班在讨论这个问题。

###从Java 5 带过来的其它的好东西

随着我对AtomicReference 的关注，Java5 伴随着它带来了更多伟大工具。

####CountDownLatch

CountDownLatch 是一个多线程之间通讯的简单机制。

{% highlight scala %}

val doneSignal = new CountDownLatch(2)
doAsyncWork(1)
doAsyncWork(2)

doneSignal.await()
println("both workers finished!")

{% endhighlight %}

除这些外，用它来做单元测试真是太棒了。让我们谈论一下你在做的一些同步的工作然后你想要确保函数被完成了。简单的用countDown 函数 锁上，然后用在测试时用 await。

####AtomicInteger/Long

由于增加的Int 和Long 是一个如此普遍的任务，AtomicInteger 和AtomicLong 被增加了。

####AtomicBoolean

我可能不必解释这个是干嘛的了。

####ReadWriteLocks

ReadWriteLock 让你接受一个reader 以及写保护锁，reader 锁当写保护锁打开的时候才锁定。

##让我们来构建一个不安全的搜索引擎

这是一个简单的不是线程安全的倒排索引。我们的倒排索引映射给定用户的部分名字。

这是我们用幼稚的方式假设我们只有一个线程访问。

注意可选的默认构造器this() 使用mutable.HashMap。

{% highlight scala %}

import scala.collection.mutable

case class User(name: String, id: Int)

class InvertedIndex(val userMap: mutable.Map[String, User]) {

  def this() = this(new mutable.HashMap[String, User])

  def tokenizeName(name: String): Seq[String] = {
    name.split(" ").map(_.toLowerCase)
  }

  def add(term: String, user: User) {
    userMap += term -> user
  }

  def add(user: User) {
    tokenizeName(user.name).foreach { term =>
      add(term, user)
    }
  }
}

{% endhighlight %}


目前我没有处理怎样把用户踢出我们的索引。我们稍后会处理的。

##让我们把它变得更安全

在我们上面的倒序索引里，userMap 不保证是安全的。多个客户端可能尝试在同一时刻添加项目同时有相同的可见的我们在第一个Person 例子里面看到的错误。

由于userMap 不是线程安全的，我们怎样确保在同一时刻只有一个线程改变它呢？

你可以考虑在添加的时候锁住userMap。

{% highlight scala %}

def add(user: User) {
  userMap.synchronized {
    tokenizeName(user.name).foreach { term =>
      add(term, user)
    }
  }
}

{% endhighlight %}

这也太特么的粗糙了。经常尝试着尽可能在mutex 外面做更昂贵的工作。记住我说过的，如果没有冲突尽量保持锁更轻便。如果你在锁里面做更少的工作，那么冲突就会更少。

{% highlight scala %}

def add(user: User) {
  // tokenizeName was measured to be the most expensive operation.
  val tokens = tokenizeName(user.name)

  tokens.foreach { term =>
    userMap.synchronized {
      add(term, user)
    }
  }
}

{% endhighlight %}

##SynchronizedMap

我们可以使用SynchronizedMap 接口混合synchronizatio 和mutable HashMap。

我们通过给用户一个容易的方式构建同步索引扩展了我们存在的倒序索引。

{% highlight scala %}

import scala.collection.mutable.SynchronizedMap

class SynchronizedInvertedIndex(userMap: mutable.Map[String, User]) extends InvertedIndex(userMap) {
  def this() = this(new mutable.HashMap[String, User] with SynchronizedMap[String, User])
}

{% endhighlight %}

如果你看到实现，你会意识到这是简单的在每个方法上的同步，因此它是安全的，它可能没有你希望的行为。

##Java ConcurrentHashMap

Java 带来了一个不错的线程安全的ConcurrentHaspMap。非常感谢，我们可以使用JavaConverters 来给我们一个良好的Scala 语义。

实际上，我们可以无缝的集成我们新的，线程安全的倒序索引作为我们之前的不安全版本的一个扩展。

{% highlight scala %}

import java.util.concurrent.ConcurrentHashMap
import scala.collection.JavaConverters._

class ConcurrentInvertedIndex(userMap: collection.mutable.ConcurrentMap[String, User])
    extends InvertedIndex(userMap) {

  def this() = this(new ConcurrentHashMap[String, User] asScala)
}

{% endhighlight %}

##让我们来载入我们的倒序索引

###幼稚的方法

{% highlight scala %}

trait UserMaker {
  def makeUser(line: String) = line.split(",") match {
    case Array(name, userid) => User(name, userid.trim().toInt)
  }
}

class FileRecordProducer(path: String) extends UserMaker {
  def run() {
    Source.fromFile(path, "utf-8").getLines.foreach { line =>
      index.add(makeUser(line))
    }
  }
}

{% endhighlight %}

我们文件的每一行，我们可以在我们的倒序索引上调用makeUser 然后掉用add 。如果我们使用一个并发的倒序索引，我们可以调用并行上的add，由于makeUser 没有副作用，所以它一直是线程安全的。

我们不能读取并行上的我文件但是我们可以构建User 然后把它添加到并行（parallel）上。

###一个解决方案：生产者和消费者

一个同步计算普遍的模式是把生产者和消费者分开，它们之间仅仅通过Queue 通讯。让我们看看搜索引擎索引是怎么工作的。

{% highlight scala %}

import java.util.concurrent.{BlockingQueue, LinkedBlockingQueue}

// Concrete producer
class Producer[T](path: String, queue: BlockingQueue[T]) extends Runnable {
  def run() {
    Source.fromFile(path, "utf-8").getLines.foreach { line =>
      queue.put(line)
    }
  }
}

// Abstract consumer
abstract class Consumer[T](queue: BlockingQueue[T]) extends Runnable {
  def run() {
    while (true) {
      val item = queue.take()
      consume(item)
    }
  }

  def consume(x: T)
}

val queue = new LinkedBlockingQueue[String]()

// One thread for the producer
val producer = new Producer[String]("users.txt", q)
new Thread(producer).start()

trait UserMaker {
  def makeUser(line: String) = line.split(",") match {
    case Array(name, userid) => User(name, userid.trim().toInt)
  }
}

class IndexerConsumer(index: InvertedIndex, queue: BlockingQueue[String]) extends Consumer[String](queue) with UserMaker {
  def consume(t: String) = index.add(makeUser(t))
}

// Let's pretend we have 8 cores on this machine.
val cores = 8
val pool = Executors.newFixedThreadPool(cores)

// Submit one consumer per core.
for (i <- i to cores) {
  pool.submit(new IndexerConsumer[String](index, q))
}

{% endhighlight %}


