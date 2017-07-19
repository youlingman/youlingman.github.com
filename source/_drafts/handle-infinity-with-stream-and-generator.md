---
title: 处理无穷：Stream与generator
tags: [scala,python]
---
[上一篇介绍stream](http://youlingman.info/2017/07/01/stream_in_scala/)里提到，scala中的Stream可以利用其延迟计算的特性表达和处理无穷序列，而python中则提供了一种称为generator生成器的机制。

### 无穷序列的表达/处理

当然，任何程序都无法对无穷进行可用的整体操作，而一般对无穷序列的处理，主要是对无穷序列进行相应的筛选/截取，以得到想要的有穷数据/信息。比如 [Fibonacci number](https://en.wikipedia.org/wiki/Fibonacci_number) 就是一个无穷大的序列，对这个序列整体进行处理是不可用的，比如统计序列长度、计算均值等等，我们一般会进行的操作是获取前十个数、获取在[100,200]区间内的fibonacci数等等。

生成一个fibonacci序列是通过[递推](https://en.wikipedia.org/wiki/Corecursion)（这个wiki挺有意思）算法来完成，下面是fibonacci序列的递推算法。

```python
def fib(n):
	if n==1 or n==2:
		return 1
	return fib(n-1)+fib(n-2)
```

而获取前五个fibonacci数最直接的算法，就是直接通过递推算法逐个获取对应的fibonacci数（废话）。

```python
>>> fibs = []
>>> for i in range(1, 6):
...   fibs.append(fib(i))
...
>>> fibs
[1, 1, 2, 3, 5]
```

### Stream：函数式、延迟计算

在scala中，通过Stream可以获得对无穷序列的表达能力，这里的表达能力是指通过将序列的递推生成算法映射到Stream对象的构造逻辑，然后就可以指着这个Stream对象说它就是一个对应的无穷序列。下面从[Stream docs](https://www.scala-lang.org/api/current/scala/collection/immutable/Stream.html)抄了fibonacci序列的对应Stream对象（#::改成cons以便理解）。

```scala
scala> val fibs:Stream[Int] = Stream.cons(0, Stream.cons(1, fibs.zip(fibs.tail).map { n => n._1 + n._2 }))
fibs2: Stream[Int] = Stream(0, ?)
```

这里就不重复上一篇的源码剖析思路去分析Stream原理了，这里关注的点是，基于延迟计算和高阶函数，scala的Stream可以从较高的抽象层面去表达和处理无穷序列（直接将递推算法映射到Stream对象里）。

而在非函数式语言里，一般的思路则是通过迭代和状态流（if-else-return）来实现对无穷序列的处理。

### generator：迭代器、协程

在[Stream docs](https://www.scala-lang.org/api/current/scala/collection/immutable/Stream.html)里提到，当Stream被定义为val变量（而不是函数）时，其会缓存head（以及其它被访问过的元素），某些操作可能会在返回结果前产生大量的中间过程数据，从而占用大量内存。如果这些缓存是不必要的，docs推荐尽量使用迭代器。下面是Fibonacci序列的scala迭代器版本。

```scala
val it = new Iterator[Int] {
  var a = 0
  var b = 1
  def hasNext = true
  def next(): Int = { val c = b; b = a + b; a = c; a }
}
```

这里就引出了生成无穷序列的另外一个思路：迭代器。不同于函数式编程的思路，迭代器从过程的角度来实现每一步的递推，引入可变量来保存每步递推的上下文，利用控制流（hasNext = true对应while true）来重复递推逻辑。




[generator](https://en.wikipedia.org/wiki/Generator_(computer_programming))

```python
>>> def fib():
...   a = 0
...   b = 1
...   while True:
...     c = a + b
...     a = b
...     b = c
...     yield a
...
>>> fibs = fib()
>>> fibs
<generator object fib at 0x0000000002BBBA68>
```