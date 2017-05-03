---
title: 'GNU Readline, 让命令行编辑倍速提升'
date: 2017-05-03 16:02:10
tags: efficiency
---

你有没有遇到过这种场景？写了很长一串命令，运行时发现第一个字母写错了。然后一个
字符一个字符的删除，改完再重新输入一遍。

没有必要！！！比如上面的问题，你仅仅需要`Control + a` 回到开头位置修正你的问题即可。这种在行内编辑文本的功能通常是由 GNU Readline提供的。

<!-- more -->

## GNU Readline是什么？

GNU 的一个库，提供行内编辑，历史管理等功能。bash, mysql, zsh, python, node等的
shell中都有类似的功能。

## 常用的 Shotcuts

虽然 GNU Readline 作为一个类库存在，但我们并不关注如何利用其接口实现功能。此处
只关注如何使用。感谢 Readline，使我们在不同的 shell 中有一致的体验。



### Moving

| Command | Description |
| --- | --- |
| Ctrl-a | 移至行首 |
| Ctrl-e | 移至行尾 |
| Ctrl-f | 向前移动一个字符 |
| Ctrl-b | 向后移动一个字符 |
| Alt-f | 向前移动一个单词。 |
| Alt-b | 向后移动一个单词。 |
| Ctrl-l | 清屏 |


### Editing

| Command | Description |
| --- | --- |
| Ctrl-d | 删除光标下的字符 |
| Ctrl-t | 交换字符位置 |
| Alt-t | 交换单词位置 |
| Alt-u | 将光标开始后的单词大写。 |
| Alt-l | 与上相反 |
| Alt-c | 大写当前单词（从光标处开始）。 |


### Cutting and Pasting

| Command | Description |
| --- | --- |
| Ctrl-k | 剪切到行尾 |
| Ctrl-u | 剪切到行首 |
| Alt-d | 剪切到词尾 |
| Ctrl-w | 剪切到词首 |
| Alt-\ | 删除光标附近空白 |
| Ctrl-y | 粘贴 |
| Alt-y | 交换粘贴 |


### History

| Command | Description |
| --- | --- |
| Ctl-p | 前一个命令 |
| Ctl-n | 下一个命令 |
| Alt-< | 第一个命令 |
| Alt-> | 最后一个命令 |
| Ctrl-r | 向后搜索 |
| Ctrl-s | 向前搜索 |
| Alt-p | 按字符串向前搜索 |
| Alt-n | 按字符串向后搜索 |


### Misc

| Command | Description |
| --- | --- |
| Alt-# | 注释当前行，并跳到下一行 |
| Ctrl-] | 向后搜索字符 |
| Ctrl-Alt-] | 向前搜索字符 |
