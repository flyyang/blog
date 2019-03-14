---
title: 深入浅出 Vue 数据驱动（二） 
date: 2019-03-15 02:57:05
tags:
---
数据驱动开发是 Vue 的一大特征。

那么什么是数据驱动呢？在 Vue 的概念下，我们可以通过 data 来初始化页面；后续可以通过操作data 的值，来改变页面。整个过程都是围绕 data 来变化，所以称之为数据驱动，其中操作数据更新页面又常被称为响应式。

在 [深入浅出 Vue 数据驱动 （一）](https://github.com/flyyang/blog/issues/18) 中，我们已经介绍了初始化的部分，本节主要介绍响应式是如何实现的。

<!--more-->

## 写在前面

![image](https://user-images.githubusercontent.com/3912408/54339498-f9562f80-466f-11e9-9d15-29365b00bca5.png)

由上图可知，我们改变了 message 的值，对应的 ui 就会发生变化。
```
App.message = 'Some one say hello to Vue!';
```
而在正常情况下，给属性赋值就是赋值，没有任何特别之处：

```
const a = { b: 1}
a.b = 2;

a.b // 输出 2
```

在 Vue 里面却变成 ui 变更，跟我们赋值操作做的看起来不是一件事儿。这说明 Vue 在把自己**挂载到dom之前**，做了一些工作。我们知道在 es5 中，可以通过 Object.defineProperty 来实现赋值 set 添加其他功能。

![image](https://user-images.githubusercontent.com/3912408/54348627-5e1b8500-4684-11e9-9e2b-787c5877fadd.png)

在 Vue 的源码分析过程中，一个重要的点就是 找到 Object.defineProperty 的定义。

另外 message 可以形成 getter 、computed 等，相互之间的依赖关系会越来越复杂。Vue 通过一个 Pub / Sub 模型来管理这些依赖。

总结一下上面的流程：

在挂载到 Dom 前， Vue 需要完成两件事：

1. 将属性转换为 get 、set
2. 将所有依赖关系收集起来。

```
虽然这部分也属于 new Vue 到 dom， 但是为了减小复杂度，我们在 深入浅出 Vue 数据驱动 （一） 中，故意省略了这部分。
```
在挂载到 dom 后：

1. 调用 set ，执行所有依赖，更新 dom。

## 源码分析
以一个最简单的例子开始：

```js
new Vue({
  template: '<div>{{message}}</div>',
  el: '#app',
  data: {
    message: 'Flyyang say hello to Vue!',
  },
});
```

下面分两个部分来分析源码。

### 属性转换与依赖收集

我们直接从  Vue.prototype._init 开始（参考前一篇文章）

```js
Vue.prototype._init__ = function( options ) {
   //...
    initLifecycle(vm);
    initEvents(vm);
    initRender(vm);
    callHook(vm, 'beforeCreate');
    initInjections(vm); // resolve injections before data/props
    initState(vm);
    initProvide(vm); // resolve provide after data/props
    callHook(vm, 'created');
  // ...
   if (vm.$options.el) {
      vm.$mount(vm.$options.el);
   }
}
```

找到 initState：

```js
function initState (vm) {
  debugger;
  vm._watchers = [];
  var opts = vm.$options;
  if (opts.props) { initProps(vm, opts.props); }
  if (opts.methods) { initMethods(vm, opts.methods); }
  if (opts.data) {
    initData(vm);
  } else {
    observe(vm._data = {}, true /* asRootData */);
  }
  if (opts.computed) { initComputed(vm, opts.computed); }
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch);
  }
}
```
忽略不相干的代码，直接看 initData:

```js
function initData (vm) {
    var data = vm.$options.data;
    data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {};
    // ...
    if (props && hasOwn(props, key)) {
      warn(
        "The data property \"" + key + "\" is already declared as a prop. " +
        "Use prop default value instead.",
        vm
      );
    } else if (!isReserved(key)) {
      proxy(vm, "_data", key);
    }
  }
  // observe data
  observe(data, true /* asRootData */);
}
```

initData 做了许多事情，我们主要关注三点：1. vm._data 2. proxy 3. observe。

vm._data 是 data 的内部表示。所以 `proxy(vm, "_data", key);` 是对 data 的访问代理。

```js
function proxy (target, sourceKey, key) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  };
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val;
  };
  Object.defineProperty(target, key, sharedPropertyDefinition);
}
```
针对我们上面的例子，vm.message 访问代理到 vm._data.message。

在开始分析 observe 之前，我们先梳理一下到此为止的整个流程，如图所示：

![image](https://user-images.githubusercontent.com/3912408/54357822-dfc9dd80-4699-11e9-8f2d-11ad5cfaed8b.png)

可以看出我们在逐步细化这个流程，比如在第二步，不仅有 initData， 还有initProps。我们故意忽略了这个细节，方便我们整体去把控流程。

```js
function observe (value, asRootData) {
   if (!isObject(value) || value instanceof VNode) {
    return
  }
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__;
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value);
  }
  if (asRootData && ob) {
    ob.vmCount++;
  }
  return ob
}
```
同样的忽略所有相关细节， observe 函数主要作用是建立一个 Observer 类。如果传给 observe 的不是一个对象的话，返回 undefined，否则返回一个 Observer 实例（后续会利用这个特性做深度响应式处理）。

```js
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }
}
```

此时传给 observer 的 value 为：{ message: 'Flyyang say hello to Vue '}。

将会走到 this.walk(value)：

```js
  /**
   * Walk through all properties and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }
```
walk 的作用是循环所有的对象属性，转换为 geter/setter。转换操作在 defineReactive 里：

```js
function defineReactive (
  obj,
  key,
  val,
  customSetter,
  shallow
) {
  var dep = new Dep();
  var childOb = !shallow && observe(val);
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      var value = getter ? getter.call(obj) : val;
      if (Dep.target) {
        dep.depend();
        if (childOb) {
          childOb.dep.depend();
          if (Array.isArray(value)) {
            dependArray(value);
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      var value = getter ? getter.call(obj) : val;
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (customSetter) {
        customSetter();
      }
      // #7981: for accessor properties without setter
      if (getter && !setter) { return }
      if (setter) {
        setter.call(obj, newVal);
      } else {
        val = newVal;
      }
      childOb = !shallow && observe(newVal);
      dep.notify();
    }
  });
}
```

饶了这么一大圈，终于看到了 Object.defineProperty 的庐山真面目。我们将我们的参数代入进去：

1. obj: { message: ' Flyyang say hello to Vue'}
2. key: 'message'
3. value: 'Flyyang say hello to Vue'。

首先新建了一个 dep，我们先理解为依赖管理器。然后定义一个 childOb， 也就是 子的 Obsever。
由上面 observe 函数可知，当传入的 value 不是对象时，返回 undefind。所以 childOb 应为 false。

```
如果我们定义的 data 包含对象时，会递归调用 observe ，重走上面的流程知道 value 非 object。对这一块的理解非常重要。
```
由于这里只是定义 getter setter，我们先将分析到此为止。回忆一下我们的 __init__ 方法：

![image](https://user-images.githubusercontent.com/3912408/54360571-202c5a00-46a0-11e9-8703-38db592fdf95.png)

我们在 initState 阶段对数据做了响应式处理。然后走入 mount 的流程。由上一节可知，在 mount 的流程里
会新建一个 Watcher:

```js
  new Watcher(vm, updateComponent, noop, {
    before: function before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate');
      }
    }
  }, true /* isRenderWatcher */);
```
```js
/**
 * A watcher parses an expression, collects dependencies,
 * and fires callback when the expression value changes.
 * This is used for both the $watch() api and directives.
 */
export default class Watcher {
  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = noop
        process.env.NODE_ENV !== 'production' && warn(
          `Failed watching path: "${expOrFn}" ` +
          'Watcher only accepts simple dot-delimited paths. ' +
          'For full control, use a function instead.',
          vm
        )
      }
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
  get () {
    pushTarget(this)
    // ...
    value = this.getter.call(vm, vm)
   // ...
}
```

我们来回忆一下上一节中的流程，新建一个 wathcer， 然后构造函数中将 `updateComponent` 付给 watcher 的 getter。最后 在赋值 this.value 中调用 get 方法，同时执行 pushTarget 和 updateComponent。

我们先来看 pushTarget

```js
// The current target watcher being evaluated.
// This is globally unique because only one watcher
// can be evaluated at a time.
Dep.target = null
const targetStack = []

export function pushTarget (target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}

export function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}
```
`pushTarget(this)` 将当前执行的 Watcher 实例 当做 Dep 对象的静态属性。这种黑科技相当于我**在一个对象上面挂了一个全局变量**。

然后我们看下 `updateComponent` 部分。根据上篇文章介绍，在生成 dom 的过程中，会先将模板变异成 render 函数，并执行render 函数:

在 `/src/core/instance/render.js` 中：

```
 Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const { render, _parentVnode } = vm.$options
    // ...
    vnode = render.call(vm._renderProxy, vm.$createElement)
   // ... 
   return vnode
  }
}
```
vm._renderProxy 其实就是 vm 本身（或者proxy 过得 vm）。那么我们上面示例模板会编出什么代码呢？

![image](https://user-images.githubusercontent.com/3912408/54372287-0138c280-46b6-11e9-8b8c-307f2aac6ebe.png)

如上图所示，render 函数中访问了 message 属性。我们知道它是被代理过得，并且也转换了 getter /setter。

访问意味着会走到其get 访问器。

```js
function defineReactive (
  obj,
  key,
  val,
  customSetter,
  shallow
) {
  var dep = new Dep();
  var childOb = !shallow && observe(val);
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      var value = getter ? getter.call(obj) : val;
      if (Dep.target) {
        dep.depend();
        if (childOb) {
          childOb.dep.depend();
          if (Array.isArray(value)) {
            dependArray(value);
          }
        }
      }
      return value
    },
}
```
我们看 Dep.target ，在新建 Watcher 的时候，我们把当前 Watcher 赋值给了 Dep 对象的静态属性 target，那么此时 Dep.target 是有值的。

我们只有一个 属性 `message` 并且其值不是对象也不是 Array。那么只会执行 dep.depend()方法:

```
Dep.prototype.depend = function depend () {
  if (Dep.target) {
    Dep.target.addDep(this);
  }
};
```
其作用是把 dep 实例 添加到 watcher 上。

```js
  // watcher.js
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }
```

```js
  // dep
  addSub (sub: Watcher) {
    this.subs.push(sub)
  }
```

同时又将 watcher 添加到添加到 dep.subs 内。至此依赖收集已经做完了。当前几个对象的关系用图片来表示为：

![image](https://user-images.githubusercontent.com/3912408/54376419-00a42a00-46be-11e9-8c9d-18e37d34baaa.png)

```
我们这里只描述了一个属性对应的 dep 。当你初始化的属性越多，包含嵌套对象和数组越多，那么生成的 dep 实例也就越多。

关于将会有多少个 watcher，我们后续章节再讨论。
```

### 派发更新

接下来分析当我们修改 App.message 时会发生什么：

```js
App.message = 'Some one say hello to Vue'
```
由上面的分析可知 App.message 时代理过后的属性，最终会走到属性的 setter:

```js
function defineReactive (
  obj,
  key,
  val,
  customSetter,
  shallow
) {
  var dep = new Dep();
  var childOb = !shallow && observe(val);
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    set: function reactiveSetter (newVal) {
      var value = getter ? getter.call(obj) : val;
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (customSetter) {
        customSetter();
      }
      // #7981: for accessor properties without setter
      if (getter && !setter) { return }
      if (setter) {
        setter.call(obj, newVal);
      } else {
        val = newVal;
      }
      childOb = !shallow && observe(newVal);
      dep.notify();
    }
  });
}
```

我们关注两个细节：

```js
childOb = !shallow && observe(newVal);
```
当你 set 一个新值时，同样也会判断是否为对象数组等，仍然会走一遍 observe 的流程。

最后调用 `dep.notify()`。

```js
  // dep.js
  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
```
由依赖分析小节可知， sub 内存放的是 watcher 实例。notify 的作用是按顺序触发所有 watcher。

```js
  // watcher.js
  update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }
```
忽略特殊选项，将会执行到 queueWatcher。 在 `scheduler.js` 中

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

queueWatcher 作用是，如果当前没有在 flushing 的状态，那么就进入队列排队。如果在的话，在 nextTick 阶段则 flush 队列。

```js
  // core/util/next-tick.js

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
我们不对 nextTick 做过多分析。以一个最简单的例子来说明，假设nextTick 是 new 了一个 Promise，那么他的回调会在下一个 event loop 过程中执行。也就是说要走一遍 js 的 event loop 流程。

```
依赖变化并不会直接更新 dom ，而是先入队做处理。在 nextTick 更新。
```

接下来看一下 nextTick 的 cb 函数: `flushSchedulerQueue`。

```js
// scheduler.js

function flushSchedulerQueue () {
  // ...
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run()
    // in dev build, check and stop circular updates.
    if (process.env.NODE_ENV !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? `in watcher with expression "${watcher.expression}"`
              : `in a component render function.`
          ),
          watcher.vm
        )
        break
      }
    }
  }

  // call component updated and activated hooks
  callActivatedHooks(activatedQueue)
  callUpdatedHooks(updatedQueue)
}
```
flush 的过程中会调用 watcher 的 run 方法：

```js
// watcher.js

  run () {
    if (this.active) {
      const value = this.get()
    // ...
  }

```

`run` 方法会调用 this.get() 。其实就是我们的 updateComponent 函数。这样就回到了我们上一章中的流程。

唯一不同的是我们的 message 变了。此时生成的 vnode 也就变了:

<img src="https://user-images.githubusercontent.com/3912408/54383284-abbbe000-46cc-11e9-86d3-f72cca7c96cd.png" />

剩下的就是做 dom diff 和 patch，最后更新页面。

以上。

## ISSUE

有问题？来 [GitHub](https://github.com/flyyang/blog/issues/19) 一起讨论。