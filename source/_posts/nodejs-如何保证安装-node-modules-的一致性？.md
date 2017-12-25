---
title: nodejs 如何保证安装 node_modules 的一致性？
date: 2017-12-25 12:17:38
tags:
---
## package.json 的困境

package.json 不能够保证每次安装的依赖的唯一性。 举例来说：

A 模块：
```
{
  "name": "A",
  "version": "0.1.0",
  "dependencies": {
    "B": "<0.1.0"
  }
}
```
依赖版本号小于 0.1.0 的 B 模块。
```
{
  "name": "B",
  "version": "0.0.1",
  "dependencies": {
    "C": "<0.1.0"
  }
}
```

<!--more-->

我们在开发的时候 B 模块时 0.0.1。下一次执行 npm install 的 B 模块发布了 0.0.2 版本。此时安装到的版本时 B 的 0.02 版。出现了不一致情况。

npm 推荐 [sermer](https://semver.org/) 的规则来管理自己的版本发布：
```
- MAJOR version when you make incompatible API changes,
- MINOR version when you add functionality in a backwards-compatible manner, and
- PATCH version when you make backwards-compatible bug fixes.
```
sermer 的目的是让代码升级到最新的 bug fix。但是，通常一个项目依赖成百上千个模块，你并不能确定哪一个模块会出问题。

**[一定要选择靠谱的开源模块](https://zhuanlan.zhihu.com/p/22934066)** 并不能解决你的忧虑。

相信别人，还是相信自己？从可控性角度来说，当然是相信自己。**我们需要的是一个 single of truth 的依赖树。**

## yarn.lock

npm 的 对待此问题的行动迟缓 （君不见 nodejs、io-js）， facebook 开发出了 yarn 来解决npm 的 lock 和 cache 等问题。

![image](https://user-images.githubusercontent.com/3912408/34292451-f25806a2-e73b-11e7-96f2-1192e96e7398.png)

版本锁定 与  install 等操作同步，保证了 node_modules 的一致性。实现了我们想要的 single of truth 的依赖树。

## package-lock.json

有竞争就有改进的动力。 npm 5 发布，默认支持 package-lock.json。

```
package-lock.json is automatically generated for any operations where npm modifies either the node_modules tree, or package.json. It describes the exact tree that was generated, such that subsequent installs are able to generate identical trees, regardless of intermediate dependency updates.
```
一个简单的例子：

```
{
  "name": "mobi-pandaren-front-web",
  "version": "0.0.0",
  "lockfileVersion": 1,
  "requires": true,
  "dependencies": {
    "align-text": {
      "version": "0.1.4",
      "resolved": "http://npm.pandatv.com/align-text/-/align-text-0.1.4.tgz",
      "integrity": "sha1-DNkKVhCT810KmSVsIrcGlDP60Rc=",
      "requires": {
        "kind-of": "3.2.2",
        "longest": "1.0.1",
        "repeat-string": "1.6.1"
      }
    },
    "amdefine": {
      "version": "1.0.1",
      "resolved": "http://npm.pandatv.com/amdefine/-/amdefine-1.0.1.tgz",
      "integrity": "sha1-SlKCrBZHKek2Gbz9OtFR+BfOkfU="
    },
    "asap": {
      "version": "2.0.6",
      "resolved": "http://npm.pandatv.com/asap/-/asap-2.0.6.tgz",
      "integrity": "sha1-5QNHYR1+aQlDIIu9r+vLwvuGbUY="
    },
   ...
```
package-lock 描述了所有相关依赖的具体版本，并且会随着 npm install 变化而变化。弥补了 package.json 的不足。

##  npm-shrinkwrap.json

npm-shrinkwrap.json 和 package-lock.json   内容是相同的。不同之处在于：

1.  npm-shrinkwrap.json  优先级较高。install 时会优先采用。
2.  可发布到 registry。使用场景仅在：模块通常以 daemon 形式后台运行，或者依赖在dev中。其他请用package-lock.json。

## 结论

要使得避免“我这里是好的”这种情况。npm 5 是不错的选择。低版本推荐用 yarn 替代。
