---
title: 深入浅出 Vue 数据驱动（一）
date: 2019-03-04 15:51:49
tags:
---
数据驱动开发，与传统的 jQuery 开发相比，有很多优势。最明显的两点是：

1. 不需要关注 dom。不仅不需要关注如何初始化dom，也不需要关心状态变更时如何处理dom。整个流程围绕着如何操作数据。
2. 可以方便做优化。因为整个流程都是数据，加上配合 vdom 对底层的抽象，我们可以做类似于 diff patch 算法的优化。多了层抽象意味着有了很多优化空间。

在做UI 编程时，通常有两个流程需要考虑：

* 第一次进来时如何展示。
* 当后续有变化时如何展示。

这是一个动态的**时间序**的考量。对应在 Vue 的流程中：

* 从 new Vue 到 dom。
* 数据变化时更新 dom（很多人称之为响应式）。

![image](https://user-images.githubusercontent.com/3912408/53619426-57bfee80-3c2a-11e9-965b-1e94d2cab39d.png)

本节主要分析从 new Vue 到最终 dom 的过程。

<!--more -->

###  从 new Vue 开始

我们以最简单的 Hello world 示例：

```js
import Vue from 'vue/dist/vue.esm';

new Vue({
  template: '<div>{{message}}</div>',
  el: `#app`,
  data: {
    message: 'Flyyang say hello to Vue.js!',
  },
});
```

由[深入浅出 Vue 实例化](https://github.com/flyyang/blog/issues/17) 可知，构造函数 Vue 位于 `
src/core/instance/index.js`:

```js
import { initMixin } from './init'

function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)

export default Vue
```

当我们调用 new Vue 的时候，会使用内部方法 _init 。定义在 initMixin 内。注意由于 js 是动态语言，我们可以先使用，后定义。在 `src/core/instance/init.js`

```js
export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
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

initMixin 定义了一个原型方法。做了一些初始化操作，然后调用 $mount 方法。

### 平台无关的 $mount 方法

$mount 方法是一个平台无关的方法。无论是 weex 运行时，还是 web 的运行时，都需要有 $mount 方法。相当于一个运行时接口。

我们研究 `entry-runtime-with-compiler.js` 时，发现有如下代码：

```js
import Vue from './runtime/index'

const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && query(el)
 if (template) {
      const { render, staticRenderFns } = compileToFunctions(template, {
        outputSourceRange: process.env.NODE_ENV !== 'production',
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
  }
  return mount.call(this, el, hydrating)
}
```
在 `import Vue from './runtime/index'` 时，已经定义了 $mount 方法，为什么在这里要缓存并重写呢？

1. runtime 版本的mount 可以被多个入口使用。比如 `entry-runtime.js` 。
2. 这里重写是要加自己的逻辑，对于带compiler版本的 Vue 来说，需要编译 template 到 render function。最终还是会调用 runtime 内的 $mount 方法。

`compileToFunctions` 会将模板 `<div>{{message}}</div>` 编译成 render function。 Vue2.0 之后都会变成 render fucntion。细节在后续章节详述。

![image](https://user-images.githubusercontent.com/3912408/53626308-539ecb80-3c40-11e9-9c16-a53766db7160.png)

### mountComponent 到 dom

由上图可知，$mount 会调用 mountComponent 方法。找到 mountComponent 其实就已经找到终点了。

什么？这么简单？也不是，还有一些细节需要补充。

```js
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  let updateComponent
    updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }

  // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)

  return vm
}
```

mountComponent 定义了一个 updateComponent 方法，然后新建了一个 Watcher 实例。

```js

export default class Watcher {
  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {

    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
  get () {
    value = this.getter.call(vm, vm)
    return value
  }

}

```

Watcher 的 构造函数将第二个参数 `updateComponent` 赋值给 getter。然后有调用其 get 方法，触发 getter。执行了 updateComponent 方法。

```js
    updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }
```

#### vm._render

vm._render 定义在 `src/core/instance/render.js`:

```js
export function renderMixin (Vue: Class<Component>) {
  Vue.prototype._render = function (): VNode {
    const { render, _parentVnode } = vm.$options
      // There's no need to maintain a stack becaues all render fns are called
      // separately from one another. Nested component's render fns are called
      // when parent component is patched.
      currentRenderingInstance = vm
      vnode = render.call(vm._renderProxy, vm.$createElement)

    return vnode
  }
}
```

我们关注其核心部分。 _render 函数调用编译阶段生成的 render 函数。执行生成，vnode。最后返回 vnode。

#### vm._update

update 方法定义于 `src/core/instance/lifecycle.js`。

```js

export function lifecycleMixin (Vue: Class<Component>) {
  Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    const restoreActiveInstance = setActiveInstance(vm)
    vm._vnode = vnode
    // Vue.prototype.__patch__ is injected in entry points
    // based on the rendering backend used.
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    restoreActiveInstance()
    // update __vue__ reference
    if (prevEl) {
      prevEl.__vue__ = null
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm
    }
    // if parent is an HOC, update its $el as well
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el
    }
    // updated hook is called by the scheduler to ensure that children are
    // updated in a parent's updated hook.
  }
}
```

update 第一个参数接受 vnode。第二个参数是服务端渲染标记，暂不考虑。由上面可知，vm._render 返回的就是 vnode。而 update 内部又调用了 __patch__ 方法。

#### vm.\__patch\__

\__patch\__ 是一个平台无关的方法。和 $mount 一样，每个平台有不同的 \__patch\__ 方法，定义在 `src/platforms/(web/weex)/runtime/index.js`内。我们只看 web 部分:

```js
import { patch } from './patch'
// install platform patch function
Vue.prototype.__patch__ = inBrowser ? patch : noop
```

``` js
// patch.js
import * as nodeOps from 'web/runtime/node-ops'
import { createPatchFunction } from 'core/vdom/patch'
import baseModules from 'core/vdom/modules/index'
import platformModules from 'web/runtime/modules/index'

// the directive module should be applied last, after all
// built-in modules have been applied.
const modules = platformModules.concat(baseModules)

export const patch: Function = createPatchFunction({ nodeOps, modules }
```

nodeOps 定义了**平台相关**的节点操作方法。modules 定义了一些**平台相关**的属性事件操作。

![image](https://user-images.githubusercontent.com/3912408/53715480-e59b1400-3e8c-11e9-9372-4fd1c9a91fba.png)

我们注意到虽然平台相关，但是对外仍是接口的形式。不同平台需要实现**相同的方法**。

![image](https://user-images.githubusercontent.com/3912408/53716549-4546ee80-3e90-11e9-9445-8f8f4843d484.png)

```js
export function createPatchFunction (backend) {
    return function patch (oldVnode, vnode, hydrating, removeOnly) {
    ....
   }
}
```

createPatchFunction 是一个高阶函数。通过传入不同的 node_ops 和 modules，生成不同的 patch function。这里用到了一个函数柯里化的技巧，通过 createPatchFunction 把差异化参数提前固化，这样不用每次调用 patch 的时候都传递 nodeOps 和 modules 了。

在 返回的 patch 函数中：
```js
// create new node
createElm(
  vnode,
  insertedVnodeQueue,
  // extremely rare edge case: do not insert if old element is in a
  // leaving transition. Only happens when combining transition +
  // keep-alive + HOCs. (#4590)
  oldElm._leaveCb ? null : parentElm,
  nodeOps.nextSibling(oldElm)
)
```

将会实际创建 dom。至此，我们就看到了从 new Vue 到 dom 的整个过程。

## ISSUE

有问题？来 [GitHub](https://github.com/flyyang/blog/issues/18) 一起讨论。