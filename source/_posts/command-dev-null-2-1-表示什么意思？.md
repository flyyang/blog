---
title: command > /dev/null 2>&1 表示什么意思？
date: 2017-06-27 23:39:30
tags: linux
---

## 文件描述符

每开启一个 linux(unix) 应用程序（daemon 除外），默认会打开三个文件描述符。即
stdin (标准输入， 文件描述符 0)， stdout(标准输出， 文件描述符1),
stderr(标准错误， 文件描述符 2)。


