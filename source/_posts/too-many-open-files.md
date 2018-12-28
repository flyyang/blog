---
title: Too many open files in system 问题排查记录
date: 2018-12-28 17:47:00
tags:
---

同事新上线了一个日志 sdk。运行一段时间后导致线上服务器打开太多文件而拒绝服务。

原因很简单每一个日志实例化，都持有了一个写文件的流，写完没有关闭。

随着业务运行导致服务器崩溃。

<!--more-->

## fd （file descriptor） 是什么？

我们先来介绍一下 fd，一个进程所有打开的文件可以通过 fd 查询。

举例来说，当一个程序（进程）要写文件时，会像操作系统申请权限。操作系统会授予一个标记（通常是数字）来指向所描述的文件。这个标记就是 fd。

在 centos 中，一个进程的所有打开的 fd 在 `/proc/进程id/fd` 下。

## 如何排查

比如一个 node 服务, 我们先找一下他的进程 id


```
ps aux | grep node

```

![image](https://user-images.githubusercontent.com/3912408/50510523-04fc7600-0ac5-11e9-81a1-5b7f0bfae4b3.png)

第二列就是进程 id。有了进程 id 就可以查一下具体的 fd

```
sudo ls -l /proc/29027/fd
```

![image](https://user-images.githubusercontent.com/3912408/50510359-59ebbc80-0ac4-11e9-8943-317478b2b71e.png)

当打开文件数量过多时，可以通过命令查看打开连接的总数：

```
sudo ls -l /proc/29027/fd | wc -l
```

## 结论

从现象定位问题是比较简单的。上面写了一些排查还有测试过程。这里在描述一下解决方案：

在 nodejs 中，流是一个非常重要的概念。在日志这个场景中。我们打开一个流，日志直接往内部写就可以了。在进程退出或者日志路径切换过程中销毁并新建即可。而不需要每次都新建一个流。

所以用全局的流来写日志是一个不错的方案。即便是多进程，以及按等级分不同的流，它的复杂度也是 O(n)。高效并且可控。

```
类似 appendFile 这种 buffer 形式写文件，每次写入都需要打开关闭，不适合来做日志。
```

## 参考

* [file descriptor](https://www.bottomupcs.com/file_descriptors.xhtml)
* [fs.createWriteStream](https://nodejs.org/api/fs.html#fs_fs_createwritestream_path_options)


## ISSUE
有问题？来 [github](https://github.com/flyyang/blog/issues/12) 一起讨论。