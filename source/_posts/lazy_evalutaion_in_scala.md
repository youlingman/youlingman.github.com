---
title: scala中的延迟求值
date: 2017-02-15 19:57:30
tags: [scala]
---
在做[scala99](http://aperiodic.net/phil/scala/s-99/)的P91 Knight's tour时，需要递归搜索整个解题空间找到所有可能的解，如果只想要得到部分解，就要先算出全量解再作筛选，这样就要作多余的运算了。在其他编程语言一般利用控制流来实现，即利用if-else条件判断和return返回提前终止搜索过程，而在scala里更functional的写法是利用延迟求值，这里整理下相关的概念。
<!--more-->

### 求值策略

表达式本身是递归定义的，一个表达式一般由子表达式、运算符、保留关键字等构成，所以对表达式的求值也需要递归去执行。SICP里提到两种求值策略，应用序 (applicative-order) 求值和正则序 (normal-order) 求值，如果将表达式展开成一棵符号树，这两种不同的求值策略对应了对符号树的不同的遍历和规约策略。

- 应用序 (applicative-order) ：以类似depth-first的搜索策略，展开表达式后求所有子表达式的值；
- 正则序 (normal-order) ：以类似breadth-first的搜索策略，展开表达式，展开时只作参数的替换，只有当到达符号树的叶子节点时才执行求值/规约；

在scala中，对函数的参数求值，对应应用序和正则序有call by value和call by name两种求值策略，默认使用call by value策略。根据维基的[Evaluation_strategy](https://en.wikipedia.org/wiki/Evaluation_strategy)，call by value是一种strict/eager求值，可以认为是在函数被调用时，函数的每个参数都会被求值，而call by name是一种non-strict/lazy求值，在函数被调用时将函数的参数展开，而参数的具体求值动作则仅在需要时进行，同时如果同一个参数被传入了多次，可能出现多次求值的情况。

在scala 2.11.7上，分别实现参数为call by value和call by name的函数如下：

```scala
class SimpleTest {
  def callByValue(a : Int) = a
  def callByName(a : => Int) = a
}
```

反编译后得到的java代码如下：

```java
public int callByValue(int a)
{
  return a;
}
public int callByName(Function0<Object> a)
{
  return a.apply$mcI$sp();
}
```

可以看出call by name的参数被包裹成一个Function0对象，而对该参数的使用则转化成对该包裹函数的调用，表达式展开求值时传入的Function0对象自然不会被展开求值，被包裹的逻辑要到该函数对象被实际调用时才会执行，利用函数占位来代替参数就地展开求值达到call by name的效果。

看起来scala里的call by name语法和无参匿名函数的语法很相似，试试以无参函数作为参数，实现一个伪call by name的函数如下：

```scala
def fakeCallByName(a : () => Int) = a.apply
```

反编译结果如下：

```java
public int fakeCallByName(Function0<Object> a)
{
  return a.apply$mcI$sp();
}
```

无参函数作为参数和call by name参数反编译出来的代码是等价的，感觉scala利用了函数作为语法糖来实现call by name。

### var/val和def

有些文章提到val/var和def前缀都可以用来定义变量，其中def利用了call by name，如以下代码：

```
scala> val v_val = {
     |     println("println in val")
     |     2
     |   }
println in val
v_val: Int = 2

scala>   var v_var = {
     |     println("println in var")
     |     2
     |   }
println in var
v_var: Int = 2

scala>   def v_def = {
     |     println("println in def")
     |     2
     |   }
v_def: Int

scala> v_def
println in def
res0: Int = 2

scala> v_def
println in def
res0: Int = 2
```

在实际执行时，"println in def"只有到v_def被实际使用到时才会打印出来，且每次使用到v_def都会执行println语句，而"println in val"和"println in var"则在执行到对应变量赋值语句时就会马上打印出来，反编译得到java代码如下：

```java
public class SimpleTest
{
  private final int v_val;
  private int v_var;
  
  public int v_val()
  {
    return this.v_val;
  }
  
  public void v_var_$eq(int x$1)
  {
    this.v_var = x$1;
  }
  
  public int v_var()
  {
    return this.v_var;
  }
  
  public SimpleTest()
  {
    Predef..MODULE$.println("println in val");this.v_val = 
      2;
    

    Predef..MODULE$.println("println in var");this.v_var = 
      2;
  }
  
  public int v_def()
  {
    Predef..MODULE$.println("println in def");
    return 2;
  }
}
```

var和val变量的右值表达式执行和变量赋值放在的当前类的构造函数内，而def定义的‘变量’则没有对应的java变量，而是把右值表达式放在一个函数内执行，并将函数的返回值作为‘变量’的值。

总结一下，scala里的call by name的思路是将参数/变量的表达式放在一个函数里，变成一个类似“生成器”的东西，因此表达式的执行可以延迟到函数被调用的时候，实现延迟求值。

### lazy关键字

scala里的lazy关键字也可以实现延迟求值的效果，下面这一段代码：

```
scala> def v_def = {
     |     println("println in def")
     |     2
     |   }
v_def: Int

scala>   lazy val v_lval = {
     |     println("println in def")
     |     2
     |   }
v_lval: Int = <lazy>

scala> v_def
println in def
res0: Int = 2

scala> v_def
println in def
res1: Int = 2

scala> v_lval
println in def
res2: Int = 2

scala> v_lval
res3: Int = 2
```

lazy修饰的val变量也可以实现和def类似的延迟求值效果，不同在于v_def对应的表达式在每次被用到时都会执行一次，而v_lval对应的表达式则只会在第一次被用到时被执行，继续看看反编译出来的java代码：

```java
public class SimpleTest
{
  private int v_lval;
  private volatile boolean bitmap$0;
  
  public int v_def()
  {
    Predef..MODULE$.println("println in def");
    return 2;
  }
  
  public int v_lval()
  {
    return this.bitmap$0 ? this.v_lval : v_lval$lzycompute();
  }
  
  private int v_lval$lzycompute()
  {
    synchronized (this)
    {
      if (!this.bitmap$0)
      {
        Predef..MODULE$.println("println in lazy val");this.v_lval = 2;this.bitmap$0 = true;
      }
      return this.v_lval;
    }
  }
}
```

用lazy关键字修饰的变量，scala编译器为该变量自动生成了一个类似double-check单例的机制，其中包括一个bitmap位图和lzycompute方法，当变量被用到时，利用bitmap位图中对应的bit来确定该变量是否已赋值，进而决定是调lzycompute方法执行对应表达式并赋值还是直接返回变量的值，一个类里利用一个bitmap位图来统一控制类里所有lazy变量的延迟执行。而由于lazy修饰的变量只能被初始化一次，用来修饰var变量是没有意义的，编译器直接会报错：

```
scala> lazy var b = 2
<console>:1: error: lazy not allowed here. Only vals can be lazy
lazy var b = 2
     ^
```

写不动了，下一篇再整理下Stream类。