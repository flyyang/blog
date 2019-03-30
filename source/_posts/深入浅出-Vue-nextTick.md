---
title: 深入浅出 Vue nextTick
date: 2019-03-31 02:38:14
tags: Vue
---

Vue nextTick 是 Vue 内部非常重要的机制。本文假设你已经了解 microtask 和 macrotask 的区别，将从以下三个角度来介绍 nextTick：

1. 静态方法 Vue.nextTick 挂载
2. 实例方法 Vue.prototype.$nextTick 挂载
3. nextTick 源码分析。

<!--more-->

## 静态方法 Vue.nextTick

Vue.nextTick 定义于 `src/core/global-api/index.js`:

```js
export function initGlobalAPI (Vue: GlobalAPI) {
  // ...
  Vue.set = set
  Vue.delete = del
  Vue.nextTick = nextTick
  // ...
}
```

我们很少在全局中使用 nextTick 处理业务，但要知道 Vue 在初始化 globalApi 的时候暴露了这个方法。

## 实例方法 Vue.prototype.$nextTick

由[深入浅出 Vue 实例化](https://github.com/flyyang/blog/issues/17) 一节中可知，最终的构造函数位于 `src/core/instance/index.js`:

```js
import { renderMixin } from './render'

function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)
```

在 `renderMixin(Vue)` 中定义了实例方法：

```js
export function renderMixin (Vue: Class<Component>) {
  // install runtime convenience helpers
  installRenderHelpers(Vue.prototype)

  Vue.prototype.$nextTick = function (fn: Function) {
    return nextTick(fn, this)
  }
  // ...
}
```

实例方法在我们的业务代码中相对常见。用来解决在数据变更后，“立即”获取 dom 更新后的结果。

```
注意这里面为 callback 传入了上下文 this，也就是 Vue 实例。
所以在下面的例子中可以直接访问 Vue 实例内容。
```

```js
new Vue({
  // ...
  methods: {
    // ...
    example: function () {
      // modify data
      this.message = 'changed'
      // DOM is not updated yet
      this.$nextTick(function () {
        // DOM is now updated
        // `this` is bound to the current instance
        this.doSomethingElse()
      })
    }
  }
})
```

## nextTick 源码分析

nextTick 源码位于 `src/core/util/next-tick.js`,  在2.6.10 的版本中，代码如下：

```js
import { noop } from 'shared/util'
import { handleError } from './error'
import { isIE, isIOS, isNative } from './env'

export let isUsingMicroTask = false

const callbacks = []
let pending = false

function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}

let timerFunc

// The nextTick behavior leverages the microtask queue, which can be accessed
// via either native Promise.then or MutationObserver.
// MutationObserver has wider support, however it is seriously bugged in
// UIWebView in iOS >= 9.3.3 when triggered in touch event handlers. It
// completely stops working after triggering a few times... so, if native
// Promise is available, we will use it:
/* istanbul ignore next, $flow-disable-line */
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
    // In problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  // Use MutationObserver where native Promise is not available,
  // e.g. PhantomJS, iOS7, Android 4.4
  // (#6466 MutationObserver is unreliable in IE11)
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  // Fallback to setImmediate.
  // Techinically it leverages the (macro) task queue,
  // but it is still a better choice than setTimeout.
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  // Fallback to setTimeout.
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}

export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    timerFunc()
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

我们忽略变量定义和函数定义部分。那么 nextTick 主要由两部分组成。一个是选择 microtask 还是 macrotask:

1. 如果原生支持 promise。使用 promise。如果是 ios webview，需额外触发一个 setTimeout。此时表示使用 microtask。
2. 如果不支持 promise, 但是支持 MutationObserver（ios7 、platformjs， android 4.4）,并且不是 ie 的话，选择 mutation observer。此时表示使用 microtask。
3.  以上都不支持的话，回退到 setImmediate（ie10-11)。
4. 以上都不支持的话，回退到 setTimout(ie9)。

我们以 4 为例子，最终将生成一个函数 `timerFunc`:

```
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
```

另外就是 nextTick 函数的定义。

nextTick 接收两个参数，cb 和上下文参数。首先将 cb 包装成一个匿名函数，push 到 callbacks 数组里。
如果当前 nextTick 在执行的话，就表示处于 pending 状态。如果非 pending 状态，则执行我们的 timerFunc。而 timeFunc 则会调用 flushCallbacks，执行所有的 callback 函数。

了解了 nextTick 行为后，我们来回顾一下[深入浅出 Vue 数据驱动 （二](https://github.com/flyyang/blog/issues/19)，中 nextTick 在派发更新的流程中，是如何调用的。

```js
/**
 * Push a watcher into the watcher queue.
 * Jobs with duplicate IDs will be skipped unless it's
 * pushed when the queue is being flushed.
 */
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
```

当我们改变了数据时，watcher 并不会立即出发，而是会放到队列里。以防重复触发一个 watcher，造成的不必要的 dom 更新。并且当前 tick 的变更会在 nextTick 去响应，在 nextTick 的流程里更新 dom。

除了在数据变化时会调用 nextTick，另外一种场景是手动调用 nextTick。我们仍以上面的例子为例：

```js
new Vue({
  // ...
  methods: {
    // ...
    example: function () {
      // modify data
      this.message = 'changed'
      // DOM is not updated yet
      this.$nextTick(function () {
        // DOM is now updated
        // `this` is bound to the current instance
        this.doSomethingElse()
      })
    }
  }
})
```
当我们改变了 this.message 时，会调用 nextTick，最终更新 dom。如果以同步访问的形式是拿不到变更后的 dom 的。所以新开一个 nextTick 来做 dom 更新之后的操作。

## 参考

* [https://vuejs.org/v2/guide/reactivity.html#Async-Update-Queue](https://vuejs.org/v2/guide/reactivity.html#Async-Update-Queue)
* [https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)


## ISSU

有问题？来 [GitHub](https://github.com/flyyang/blog/issues/21) 一起讨论。
