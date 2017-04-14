---
title: 使用Hexo在GitHub上搭建博客
date: 2016-07-29 01:31:48
tags: hexo
---

越来越多的人利用[GitHub Pages](https://pages.github.com/)创建静态网站。这样有很多好处，比如你不再需要维护自己的VPS,也不需要配置Nginx服务器，更不必担心域名何时续费等等。对于一个不需要那么多"动态"业务的博客来说，GitHub提供给我们的已经足够了。

`GitHub`原生支持`Jekll`。`Jekyll`是一个用来生成静态网页的工具,跟`Hexo`类似。本文介绍如何使用`Hexo`在`GitHub`上搭建博客。

<!--more-->

## 安装hexo

```
npm install hexo-cli -g
```

## 创建本地环境

假设我们新建一个叫做blog的本地文件夹，在blog下安装依赖。安装完成后启动`hexo-server`。可以在本地看到一个类Hello world的页面。

```
hexo init blog
cd blog
npm install
hexo server
```

## 创建GitHub仓库

`GitHub`中有一个特殊的仓库，叫`username.github.io`。Github Pages会默认使用这个仓库。如果在这个库里有一个index.html。那么访问[http://username.github.io](http://username.github.io)就会看到这个静态页面。

此步骤可以创建一个空的仓库：`username.github.io`。

## 本地生成静态文件

由上一步我们知道，我们需要一个index.html的静态文件。在本地我们可以由`hexo`生成。

```
hexo generate
```

此过程会将hexo的主题包括css和js，以及你本地的博客文件编译成静态文件。

## 部署到GitHub

安装`hexo-deployer-git"。
```
npm install hexo-deployer-git --save
```
修改_config.yml。将刚才创建的空得仓库信息填写到下面。messges可以删除。

```
deploy:
  type: git
  repo: <repository url>
  branch: [branch]
  message: [message]
```

然后`hexo deploy`。查看username.github.io，会看到和本地server一样的结果。
