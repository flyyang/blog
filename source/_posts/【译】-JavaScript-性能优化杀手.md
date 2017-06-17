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

1. 工具
2. 不支持的语法
3. 管理 `arguments`
4. `Switch-case`
5. `For-in`
6. 无明显退出条件或退出条件逻辑嵌套太深的无限循环

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

### 不支持的语法

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

Likely never optimizable:

Functions that contain a debugger statement
Functions that call literally eval()
Functions that contain a with statement

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

3. 管理 `arguments`

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

