---
title: 如何贡献一篇文章
date: 2017-04-05 17:52:26
---

本博客由 `hexo` 生成。[hexo](https://hexo.io/) 类似于 `sphinx` ，相比 `sphinx`，更适合作为团队的技术博客输出。

`hexo` 安装依赖 [npm](https://github.com/npm/npm)。以下假设你已经安装了 `npm`。

首先，将 repo 下载到本地:

```
git clone ssh://yourname@gerrit.x.lan:29418/blog
```

安装依赖：

```
cd blog

npm run preinstall && npm install

```

此时hexo-cli已经全局安装到了你的开发机上。执行：

```
hexo new post 'some-title-you-made'
```

在 `source/_posts/some-title-you-made.md` 编写你的博客文章。

运行 `hexo server` 查看编译结果。That's all。

> 运行 `npm run pub` 将改动发送到 x.lan 服务器。仅用来测试和快速发布用，推荐jekins来做自动发布。
> md 支持 GFM（GitHub flavored Markdown）格式。猛击[这里](https://guides.github.com/features/mastering-markdown/)查看。
> 保持 `hexo-server` 运行，可以实现 `live reload` 的效果。每当文件保存，server会自动生成新文件，并自动刷新页面，实现保存即查看结果的效果。
