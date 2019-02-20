---
title: 深入浅出 Vue 构建流程
date: 2019-02-20 16:31:17
tags: vue
---
很多同学看源码分析，一般直接从 src 看起。像 `Vuejs` 这种项目，目录繁杂，入口都很难找到。不从中拎出几个线头来，很难从全局把握源码的流程。

理解源码是如何构建的，能帮助我们梳理这个过程。

<!-- more -->

## 从 `package.json` 开始

我们关注三个字段：

* "main": "dist/vue.runtime.common.js"。

默认 `require('vue')` , `import('vue')` 会读取的文件。格式为 `commonjs`。

* "module": "dist/vue.runtime.esm.js"

webpack 2 以上会默认读 module 字段指定的文件。esm 模块方便做 tree shaking。

* "scripts"

现在很多的包使用 npm scripts 来做构建流程脚本。我们可以看到 vue 内部有许多研发流程脚本在 scripts字段内， 我们关注 build 部分代码：

```
    "build": "node scripts/build.js",
    "build:ssr": "npm run build -- web-runtime-cjs,web-server-renderer",
    "build:weex": "npm run build -- weex",
```

会发现所有构建相关的逻辑都在 `scripts/build.js`  内。

##  rollup

在分析 `scripts/build.js` 脚本之前，我们先来简单介绍一下 rollup。

业界有一种说法：webpack 适合打包应用程序，而 rollup 适合打包应用库。rollup 是一个以 利用 es 模块特性，treeshaking 起家的库，相比webpack ，可以打包出各种模式（cmd, umd, es）的类库。

而 vue 的源码就是用 es 模块编写的，它的构建脚本整个都是围绕如何组合 rollup 的配置。

打包工具的通常关注的配置是，入口是什么，输出文件放哪里，输出什么格式，用什么插件，不同的环境怎么处理。vue 的构建脚本并不例外。

## 构建脚本分析

如上所诉，vue 的构建脚本就是围绕 rollup ，做一些自定义配置。相关代码在：

* `scripts/build.js` 主构建流程
* `scripts/alias.js`  文件路径 alias
*  `scripts/config.js` 生成 rollup 配置

我们简要分析一下这几个文件。在 `scripts/build.js` 中：

```
function buildEntry (config) {
  const output = config.output
  const { file, banner } = output
  const isProd = /(min|prod)\.js$/.test(file)
  return rollup.rollup(config)
    .then(bundle => bundle.generate(output))
    .then(({ output: [{ code }] }) => {
      if (isProd) {
        const minified = (banner ? banner + '\n' : '') + terser.minify(code, {
          toplevel: true,
          output: {
            ascii_only: true
          },
          compress: {
            pure_funcs: ['makeMap']
          }
        }).code
        return write(file, minified, true)
      } else {
        return write(file, code)
      }
    })
}
```
buildEntry 方法暴露了 vue 的构建流程是依赖 rollup 的。

在 `scripts/config.js` 中：

```
const resolve = p => {
  const base = p.split('/')[0]
  if (aliases[base]) {
    return path.resolve(aliases[base], p.slice(base.length + 1))
  } else {
    return path.resolve(__dirname, '../', p)
  }
}
const builds = {
  // Runtime only ES modules build (for bundlers)
  'web-runtime-esm': {
    entry: resolve('web/entry-runtime.js'),
    dest: resolve('dist/vue.runtime.esm.js'),
    format: 'es',
    banner
  },
}
```
以生成的 es 模块为例，他的入口文件为 `resolve('web/entry-runtime.js')`。

config 的 resolve 遇到在 alias 中的文件会特殊处理。alias 定义在 `scripts/alias.js`。

```
const resolve = p => path.resolve(__dirname, '../', p)

module.exports = {
  vue: resolve('src/platforms/web/entry-runtime-with-compiler'),
  compiler: resolve('src/compiler'),
  core: resolve('src/core'),
  shared: resolve('src/shared'),
  web: resolve('src/platforms/web'),
  weex: resolve('src/platforms/weex'),
  server: resolve('src/server'),
  sfc: resolve('src/sfc')
}
```

这些alias 会贯穿整个 vue 源码中。`resolve(web/entry-runtime.js)` 最终地址位于 src/platforms/web/entry-runtime.js。

```
Vue 的 入口 都在 plaforms 的 web 、 weex、sever 下。具体参考 scripts/config.js。
出口位于 dist 和 packages 内，其中 packages 为独立发布的包。下图标注了 web 和 weex 部分的入口。
```
![image](https://user-images.githubusercontent.com/3912408/53076663-29a42580-352b-11e9-9fad-cf8ede480f81.png)

找到这个入口文件之后，我们就拿到了分析源码的一条绳。从此文件开始，便可以分析 Vue 的源码是如何工作的。

## 参考资料

* [rollup](https://rollupjs.org/guide/en)

## ISSUE

有问题，来 【GitHub】(https://github.com/flyyang/blog/issues/16) 一起讨论。
