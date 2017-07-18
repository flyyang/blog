---
title: JavaScript 中的 memoization
date: 2017-07-11 23:58:56
tags: memoization
---

# JavaScript 中的 memoization

Memoization, 源自于拉丁文 `memorandum`，和 `memorization` 属于近义词，但是
`memoization` 在计算机世界中有特殊的意义：函数调用结果被缓存，当下次以同样的结
果调用时，返回已经被记住的结果。是一种常用的优化手段。

相同的参数，返回相同的结果，所以 memorization 技术特别适合于 `纯函数`。尤其适用
于那些大计算量的函数。

所以通常采用递归算法的函数非常适合用 memoization技术优化，如 fibnacci 数列问题。

<!--more-->

## fibnacci 数列问题

```
function fibnacci(n) {
  return n === 1 || n===2 ? 1 : fibnacci(n-1) + fibnacci(n -2)
}
```
fibnacci为什么这么慢? 我们可以看看它的算法：

```
                                                   Fib(5)
                                                    |
                              _____________________/ \__________________
                             |                                          |
                           Fib(4)                   +                 fib(3)
                             |                                          |
                     _______/ \_______                         ________/ \_______
                    |        +        |                        |        +        |
                 Fib(3)             Fib(2)                   Fib(2)           Fib(1)
                    |                  |                        |
            _______/ \____        ____/ \_______        _______/ \_____
           |        +     |      |     +        |      |         +      |
        Fib(2)        Fib(1)    Fib(1)      Fib(0)     Fib(1)        Fib(0)
   _______/ \_______
  |        +        |
Fib(1)             Fib(0)

```

算法复杂度是 2^n。可以想象，fib(50) 是一个什么样的结果。这个 fib(50)，在我的PC
上是算不出来的。

由于 memoization 在计算量较大的时候会有明显的优势，我们选定 40 为我们的测试
fibnacci 的 n 值：

```
const perf = require('perf-function')
const _ = require('lodash')


function fibnacci(n) {
  return n === 1 || n === 2 ? 1 : fibnacci(n-1) + fibnacci(n-2)
}

const mfib = _.memoize(fibnacci)

console.log(perf(fibnacci)(40))
console.log(perf(fibnacci)(40))
console.log(perf(fibnacci)(40))
console.log(perf(fibnacci)(40))
console.log(perf(fibnacci)(40))

console.log(perf(mfib)(40))
console.log(perf(mfib)(40))
console.log(perf(mfib)(40))
console.log(perf(mfib)(40))
console.log(perf(mfib)(40))
console.log(perf(mfib)(40))
```

我们使用 lodash 的 memoize 方法得到如下的结果(单位秒)：

n  | 40 | 40 | 40 | 40 | 40
--- | --- | --- | --- | --- | ---
fib | 1059.210915 | 1071.314894 | 1076.098233 | 1063.890243 | 1085.940257
mfib | 1049.419985 | 0.145781 | 0.015948 | 0.003151 | 0.002541

在二次以上调用时，复杂度有 O(2^n) 降到 O(1)。由此可见，memorize 在需要大量计
算的函数优化中会有很大的作用。

## 实现一个memoize

实现 memoize 是比较简单的。翻遍网络上各种教程，你会发现很少见在函数内部去实现，
通常是实现一个 util 函数来辅助做记忆功能。为什么呢？

因为通常函数执行即不能再次访问，要实现记忆功能，要么用一个全局变量，要么用一个
闭包。闭包的方案更优雅一些。

下面来实现一个 fibnacci 的简单 memoize 功能：

```

function memoize(fib) {
  const cache = {}

  return function (n) {
    return cache[n] ? cache[n] : cache[n] = fib(n)
  }
}
```

与 lodash 的 memoize 对比，性能差别不大。

当然要做一个详细的 memoization 功能，需要考虑的不止这些内容。比如：参数 key 如何
序列化成 string? 如何接受多个参数？当传入多个参数，参数中又包含对象时，cache 的
命中 key 不再是 n 这种简单（转换）的字符串了。方案有 WeakMap,JSON.stringify 等。
可以参考掘金的一篇译文 [我是如何实现世界上最快的 JS 记忆化的](https://juejin.im/post/5912b635a0bb9f0058b44c60)
