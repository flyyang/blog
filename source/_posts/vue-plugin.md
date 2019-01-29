---
title: 深入浅出 Vue plugin
date: 2019-01-30 02:43:25
tags:
---

## 什么是插件

按照 wiki 的解释：插件是一个用来向现有系统添加新功能的组件。当你的程序支持插件时，我们称此程序可定制。

一个插件系统的设计通常如下图所示：

![image](https://user-images.githubusercontent.com/3912408/51927841-79f9fd00-242f-11e9-97ea-e53f813305e6.png)

* 程序主体要提供一个注册插件的机制（plugin manager）。
* 插件要拥有一个可供调用的接口，在注册时使用(plugin interface)。
* 插件调用主体程序提供的服务，来扩展功能(service interface)。

我们将上面三点放在  Vue 的环境来说：

* Vue 使用 Vue.use 注册插件。使用方法为： `Vue.use(plugin, options)`。
* 插件要提供一个 install 方法，供 Vue 调用。
* 插件内部可以通过使用 Vue 提供的功能来扩展 Vue 的能力。

细节在代码分析部分介绍。

<!-- more -->

## 什么是 Vue 插件

Vue 插件官方并没具体的定义。只是描述作为全局功能的扩展，可以放到插件内实现。

我们当然可以通过在实例化前，自己定义这些全局扩展。放在插件内的好处是：

1. 功能独立，适合分发给第三方使用的扩展。如 vue-router, vuex 等。
2. 提供作用域，避免冲突。比如 a plugin 扩展了 Vue 原型 $a, b 扩展了 $b。可以在插件描述清楚。

其他情况请放到源码中，进行全局扩展。

插件的使用场景如下：
``` js
MyPlugin.install = function (Vue, options) {
  // 1. add global method or property
  Vue.myGlobalMethod = function () {
    // some logic ...
  }

  // 2. add a global asset
  Vue.directive('my-directive', {
    bind (el, binding, vnode, oldVnode) {
      // some logic ...
    }
    ...
  })

  // 3. inject some component options
  Vue.mixin({
    created: function () {
      // some logic ...
    }
    ...
  })

  // 4. add an instance method
  Vue.prototype.$myMethod = function (methodOptions) {
    // some logic ...
  }
}
```
## 源码分析

### Vue.use 分析

我们以 vuex 为例：

```js
import Vue from 'vue';
import Vuex from 'vuex'

// 注册 vuex
Vue.use(Vuex);
```
我们先来看 Vuex 是什么，以 `dist/vuex.esm.js` 为例，在最末尾：

```js
export { Store, install, mapState, mapMutations, mapGetters, mapActions, createNamespacedHelpers };
```
我们注意到 Vuex 是一个对象。其包含了一个 `install` 方法。 实现细节后续分析。

再来看一下 Vue.use 的实现，其定义在 `src/core/global-api/use.js`:

```js
import { toArray } from '../util/index'

export function initUse (Vue: GlobalAPI) {
  Vue.use = function (plugin: Function | Object) {
    const installedPlugins = (this._installedPlugins || (this._installedPlugins = []))
    if (installedPlugins.indexOf(plugin) > -1) {
      return this
    }

    // additional parameters
    const args = toArray(arguments, 1)
    args.unshift(this)
    if (typeof plugin.install === 'function') {
      plugin.install.apply(plugin, args)
    } else if (typeof plugin === 'function') {
      plugin.apply(null, args)
    }
    installedPlugins.push(plugin)
    return this
  }
}
```

我们注意下 Vue.use 在定义时接收一个参数，plugin。

然后初始化 Vue._installedPlugins。这个 this 在调用时就是 Vue 本身。如果同一个 plugin 被多次 use，那么第二次会直接返回。

**返回 `this` 意味着我们可以链式注册**。

```js
Vue.use(plugin1).use(plugin2);
```

通常我们需要给插件一些定制传参。如：

```js
Vue.use(plugin1, { showLog: false }),
```

但是我们的 Vue.use 方法定义时只提供了一个参数，那么 Vue 是怎么实现的呢？ `arguments`。

```js
  const args = toArray(arguments, 1)
  args.unshift(this)
```

这两段代码表示，把 arguments 转换成 array, 并移除第一个 plugin 参数，保留后续参数。将 Vue 作为第一个参数，放到数组内。

```js
  if (typeof plugin.install === 'function') {
    plugin.install.apply(plugin, args)
  } else if (typeof plugin === 'function') {
    plugin.apply(null, args)
  }
```

如果存在 plugin.install, 调用 plugin install 的方法。并将 Vue作为第一个参数传递给插件。
如果不存在 plugin.install 并且 plugin 本身是一个函数的话，执行 plugin 函数。

**由此我们得到，plugin 是一个函数也可以在 Vue 里工作**。

剩下的代码就比较简单，存储已安装的 plugin，并返回 this，供链式调用。

### 插件 install 分析

我们仍然以 Vuex 为例, 其install 方法位于 `src/store.js`

```js
let Vue // bind on install

// remove unrelated code

export function install (_Vue) {
  if (Vue && _Vue === Vue) {
    if (process.env.NODE_ENV !== 'production') {
      console.error(
        '[vuex] already installed. Vue.use(Vuex) should be called only once.'
      )
    }
    return
  }
  Vue = _Vue
  applyMixin(Vue)
}
```

从上面分析可以得到，install 方法的第一个参数是 Vue 本身。Vuex 不需要额外参数。另外我们注意到 install 方法定义了一个 _Vue, 有什么用呢？防止多次调用。

第一次注册此插件时，我们将 _Vue 缓存起来，放到模块的变量 Vue 内。下次来了一对比，如果一样，则说名被多次调用。

applyMixin(Vue) 功能是为所有组件添加一个 this.$store。具体分析见[深入浅出 Vue mixin](https://flyyang.me/2019/01/24/vue-mixin/)。

以上。

## 参考

* [https://en.wikipedia.org/wiki/Plug-in_(computing)](https://en.wikipedia.org/wiki/Plug-in_(computing))