---
title: '指定Python源文件编码, why，where，what and who'
date: 2017-05-09 11:17:07
tags: Python
---
在看 Python 源代码的时候，经常会碰到如下注释：

``` bash
 #!/usr/bin/env python
 # -*- coding: utf-8 -*-
```

```
 # -*- coding: utf-8 -*-
```

见名知意，大概可以猜测到本文件是utf-8编码，但是为什么呢？

<!-- more -->

## why

Python 是一门解释性语言。在正式执行之前，需要经过Decoding -> Tokenizing -> Parsing -> AST -> Compiling这么多流程。

解释器如何在Decoding阶段就知道文件如何编码的呢？[PEP 263](https://www.python.org/dev/peps/pep-0263/) 提出了解决方案：在源文件中以上面的格式指定，解释器按照格式解析出对应的编码。

如果没有指定编码呢？

Python 2.x 默认会按照 `ascii`读取。所以，如果在 2.x 源码中加入中文，会报编码错误。

Python 3.x 则默认读取 `utf-8` 模式。

## where

源文件第 `1` 或者 `2` 行。

## what

指定的格式必须满足以下格式

[PEP 263](https://www.python.org/dev/peps/pep-0263/):
```
^[ \t\v]*#.*?coding[:=][ \t]*([-_.a-zA-Z0-9]+)
```
或, [Python Language Specification](https://docs.python.org/3/reference/lexical_analysis.html#encoding-declarations)：

```
coding[=:]\s*([-\w.]+)
```

解释一下这个正则表达式:
所有的编码指定必须在一行内。

不必以 # 号开头，# 号前可以有0个或者多个空白符。# 到coding 可以是除换行符意外的任何字符。coding后可以是 `=` 或者 `:` , 再之后是可选的空白符，然后是编码(参考[Python encoding list](https://docs.python.org/2.4/lib/standard-encodings.html))。

## who

指定的格式给谁看的呢？

1. 主要是给解释器看的。
2. 有的编辑器也能理解某种格式的注释。
3. 给人看。

针对第2点：
Emacs用户：

``` bash
 # -*- coding: utf-8 -*-
```
Vim 用户：

``` bash
# vim:fileencoding=<encoding-name>
```

> 参考： [correct-way-to-define-python-source-code-encoding](http://stackoverflow.com/questions/728891/correct-way-to-define-python-source-code-encoding`)
