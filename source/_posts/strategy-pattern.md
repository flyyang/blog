---
title: 简明 js 设计模式 —— 策略模式
date: 2019-03-24 23:06:48
tags: 设计模式
---
## 场景

假如要写一个文件解析器。针对不同的文件类型调用不同的方法。如果用一个函数来表达，可能会写出如下代码：

```js
function fileParser(fileType) {
  if (fileType === 'js') jsParser();
  if (fileType === 'txt') txtParser();
  // ....
}
```
每增加一种类型，我们需要修改我们的 fileParser 函数，增加一个条件判断。这样不够优雅。

## 实现

nodejs 的 require 函数也会遇到上面的情况，它需要针对三种不同的类型 —— js、json、node 分别做处理。我们来看看它是如何[实现](https://github.com/nodejs/node/blob/master/lib/internal/modules/cjs/loader.js)的：

```js
// Native extension for .js
Module._extensions['.js'] = function(module, filename) {
  var content = fs.readFileSync(filename, 'utf8');
  module._compile(stripBOM(content), filename);
};


// Native extension for .json
Module._extensions['.json'] = function(module, filename) {
  const content = fs.readFileSync(filename, 'utf8');

  if (manifest) {
    const moduleURL = pathToFileURL(filename);
    manifest.assertIntegrity(moduleURL, content);
  }

  try {
    module.exports = JSON.parse(stripBOM(content));
  } catch (err) {
    err.message = filename + ': ' + err.message;
    throw err;
  }
};


// Native extension for .node
Module._extensions['.node'] = function(module, filename) {
  if (manifest) {
    const content = fs.readFileSync(filename);
    const moduleURL = pathToFileURL(filename);
    manifest.assertIntegrity(moduleURL, content);
  }
  // Be aware this doesn't use `content`
  return process.dlopen(module, path.toNamespacedPath(filename));
};
```
这些函数在 `Module.prototype.load` 过程中调用。

```js
Module.prototype.load = function(filename) {

  Module._extensions[extension](this, filename);
  // ...
}
```

我们看到这个过程没有了条件判断的流程。而且对多类型支持也非常方便，只需要扩展类型方法即可，不需要修改所谓的 fileParser 方法（符合 open / close 原则）。

策略模式便是在运行时选择算法，并且消除了判断流程的一种模式。

## 参考

* [https://en.wikipedia.org/wiki/Behavioral_pattern](https://en.wikipedia.org/wiki/Behavioral_pattern)
* [https://en.wikipedia.org/wiki/Strategy_pattern](https://en.wikipedia.org/wiki/Strategy_pattern)

## ISSUE

有问题？来 [Github](https://github.com/flyyang/blog/issues/20) 一起讨论。