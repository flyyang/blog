---
title: xdomain.js 实现原理分析
date: 2018-12-21 11:42:52
tags: 跨域请求
---

## xdomain 出现的背景

由于浏览器的同源策略，跨域请求需要做额外的配置才能工作。

CORS 是一个通用的解决方案，唯一缺点是低版本浏览器（< IE 9）不支持。低版本 IE 支持一个私有的 [XDomainRequest](https://developer.mozilla.org/en-US/docs/Web/API/XDomainRequest)，是个鸡肋 —— 如不能带 cookie。

![image](https://user-images.githubusercontent.com/3912408/50285608-a9643400-0497-11e9-8d5c-fee5c1e30747.png)

如果需要兼容  IE 9 以下, 需要另外一种方案来实现跨域请求。xdomian 便是其中一种。

<!--more-->

## xdomain 实现原理

xodomain 本质是利用 iframe 和 postMessage 技术来实现跨域请求的。

以一个请求为例。localhost:8000 请求 localhost:58000 的 api。

```
// localhost:8000 的 index.html
<html>

<head>

</head>

<body>
  <script src="./xdomain.js"></script>
  <script src="https://cdn.bootcss.com/jquery/3.3.1/jquery.min.js"></script>
  <script>
    xdomain.slaves({
      "http://localhost:58000": "/proxy.html"
    })
    // xdomain.debug = true;
    $.get('http://localhost:58000/api/test').then(function (data) { debugger; console.log(data) });

    window.addEventListener('message', data => console.log(data.data, 'slave to master'));
  </script>
</body>

</html>
```
localhost:58000 proxy.html:

```
<!doctype html>
<html>

<head>

</head>

<body>
  <script src="./xdomain.js"></script>
  <script>
    xdomain.masters({
      "http://localhost:8000": "*"
    })
    window.addEventListener('message', (data) => { console.log(data.data, 'master to slave') })
  </script>
</body>

</html>
```

完整测试用例在[这里](https://github.com/flyyang/xdomain-test)。

xdomain 的思路是：
```
1. 在 localhost:8000 建立一个 `**iframe**`, 指向 localhost:58000 的一个页面，通常是 proxy.html。
2. 当 a 向 b 发起请求时，将请求转给 localhost:58000/proxy.html 来真正发起，避免跨域问题。
3. 最后结果从 /proxy.html 传递给 localhost:8000。
4. 通过 xhook，将返回最终传递给 a 的请求。
```
这里涉及到的消息传递通过 `**postMessage*`（兼容 IE8） 来实现。

如果将 localhost:8000 称为 master, localhost:58000称为 slave。那么通常一个经过代理的请求消息传递如下：

![image](https://user-images.githubusercontent.com/3912408/50285105-22fb2280-0496-11e9-84a1-900373db7f5c.png)

```
0. master 和 slave 各建立一个 socket。
1. localhost:8000 iframe 加载完毕后，从slave 发一条 XDPING_V1 消息给 master。 告知 master 我已加载完毕。
2. master 发一条 XD_CHECK 给 slave。 检查slave 状态。
3. slave 发给 master 一条 ready 信息。告知 master已准备好。
4. slave 发给 master 一条 XD_CHECK 信息。检查 master 状态。
5. master 发给 slave 一条 ready 信息。告知 master已准备好。并将请求的信息给slave。如请求的地址，是否带cookie 等信息。
6. slave 告知 master 开始 request。
7. 中间触发一系列的 xhr-event。从 slave 到 master。
8. slave 触发 response 告知已经请求完毕，并带上请求的数据。至此，master 已经拿到了数据。
9. master 告知 slave 关闭 socket。
10. slave 告知 master 关闭 socket。
```
注意在第7条中， xhr-event 是 proxy.html 发 ajax 触发一系列请求过程的消息转发。通过 [xhook]() 控制  master 的 beforeXHR 将数据最终放到 a.com 的 请求 response 里。
## 其他

* xdomian 实现了一套自己的通信协议。并且主从代码打在了一起。从设计到实现都很精妙。

* xdomain 作者不推荐 CORS，[理由](https://github.com/jpillora/xdomain#faq--troubleshooting) 有点站不住脚。通常只需要在 IE 9 以下 做兼容。

```
<!--[if lte IE 9]>
    <script  src="xdomain.js"></script>
<![endif]-->
```

* 面向事件编程代码难以阅读。很难理清楚各个事件的顺序，以及调用关系。个人感觉可以通过事件代码汇总，文档来降低复杂度。

## 参考链接

* [xdomain](https://github.com/jpillora/xdomain)
* [XDomainRequest](https://developer.mozilla.org/en-US/docs/Web/API/XDomainRequest)
* [contentWindow](https://developer.mozilla.org/en-US/docs/Web/API/HTMLIFrameElement/contentWindow)
* [Event-driven architecture](https://en.wikipedia.org/wiki/Event-driven_architecture)

ISSUE
有问题？来 [github](https://github.com/flyyang/blog/issues/11) 一起讨论。