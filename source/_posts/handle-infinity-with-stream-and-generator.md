---
title: 无穷序列：Stream与generator
date: 2017-08-01 19:57:30
tags: [scala,python]
---
[上一篇介绍stream](http://youlingman.info/2017/07/01/stream_in_scala/)里提到，scala中的Stream可以利用其延迟计算的特性表达和处理无穷序列，而python中则提供了一种称为generator生成器的机制。下面讨论下stream和generator如何表达无穷序列。

### 无穷序列的生成/处理

如何生成一个无穷序列，可以有以下几种方式：

1. 重复某（几）个元素生成序列；
2. 通过随机生成每个元素生成序列；
3. 通过特定的递推算法生成序列；

而重复和随机元素都可以归为一种特定的递归算法，下面就以递推生成序列为例。

当然，任何程序都无法对无穷进行整体操作，一般对无穷序列的处理，主要是对无穷序列进行相应的筛选/截取，以得到想要的有穷数据/信息。比如 [Fibonacci number](https://en.wikipedia.org/wiki/Fibonacci_number) 就是一个无穷大的序列，对这个序列整体进行处理是不可用的，比如统计序列长度、计算均值等等，我们一般会进行的操作是获取前十个数、获取在[100,200]区间内的fibonacci数等等。

生成一个fibonacci序列可以通过一个[递推](https://en.wikipedia.org/wiki/Corecursion)（这个wiki挺有意思）算法来完成，下面是获取第n个fibonacci数的递推算法。

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
<!--more-->
### Stream：函数式编程、延迟计算、不可变容器

在scala中，通过Stream可以获得对无穷序列的表达能力，这里的表达能力是指通过将序列的递推生成算法映射到Stream对象的构造逻辑，然后就可以指着这个Stream对象说它就是一个对应的无穷序列。下面从[Stream docs](https://www.scala-lang.org/api/current/scala/collection/immutable/Stream.html)抄了fibonacci序列的对应Stream（#::改成cons以便理解）。

```scala
scala> val fibs:Stream[Int] = Stream.cons(0, Stream.cons(1, fibs.zip(fibs.tail).map { n => n._1 + n._2 }))
fibs2: Stream[Int] = Stream(0, ?)
```

不重复上一篇的源码剖析思路去分析Stream原理了，这里关注的点是，基于延迟计算和高阶函数，scala的Stream可以从较高的抽象层面去表达和处理无穷序列（直接将递推算法映射到Stream的构造逻辑，生成序列对应的容器）。

而在命令式编程语言里，一般的思路则是通过可变量和状态流（if-else-return）来实现对应递推进而对无穷序列进行生成和处理。

### generator：迭代器、延迟计算？协程？

在[scala Stream docs](https://www.scala-lang.org/api/current/scala/collection/immutable/Stream.html)里提到，当Stream被声明为val变量（而不是函数）时，其会缓存head（以及其它被访问过的元素），某些操作可能会在返回结果前产生大量的中间过程数据，从而占用大量内存。如果这些缓存是不必要的，docs推荐尽量使用迭代器。继续从docs里抄Fibonacci序列的scala迭代器版本。

```scala
val it = new Iterator[Int] {
  var a = 0
  var b = 1
  def hasNext = true
  def next(): Int = { val c = b; b = a + b; a = c; a }
}

scala> it.next
res0: Int = 1

scala> it.next
res1: Int = 1

scala> it.next
res2: Int = 2

scala> it.next
res3: Int = 3
```

这里就引出了生成无穷序列的另外一个思路：迭代器。不同于函数式编程的思路，迭代器更关注怎么做（how），而不是做什么（what），这里存在所谓的[声明式编程](https://en.wikipedia.org/wiki/Declarative_programming)和[命令式编程](https://en.wikipedia.org/wiki/Imperative_programming)的[差异](https://en.wikipedia.org/wiki/Comparison_of_multi-paradigm_programming_languages)。总而言之，迭代器从过程的角度，引入可变量来保存每步递推的上下文，利用控制流来重复递推逻辑。

在Python里，实现了迭代器协议的对象都可以被当作迭代器（[鸭子类型](https://en.wikipedia.org/wiki/Duck_typing)），迭代器协议包括两个方法（Python3）：

- __iter__()，返回迭代器对象自身；
- __next__()，返回迭代器的下一个元素，如果不存在下一个元素则抛出一个StopIteration异常；

在scala中自定义一个迭代器只要实例化一个支持Iterator trait的匿名类就好，而不支持匿名类的python里则没有这么方便了，于是python提供了一个**快速自定义迭代器的语法糖**，generator生成器。下面尝试利用生成器得到fibonacci序列的迭代器。

```python
>>> def fib():
...     a, b = 0, 1
...     while Tru:
...         yield a
...         a, b = b, a + b
...
>>> fibs = fib()
>>> fibs
<generator object fib at 0x0000000002BBBA68>
>>> next(fibs)
0
>>> next(fibs)
1
>>> next(fibs)
1
>>> next(fibs)
2
```

在一个函数内用到yield关键字，函数会返回一个生成器对象，这个生成器对象实现了迭代器协议，可以认为是一个迭代器。下面是Python2打出来的生成器对象方法。

```python
>>> help(fibs)
Help on generator object:

fib = class generator(object)
 |  Methods defined here:
 |
 |  __getattribute__(...)
 |      x.__getattribute__('name') <==> x.name
 |
 |  __iter__(...)
 |      x.__iter__() <==> iter(x)
 |
 |  __repr__(...)
 |      x.__repr__() <==> repr(x)
 |
 |  close(...)
 |      close() -> raise GeneratorExit inside generator.
 |
 |  next(...)
 |      x.next() -> the next value, or raise StopIteration
 |
 |  send(...)
 |      send(arg) -> send 'arg' into generator,
 |      return next yielded value or raise StopIteration.
 |
 |  throw(...)
 |      throw(typ[,val[,tb]]) -> raise exception in generator,
 |      return next yielded value or raise StopIteration.
```

最后讨论两个问题，生成器是否使用到延迟计算？生成器是否是一种协程？

第一个问题，生成器本身是一个迭代器，一方面，迭代器和序列/列表并不完全对等，实际还需要额外的逻辑才能从迭代器得到对应的列表数据，而另一方面，如果只针对单次遍历/访问序列元素而言，迭代器的确能起到按需计算的效果（虽然是阅后即焚），这种场景下迭代器可以认为是一种lazy sequence。

第二个问题，引出协程的概念，对这个概念还是一知半解，后续再学习。
