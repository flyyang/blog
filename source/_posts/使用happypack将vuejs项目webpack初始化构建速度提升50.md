---
title: 使用happypack将vuejs项目webpack初始化构建速度提升50%
date: 2017-03-09 23:28:08
tags:
  - happypack
  - webpack
  - vuejs
  - 性能优化
---

[Vuejs](https://github.com/vuejs/vue)在经过一两年发力之后，越来越受广大开发者的欢迎。Vue 全家桶带来便捷的同时，也带来一些问题。比如[webpack](https://webpack.js.org/)的初始化构建速度问题。以我手中的某一个中小型项目来说，在开发模式下，初次编译耗时约50秒。在某些场景下，这是不可接受的。

网上查到的webpack的优化方式大概有四五种，本文介绍如何使用[happypack](https://github.com/amireh/happypack)将初始化构建速度提升50%or更多。

<!-- more -->

## 环境

* PC: 2核2G
* Webpack 2.1 
* Vue 2.2.1 以及Vue 2.0 全家桶
* happypack 3.0.3

## 背景

Webpack 打包工具初始化构建时间随着项目的变大，变得越来越慢。一次构建需要一两分钟，在紧急查看一个问题时，特别让人懊恼。从原理上来讲，Webpack本身无非是管理所有依赖的文件，解析他们，将他们pipe起来，交给一个个loader去处理这些文件，最后构建出我们需要的bundle出来。Happypack的出发点就是，让可以并发执行的loader，并发起来，加快构建。

安装happypack:

``` bash
npm install --save-dev happypack

```

使用happypack:

```
var HappyPack = require('happypack');

// 然后在webpack.config.js的plugin处添加

new HappyPack({
  // loaders is the only required parameter:
  loaders: [ 'babel?presets[]=es2015' ],

  // customize as needed, see Configuration below
})

// 在loaders处替换为happypack的loader

loaders: {
  test: /.js$/,
  loaders: [ 'happypack/loader' ],
  include: [
    // ...
  ],
}

```

在配置多个 happypack loader 的情况下，一般可指明 happypack 的id:

``` javascript
  new HappyPack({
    id: 'babel',
    loaders: [ 'babel-loader' ],
  }),
```

```
   {
    test: /\.js$/,
    include: path.join(projectRoot, 'src'),
    exclude: /node_modules/,
    loader: 'happypack/loader?id=babel',
  }, 
```

## 实验

### 实验一

![实验一](/public/images/happypack1.png)

1. happypack + babel。将js文件交由happypack去处理，性能提升9%。

  ``` javascript
      new HappyPack({
        id: 'babel',
        loaders: [ 'babel-loader' ],
      }),
  ```
  ``` javascript
    {
      test: /\.js$/,
      include: path.join(projectRoot, 'src'),
      exclude: /node_modules/,
      loader: 'happypack/loader?id=babel',
    }
  ```
2. 由于看到性能越来越差，怀疑是happypack的cache问题，禁止cache。飙升至22%。

  ``` javascript
      new HappyPack({
        id: 'babel',
        cache: false,
        loaders: [ 'babel-loader' ],
      }),
  ```
3. 打开babel cache。性能变低。怀疑babel cache并不好用，命中率过低。

4. 在2的结果上加url loader，性能低了一点。`url-loader`本身做的事情[很少](https://github.com/amireh/happypack/issues/14)，通过happypack处理，反而变慢。

5. 配合less-loader , css-loader，Extract plugin。相对于2，略有提升。

6. 由于目 happypack 暂时不支持 vue-loader。所以只能在vue-loader内部搞一些[事情](http://vue-loader.vuejs.org/en/configurations/advanced.html)。vue 模板本身是通过三个部分来组成的。那么通过配置vue-loader的options部分，将js部分交由happypack处理，得到了最高分。29.52%。

  ```
    {
      test: /\.vue$/,
      include:  src,
      exclude: /node_modules/,
      use: [
        {
          loader: 'vue-loader',
          options: {
            loaders: {
              css: xxx, 
              less: xxx,
              js: 'happypack/loader?id=babel'
            },
         }
        }
      ]
    },
  ```

### 实验二——cache再测试

![实验二](/public/images/happypack-cache.png)

1. happypack在处理越来越多的文件后，cache命中率提升，效率也有不少提升—— 2.79 %
2. babel的cache似乎仍然会有regression问题。

> 实践中发现，js文件的cache打开会加快速度，css、less等cache应设置为false，否则会有编译不完全的问题，原因未知。

### 实验三—— 调节threads，共享threads pool

直接上结论。不要自己去定义，在我的环境里默认的就是最OK的，线程数多了regression比较严重。

### 实验四 -- eslint

![实验四](/public/images/happypack-eslint.png)

eslint通过happyloader提升惊人。达到50.84%。

## 结论

1. happypack不完全兼容vue-loader。[兼容列表](https://github.com/amireh/happypack/wiki/Loader-Compatibility-List)。但可以通过options来将不同的配置交由 happypack 处理。
2. eslint和babel是最大的提升点。不要盲目全上 happypack。有些本身没几个文件，或者loader本身不太耗时，不需要由 happypack处理。
3. babel的缓存帮助不大，happypack的缓存可开启，但是会导致css编译不全（原因未知），所以css、less等应禁用缓存。
4. 谨慎调节threads，和共享pool。

配置示例：

```
    new HappyPack({
      id: 'babel',
      loaders: [ 'babel-loader?cacheDirectory=true' ],
    }),
    new HappyPack({
      id: 'css',
      cache: false,
      loaders: [ 'css-loader?mportLoaders=1' ],
    }),
    new HappyPack({
      id: 'less',
      cache: false,
      loaders: [ 'less-loader' ],
    }),
    new HappyPack({
      id: 'eslint',
      cache: false,
      loaders: ['eslint-loader'],
    }),

   ...

  {
    test: /\.js$/,
    include: path.join(projectRoot, 'src'),
    exclude: /node_modules/,
    loader: 'happypack/loader?id=babel',
  },

  ...
    
     var cssLoader = ExtractTextPlugin.extract({
      use: [
        {
          loader: 'happypack/loader?id=css',
        },
      ],
      fallback: 'vue-style-loader'
    })

    var lessLoader = ExtractTextPlugin.extract({
      use: [
        'happypack/loader?id=css',
        'happypack/loader?id=less',
      ],
      fallback: 'vue-style-loader'
    })
    
    {
    test: /\.vue$/,
    include: path.join(projectRoot, 'src'),
    exclude: /node_modules/,
    use: [
      {
        loader: 'vue-loader',
        options: {
          loaders: {
            css: cssLoader,
            less: lessLoader,
            js: 'happypack/loader?id=babel'
          },
          postcss: {
            plugins: [
              require('postcss-cssnext')(),
            ],
          }
       }
      }
    ]
  },
```
