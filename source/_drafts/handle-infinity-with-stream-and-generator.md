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

在scala中，通过Stream可以获取对无穷序列的表达能力，这里的表达能力是指通过将序列的递推生成算法映射到Stream对象的构造逻辑，然后就可以指着这个Stream对象说它就是一个对应的无穷序列。下面从[Stream docs](https://www.scala-lang.org/api/current/scala/collection/immutable/Stream.html)抄了fibonacci序列的对应Stream对象（#::改成cons以便理解）。

```scala
scala> val fibs:Stream[Int] = Stream.cons(0, Stream.cons(1, fibs.zip(fibs.tail).map { n => n._1 + n._2 }))
fibs2: Stream[Int] = Stream(0, ?)
```

这里就不重复上一篇的源码剖析思路去分析Stream原理了，这里关注的点是，基于延迟计算和高阶函数，scala的Stream可以从较高的抽象层面去表达无穷序列（直接将递推算法映射到Stream对象里）。

### generator：迭代器、协程



[generator](https://en.wikipedia.org/wiki/Generator_(computer_programming))