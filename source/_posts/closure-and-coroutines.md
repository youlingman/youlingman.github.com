---
title: 闭包与协程
tags: functional
date: 2020-05-17 19:48:14
---


最近学习[SICP Python](https://wizardforcel.gitbooks.io/sicp-py/content/)（感谢译者大大），看到第五章序列和协程部分，发现和之前写的[scala stream](http://icyl.rocks/2017/07/01/stream-in-scala/)和[无穷序列](http://icyl.rocks/2017/08/01/handle-infinity/)的学习记录基本一致，当时立了个flag说后续学习下协程，这里尝试补一下坑。
<!--more-->

# 闭包

首先从函数说起，纯函数本身是无状态的，其接收输入，完成其中的逻辑处理，最后返回输出。对于相同的输入，同一个函数总是返回相同的输出（这里暂时不考虑side effect）。

对于支持函数式的编程语言，即函数是“first class一等公民”，函数本身可以作为函数的返回，基于函数的封装和传递，可以完成对程序控制流的函数式抽象，当然无状态的函数表现能力比较弱，需要依赖输入作为状态信息，就是说调用函数方仍然需要维护相关上下文状态信息，是否可以直接在函数上绑定状态信息呢？

闭包的作用就是为无状态的函数绑定额外状态信息。基于函数的词法作用域，函数被传递出去后仍然可以保持对定义时变量的访问能力，这就可以实现在函数上增加额外上下文状态信息。这个用法在JavaScript中十分常见，通过在函数内（outter）定义函数（inner）并返回，后续inner函数可以保持对outter函数作用域内对应变量的访问能力，下面用scala作为示例，实现在传递出去的inner函数上绑定变量a。

```scala
scala> def outter():() => Unit = { var a = 10;  def inner():Unit = { a = a + 1; println(a); }; return inner; }
outter: ()() => Unit

scala> var innerF = outter()
innerF: () => Unit = $$Lambda$1079/2053491093@6806468e

scala> innerF()
11

scala> innerF()
12
```

注意Python对闭包和匿名函数的支持是不完备的，同样的闭包实现在Python中直接报错。用对函数支持不完备的Python来讲SICP也是挺魔幻的……

```python
>>> def outter():
...    a = 10
...    def inner():
...      a = a + 1
...      println(a)
...    return inner
...
>>> innerF = outter()
>>> innerF()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 4, in inner
UnboundLocalError: local variable 'a' referenced before assignment
```

虽然Python的函数不支持闭包，但是可以利用Python特有的生成器函数来实现类似的功能，生成器函数可以保存函数执行的当前上下文，实现可重入函数的效果，生成器的yield和next/send对应普通函数的返回和调用，差别在于普通函数的调用中断点和重入点在函数的return语句和函数的起始句，而生成器函数的中断点和重入点都在函数的yield语句处。

# 协程

按照SICP Python的说法，协程是一种任务处理方式，其将任务分解成独立的函数步骤，然后不依赖于额外的协调组件，协程可以自发链接到一起来组成流水线，完成输入、执行和输出流程。

按课程中介绍的协程 in Python way，利用生成器函数作为执行子任务的协程，通过yield/send完成子任务之间的同步，最后实现任务流的串联执行。

每个协程可能是如下三个角色：

* 生产者创建序列中的物品，并使用send()，而不是(yield)。
* 过滤器使用(yield)来消耗物品并将结果使用send()发送给下一个步骤（传入的下一个协程/生成器）。
* 消费者使用(yield)来消耗物品，但是从不发送。

考虑到生成器这个东西是Python自己的专有概念，如果把这套协程机制按functional way的角度来看，本质上可以理解为是基于闭包传递的continuation-passing style模式，解决的问题是对任务的子任务/函数划分和任务拓扑流的执行。

1. 首先，将复杂任务解构划分为一个个独立的子任务/函数，利用闭包的方式绑定相关上下文状态到函数上，以支持带状态的可重入调用；
2. 按照任务执行流程，对每个子任务确定其后续执行的子任务，下一步执行的实际函数（们）可以通过定义当前子任务函数时确认，也可以在实际执行时通过上一步子任务传入；
3. 当每个子任务的后续操作都确定之后，整体的任务拓扑流程图也随之完整，这时从拓扑图的起始节点开始执行就可以驱动整体任务的执行。

所以，协程其实是任务执行场景的一种工具/编程模式，其抽象层面相对于进程/线程要更高。从工具来说，可重入的子例程都可以称为协程，因此闭包函数也可以作为协程。从任务执行模式来说，协程可以实现复杂任务的划分和执行，而这套机制和continuation-passing style模式似乎是一致的。