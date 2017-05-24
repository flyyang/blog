---
title: git 找回删除的文件
date: 2017-05-24 16:50:56
tags: git
---
误删除的文件可以被恢复，前提是在此之前进行过 `git add` 或者 `git stash` 操作。否则你就需要找对应的IDE或编辑器（甚至文件系统）的备份了。

通过以下两步可以找回 `git add (stash)` 的文件：

<!-- more -->

第一步：

```
git fsck --lost-found
```

此操作会获取所有 `git add(stash)` 操作生成的 id 。

第二步：

```
git show <ID>
```

显示所有相关的文件到控制台。这样就找回了你所提交的文件。

从此过程中我们学习到：**时常 `add` 本地改动是一个好的习惯**。

> [recover from added file](https://stackoverflow.com/questions/23728769/how-to-recover-files-added-to-git-but-overwritten-by-checkout)
---
