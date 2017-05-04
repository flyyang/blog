---
title: JavaScript中的microtask与task
date: 2017-03-07 01:48:35
tags:
 - javascript
 - microtask
 - task
 - event loop
---

[Philip Roberts](https://www.youtube.com/watch?v=8aGhZQkoFbQ)的一个演讲对 JavaScript 运行机制做了一个不错的介绍。其中讲到了task queue，但并没有覆盖关于microtask的问题，这里对microtask做下简单介绍。

<!-- more -->

先看一段代码：

``` javascript
  console.log('1')

  setTimeout(function(){
    console.log('2')
  }, 0)

  Promise.resolve().then(function() {
    console.log('3')
  })

  Promise.resolve().then(function() {
   console.log('4')
  })

  console.log('5')
```
该段代码打印什么结果？答案是： 1,5,3,4,2。

下面我们来分析一下这段代码。

由于 JavaScript 是单线程的，所以它只有一个 Call Stack，使得 JavaScript 在执行时有一个非常重要的特性：`run to complete`,只要运行就直到完成。另外，JavaScript 编译器并不会做太多的事情，比如异步请求、事件操作、`setTimeout`、`Promise`等，并不会自己直接等待处理，
而是交由其宿主代理。

![event loop](/public/images/javascript_event_loop.png)

参照上图和上面的解释，很容易得出。先输出1和5。

setTimeOut和Promise哪个先输出呢，答案是Promise。Why? Promise 属于一个microtask。Eventloop在执行完堆栈，或者一个task后，会**优先**询问microtask queue，如果队列中有任务要执行，则执行，一直到队列为空，然后再执行下一个task(每一个microtask、task都遵循`run to complete规则)。

所以以上会优先输出3，4，然后是2。

Why microtask? 有时候task太重了，每一个task结束后，都会重新渲染页面。microtask比task拥有更高的优先级，可以做一些比较有意思的事情。

## 常见的一些task分类

macrotasks(task): setTimeout, setInterval, setImmediate, I/O, UI rendering
microtasks: process.nextTick, Promises, Object.observe, MutationObserver

## 参考

* http://stackoverflow.com/questions/25915634/difference-between-microtask-and-macrotask-within-an-event-loop-context
* https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/
* https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model
* https://www.zhihu.com/question/55364497
* https://www.youtube.com/watch?v=8aGhZQkoFbQ
