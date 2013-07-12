---
layout: post
title:  "SearchBird"
next_link:  
prev_link:  finagle
date:  2013-07-09 00:00:00
categories: jekyll update
---

我们将要使用Scala 以及我们之前讨论的Finagle 框架来构建一个简单的分布式搜索引擎。

##设计目标：一个大图片

广泛来讲，我们的设计目标包括抽象化（在没有知道所有的内部详细信息的时候使用resulting 系统的能力）；模块化（可以把系统分成更小，更简单的小片，使它能够更容易明白或者更容易替换成其它的小块的能力）;以及稳定性（以一种简单的方法增加系统容量的能力）。

我们将要描述的系统有三个小片：（1）发送请求的客户端（2）给请求发送响应的服务器（3）打包这些通讯的传输机制。通常客户端和服务器应该位于不容的机器上，它们通过网络上一个特殊的数字端口来通讯。但是在这个例子里它们会在同一台机器上（仍然使用端口通讯）。在我们的例子里，客户端和服务器会用Scala 来写，传输会使用Thrift 来处理。这个教程的原始意图是展示一个简单的可扩展的提供稳定性能的客户端和服务器。

##探索默认的启动工程

首先使用scala-bootstrapper 创建一个骨架工程（“Searchbird”）。它创建一个简单的基于Finagle 的Scala 服务，它导出一个内存键值对存储。我们会通过支持搜索值来扩展它，接着通过一些处理来支持搜索多个内存存储来扩展它。

{% highlight bash %}

$ mkdir searchbird ; cd searchbird
$ scala-bootstrapper searchbird
writing build.sbt
writing config/development.scala
writing config/production.scala
writing config/staging.scala
writing config/test.scala
writing console
writing Gemfile
writing project/plugins.sbt
writing README.md
writing sbt
writing src/main/scala/com/twitter/searchbird/SearchbirdConsoleClient.scala
writing src/main/scala/com/twitter/searchbird/SearchbirdServiceImpl.scala
writing src/main/scala/com/twitter/searchbird/config/SearchbirdServiceConfig.scala
writing src/main/scala/com/twitter/searchbird/Main.scala
writing src/main/thrift/searchbird.thrift
writing src/scripts/searchbird.sh
writing src/scripts/config.sh
writing src/scripts/devel.sh
writing src/scripts/server.sh
writing src/scripts/service.sh
writing src/test/scala/com/twitter/searchbird/AbstractSpec.scala
writing src/test/scala/com/twitter/searchbird/SearchbirdServiceSpec.scala
writing TUTORIAL.md

{% endhighlight %}

我们首先来看看为我们创建的默认的scala-bootstrapper 工程。它是一个模板。你最终会取代其中的大部分，但是它是一个很方便的脚手架。它定义一个简单的（但是很完整）键值存储。配置，一个thrift 接口，统计出口以及日志都包括在内。

在我们看代码之前，我们将要运行一个客户端和服务器来看看它是怎么工作的。这里是我们将要构建的：

![Searchbird implementation,revision 1](http://twitter.github.io/scala_school/searchbird-1.svg)

这里是我们服务导出的接口。由于Searchbird 服务是一个Thrift，服务（更像我们的服务），它的内部接口定义在Thrift IDL（“接口描述语言”）。

**src/main/thrift/searchbird.thrift**

{% highlight scala %}

service SearchbirdService {
  string get(1: string key) throws(1: SearchbirdException ex)

  void put(1: string key, 2: string value)
}

{% endhighlight %}

这很简单：我们的服务SearchbirdService 导出了两个RPC 方法，get 和put。它们包含一个简单的键值存储接口。

想在让我们运行默认的服务并且用一个客户端来连接这个服务，接着通过这个接口来探索他们。打开两个窗口，一个是服务器一个是客户端。

在第一个窗口里，在交互模式下运行sbt(在命令行里运行 ./sbt)接着在sbt 里面构建运行工程。在Main.scala 里运行main。

{% highlight bash %}

$ ./sbt
...
> compile
> run -f config/development.scala
...
[info] Running com.twitter.searchbird.Main -f config/development.scala

{% endhighlight %}

配置文件（development.scala）初始化一个新的服务，并在我们的local 机器上的9999 端口暴漏我们的服务。客户端可以通过连接9999 端口和这个服务通讯。

现在我们将要使用提供的终端shell 脚本来运行一个客户端，它初始化一个SearchbirdConsoleClient 实例（从SearchbirdConsoleClient.scala）。在另一个窗口里运行这个脚本：

{% highlight bash %}

$ ./console 127.0.0.1 9999
[info] Running com.twitter.searchbird.SearchbirdConsoleClient 127.0.0.1 9999
'client' is bound to your thrift client.

finagle-client> 

{% endhighlight %}

客户端object client 现在在我们的local 电脑上连接了9999 端口，可以通过之前我们开始的那个端口和服务会话。因此让我们发送一些请求：

{% highlight bash %}

scala> client.put("marius", "Marius Eriksen")
res0: ...

scala> client.put("stevej", "Steve Jenson")
res1: ...

scala> client.get("marius")
res2: com.twitter.util.Future[String] = ...

scala> client.get("marius").get()
res3: String = Marius Eriksen

{% endhighlight %}

(第二个get() 可以解析Future 它在值准备好之前一直阻塞返回client.get() 类型。)

服务器还导出运行统计（配置文件指定这些可以在9900 端口里找到）。不管是个人检查服务器还是作为全局服务的汇总统计这都很方便。打开第三个窗口检查如下统计：

{% highlight bash %}

$ curl localhost:9900/stats.txt
counters:
  Searchbird/connects: 1
  Searchbird/received_bytes: 264
  Searchbird/requests: 3
  Searchbird/sent_bytes: 128
  Searchbird/success: 3
  jvm_gc_ConcurrentMarkSweep_cycles: 1
  jvm_gc_ConcurrentMarkSweep_msec: 15
  jvm_gc_ParNew_cycles: 24
  jvm_gc_ParNew_msec: 191
  jvm_gc_cycles: 25
  jvm_gc_msec: 206
gauges:
  Searchbird/connections: 1
  Searchbird/pending: 0
  jvm_fd_count: 135
  jvm_fd_limit: 10240
  jvm_heap_committed: 85000192
  jvm_heap_max: 530186240
  jvm_heap_used: 54778640
  jvm_nonheap_committed: 89657344
  jvm_nonheap_max: 136314880
  jvm_nonheap_used: 66238144
  jvm_num_cpus: 4
  jvm_post_gc_CMS_Old_Gen_used: 36490088
  jvm_post_gc_CMS_Perm_Gen_used: 54718880
  jvm_post_gc_Par_Eden_Space_used: 0
  jvm_post_gc_Par_Survivor_Space_used: 1315280
  jvm_post_gc_used: 92524248
  jvm_start_time: 1345072684280
  jvm_thread_count: 16
  jvm_thread_daemon_count: 7
  jvm_thread_peak_count: 16
  jvm_uptime: 1671792
labels:
metrics:
  Searchbird/handletime_us: (average=9598, count=4, maximum=19138, minimum=637, p25=637, p50=4265, p75=14175, p90=19138, p95=19138, p99=19138, p999=19138, p9999=19138, sum=38393)
  Searchbird/request_latency_ms: (average=4, count=3, maximum=9, minimum=0, p25=0, p50=5, p75=9, p90=9, p95=9, p99=9, p999=9, p9999=9, sum=14)

{% endhighlight %}

除了我们自己的服务统计，我们也给了一些一般的经常使用的JVM 统计。

现在让我们来看看实现配置的代码，客户端和服务器。

**../config/SearchbirdServiceConfig.scala**

一个配置是一个有一个为一些T 想要创建的apply: RuntimeEnviroment => T 方法 Scala 接口。在这里配置是“工厂”。在运行时，一个配置文件相当与一个脚本（通过使用scala 编译类库），它期待产生诸如配置object。一个RuntimeEnvironment 是一个查询不同运行参数的object（命令行标志，JVM 版本，构建时间等）。

SearchbirdServiceConfig 类指定这样一个类。它和它的默认配置一起指定配置参数。（Finagle 支持一个我们可以忽略这个教程的意图的通用的追踪系统；Zipkin 分布跟踪系统是一个这样追踪的一个collector/aggregator）

{% highlight scala %}

class SearchbirdServiceConfig extends ServerConfig[SearchbirdService.ThriftServer] {
  var thriftPort: Int = 9999
  var tracerFactory: Tracer.Factory = NullTracer.factory

  def apply(runtime: RuntimeEnvironment) = new SearchbirdServiceImpl(this)
}

{% endhighlight %}

在我们的类里，我们想要创建一个SearchbirdService.ThriftServer,然后启动它。它读取配置（在development.scala 以及作为运行时的一个参数），创建SearchbirdService。ThriftServer，然后启动它。RuntimtEnvironment.loadRuntimeConfig 执行这个配置评估同时做为参数调用apply 方法。

{% highlight scala %}

object Main {
  private val log = Logger.get(getClass)

  def main(args: Array[String]) {
    val runtime = RuntimeEnvironment(this, args)
    val server = runtime.loadRuntimeConfig[SearchbirdService.ThriftServer]
    try {
      log.info("Starting SearchbirdService")
      server.start()
    } catch {
      case e: Exception =>
        log.error(e, "Failed starting SearchbirdService, exiting")
        ServiceTracker.shutdown()
        System.exit(1)
    }
  }
}

{% endhighlight %}

**../SearchbirdServiceImpl.scala**

这意味着服务：我们用我们的定制实现展开我们的SearchbirdService.ThriftServer。重新调用已经被我们用thrift 代码构建器创建的SearchbirdService.ThriftServer 。每一个thrift 方法生成一个scala 方法。目前为止在我们的例子里面，生成接口如下：

{% highlight scala %}

trait SearchbirdService {
  def put(key: String, value: String): Future[Void]
  def get(key: String): Future[String]
}

{% endhighlight %}

Future\[Value\] 将代替values 返回，所以计算可以被延期，（finagle [文档](http://twitter.github.io/scala_school/finagle.html)里面有更详细的信息。）对于本教程，所有关于Future 你需要了解的就是你可以通过get() 取回它的值。

通过scala-bootstrapper 提供的默认键值存贮的实现也很简单：它提供一个数据库结构以及get 和put 调用访问数据结构。

{% highlight scala %}

class SearchbirdServiceImpl(config: SearchbirdServiceConfig) extends SearchbirdService.ThriftServer {
  val serverName = "Searchbird"
  val thriftPort = config.thriftPort
  override val tracerFactory = config.tracerFactory

  val database = new mutable.HashMap[String, String]()

  def get(key: String) = {
    database.get(key) match {
      case None =>
        log.debug("get %s: miss", key)
        Future.exception(SearchbirdException("No such key"))
      case Some(value) =>
        log.debug("get %s: hit", key)
        Future(value)
    }
  }

  def put(key: String, value: String) = {
    log.debug("put %s", key)
    database(key) = value
    Future.Unit
  }

  def shutdown() = {
    super.shutdown(0.seconds)
  }
}

{% endhighlight %}

结果是一个简单的thrift 向Scala HashMap 的接口。

##一个简单的搜索引擎

现在我们来扩展我们的例子来创建一个简单的搜索引擎，我们将要把它变成一个由多个碎片组成的分布式搜索引擎，这样我们就可以适应更大量的文集而不是单个机器里面的内存。

为了让事情更简单一点，为了支持搜索操作，我们将稍微的扩展我们当前的thrift 服务。model 用例把文档放到搜索引擎上，每个文档都是由一系列分割的单词组成;接着我们搜索一个关键字符串来返回所有包含所有关键字集合的文档。它的结构和前一个例子相同，但是增加了一个新的search 调用。

![Searchbird implementation,revision 2](http://twitter.github.io/scala_school/searchbird-2.svg)

要想实现上面的像搜索引擎，改变如下两个文件：

**src/main/thrift/searchbird.thrift**

{% highlight scala %}

service SearchbirdService {
  string get(1: string key) throws(1: SearchbirdException ex)

  void put(1: string key, 2: string value)

  list<string> search(1: string query)
}

{% endhighlight %}

我们新增了一个搜索当前哈希表的search 方法，返回匹配查询的值的键。实现同样也非常简单：

**../SearchbirdServiceImpl.scala**

我们所有的改变都几乎发生在这个文件里。

当前的数据库hashmap 保持了一个当前的映射到文档的键的目录。我们把它重命名为forward 然后给reverse 索引添加第二个map（它映射到一个包含token 的文档集合）。因此，在SearchbirdServiceImpl.scala 里面，把数据库定义替换成：

{% highlight scala %}

val forward = new mutable.HashMap[String, String]
  with mutable.SynchronizedMap[String, String]
val reverse = new mutable.HashMap[String, Set[String]]
  with mutable.SynchronizedMap[String, Set[String]]

{% endhighlight %}

在get 调用里，把database 替换成为forward，否则，get 会一直返回相同的值（它只是执行正向查找）。不管怎样，put 也需要改变：我们也需要为每个文档的taken 计算反转索引，可以通过给和taken 联系的list 添加文档键。把put 调用替换成下面的代码。给定一个特殊的搜索taken，我们现在可以使用reverse map 来搜索文档。
 
{% highlight scala %}

val forward = new mutable.HashMap[String, String]
  with mutable.SynchronizedMap[String, String]
val reverse = new mutable.HashMap[String, Set[String]]
  with mutable.SynchronizedMap[String, Set[String]]

{% endhighlight %}

注意这里（尽管HashMap 是线程安全的）仅仅一个线程可以更新reverse map，同时确保特殊map入口的读-修改-写是原子操作。（这里的代码非常保守;它锁住了整个的map 而不是锁住每一个单独的取回-修改-写操作。）同时注意Set 作为数据结构的的使用；它确保了如果相同的token 在一个文档里出现两次的情况，它将要被foreach 循环执行一次。

这个实现仍然有一个问题，它给读者留下了一个小练习：当我们用一个新文档重写一个键时，我们没有删除任何在反转索引里面的相关老文档。

现在搜索引擎的菜是：新的搜索方法。它应该接受它的查询，寻找所有匹配的文档，然后和这个列表相交。这会产生一个包含所有接受参数的文档的列表。这在Scala 里面很容易表达，这里是SearchbirdServiceImpl 类：

{% highlight scala %}

def search(query: String) = Future.value {
  val tokens = query.split(" ")
  val hits = tokens map { token => reverse.getOrElse(token, Set()) }
  val intersected = hits reduceLeftOption { _ & _ } getOrElse Set()
  intersected.toList
}

{% endhighlight %}

在这小段代码里面没有什么值得调用的。当构造hit 列表时，如果键（token）没有找到，getOrElse 会应用它第二个参数的值（在这里是一个空的Set）。我们使用left-reduce 来执行真实的交叉。特殊的flavor ，reduceLeftOption，在hi ts 为空时也将不尝试执行减少，相反返回None 。 这允许我们应用一个默认的值而不是抛出一个异常。事实上，这和下面相当：

{% highlight scala %}

def search(query: String) = Future.value {
  val tokens = query.split(" ")
  val hits = tokens map { token => reverse.getOrElse(token, Set()) }
  if (hits.isEmpty)
    Nil
  else
    hits reduceLeft { _ & _ } toList
}

{% endhighlight %}

使用哪一个大多看个人喜好，即使函数式风格经常避开了明智默认条件。

我们现在可以使用console 来体验一下我们的新实现。在一次开启你的服务端：

{% highlight bash %}

$ ./sbt
...
> compile
> run -f config/development.scala
...
[info] Running com.twitter.searchbird.Main -f config/development.scala

{% endhighlight %}

然后从searchbird 路径下，开启客户端：

{% highlight bash %}

$ ./console 127.0.0.1 9999
...
[info] Running com.twitter.searchbird.SearchbirdConsoleClient 127.0.0.1 9999
'client' is bound to your thrift client.

finagle-client> 

{% endhighlight %}

把下面的句子粘贴到你的控制台：

{% highlight scala %}

client.put("basics", " values functions classes methods inheritance try catch finally expression oriented")
client.put("basics", " case classes objects packages apply update functions are objects (uniform access principle) pattern")
client.put("collections", " lists maps functional combinators (map foreach filter zip")
client.put("pattern", " more functions! partialfunctions more pattern")
client.put("type", " basic types and type polymorphism type inference variance bounds")
client.put("advanced", " advanced types view bounds higher kinded types recursive types structural")
client.put("simple", " all about sbt the standard scala build")
client.put("more", " tour of the scala collections")
client.put("testing", " write tests with specs a bdd testing framework for")
client.put("concurrency", " runnable callable threads futures twitter")
client.put("java", " java interop using scala from")
client.put("searchbird", " building a distributed search engine using")

{% endhighlight %}

我们现在可以执行一些搜索，它返回包含搜索条款的文档的键。

{% highlight bash %}

> client.search("functions").get()
res12: Seq[String] = ArrayBuffer(basics)

> client.search("java").get()
res13: Seq[String] = ArrayBuffer(java)

> client.search("java scala").get()
res14: Seq[String] = ArrayBuffer(java)

> client.search("functional").get()
res15: Seq[String] = ArrayBuffer(collections)

> client.search("sbt").get()
res16: Seq[String] = ArrayBuffer(simple)

> client.search("types").get()
res17: Seq[String] = ArrayBuffer(type, advanced)

{% endhighlight %}

如果调用返回Future 就重新调用它，我们不得不使用一个blocking get() 调用来解决包括future 的值。我们可以使用Future.collect 命令来使用多并发请求并且等待它们的成功。

{% highlight bash %}

> import com.twitter.util.Future
...
> Future.collect(Seq(
    client.search("types"),
    client.search("sbt"),
    client.search("functional")
  )).get()
res18: Seq[Seq[String]] = ArrayBuffer(ArrayBuffer(type, advanced), ArrayBuffer(simple), ArrayBuffer(collections))

{% endhighlight %}

##分发我们的服务

在单个机器上，我们简陋的内存搜索引擎不会搜索到大于内存值的文集。我们现在尝试通过含有简单分片方案的分布式节点来弥补一下。这里是代码图：

![Distributed Searchbird service](http://twitter.github.io/scala_school/searchbird-3.svg)

###抽象

为了补救我们的工作，我们首先引入了一个抽象--Index--为了从SearchbirdService 里面脱钩index 实现。这是一个简单的重构。我们开始给build 添加一个Index 文件（创建一个searchbird/src/main/scala/com/twitter/searchbird/Index.scala）:

**../Index.scala**

{% highlight scala %}

package com.twitter.searchbird

import scala.collection.mutable
import com.twitter.util._
import com.twitter.conversions.time._
import com.twitter.logging.Logger
import com.twitter.finagle.builder.ClientBuilder
import com.twitter.finagle.thrift.ThriftClientFramedCodec

trait Index {
  def get(key: String): Future[String]
  def put(key: String, value: String): Future[Unit]
  def search(key: String): Future[List[String]]
}

class ResidentIndex extends Index {
  val log = Logger.get(getClass)

  val forward = new mutable.HashMap[String, String]
    with mutable.SynchronizedMap[String, String]
  val reverse = new mutable.HashMap[String, Set[String]]
    with mutable.SynchronizedMap[String, Set[String]]

  def get(key: String) = {
    forward.get(key) match {
      case None =>
        log.debug("get %s: miss", key)
        Future.exception(SearchbirdException("No such key"))
      case Some(value) =>
        log.debug("get %s: hit", key)
        Future(value)
    }
  }

  def put(key: String, value: String) = {
    log.debug("put %s", key)
    
    forward(key) = value

    // admit only one updater.
    synchronized {
      (Set() ++ value.split(" ")) foreach { token =>
        val current = reverse.get(token) getOrElse Set()
        reverse(token) = current + key
      }
    }

    Future.Unit
  }

  def search(query: String) = Future.value {
    val tokens = query.split(" ")
    val hits = tokens map { token => reverse.getOrElse(token, Set()) }
    val intersected = hits reduceLeftOption { _ & _ } getOrElse Set()
    intersected.toList
  }
}

{% endhighlight %}

我们现在把我们的thrift 服务转换成为简单的dispatch 机制：它给所有Index 实例提供一个thrift 接口。这是一个很强大的抽象，因为它把服务的实现从index 的实现分离了出来。服务不必知道任何关于相关index 的细节;index 可能是local 或者可能是远程或者可能是一些远程indices 的复合物，但是服务什么也不关心，index 的实现可能在服务未变时而改变。

把你的SearchbirdServiceImpl 类定义成（很简单的）一个（不在包含index 详细实现）。注意现在初始化一个服务需要第二个参数，Index。

**../SearchbirdServiceImpl.scala**

{% highlight scala %}

class SearchbirdServiceImpl(config: SearchbirdServiceConfig, index: Index) extends SearchbirdService.ThriftServer {
  val serverName = "Searchbird"
  val thriftPort = config.thriftPort

  def get(key: String) = index.get(key)
  def put(key: String, value: String) =
    index.put(key, value) flatMap { _ => Future.Unit }
  def search(query: String) = index.search(query)

  def shutdown() = {
    super.shutdown(0.seconds)
  }
}

{% endhighlight %}

**../config/SearchbirdServiceConfig.scala**

根据下面的代码更新SearchbirdServiceconfig 里面的apply 调用：

{% highlight scala %}

class SearchbirdServiceConfig extends ServerConfig[SearchbirdService.ThriftServer] {
  var thriftPort: Int = 9999
  var tracerFactory: Tracer.Factory = NullTracer.factory

  def apply(runtime: RuntimeEnvironment) = new SearchbirdServiceImpl(this, new ResidentIndex)
}

{% endhighlight %}

为了有一个查询子节点坐标分布式节点，我们将要建立我们的简单分布式系统。为了做到这一点，我们需要两个新的Index 类型。一个代表远程index，另一个是通过一个其他Index 实例的综合index。我们通过这种方法我们可以通过实例化一个远程indices 的综合索引来构建分布式索引。注意所有Index 类型都有相同的接口，因此服务不需要知道它们连接的是远程还是综合的。

**../Index.scala**

在Index.scala 里，定义一个CompositeIndex：

{% highlight scala %}

class CompositeIndex(indices: Seq[Index]) extends Index {
  require(!indices.isEmpty)

  def get(key: String) = {
    val queries = indices.map { idx =>
      idx.get(key) map { r => Some(r) } handle { case e => None }
    }

    Future.collect(queries) flatMap { results =>
      results.find { _.isDefined } map { _.get } match {
        case Some(v) => Future.value(v)
        case None => Future.exception(SearchbirdException("No such key"))
      }
    }
  }

  def put(key: String, value: String) =
    Future.exception(SearchbirdException("put() not supported by CompositeIndex"))

  def search(query: String) = {
    val queries = indices.map { _.search(query) rescue { case _=> Future.value(Nil) } }
    Future.collect(queries) map { results => (Set() ++ results.flatten) toList }
  }
}

{% endhighlight %}

综合的index 通过一个相关的Index 实例起作用。注意它不在乎这些实际上是怎么实现的。这个综合的类型允许在构造多种schemes 查询时有很大的灵活性，因此综合索引不需要支持put 操作。这个直接被字节点替代了。get 被实现为查询所有的字节点然后选出第一个成功的结果。如果没有，我们就抛出一个异常。注意在值被通知抛出异常的情况下，我们在Future 中处理它，把任何异常类型转换为None 值。在一个真实的系统里，我们可能针对错误的值有适当的错误代码而不是使用异常。异常是原型之间的权宜，但是构成很不好。为了区别真实的异常还是值丢失，我们不得不检查异常本身。这返回值类型里面的最好的方式直接识别嵌入。

search 和之前类似的方法工作。和找到第一个值不一样，我们把它们组合了，确保他们使用Set 构造器来去重。

RemoteIndex 来给服务端提供提供一个Index 接口。

{% highlight scala %}

class RemoteIndex(hosts: String) extends Index {
  val transport = ClientBuilder()
    .name("remoteIndex")
    .hosts(hosts)
    .codec(ThriftClientFramedCodec())
    .hostConnectionLimit(1)
    .timeout(500.milliseconds)
    .build()
  val client = new SearchbirdService.FinagledClient(transport)

  def get(key: String) = client.get(key)
  def put(key: String, value: String) = client.put(key, value) map { _ => () }
  def search(query: String) = client.search(query) map { _.toList }
}

{% endhighlight %}

这个结构是一个含有一些智能默认的finagle fhrift 客户端，仅仅代理调用，能简单的调整类型。

##把它们都放在一起

我们现在已经有了我们需要的所有碎片。为了能够既做为一个分布式节点有能做为一个共享节点调用给定的节点我们需要调整一下配置。为了做到这点，我们通过为它创建一个新的配置项目来枚举我们系统里面的碎片。我们也需要给我们SearchbirdServiceImpl 的初始化添加一个Index 参数。我们接着使用命令行参数（重新调用已经访问过它们的Config）来每一端开始服务。

**../config/SearchbirdServiceConfig.scala**

{% highlight scala %}

class SearchbirdServiceConfig extends ServerConfig[SearchbirdService.ThriftServer] {
  var thriftPort: Int = 9999
  var shards: Seq[String] = Seq()

  def apply(runtime: RuntimeEnvironment) = {
    val index = runtime.arguments.get("shard") match {
      case Some(arg) =>
        val which = arg.toInt
        if (which >= shards.size || which < 0)
          throw new Exception("invalid shard number %d".format(which))

        // override with the shard port
        val Array(_, port) = shards(which).split(":")
        thriftPort = port.toInt

        new ResidentIndex

      case None =>
        require(!shards.isEmpty)
        val remotes = shards map { new RemoteIndex(_) }
        new CompositeIndex(remotes)
    }

    new SearchbirdServiceImpl(this, index)
  }
}

{% endhighlight %}

现在我们来调整参数，给SearchbirdServiceConfig（我们可以叫做从0到9000端口共享，通过9001端口共享0,等等）实例添加碎片初始化。

**config/development.scala**

{% highlight scala %}

new SearchbirdServiceConfig {
  // Add your own config here
  shards = Seq(
    "localhost:9000",
    "localhost:9001",
    "localhost:9002"
  )
  ...

{% endhighlight %}

注释掉admin.httpPort 的设置（我们不想好多服务都运行在同一台机器上，而所有的服务还试着打开相同的端口）：

{% highlight bash %}

  // admin.httpPort = 9900

{% endhighlight %}

现在如果我们不通过任何参数来调用我们的服务，它就会启动一个杰出的和所有给定碎片通讯的节点。如果我们指定一个碎片参数，它会在属于这个碎片索引的端口上启动一个服务。

让我们来试试吧！我们将运行3 个服务：2个碎片一个distinguished 节点。第一个编译改变。

{% highlight bash %}

$ ./sbt
> compile
...
> exit

{% endhighlight %}

运行3 个服务端：

{% highlight bash %}

$ ./sbt 'run -f config/development.scala -D shard=0'
$ ./sbt 'run -f config/development.scala -D shard=1'
$ ./sbt 'run -f config/development.scala'

{% endhighlight %}

你既可以把它们运行在3个不同的窗口或者（在一个窗口里）按顺序启动它们，等待它们启动，用ctrl + Z 来暂停，bg 来让它们在后台运行。

然后我们就会在控制台里和它们交互。首先，让我们在两个碎片节点里填充一些数据。在searchbird 路径里运行：

{% highlight bash %}

$ ./console localhost 9000
...
> client.put("fromShardA", "a value from SHARD_A")
> client.put("hello", "world")

{% endhighlight %}

{% highlight scala %}

$ ./console localhost 9001
...
> client.put("fromShardB", "a value from SHARD_B")
> client.put("hello", "world again")

{% endhighlight %}

你可以在你完成它们之后退出控制台会话。现在从distinguished 节点里查询我们的数据库（端口9999）：

{% highlight bash %}

$ ./console localhost 9999
[info] Running com.twitter.searchbird.SearchbirdConsoleClient localhost 9999
'client' is bound to your thrift client.

finagle-client> client.get("hello").get()
res0: String = world

finagle-client> client.get("fromShardC").get()
SearchbirdException(No such key)
...

finagle-client> client.get("fromShardA").get()
res2: String = a value from SHARD_A

finagle-client> client.search("hello").get()
res3: Seq[String] = ArrayBuffer()

finagle-client> client.search("world").get()
res4: Seq[String] = ArrayBuffer(hello)

finagle-client> client.search("value").get()
res5: Seq[String] = ArrayBuffer(fromShardA, fromShardB)

{% endhighlight %}

这个设计有多个允许更多模块和稳定实现的数据抽象：

*  ResidentIndex 数据结构对网络、服务或者客户端一无所知。
*  CompositeIndex 不知道consitituent indeices 是如何实现的或者它们相关的数据结构；它简单的给它们分布了一个请求
*  服务端相同的search 接口允许服务查询本地数据结构（ResidenIndex）或者在不需要知道对调用者隐藏的分别来分布查询其它服务（CompositeIndex）
*  SearchbirdServiceImpl 和Index 现在是分隔的模块了，允许一个简单的服务实现以及分隔服务和访问者的数据结构的实现
*  设计足够的稳定允许一个或者更多的远程indices，位于本地机器或者一个远程机器

对这个实现的可能的改善如下

*   当前的实现给所有的节点发送put() 调用。作为代替，我们可以使用一个hash 表来给一个节点和所有的分布存储节点发送一个put（） 调用。
    *   注意，不管怎样我们丢掉了这种方法的冗余。我们怎样获得一些redundancy 却不需要完整的复制？
*    我们没有处理系统里任何含有失败的感兴趣的信息（例如，我们没有处理任何异常）。

1.  local ./sbt 脚本简单的保证了sbt 版本是和我们已知的合适的可见类库一致
2.  在target/gen-scala/com/twitter/searchbird/SearchbirdService.scala
3.  见Ostrich 的[README](https://github.com/twitter/ostrich/blob/master/README.md) 的更多信息。

