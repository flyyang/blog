---
title: 深入浅出 Vue 实例化
date: 2019-02-21 18:02:25
tags: vue
---

由[深入浅出 Vue 构建流程](https://github.com/flyyang/blog/issues/16)可知，当我们使用：

```js
import Vue from 'Vue'
```

时，默认查找的文件是 `dist/vue.runtime.esm.js`。而构建出这个文件的入口文件是：`src/platforms/web/entry-runtime.js`。

runtime 版本是不包含 compiler 的，也就是没有编译 Vue 模板的过程。通常编译的工作交给 vue-loader，也就是 webpack 来代劳。但是从分析源码的角度来看，我们还是有必要要了解一下编译过程。所以我们从带 compiler 的入口开始:

```js
// 源码分析从此开始
src/platforms/web/entry-runtime-with-compiler.js
```

<!--more-->

### Vue 是一个构造函数
在 Vue 里，一个简单的 Hello World 程序如下：

```html
<div id="app">
  {{ message }}
</div>
```
```js
var app = new Vue({
  el: '#app',
  data: {
    message: 'Hello Vue!'
  }
})
```

我们可以注意到一个通常的 Vue 应用程序，是通过 new Vue 开始的。我们来看看 Vue 到底是什么。

![image](https://user-images.githubusercontent.com/3912408/53155852-eadca100-35f8-11e9-95b7-db3b43c4e4e1.png)

由上图的调用流程可知 Vue 的构造函数位于 `src/core/instance/index.js`:

```js
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'

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

export default Vue
```

我们绕了这么一大圈（如下图），终于找到了Vue 的原型。这样写有什么好处呢？


![image](https://user-images.githubusercontent.com/3912408/53155787-c385d400-35f8-11e9-98e7-123d66c65651.png)

这样写最大的好处是功能拆分，**可以按功能模块来划分，逐步扩展 Vue 的功能**。

比如在 `core/index.js` 中，我们扩展了 Global API, 在 `instance/index.js` 中 对 Vue做了很多 mixin。

### 静态属性，实例属性，和原型属性

Vue 的源码生成的整个过程中，有一个非常重要的点是，将各种工具函数，实例方法等挂载在 Vue 中。

挂载的形式主要有三种

1. 静态方法挂载。适用于通用功能，不需要实例化就能使用。

比如 Vue.use 用来初始化插件，Vue.config 用来配置 Vue 的特性。基本上可以对应 api 文档中的 [Global API](https://vuejs.org/v2/api/#Global-API).

```js
// src/core/gloal-api/use.js

  Vue.use = function (plugin: Function | Object) {
    //
    return this
  }
```

2. 原型属性挂载。需要实例化的通用功能。

比如 $on, $off, $nextTick, $watch, $set, $delete 等。通用功能放在原型是一个时间换空间的做法。

项目越大，业务越负杂，内存越节省，而原型查找的效率损失可以忽略不计。

3. 实例属性。需要实例化才能使用。

如 vm.$slots等。Vue 在实现过程中还使用了很多内部属性，在整个编码过程中使用。

![image](https://user-images.githubusercontent.com/3912408/53159966-e5378900-3601-11e9-846e-c70c20a5910c.png)


第二种和第三种挂载形式对应于文档[实例属性方法等章节](https://vuejs.org/v2/api/#Instance-Properties)。

以上。

## ISSUE

有问题？来 (Github)[https://github.com/flyyang/blog/issues/17] 一起讨论。