---
title: 深入浅出 vue mixin
date: 2019-01-24 15:36:50
tags:
---
## 什么 mixin

mixin， 中文意思为混入。

比如去买冰激凌，我先要一点奶油的，再来点香草的。我就可以吃一个奶油香草的冰激凌。如果再加点草莓，我可以同时吃三个口味的冰激凌。

### js 代码表示
假设把你已有的奶油味的称为 base，把要添加的味道称为 mixins。用 js 伪代码可以这么来写：

```javascript
const base = {
  hasCreamFlavor() {
    return true;
  }
}
const mixins = {
  hasVanillaFlavor() {
    return true;
  },
  hasStrawberryFlavor() {
    return true;
 }
}

function mergeStrategies(base, mixins) {
  return Object.assign({}, base, mixins);
}
// newBase 就拥有了三种口味。
const newBase = mergeStrategies(base, mixins);
```

注意一下这个 `mergeStrategies`。

合并策略可以你想要的形式，也就是说你可以自定义自己的策略，这是其一。另外要解决冲突的问题。上面是通过 Object.assign 来实现的，那么 mixins 内的方法会覆盖base 内的内容。如果这不是你期望的结果，可以调换 mixin 和 base 的位置。

<!--more-->

### 组合大于继承 && DRY

想象一下上面的例子用继承如何实现？由于 js 是单继承语言，只能一层层继承。写起来很繁琐。这里就体现了 mixin 的好处。符合组合大于继承的原则。

mixin 内通常是提取了公用功能的代码。而不是每一个地方都写一遍。符合 DRY 原则。

## 什么是 vue mixin

vue mixin 是针对组件间功能共享来做的。可以对组件的任意部分（生命周期， data等）进行mixin，但不同的 mixin 之后的合并策略不同。在源码分析部分会介绍细节。

### 组件级 mixin

假设两个功能组件 model 和 tooltip ，他们都有一个显示和关闭的 toggle 动作：

```javascript
//modal
const Modal = {
  template: '#modal',
  data() {
    return {
      isShowing: false
    }
  },
  methods: {
    toggleShow() {
      this.isShowing = !this.isShowing;
    }
  }
}

//tooltip
const Tooltip = {
  template: '#tooltip',
  data() {
    return {
      isShowing: false
    }
  },
  methods: {
    toggleShow() {
      this.isShowing = !this.isShowing;
    }
  }
}
```

可以用 mixin 这么写：

```javascript
const toggleMixin = {
  data() {
    return {
      isShowing: false
    }
  },
  methods: {
    toggleShow() {
      this.isShowing = !this.isShowing;
    }
  }
}

const Modal = {
  template: '#modal',
  mixins: [toggleMixin]
};

const Tooltip = {
  template: '#tooltip',
  mixins: [toggleMixin],
};
```

### 全局 mixin

全局 mixin 会作用到每一个 vue 实例上。所以使用的时候要慎重。通常会用 plugin 来显示的声明用到了那些 mixin。

比如 vuex。我们都知道它在每一个实例上扩展了一个 $store, 在任意一个组件内可以调用 this.$store。那么他是如何实现的呢？

在 `src/mixin.js` 内
```javascript
export default function (Vue) {
  const version = Number(Vue.version.split('.')[0])

  if (version >= 2) {
    Vue.mixin({ beforeCreate: vuexInit })
  } else {
    // override init and inject vuex init procedure
    // for 1.x backwards compatibility.
    const _init = Vue.prototype._init
    Vue.prototype._init = function (options = {}) {
      options.init = options.init
        ? [vuexInit].concat(options.init)
        : vuexInit
      _init.call(this, options)
    }
  }
  /**
   * Vuex init hook, injected into each instances init hooks list.
   */

  function vuexInit () {
    const options = this.$options
    // store injection
    if (options.store) {
      this.$store = typeof options.store === 'function'
        ? options.store()
        : options.store
    } else if (options.parent && options.parent.$store) {
      this.$store = options.parent.$store
    }
  }
}
```

我们看到 在 Vue 2.0 以上版本，通过 `Vue.mixin({ beforeCreate: vuexInit })`实现了在每一个实例的 `beforeCreate` 生命周期调用vuexInit 方法。

而 vuexInit 方法则是：在跟节点我们会直接把store 注入，在其他节点则拿父级节点的 store，这样this.$store 永远是你在根节点注入的那个store。

## vue mixin 源码实现

在 Vuex 的例子中，我们通过 `Vue.mixin({ beforeCreate: vuexInit })` 实现对实例的 $store 扩展。

### 全局 mixin 注册

我们先看一下 mixin 是如何挂载到原型上的。

在 `src/core/index.js` 中：

```javascript
import Vue from './instance/index'
import { initGlobalAPI } from './global-api/index'

initGlobalAPI(Vue)

export default Vue
```

我们发现有一个 initGlobalAPI。在 `src/global-api/index` 中：

```javascript
/* @flow */

import config from '../config'
import { initUse } from './use'
import { initMixin } from './mixin'
import { initExtend } from './extend'
import { initAssetRegisters } from './assets'
import { set, del } from '../observer/index'
import { ASSET_TYPES } from 'shared/constants'
import builtInComponents from '../components/index'

import {
  warn,
  extend,
  nextTick,
  mergeOptions,
  defineReactive
} from '../util/index'

export function initGlobalAPI (Vue: GlobalAPI) {
  // config
  const configDef = {}
  configDef.get = () => config
  if (process.env.NODE_ENV !== 'production') {
    configDef.set = () => {
      warn(
        'Do not replace the Vue.config object, set individual fields instead.'
      )
    }
  }
  Object.defineProperty(Vue, 'config', configDef)

  // exposed util methods.
  // NOTE: these are not considered part of the public API - avoid relying on
  // them unless you are aware of the risk.
  Vue.util = {
    warn,
    extend,
    mergeOptions,
    defineReactive
  }

  Vue.set = set
  Vue.delete = del
  Vue.nextTick = nextTick

  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })

  // this is used to identify the "base" constructor to extend all plain-object
  // components with in Weex's multi-instance scenarios.
  Vue.options._base = Vue

  extend(Vue.options.components, builtInComponents)

  initUse(Vue)
  initMixin(Vue)
  initExtend(Vue)
  initAssetRegisters(Vue)
}
```

所有全局的方法都在这里注册。我们关注 initMixin 方法，定义在 `src/core/global-api/mixin.js`:

```javascript
import { mergeOptions } from '../util/index'

export function initMixin (Vue: GlobalAPI) {
  Vue.mixin = function (mixin: Object) {
    this.options = mergeOptions(this.options, mixin)
    return this
  }
}
```
至此我们发现了 Vue 如何挂载全局 mixin。

### mixin 合并策略

vuex 通过 beforeCreate Hook 实现为所有 vm 添加 $store 实例。 让我们先把 hook 的事情放一边。看一看 beforeCreate 如何实现。

在 `src/core/instance/init.js` 中：

```javascript
export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    // remove unrelated code
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')

    // remove unrelated code
    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```
我们可以看到在 initRender 完成后，会调用 `callHook(vm, 'beforeCreate')`。而 init 实在 vue 实例化会执行的。

在 `src/core/instance/lifecycle.js` 中：

```javascript
export function callHook (vm: Component, hook: string) {
  // #7573 disable dep collection when invoking lifecycle hooks
  pushTarget()
  const handlers = vm.$options[hook]
  if (handlers) {
    for (let i = 0, j = handlers.length; i < j; i++) {
      try {
        handlers[i].call(vm)
      } catch (e) {
        handleError(e, vm, `${hook} hook`)
      }
    }
  }
  if (vm._hasHookEvent) {
    vm.$emit('hook:' + hook)
  }
  popTarget()
}

```

在对 beforeCreate 执行 callHook 过程中，会先从 vue 实例的 options 中取出所有挂载的 handlers。
然后循环调用 call 方法执行所有的 hook:

```
handlers[i].call(vm)
```

由此我们可以了解到全局的 hook mixin 会和要 mixin 的组件合并 hook，最后生成一个数组。

回头再看：

```javascript
import { mergeOptions } from '../util/index'

export function initMixin (Vue: GlobalAPI) {
  Vue.mixin = function (mixin: Object) {
    this.options = mergeOptions(this.options, mixin)
    return this
  }
}
```

this.options 默认是 vue 内置的一些 option:

![image](https://user-images.githubusercontent.com/3912408/51659312-17df6900-1fe5-11e9-9682-ba0543c32cfc.png)

mixin 就是你要混入的对象。我们来看一看 mergeOptions。定义在 `src/core/util/options.js`:

```javascript
export function mergeOptions (
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  if (process.env.NODE_ENV !== 'production') {
    checkComponents(child)
  }

  if (typeof child === 'function') {
    child = child.options
  }

  normalizeProps(child, vm)
  normalizeInject(child, vm)
  normalizeDirectives(child)
  const extendsFrom = child.extends
  if (extendsFrom) {
    parent = mergeOptions(parent, extendsFrom, vm)
  }
  if (child.mixins) {
    for (let i = 0, l = child.mixins.length; i < l; i++) {
      parent = mergeOptions(parent, child.mixins[i], vm)
    }
  }
  const options = {}
  let key
  for (key in parent) {
    mergeField(key)
  }
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }
  function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
  return options
}
```
忽略不相干代码我们直接跳到：

```javascript
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }
  function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
```

此时 child 为 `{ beforeCreate: vuexInit }`。走入到 mergeField 流程。mergeField 先取合并策略。

`const strat = strats[key] || defaultStrat`，相当于取 strats['beforeCreate'] 的合并策略。定义在通文件的上方：

```javascript
/**
 * Hooks and props are merged as arrays.
 */
function mergeHook (
  parentVal: ?Array<Function>,
  childVal: ?Function | ?Array<Function>
): ?Array<Function> {
  return childVal
    ? parentVal
      ? parentVal.concat(childVal)
      : Array.isArray(childVal)
        ? childVal
        : [childVal]
    : parentVal
}

LIFECYCLE_HOOKS.forEach(hook => {
  strats[hook] = mergeHook
})

// src/shared/constants.js

export const LIFECYCLE_HOOKS = [
  'beforeCreate',
  'created',
  'beforeMount',
  'mounted',
  'beforeUpdate',
  'updated',
  'beforeDestroy',
  'destroyed',
  'activated',
  'deactivated',
  'errorCaptured'
]
```

在  mergeHook 中的合并策略是把所有的 hook 生成一个函数数组。其他相关策略可以在options 文件中查找（如果是对象，组件本身的会覆盖上层，data 会执行结果，返回再merge，hook则生成数组）。

### mixin 早于实例化

mergeOptions 会多次调用，正如其注释说描述的那样：

``` javascript
/**
 * Merge two option objects into a new one.
 * Core utility used in both instantiation and inheritance.
 */
```

上面介绍了全局 mixin 的流程，我们来看下 实例化部分的流程。在 `src/core/instance/init.js` 中:

```js
export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    if (options && options._isComponent) {
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    // expose real self
    vm._self = vm
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')
    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```

由于 全局 mixin 通常放在最上方。所以一个 vue 实例，通常是内置的 options + 全局 mixin 的 options +用户自定义options，加上合并策略生成最终的 options.

那么对于 hook 来说是[mixinHook, userHook]。mixin 的hook 函数优先于用户自定义的 hook 执行。

### local mixin

在 组件中书写 mixin 过程中:

```js
const Tooltip = {
  template: '#tooltip',
  mixins: [toggleMixin],
};
```

在 mergeOptions 的过程中有下面一段代码：

```javascript
  if (child.mixins) {
    for (let i = 0, l = child.mixins.length; i < l; i++) {
      parent = mergeOptions(parent, child.mixins[i], vm)
    }
  }
```

当 tooltip 实例化时，会将对应的参数 merge 到实例中。

### 定制合并策略

``` javascript
Vue.config.optionMergeStrategies.myOption = function (toVal, fromVal) {
  // return mergedVal
}
```

以上。

## 参考

* [http://techsith.com/mixins-in-javascript/](http://techsith.com/mixins-in-javascript/)
* [https://vuejs.org/v2/guide/mixins.html](https://vuejs.org/v2/guide/mixins.html)
* [https://css-tricks.com/using-mixins-vue-js/](https://css-tricks.com/using-mixins-vue-js/)