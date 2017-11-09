---
title: scala中的Stream
date: 2017-07-01 19:57:30
tags: [scala,functional]
---
在之前[scala中的延迟求值](http://youlingman.info/2017/02/15/lazy_evalutaion_in_scala/)中提到，在使用递归遍历的方法解[骑士巡游问题](https://en.wikipedia.org/wiki/Knight%27s_tour)时，可以利用延迟求值的方法来获取需要的部分解（而不用依赖if-else等条件控制流），scala提供了一个lazy list来支持按需计算，称为Stream，这里整理一下相关用法和原理。
<!--more-->

### 问题简化

对骑士巡游问题作一下抽象，可以将问题描述为“对一个列表（解题空间，骑士所有可能的走法）进行筛选（是否填满棋盘/出发点和结束点是否一致）”，这里将问题简化成同构的问题“对一个列表（整数列表）进行筛选（是否素数）”。

首先先实现一个判断素数函数，并随机生成一个长度为50的整数列表。

```scala
scala> def isPrime(n : Int): Boolean = n == 2 || List.range(2, n / 2 + 1).forall(i => n % i != 0)
isPrime: (n: Int)Boolean

scala> val list = Seq.fill(50)(Random.nextInt(1000))
list: Seq[Int] = List(403, 615, 930, 197, 599, 841, 439, 153, 655, 809, 861, 942, 858, 381, 299, 550, 32, 343, 542, 146, 872, 522, 23, 569, 959, 165, 631, 438, 444, 589, 634, 997, 145, 675, 873, 155, 51, 326, 610, 623, 938, 343, 442, 728, 364, 0, 866, 188, 892, 495)
```

然后对列表作filter。

```scala
scala> list.filter(isPrime).take(5)
res6: Seq[Int] = List(197, 599, 439, 809, 23)

scala> list.filter(isPrime)
res5: Seq[Int] = List(197, 599, 439, 809, 23, 569, 631, 997, 0)
```

可见列表中一共有9个素数，如果只需要获取前5个素数，则后面4个素数的筛选都是不必要的。而使用Stream可以免去后续的多余计算，实现按需计算的效果。

### 按需计算

要使用Stream，首先可以将List转成Stream，Stream和List用的同一套列表操作API，后续操作也和List版本一致。

```scala
scala> val stream = list.toStream
stream: scala.collection.immutable.Stream[Int] = Stream(403, ?)

scala> stream.filter(isPrime)
res7: scala.collection.immutable.Stream[Int] = Stream(197, ?)

scala> stream.filter(isPrime).take(5)
res8: scala.collection.immutable.Stream[Int] = Stream(197, ?)
```

可以看到，List转成Stream后得到的是一个Stream对象 Stream(403, ?)，而执行完筛选素数并取前5个数后的到的仍然是Stream对象 Stream(197, ?)，从中仅能得到第一个素数，后面的仍然是问号。这时候回头看看stream：

```scala
scala> stream
res9: scala.collection.immutable.Stream[Int] = Stream(403, 615, 930, 197, ?)
```

stream也是仅仅看到原始列表到第一个素数为止的内容，其后同样是问号。下面我们将前5个素数打印出来再回头看看差异。

```scala
scala> stream.filter(isPrime).take(5).foreach(println)
197
599
439
809
23

scala> stream
res11: scala.collection.immutable.Stream[Int] = Stream(403, 615, 930, 197, 599, 841, 439, 153, 655, 809, 861, 942, 858, 381, 299, 550, 32, 343, 542, 146, 872, 522, 23, ?)
```

当把前5个素数打印出来后，stream中的数据也恰恰仅到第5个素数为止。这里就能看出按需计算的效果，Stream仅对获取到的内容（前5个素数）进行了计算，而没有计算到的部分仍然是问号。接下来进去源码看一下实现原理。

### 看看源码

首先从List的toStream方法开始，看看Stream对象的创建。

```scala
  override def toStream : Stream[A] =
    if (isEmpty) Stream.Empty
    else new Stream.Cons(head, tail.toStream)
```

先不关心Empty，看看Stream helper中的Cons方法。

```scala
  /** A lazy cons cell, from which streams are built. */
  @SerialVersionUID(-602202424901551803L)
  final class Cons[+A](hd: A, tl: => Stream[A]) extends Stream[A] {
    override def isEmpty = false
    override def head = hd
    @volatile private[this] var tlVal: Stream[A] = _
    @volatile private[this] var tlGen = tl _
    def tailDefined: Boolean = tlGen eq null
    override def tail: Stream[A] = {
      if (!tailDefined)
        synchronized {
          if (!tailDefined) {
            tlVal = tlGen()
            tlGen = null
          }
        }

      tlVal
    }
```

Cons类继承了Stream，然后有两个参数hd和tl，hd为call-by-value，对应列表的head，tl为call-by-name，对应列表的tail部分，可以看出，hd的值在对象创建时就会进行计算，而tl的值，只有在被访问到时才会进行计算，但是tl的值在哪里会被访问到呢？自然是在tail方法里面。

由于call-by-name的参数存在**每次访问都会重复计算**的问题，这里引入了tlVal、tlGen变量，var tlGen = tl _ 的写法其实是将call-by-name参数tl转成对应的匿名无参函数并赋值给tlGen，以使得变量tlGen也具有call-by-name的特性，这种方法貌似被称为[Thunk](https://en.wikipedia.org/wiki/Thunk#Functional_programming)。控制重复计算的关键是tailDefined的返回值，假定tl不为null，则初始tailDefined为false，而tail方法被访问过后，会计算tl的值并赋给tlVal，然后将tlGen置为null，其后tailDefined将变成true。tail方法中还使用了synchronized和double-check来保证线程同步。

换言之，**tl的值只有在tail方法被调用过才会进行执行/计算，这就是Stream支持按需计算的由来**。而tailDefined表示当前Stream的tail是否已经被访问/计算过，看看Stream.addString方法中的对应逻辑就能知道之前Stream中的问号从何而来。

```scala
override def addString(b: StringBuilder, start: String, sep: String, end: String): StringBuilder = {
	...
      if (!cursor.isEmpty) {
        // Either undefined or cyclic; we can check with tailDefined
        if (!cursor.tailDefined) b append sep append "?"
        else b append sep append "..."
      }
	...
}

```

再看看filter和take的实现，就大概清楚Stream实现延迟计算的思路了。

```scala
  /** An alternative way of building and matching Streams using Stream.cons(hd, tl).
   */
  object cons {

    /** A stream consisting of a given first element and remaining elements
     *  @param hd   The first element of the result stream
     *  @param tl   The remaining elements of the result stream
     */
    def apply[A](hd: A, tl: => Stream[A]) = new Cons(hd, tl)

    /** Maps a stream to its head and tail */
    def unapply[A](xs: Stream[A]): Option[(A, Stream[A])] = #::.unapply(xs)
  }

  override def take(n: Int): Stream[A] = (
    // Note that the n == 1 condition appears redundant but is not.
    // It prevents "tail" from being referenced (and its head being evaluated)
    // when obtaining the last element of the result. Such are the challenges
    // of working with a lazy-but-not-really sequence.
    if (n <= 0 || isEmpty) Stream.empty
    else if (n == 1) cons(head, Stream.empty)
    else cons(head, tail take n-1)
  )

  override private[scala] def filterImpl(p: A => Boolean, isFlipped: Boolean): Stream[A] = {
    // optimization: drop leading prefix of elems for which f returns false
    // var rest = this dropWhile (!p(_)) - forget DRY principle - GC can't collect otherwise
    var rest = this
    while (!rest.isEmpty && p(rest.head) == isFlipped) rest = rest.tail
    // private utility func to avoid `this` on stack (would be needed for the lazy arg)
    if (rest.nonEmpty) Stream.filteredTail(rest, p, isFlipped)
    else Stream.Empty
  }

  private[immutable] def filteredTail[A](stream: Stream[A], p: A => Boolean, isFlipped: Boolean) = {
    cons(stream.head, stream.tail.filterImpl(p, isFlipped))
  }
```

可以看出，take和filter的返回仍然是一个Stream/Cons，在创建返回Cons对象时，首先计算得到head的值作为hd参数，然后将计算tail的递归逻辑作为tl参数，而tl的值只有在tail方法被调用过才会进行执行/计算，因此在Stream上调用take和filter方法仍然能保持按需计算的特性。当然，如果连head的计算也想推迟，可以使用lazy val变量来声明Stream对象。

### 总结

在[另外一篇分析Stream的博文](http://cuipengfei.me/blog/2014/10/23/scala-stream-application-scenario-and-how-its-implemented/)的总结里，将List和Stream作了对比，提到“Stream构造的容器，其中不包含数据，包含的时能够生产数据的算法”，感觉挺有道理，不过根据上面的分析，Stream其实同时包括数据（head和已经被访问过的tail）和产生数据的算法（没被访问过的tail），当所有元素都被访问到之后，Stream其实和List就没什么太大的差别了。

想了想之前学习的spark，其实Stream和spark里的RDD倒是有着类似的设计思路。
> 
> **Stream**

- 获取数据：通过toStream将其它容器的数据导入到Strean内，或者利用递归定义的序列生成算法构造新的Stream；
- Stream API：使用Stream的API对数据进行操作，返回新的Stream对象，此时不会触发tail对应的数据计算逻辑；
- 非Stream API：使用非Stream的API获取/访问容器内的数据，此时可能会触发新的数据计算过程。

> 
> **spark RDD**

- 获取数据：从外部数据空间将数据输入spark作为RDD；
- transformation（转换）：在RDD上进行transformation操作，返回新的RDD，此时不会执行计算；
- action（执行）：触发Spark作业的运行，真正触发transformation转换算子的计算流程，并得到结果数据。

最后，由于延迟计算特性，Stream拥有可以表达无限序列的能力，后面打算和Python的generator作一下对比。