---
title: Resource Hint 介绍
date: 2018-12-13 20:21:41
tags:
---
## Resource Hint 是什么？

简而言之。辅助浏览器用来做资源优化的 `指令`。

为什么需要这些 `指令` 呢？

浏览器已经长大成人了，已经懂得如何做优化了。但是具体到每个应用，各有不同，要具体方案具体分析。
这就是指令的目的。

一些常见的指令：

* dns-prefetch
* preconnect
* prefetch
* preload
* prerender

这些指令通常写在 head 标签的 meta 里, 形式如下：

```
<link rel="xxx" href="yyy">
```
但也会有些不同。下面做详细介绍。

<!-- more -->

## Resource Hint  分类介绍

先介绍 `dns-prefetch` 和 `preconnect`。在介绍这两者之前， 先看一下这张图：

![image](https://user-images.githubusercontent.com/3912408/49926966-d2b11d00-fef7-11e8-9fee-7268923b83cc.png)

`dns-prefetch` 是用来解决 `DNS Lookup` 的问题。`dns-preconnect` 解决 `DNS Lookup` + `Initial connection` + `SSL` 问题。

### dns-prefetch

![image](https://user-images.githubusercontent.com/3912408/49926951-c88f1e80-fef7-11e8-919e-d9b053c3ff04.png)

dns-prefech 支持很广泛。从 ie9 到现代浏览器，都支持此特性。由于支持比较早，使用方法也比较多：

1.  服务端返回 `X-DNS-Prefetch-Control`
2. 在页面 head 部分添加 meta 标签。

```
<a href="http://a.com"> A) Default HTTPS: No prefetching </a>
<meta http-equiv="x-dns-prefetch-control" content="on">
<a href="http://b.com"> B) Manual opt-in: Prefetch domain resolution. </a>
<meta http-equiv="x-dns-prefetch-control" content="off">
<a href="http://c.com"> C) Manual opt-out: Don't prefetch domain resolution </a>
<meta http-equiv="x-dns-prefetch-control" content="on">
<a href="http://d.com"> D) Already opted out: Don't prefetch domain resolution. </a>
```
 3. 手动指定。

```
<link rel="dns-prefetch" href="hostname">
```

注意， chrome 71 似乎不再支持 dns-prefetch 。测试用例见  [dns-prefetch 测试用例](https://github.com/flyyang/resource-hint-demo/tree/master/dns-prefetch)。
### preconnect

![image](https://user-images.githubusercontent.com/3912408/49928614-7b14b080-fefb-11e8-8773-6f2bf09581f2.png)

preconnect 的作用已经介绍。使用方法也比较简单：

```
<link rel="preconnect" href="//example.com">
```

在解决了网络的问题之后, 我们再来看加载的问题：

### preload

重要资源越早加载越好。

![image](https://user-images.githubusercontent.com/3912408/49934602-c8981a00-ff09-11e8-82d8-b59131786603.png)

通常浏览器加载时散列状的，或者说是一块一块并发的。
![image](https://user-images.githubusercontent.com/3912408/49937412-95f21f80-ff11-11e8-8a11-7be7013dee18.png)

preload 却可以形成如下的效果：

![image](https://user-images.githubusercontent.com/3912408/49935783-f894ec80-ff0c-11e8-848f-38a2fc09e806.png)

什么是重要资源呢，在可控范围内，我认为所有用到的资源都可以是重要资源。当然也可以按优先级来区分。要具体问题具体分析。

更多参考 [w3c preload](https://www.w3.org/TR/2017/CR-preload-20171026/)。

```
preload 并不属于 w3c 的 resource hint，但是目标确是一致的。所以拿到了这里统一分析。
```

### prefetch

接下来可能会用到的资源偷偷加载。

![image](https://user-images.githubusercontent.com/3912408/49934634-dc438080-ff09-11e8-9c24-7c23946b1b47.png)。


需要特别说明的是，`webpack` 支持了 preload 和 prefetch 功能。

比如我们需要在点击的时候异步加载一个组件：

```
$('#test').click(() => {
  (name) =>  import(/* webpackPrefetch: true */'src/component/${name}')
})
```

以往的异步加载是点击时，拉取对应的资源。而通过该指令可以在浏览器空闲时偷偷拉取资源。

### prerender

干脆偷偷加载一个新页面得了。
![image](https://user-images.githubusercontent.com/3912408/49934650-e82f4280-ff09-11e8-8f90-76dfd525934d.png)

使用方法：

```
<link rel="prerender" src="some-url">
```
chrome 的 prerender 并不会直接渲染页面。在各个浏览器测试功能都比较鸡肋。参考[Intent to Deprecate and Remove: Prerender](https://groups.google.com/a/chromium.org/forum/#!topic/blink-dev/0nSxuuv9bBw)。

## 总结

* 所有的这些指令，都不会占用你的渲染进程。而是浏览器提供的额外 bonus。
* preconnect 比 dns-prefetch 做的更进一步，优先选用 preconnect。
* preload 和 prefetch 配合 webpack 使用更佳。
* prerender 功能有些鸡肋。慎用。

## 参考链接

* [Intent to Deprecate and Remove: Prerender](https://groups.google.com/a/chromium.org/forum/#!topic/blink-dev/0nSxuuv9bBw)
* [w3c preload](https://www.w3.org/TR/2017/CR-preload-20171026/)
* [w3c Resource Hint](https://www.w3.org/TR/resource-hints/)
* [webpack prefetch preload](https://medium.com/webpack/link-rel-prefetch-preload-in-webpack-51a52358f84c)
* [测试用例](https://github.com/flyyang/resource-hint-demo)

## ISSUE

有问题？来 [github](https://github.com/flyyang/blog/issues/10) 一起讨论。
