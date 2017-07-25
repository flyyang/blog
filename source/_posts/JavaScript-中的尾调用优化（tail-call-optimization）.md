---
title: JavaScript 中的尾调用优化（tail call optimization）
date: 2017-07-24 23:21:24
tags: JavaScript
---
# JavaScript 中的尾调用优化(tail call optimization)

我在学习尾调用优化的过程中，有两个误解:

第一个是，我们一谈优化，经常说时间的优化。但是尾调用优化却主要是指空间的优化。

第二个是，既然尾调用优化是在 es6 中支持的，那么可能又要学新的语法了。然而，尾调用优化并不需要新的语法，而只是是在解释器（如V8）中做的改进。尾调用是一直存在的，但是尾调用优化是在支持 es6 的解释器里添加的。

澄清了这两个问题之后，我们先来看看尾调用是什么。

<!--more-->

## 尾调用

尾调用，从定义上来看很简单，简而言之是函数里的最后一个动作是函数调用。

从上面了解到，尾调用优化是对空间的优化，是对解释器的改进，怎么改进的呢，看一个例子：

```
  function c() {
    throw new Error()
    return 'return from c'
  }

  function b() {
    const b = 'b'
    return c()
  }

  function a() {
    const a = 'a'
    return b(a)
  }

  a()
```
我们有意在函数 c 中抛出一个错误。在控制台会输出类似下面的结果：

![chrome-stack-error](/images/chrome-stack-error.png)

从图中可以看到一个类似于栈的结构, a 调用 b, b 调用 c，c 抛出错误。这样的一个结构我们称之为调用堆栈（call stack）。

还不够直观，对吗？我们以图的形式演示一下执行的过程。

由于 JavaScript 是一个单线程（除了webworker, child process 这种场景）的程序，它只有一个调用堆栈。以上面的代码为例，在执行a调用前，call stack 的情况是：


````
+--------------------+
|                    |
|                    |
+                    +
|                    |
|                    |
+                    +
|                    |
|                    |
+                    +
|                    |
|                    |
+--------------------+
| a      b       c   |
|       main         |  -> stack frame
+--------------------+

````
我们将全局环境用一个 main 来表示，main 在执行前只知道 a, b, c 三个函数，存在于 call stack 中。

> call stack 由 stack frame 组成。stack frame 存一些参数 本地变量，返回地址等。

执行 a(), a 入栈。在 a 的stack frame 中，执行初始化然后在其末尾调用函数 b。

这时候就有两个策略，一层一层的stack frame 网上堆。如下：

+--------------------+
|                    |
|                    |
+--------------------+
|                    |
|        c           |
+--------------------+
|                    |
|        b           |
+--------------------+
|                    |
|        a           |
+--------------------+
| a      b       c   |
|       main         |
+--------------------+

但是由于是尾调用，a 的返回仅仅依赖 b 的调用。所以 a 的 stack frame 是没有必要保存的。那么尾调用优化后的结果是：


```
+--------------------+
|                    |
|                    |
+--------------------+
|                    |
|                    |
+--------------------+
|                    |
|                    |
+--------------------+
|                    |
|        c           |
+--------------------+
| a      b       c   |
|       main         |
+--------------------+

```

空间复杂度从 O(n) 降到了 O(1)。

## 严格模式

尾递归优化只在严格模式下生效。也可以在 es6 module 中使用，因为 es6 module 默认是遵循严格模式的。

从[https://kangax.github.io/compat-table/es6/](https://kangax.github.io/compat-table/es6/)可以看到, 目前主流浏览器只有 safari 支持尾递归调用。根据我们上面的介绍。

测试一下两种不同的模式：

![safari-loose-error](/images/safari-loose-error.png)

![safari-strict-error](/images/safari-strict-error.png)

可以看到尾递归优化使得栈调用从O(n)变成了O(1)。

## 判断以及递归

如何判断是不是尾调用，有一套详细的判定规则。本文不再详述，可以参考下面的链接。

我们知道递归对堆栈要求特别高，调用层次过多的话，会导致栈溢出错误。所以尾调用优化对尾递归意义非常。其他文章已经做了很详细的介绍，本文不再赘述。

> * [es6-tail-calls](https://benignbemine.github.io/2015/07/19/es6-tail-calls/)
> * [2ality](http://2ality.com/2015/06/tail-call-optimization.html)
> * [Call Stack](https://en.wikipedia.org/wiki/Call_stack)
> * [The Spec](https://www.ecma-international.org/ecma-262/6.0/#sec-tail-position-calls)
