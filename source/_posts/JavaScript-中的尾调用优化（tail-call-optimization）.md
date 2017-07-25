---
title: JavaScript 中的尾调用优化（tail call optimization）
date: 2017-07-24 23:21:24
tags: JavaScript
---
# JavaScript 中的尾调用优化(tail call optimization)

我在学习尾调用优化的过程中，有两个误解:

第一个是，我们一谈优化，经常说的是时间的优化。但是尾调用优化却主要是指空间的优化。

第二个是，既然尾调用优化是在 es6 中支持的，那么可能又要学新的语法了。然而，尾调
用优化并不需要新的语法，而是在解释器（如V8）中做的改进。尾调用是一直存在的，但
是尾调用优化是在支持 es6 的解释器里添加的。

澄清了这两个问题之后，我们先来看看尾调用是什么。

<!--more-->

## 尾调用

尾调用，从定义上来看很简单，简而言之是函数里的最后一个动作是函数调用。

从上面了解到，尾调用优化是对空间的优化，是对解释器的改进，怎么改进的呢，看一个
例子：

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
在控制台会输出类似下面的结果：

```


````


## 尾递归

## 严格模式

