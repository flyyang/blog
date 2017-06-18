---
title: 【译】 JavaScript 性能优化杀手中文版
date: 2017-06-16 01:26:23
tags: JavaScript 性能优化
---

# JavaScript 性能优化杀手 【中文版】

## 引子

此文档包含了一些关于如何避免写出性能严重超出预期的代码的建议。尤其是那些导致 V8
（影响 Node.js ，Opeara, Chromium...）拒绝对函数进行优化的反模式。

vhf 同学也在维护一个类似的项目，试图涵盖所有 V8 Crankshaft 引擎性能优化杀手的
方方面面：[V8 Bailout Reasons](https://github.com/vhf/v8-bailout-reasons)

<!--more -->

### 一点V8背景

在 V8 内部，没有解释器，但是有两个不同的编译器：通用编译器和优化编译器。意味着
你的 JavaScript 代码总是首先被编译，然后直接以机器码运行。这样就意味着快速，对
吗？错误。由 JavaScript 编译器直接编译出的机器码对性能并不会有明显(相对解释器)
的提升。它只不过去掉了解释的过程，但是如果代码不经优化的话，仍然比较慢。

举例来说， `a + b` 经过编译器编译如下的代码：

```
mov eax, a
mov ebx, b
call RuntimeAdd
```

换话句话说，它仅仅是调用了运行时的方法。如果 `a` 和 `b` 一直是整数，那么会编译
出如下的代码：

```
mov eax, a
mov ebx, b
add eax, ebx
```

后者要比前者快很多。原因在于，后者不需要去考虑加号在 JavaScript 中的复杂语义
了。

通常来说，通用编译器编译出第一种代码，优化编译器则编译出第二种。我们可以很自信
的说，优化编译器编译出来的代速度优于普通编译器编译出来的代码几百倍。但是有一些
小的绊脚石，让你不能够写出完全被优化的 `JavaScript` 代码。这里有很多模式，甚至
有些我们日常会碰到的代码， 会被 JavaScript 的优化编译器拒绝处理(即 bails out)。

需要特别注意的是，一旦遇到拒绝优化的模式，会导致整个包含函数被拒绝优化。代码是
一个函数接一个函数进行优化的, 相互之间是透明的（除非其中一个以 in-line 的方式定
义在另一个内部）。

本文档将包含大部分的导致函数堕入拒绝优化的 hell 中的模式。不过随着优化编译器识
别越来越多的模式，文档中的主题会随之调整，一些当时适用 work around 也可能不再
必要了。


## 主题

1. [工具](#1-工具)
2. [不支持的语法](#2-不支持的语法)
3. [管理 arguments](#3-管理-arguments)
4. [`Switch-case`](#4-Switch-Case)
5. [`For-in`](#5-For-in)
6. [无明显退出条件或退出条件逻辑嵌套太深的无限循环](#6-无明显退出条件或退出条件逻辑嵌套太深的无限循环)

### 1. 工具

你应该使用 Nodejs 以及一些 V8 的参数来验证一些模式是如何影响优化的。通常情况
下，你需要创建一个包含模式的函数，通过各种方式调用他，然后调用 V8 的函数优化
它，观察对比：

test.js:

```
// Function that contains the pattern to be inspected (using an `eval`
// statement)
function exampleFunction() {
  return 3;
  eval('');
}

function printStatus(fn) {
  switch(%GetOptimizationStatus(fn)) {
    case 1: console.log("Function is optimized"); break;
    case 2: console.log("Function is not optimized"); break;
    case 3: console.log("Function is always optimized"); break;
    case 4: console.log("Function is never optimized"); break;
    case 6: console.log("Function is maybe optimized"); break;
    case 7: console.log("Function is optimized by TurboFan"); break;
    default: console.log("Unknow optimization status"); break;
  }
}

// Fill type-info
exmapleFunction();
// 2 calls are needed to go from unintialized -> pre-monomorphic -> monomorphic
exmapleFunction()

%OptimizeFunctionOnNextCall(exampleFunction);
// The next Call
exmapleFunction()

//Check
printStatus(exampleFunction);
```
运行它：

```
$ node --trace_opt --trace_deopt --allow-natives-syntax test.js
(v0.12.7) Function is not optimized
(v4.0.0) Function is optimized by TurboFan
```
[https://codereview.chromium.org/1962103003](https://codereview.chromium.org/1962103003)

如果想让他工作，注释掉 eval 再次运行脚本：

```
$ node --trace_opt --trace_deopt --allow-natives-syntax test.js
[optimizing 000003FFCBF74231 <JS Function exampleFunction (SharedFunctionInfo 00000000FE1389E1)> - took 0.345, 0.042, 0.010 ms]
Function is optimized
```

学会使用工具验证是非常重要的一个步骤。

### 2. 不支持的语法

一些语法结构不被优化编译器所支持，所以使用这些语法会导致函数不被优化。

需要指出的是，尽管一些结构可能是不可以执行到，或者不被运行，这些结构仍然是不可
优化的。

举例，这样做是无效的：

```
if (DEVELOPMENT) {
  debugger;
}
```

上面的代码片段会导致整个包含函数被拒绝优化，尽管它永远不会被访问到。

当前没有被优化的项目：

* ~~Generator 函数~~（优化于 [V8 5.7](https://v8project.blogspot.de/2017/02/v8-release-57.html)）
* ~~包含 for of 语句的函数~~ (优化于V8 commit [11e1e20](https://github.com/v8/v8/commit/11e1e20) ）
* ~~包含 try catch 的函数~~ （优化于 V8 commit [9aac80f](https://github.com/v8/v8/commit/9aac80f)/ V8 5.3 / node 7.x）
* ~~包含 try-finally 的语句~~ （优化于 V8 commit [9aac80f](https://github.com/v8/v8/commit/9aac80f)/ V8 5.3 / node 7.x）
* ~~包含复合 let 赋值的函数~~（优化于 Chrome 56/ V8 5.6）
* ~~包含复合 const 赋值的函数~~（优化于 Chrome 56/ V8 5.6）
* 对象字面量包含 `_proto` 或者，`get` `set` 的声明的函数。

可能永远不会优化的项目：

* 包含 `debugger` 语句的函数
* 调用 `eval()` 执行字符串 的函数
* 包含 `with` 语句的函数

最后要说明的一点的是,出现以下任何一种形式，整个包含函数将不被优化：

```
function containsObjectLiteralWithProto() {
    return {__proto__: 3};
}
```

```
function containsObjectLiteralWithGetter() {
    return {
        get prop() {
            return 3;
        }
    };
}
```

```
function containsObjectLiteralWithSetter() {
    return {
        set prop(val) {
            this.val = val;
        }
    };
}
```


有必要特别提到的是, 直接使用 `eval` 或 `with` 会生成动态作用域，存在污染其他函
数的可能性，因为你不知道从词法作用域的角度来确定变量到底在哪个作用域里。

### Workarounds

一些语句在生产环境中是无法避免的，如 `try-finally` 以及 `try-catch`。为了最小化
影响，他们必须被放到一个小的函数里，使主函数不被污染。

```
var errorObject = {value: null};
function tryCatch(fn, ctx, args) {
    try {
        return fn.apply(ctx, args);
    }
    catch(e) {
        errorObject.value = e;
        return errorObject;
    }
}

var result = tryCatch(mightThrow, void 0, [1,2,3]);
//Unambiguously tells whether the call threw
if(result === errorObject) {
    var error = errorObject.value;
}
else {
    //result is the returned value
}
```

### 3. 管理 `arguments`

`arguments` 有多种方法导致函数不被优化。使用 `arguments` 必须非常小心。

#### 3.1 对参数重新赋值，并且在函数体内包含了 `arguments` （非严格模式下）。
例：

```
function defaultArgsReassign(a, b) {
     if (arguments.length < 2) b = 5;
}
```

Workaround, 把参数保存到另一个新的变量内：

```
function reAssignParam(a, b_) {
    var b = b_;
    //unlike b_, b can safely be reassigned
    if (arguments.length < 2) b = 5;
}
```

如果这是函数中的唯一用例，可以使用 `undefined` 检查来替换：

```
function reAssignParam(a, b) {
    if (b === void 0) b = 5;
}
```

但如果后续又引入 `arguments` ， 维护时可能会遗忘做类似替换

Workround2: 为每一个函数，每一个文件开启 `use strict`。

#### 3.2 `arguments` 返回

```
function leaksArguments1() {
    return arguments;
}
```

```
function leaksArguments2() {
    var args = [].slice.call(arguments);
}
```

```
function leaksArguments3() {
    var a = arguments;
    return function() {
        return a;
    };
}
```
`arguments` 对象不能够被传递或者返回到任何地方。

Workaround, inline 形式返回：

```
function doesntLeakArguments() {
    //.length is just an integer, this doesn't leak
    //the arguments object itself
    var args = new Array(arguments.length);
    for(var i = 0; i < args.length; ++i) {
                //i is always valid index in the arguments object
        args[i] = arguments[i];
    }
    return args;
}

function anotherNotLeakingExample() {
    var i = arguments.length;
    var args = [];
    while (i--) args[i] = arguments[i];
    return args
}
```
完成一个 `workaround` 需要很多代码，这很烦人，所以你需要权衡一下是否真的有必要
这么做。优化通常意味着越来越多的代码，而代码越多，意味着需要深入了解 JavaScript
的语义。

不过，如果你有构建这一步骤的话，可以实现一个宏来构造soucre map，直接书写原来形
式的代码即可。

```
function doesntLeakArguments() {
    INLINE_SLICE(args, arguments);
    return args;
}
```

上面的技术应用在 blubird 里， 构建出的结果如下：

```
function doesntLeakArguments() {
    var $_len = arguments.length;
    var args = new Array($_len);
    for(var $_i = 0; $_i < $_len; ++$_i) {
        args[$_i] = arguments[$_i];
    }
    return args;
}
```

#### 3.3 赋值给 `arguments`

在非严格模式下，可以出现以下代码：

```
function assignToArguments() {
    arguments = 3;
    return arguments;
}
```

Workaround: 不要写这种白痴代码。在严格模式下，此代码会抛出异常。

那么，什么是正确使用 `arguments`  的姿势呢？

仅使用：

* arguments.length
* arguments[i], i 必须是合法的整数索引，并且不能超出范围。
* 除上两条情况外，不要直接使用 arguments
* 可以使用 fn.apply(y, arguments), 仅此一例，没有其他使用情况，如 .slice, 函数
 apply 比较特殊。
* 需要注意的是，添加属性到函数（如，fn.$inject =... ）以及绑定函数（也就是 函数
 bind 的结果）会生成隐藏类，因此使用 apply 是不安全的。

另外关于用到 arguments 会造成 arguments 对象的分配这一点的 FUD (恐惧), 在使用限
于上面提到的安全的方式时是不必要的.

### 4. Switch-Case

以前，一个 switch-case 语句最多包含 128个 case 语句，超过 128 个的包含函数将不
被优化。

```
function over128Cases(c) {
    switch(c) {
        case 1: break;
        case 2: break;
        case 3: break;
        ...
        case 128: break;
        case 129: break;
    }
}
```

你需要保持 switch-case 中 case 语句小于等于 128 条，可以使用 函数数组，或者
if-else 语句替换。

这个限制被提升了，查看[此评论]（https://bugs.chromium.org/p/v8/issues/detail?id=2275#c9）

## 5. For-in

For-in 语句在某些情况下会导致整个函数被拒绝优化。

所有的这些情况可以归结为“ For-in 太慢了”这一结论。

#### 5.1 key 不是本地变量

```
function nonLocalKey1() {
    var obj = {}
    for(var key in obj);
    return function() {
        return key;
    };
}

var key;
function nonLocalKey2() {
    var obj = {}
    for(key in obj);
}
```

因此，key 不能来自于上层或下层作用域，只能是本地变量。

#### 5.2 被遍历的对象不是简单的 enumerable 对象

##### 5.2.1 1. 对象处于 “哈希表模式” （也就是 “规约化模型”，或者 “字典模式” -
以哈希表为数据结构的对象）不是简单的 enumerable 对象

```
function hashTableIteration() {
    var hashTable = {"-": 3};
    for(var key in hashTable);
}
```

当你添加太多动态属性（在构造函数外），删除属性， 或者使用非正式的标识符等，对象
会进入哈希表模式。换话句话说，当你像哈希表一样使用对象时，那么它就是哈希表。传
递这样的对象给哈希表是错误的。你可以开启 nodejs 的 `--allow-natives-syntax` 并
使用 `console.log(%)HashFastProperties(obj)` 来判定一个对象是处于哈希表模式。

##### 5.2.2 对象原型链上具有 enumerable 属性

```
Object.prototype.fn = function() {}
```

上面的步骤给所有对象添加了一个 enumerable 的属性（除了 Object.create(null) ），
因此任何包含 for-in 的语句将不会被优化（除非仅仅遍历 Object.create(null) 对象）。

你可以通过 Object.defineProperty（不推荐在运行时调用，但是可以定义一些高效的静
态属性比如原型属性） 创建非 enumerable 对象。

##### 5.2.3 对象包含可枚举的数组片段

一个属性是不是一个数组索引，在 ecmascript 文档中定义如下：

> 一个属性名 P（String 值的形式）是否是array 索引，必须满足
ToString(ToUnit32(p)) 等于 P 并且 ToUnit32(p) 不等于 232-1。 如果一个属性的属性
名是一个数组索引，属性名也被叫做 元素。

通常情况下，这是数组。但是对象也可以有数组片段： `normalObj[0] = value`;

```
function iteratesOverArray() {
    var arr = [1, 2, 3];
    for (var index in arr) {

    }
}
```

因此使用 for-in 不仅仅遍历数组比较慢，而且会导致整个包含函数不被优化。

如果你传递一个非简单可遍历的对象给 `for-in` 语句，也会导致整个包含函数不被优化。

Workaround: 一直使用 `Object.keys` 并且 使用 for 循环替代 `for-in`。 如果真的需
要整个原型链所有的属性，使用一个隔离的帮助函数：

```
function inheritedKeys(obj) {
    var ret = [];
    for(var key in obj) {
        ret.push(key);
    }
    return ret;
}
```

## 6. 无明显退出条件或退出条件逻辑嵌套太深的无限循环

在编码的时候，你知道某一处需要一个循环，但是却不知道退出条件是什么。因此你谢了
一个 while(true) { 或者 for(;;) { , 后来写了一个 break 条件，搁在这里直到忘掉。
重构时发现函数变慢或者你看到了一些反优化的模式 —— 这可能是罪魁祸首。

将循环的退出条件重构到循环自己的条件部分可能并不容易. 如果代码的退出条件是结尾
 if 语句的一部分, 并且代码至少会执行一次, 那可以重构为 do { } while (); 循环.
 如果退出条件在循环开头, 把它放进循环本身的条件部分. 如果退出条件在中间, 你可以
尝试 “滚动” 代码: 每每从开头移动一部分代码到末尾, 也复制一份到循环开始之前. 一
旦退出条件可以放置在循环的条件部分, 或者至少是一个比较浅的逻辑判断, 这个循环应
该就不会被反优化了.

