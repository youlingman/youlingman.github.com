---
title: Trampoline和尾递归
date: 2017-11-08 20:00:35
tags: [javascript,functional]
---

### 前言

[nodeschool-Functional Javascript](https://github.com/timoxley/functional-javascript-workshop)的第13题实现了一个repeat函数，其接收两个参数，函数operation和整数num，然后利用递归对operation调用num次。当num很大时，js会报一个Maximum call stack size exceeded的错误，说明递归太深，调用栈超出限制，第14题的任务就是用一个**trampoline**函数去解决递归爆栈问题。然而这个trampoline到底是啥exercise里毛说明都木有。

<!--more-->

### 初探trampoline

抄一下[wiki](https://en.wikipedia.org/wiki/Trampoline_(computing)对trampoline的定义如下：

>As used in some Lisp implementations, a trampoline is a loop that iteratively invokes thunk-returning functions (continuation-passing style). A single trampoline suffices to express all control transfers of a program; a program so expressed is trampolined, or in trampolined style; converting a program to trampolined style is trampolining. Programmers can use trampolined functions to implement tail-recursive function calls in stack-oriented programming languages.

翻译过来就是，函数式语言里的trampoline一般是个循环，这个循环迭代地调用某些函数，这些函数会返回一个 **thunk** （？），这又被称为 **continuation-passing style** （？）。然后码农可以在基于栈的编程语言里利用这个trampoline实现**尾递归**函数。

### thunk

继续搬运[wiki](https://en.wikipedia.org/wiki/Thunk_(functional_programming))的对应内容，thunk起源于call-by-name求值策略，call-by-name指函数的参数在函数被调用（该参数被实际访问到）时才对参数求值，而一般实现上会在表达式求值时将call-by-name参数出现的地方作类似inline展开的操作，这样会导致存在参数表达式的多份副本。因此对call-by-name参数，编译器会生成一个helper把参数表达式塞进去，用来统一计算并返回该变量的值，这个helper就是thunk，在此基础上编译器还可以将thunk的计算结果缓存起来消除重复计算。

我们也可以显示地生成thunk，方法就是**将参数表达式用无参匿名函数包起来**。

### continuation-passing style

这个概念不大好理解，[wiki](https://en.wikipedia.org/wiki/Continuation-passing_style)的说法continuation-passing style和direct style相对应，continuation-passing style的函数会接收一个额外的函数变量continuation，direct style的函数执行完毕时会将结果return给调用方，而continuation-passing风格的函数执行完毕时则将结果作为参数传入并调用continuation变量对应的函数。

下面列一下wiki上两种风格函数的实现。

```javascript
direct style
function pyth(x, y) {
	return Math.sqrt(x * x + y * y);
}

continuation-passing style
function pyth(x, y, continuation) {
	(function(con1) {
		con1(x * x);
	})(function(x2) {
		(function(x2, con2) {
			con2(y * y);
		})(x2, function(y2) {
			(function(x2, y2, con3) {
				con3(x2 + y2);
			})(x2, y2, function(x2py2) {
				(function(x2py2, con4) {
					con4(Math.sqrt(x2py2));
				})(x2py2, continuation);
			})
		})
	})
}
```

实现了下wiki实例代码的javascript版本，试着理解CPS的思路，大概是将逻辑/控制的语句/块封成一个个函数，由于上一个函数结束时会将结果传递到下一个函数，分散的逻辑/控制块就可以被串成线性的逻辑/控制流。[Continuation的wiki](https://en.wikipedia.org/wiki/Continuation)提到continuation实质上是函数式版本的goto。

由于CPS函数的最后一步永远是函数调用（调用传入的新函数或者调用自身），这种情况被称为尾调用（如果是调用自身则叫做尾递归）。而在CPS的wiki里Use and implementation节找到下面这句话。

>Without tail-call optimization, techniques such as trampolining, i.e. using a loop that iteratively invokes thunk-returning functions, can be used;

就是说，trampoline这个东西可以用来代替尾调用优化的作用，复习下尾调用和尾调用优化的概念。

### 尾调用优化

回忆下本科学过的内容，调用函数时会将当前指令地址、当前环境上下文的变量、目标函数的参数压栈，目标函数返回后再弹栈恢复上下文并继续执行下一条指令。因此如果调用函数层次太深就会导致压栈内容过多而爆栈。

而对于尾调用来说，由于本函数的最后一步操作还是函数调用，因此将当前调用位置和上下文相关变量压栈就变得没有必要了（本次调用完毕后直接返回再上一层的调用位置，同时本函数不会再访问或者修改当前上下文变量），因此对尾调用来说只需要将目标函数的参数压栈然后跳转到目标函数处即可，如果调用过程中所有函数调用都是尾调用（尾递归即为典型），就能实现在调用过程中仅用一块栈帧保存所有中间目标函数的参数，从而大大优化栈内存使用情况。这就是尾调用优化的思路。

### trampoline

最后逐条回顾一下上面的内容。

- CPS函数是一种在执行完毕时进行一次尾调用的函数，尾调用的目标函数为传入的参数；
- 对CPS函数的调用，本质上是用尾调用将一系列函数块串起来的逻辑流；
- 这一系列函数的数量对应着一次CPS调用的最大调用栈深度，如果待调用的函数很多（对递归函数的调用来说很有可能），而又没有相应尾调用优化机制，就会造成爆栈；
- 对于尾递归来说，其调用栈是通过迭代地调用自身来构造的；
- trampoline是一种通过迭代地调用特定函数的方式，可以用来实现尾递归，这些特定函数会返回一个thunk，即无参匿名函数；

到这里trampoline的思路就比较清楚了。尾递归函数通过在执行结束时**同步地**调用自身，利用函数调用栈迭代地构造和遍历串行函数流；而对应的trampoline版本，则改写了尾递归函数，在函数执行完毕时将下一步调用自身的操作封装成一个thunk并返回，对应迭代调用自身的操作交由上一层的循环统一处理。以这道exercise14为例，solution的核心思路就是：

>把尾递归函数改写成一个对应串行函数流的‘迭代器’，trampoline遍历这个‘迭代器’实现对串行函数流的遍历调用。这个‘迭代器’代替了函数调用栈的作用，因此解决了调用尾递归函数时调用栈超限的问题。

```javascript
// native recursive
function repeat(operation, num) {
    // Modify this so it doesn't cause a stack overflow!
	if (num <= 0) return;
	operation();
	repeat(operation, --num);
}

// with trampoline
function repeat(operation, num) {
    // Modify this so it doesn't cause a stack overflow!
	if (num <= 0) return;
	operation();
    return function(){
		return repeat(operation, --num);
	};
}

function trampoline(fn) {
    // You probably want to implement a trampoline!
    while(fn && typeof fn === 'function') {
		fn = fn();
	}
}

module.exports = function (operation, num) {
    // You probably want to call your trampoline here!
    return trampoline(repeat(operation, num));
}
```