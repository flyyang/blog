---
title: v8-compile-cache 源码分析
date: 2019-01-16 17:37:38
tags:
---
## 背景知识

v8 是一个 JIT(Just in time) 编译器。与传统的解释器一行一行执行不同的是，JIT 会在执行脚本前，对源码先解析（parsing）、再编译（compiling)，速度相比前者提升了不少。但解析和编译仍然消耗时间。能否将中间结果缓存起来呢？

所以 v8 在 4.2（node > 5.7.0） 时，就支持了 code caching 的功能。减少二次执行的构建时间，加快脚本的整体执行速度。

### 缓存中间结果，持久化到硬盘

要使用中间文件，需要先生成中间文件。v8 提供了以下几个方法：

* `v8::ScriptCompiler::kProduceCodeCache` 是否生成 code cache
* `v8::ScriptCompiler::Source::GetCachedData` 获取生成的 code cache
* `v8::ScriptCompiler::kConsumeCodeCache` 消费生成的 cache

v8 是 c++ 代码，在 node 中有对应的 [vm.Script](https://nodejs.org/api/vm.html#vm_class_vm_script) 与上面的功能相对应。我们需要对生成的 code cache 设计一套持久化机制，来方便二次消费。

### require hook

以上解决了code cache 生成和消费的问题，但是源码从哪里来呢？

我们知道 node.js  通过 require 来连接代码，那么对所有 require 的 module 进行编译并缓存结果是不是就可以了？当然。

[v8-compile-cache](https://github.com/zertosh/v8-compile-cache) 便是这样一个仓库- 对编译中间过程持久化，加快整体执行时间。

<!-- more -->

## 源码分析


`v8-compile-cache` 的使用很简单：

```
require('v8-compile-cache')
```
没有赋值，也没有实例化。我们去源码中看看这段引用究竟执行了什么。


### 从入口出发

一个 npm 包的入口文件，在 `package.json` 的 `main` 字段：v8-compile-cache.js。

require('v8-compile-cache') 就相当于执行了这段脚本。

抛开最上层的定义，在 module.exports 前发现这样一段代码：


```
if (!process.env.DISABLE_V8_COMPILE_CACHE && supportsCachedData()) {
  const cacheDir = getCacheDir();
  const prefix = getParentName();
  const blobStore = new FileSystemBlobStore(cacheDir, prefix);

  const nativeCompileCache = new NativeCompileCache();
  nativeCompileCache.setCacheStore(blobStore);
  nativeCompileCache.install();

  process.once('exit', code => {
    if (blobStore.isDirty()) {
      blobStore.save();
    }
    nativeCompileCache.uninstall();
  });
}
```

首先检测用户是否通过环境变量 `DISABLE_V8_COMPILE_CACHE` 禁用了此功能。由于没有实例化的过程。用户可以通过配置此变量来决定是否开启 code caching。

接下来我们看一下 `supportsCachedData` 方法：

```
function supportsCachedData() {
  const script = new vm.Script('""', {produceCachedData: true});
  // chakracore, as of v1.7.1.0, returns `false`.
  return script.cachedDataProduced === true;
}
```

这里其实是一段对 chakracore 的兼容。这里比较巧妙的是，通过一段空的代码，来验证是否支持 code caching。

### 解决持久化的 blobStore

在做持久化（其实就是写硬盘）之前，我们需要决定写在哪里。

```
  const cacheDir = getCacheDir();
```

```
function getCacheDir() {
  // Avoid cache ownership issues on POSIX systems.
  const dirname = typeof process.getuid === 'function'
    ? 'v8-compile-cache-' + process.getuid()
    : 'v8-compile-cache';
  const version = typeof process.versions.v8 === 'string'
    ? process.versions.v8
    : typeof process.versions.chakracore === 'string'
      ? 'chakracore-' + process.versions.chakracore
      : 'node-' + process.version;
  const cacheDir = path.join(os.tmpdir(), dirname, version);
  return cacheDir;
}
```
我们忽略 chakracore 兼容的情况下，getCacheDir 返回一个类似于 `/tmp/v8-compile-cache-0/6.2.414.54` 地址。其中后一段数字是 v8 的版本，防止不同版本的中间文件不一致导致运行问题。

```
mac os 多用户的 tmp 地址为 ：`/var/folders/x8/pxgnmcp53gjf3llf69jqb4gh0000gq/T/`
```

默认中间文件会生成到这个地址。

有了这个写入地址后，我们需要一个类来管理写入：

```
  const prefix = getParentName();
  const blobStore = new FileSystemBlobStore(cacheDir, prefix);
```

我们先来看一下 prefix 是什么。

```
function getParentName() {
  // `module.parent.filename` is undefined or null when:
  //    * node -e 'require("v8-compile-cache")'
  //    * node -r 'v8-compile-cache'
  //    * Or, requiring from the REPL.
  const parentName = module.parent && typeof module.parent.filename === 'string'
    ? module.parent.filename
    : process.cwd();
  return parentName;
}
```
module.parent.filename 返回调用者地址绝对路径或者当前路径。作为 prefix 传递给 FileSystemBlobStore。

```
class FileSystemBlobStore {
  constructor(directory, prefix) {
    const name = prefix ? slashEscape(prefix + '.') : '';
    this._blobFilename = path.join(directory, name + 'BLOB');
    this._mapFilename = path.join(directory, name + 'MAP');
    this._lockFilename = path.join(directory, name + 'LOCK');
    this._directory = directory;
    this._load();
  }

  _load() {
    try {
      this._storedBlob = fs.readFileSync(this._blobFilename);
      this._storedMap = JSON.parse(fs.readFileSync(this._mapFilename));
    } catch (e) {
      this._storedBlob = Buffer.alloc(0);
      this._storedMap = {};
    }
    this._dirty = false;
    this._memoryBlobs = {};
    this._invalidationKeys = {};
  }
}


function slashEscape(str) {
  const ESCAPE_LOOKUP = {
    '\\': 'zB',
    ':': 'zC',
    '/': 'zS',
    '\x00': 'z0',
    'z': 'zZ',
  };
  return str.replace(/[\\:\/\x00z]/g, match => (ESCAPE_LOOKUP[match]));
}
```
在 `new FileSystemBlobStore(cacheDir, prefix)` 时，会先把 prefix 通过 slashEscape 转换成一个标准的名字，作为文件名。this._blobFilename 存储生成的二进制 code caching。this._mapFilename 存储脚本到二进制文件的映射。然后 _load 到这些中间文件到内存中。

如果本地没有中间文件， this._storedBlob 则是一个 0 长度的 buffer, this._storedMap 则是一个空对象。

### 关联 require hook 和 blobStore 的 NativeCompileCache

再上一个过程中，我们建立了一个 blobStore。它会把已经编译（如果有）的中间文件缓存到内存中。并通过 map 来定位文件和 blob 二进制的映射。接下来我们看

```
  const nativeCompileCache = new NativeCompileCache();
  nativeCompileCache.setCacheStore(blobStore);
  nativeCompileCache.install();
```

```
class NativeCompileCache {
  constructor() {
    this._cacheStore = null;
    this._previousModuleCompile = null;
  }

  setCacheStore(cacheStore) {
    this._cacheStore = cacheStore;
  }

  install() {
    const self = this;
    this._previousModuleCompile = Module.prototype._compile;
    Module.prototype._compile = function(content, filename) {
      const mod = this;
      function require(id) {
        return mod.require(id);
      }
      require.resolve = function(request, options) {
        return Module._resolveFilename(request, mod, false, options);
      };
      require.main = process.mainModule;

      // Enable support to add extra extension types
      require.extensions = Module._extensions;
      require.cache = Module._cache;

      const dirname = path.dirname(filename);

      const compiledWrapper = self._moduleCompile(filename, content);

      // We skip the debugger setup because by the time we run, node has already
      // done that itself.

      const args = [mod.exports, require, mod, filename, dirname, process, global];
      return compiledWrapper.apply(mod.exports, args);
    };
  }
}
```

我们先看 install 部分。install 重写了原生模块的 _compile 方法。并提供了一个新的 require 方法。为什么重写 require 呢？因为原生的模块内部也维护了一个 require 方法，既然重写了 _compile ，自然要提供一个 require 给后续使用。

```
const compiledWrapper = self._moduleCompile(filename, content);
```

重写 require 部分的代码比较容易理解。_compile 内部又调用了类的 _moduleCompile 方法：

```
 _moduleCompile(filename, content) {
    // https://github.com/nodejs/node/blob/v7.5.0/lib/module.js#L511
    // Remove shebang
    var contLen = content.length;
    if (contLen >= 2) {
      if (content.charCodeAt(0) === 35/*#*/ &&
          content.charCodeAt(1) === 33/*!*/) {
        if (contLen === 2) {
          // Exact match
          content = '';
        } else {
          // Find end of shebang line and slice it off
          var i = 2;
          for (; i < contLen; ++i) {
            var code = content.charCodeAt(i);
            if (code === 10/*\n*/ || code === 13/*\r*/) break;
          }
          if (i === contLen) {
            content = '';
          } else {
            // Note that this actually includes the newline character(s) in the
            // new output. This duplicates the behavior of the regular
            // expression that was previously used to replace the shebang line
            content = content.slice(i);
          }
        }
      }
    }

    // create wrapper function
    var wrapper = Module.wrap(content);

    var invalidationKey = crypto
      .createHash('sha1')
      .update(content, 'utf8')
      .digest('hex');

    var buffer = this._cacheStore.get(filename, invalidationKey);

    var script = new vm.Script(wrapper, {
      filename: filename,
      lineOffset: 0,
      displayErrors: true,
      cachedData: buffer,
      produceCachedData: true,
    });

    if (script.cachedDataProduced) {
      this._cacheStore.set(filename, invalidationKey, script.cachedData);
    } else if (script.cachedDataRejected) {
      this._cacheStore.delete(filename);
    }

    var compiledWrapper = script.runInThisContext({
      filename: filename,
      lineOffset: 0,
      columnOffset: 0,
      displayErrors: true,
    });

    return compiledWrapper;
  }
```

忽略 shebang 部分代码。 先创建一个 [wrapper function](https://nodejs.org/api/modules.html#modules_the_module_wrapper)。

```
(function(exports, require, module, __filename, __dirname) {
// Module code actually lives in here
});
```

给 vm.Script 去执行。

然后对文件内容生成散列。invalidationKey 如： `cc0579eda025ac6d18f3914d42ba60abe2b1a8e`。

接着从内存中取已经生成 code cache。this._cacheStore.get(filename, invalidationKey) 。

```
  var script = new vm.Script(wrapper, {
    filename: filename,
    lineOffset: 0,
    displayErrors: true,
    cachedData: buffer,
    produceCachedData: true,
  });
```

vm.Script 第一个参数是 code string。 也就是我们包装过的代码 warapper。第二个参数是 options。其中
cachedData 是编译好的 code cache， 如果没有提供 cacheData 的话，produceCachedData 指示是否输出 code cache。

vm.Script 并不会运行脚本，只负责编译。


```
  if (script.cachedDataProduced) {
    this._cacheStore.set(filename, invalidationKey, script.cachedData);
  } else if (script.cachedDataRejected) {
    this._cacheStore.delete(filename);
  }
```

如果生成了 code cache ，则写入到内存缓存中, 有问题则删掉缓存。

```
  var compiledWrapper = script.runInThisContext({
    filename: filename,
    lineOffset: 0,
    columnOffset: 0,
    displayErrors: true,
  });

```

接下来运行其中的代码。返回一个 compiledWraper。最后包装到 module.exports:

```
  const compiledWrapper = self._moduleCompile(filename, content);

  // We skip the debugger setup because by the time we run, node has already
  // done that itself.

  const args = [mod.exports, require, mod, filename, dirname, process, global];
  return compiledWrapper.apply(mod.exports, args);

```
compiledWrapper 返回结果其实就是一个封装好的函数, 如:

```
function(exports, require, module, __filename, __dirname) {
    const {
        b
    } = require('./b.js')
    module.exports = {
        a() {
            console.log('a')
        },
        b,
    }

}
```

在执行 compiledWrapper.apply(mod.exports, args)时， 对 mod 重新赋值，应用了新的 require 生成了新的 module.exports。

最后 return 的结果就是 module.exports 的内容。

而 Module.prototype._load 会将 Module.prototype._compile 返回的结果给用户， 两者的调用机制，可以参考[这篇文章](http://fredkschott.com/post/2014/06/require-and-the-module-system/)。

这样就完成了hook require，并取缓存中 code cache 的流程。


### 持久化的一些细节

上面只是简略的过了一下持久化的过程。下面进行详细分析。

我们看一下持久化是如何存储的, 在 code cache 生成的过程中，如果满足条件， 先写入到 _cacheStroe 内存中：

```
  if (script.cachedDataProduced) {
    this._cacheStore.set(filename, invalidationKey, script.cachedData);
  } else if (script.cachedDataRejected) {
    this._cacheStore.delete(filename);
  }
```

```
  set(key, invalidationKey, buffer) {
    this._invalidationKeys[key] = invalidationKey;
    this._memoryBlobs[key] = buffer;
    this._dirty = true;
  }
```

cacheStore 会将 code cache 写入到 _memoryBlobs 中。并标记 _dirty 为 true, 表示内存中有更新。

到这个时候，所有的变更都在内存中，我们需要写入到硬盘。在什么时机呢？

```
  process.once('exit', code => {
    if (blobStore.isDirty()) {
      blobStore.save();
    }
    nativeCompileCache.uninstall();
  });
```

当进程退出时，如果内存中有更新，就写入到文件中。

```

  save() {
    const dump = this._getDump();
    const blobToStore = Buffer.concat(dump[0]);
    const mapToStore = JSON.stringify(dump[1]);

    try {
      mkdirpSync(this._directory);
      fs.writeFileSync(this._lockFilename, 'LOCK', {flag: 'wx'});
    } catch (error) {
      // Swallow the exception if we fail to acquire the lock.
      return false;
    }

    try {
      fs.writeFileSync(this._blobFilename, blobToStore);
      fs.writeFileSync(this._mapFilename, mapToStore);
    } catch (error) {
      throw error;
    } finally {
      fs.unlinkSync(this._lockFilename);
    }

    return true;
  }

   _getDump() {
    const buffers = [];
    const newMap = {};
    let offset = 0;

    function push(key, invalidationKey, buffer) {
      buffers.push(buffer);
      newMap[key] = [invalidationKey, offset, offset + buffer.length];
      offset += buffer.length;
    }

    for (const key of Object.keys(this._memoryBlobs)) {
      const buffer = this._memoryBlobs[key];
      const invalidationKey = this._invalidationKeys[key];
      push(key, invalidationKey, buffer);
    }

    for (const key of Object.keys(this._storedMap)) {
      if (hasOwnProperty.call(newMap, key)) continue;
      const mapping = this._storedMap[key];
      const buffer = this._storedBlob.slice(mapping[1], mapping[2]);
      push(key, mapping[0], buffer);
    }

    return [buffers, newMap];

```

在 save 方法中，先调用了 _getDump 方法，内部细节不在赘述。最终写入到机器的 MAP 文件类似如下:

```
{"/Users/flyyang/devspace/test-v8-file-size/b.js":["ba9069dd2de36ca9d7a51fb6f6d2d00c8d4b11a8",0,1064],"/Users/fl
yyang/devspace/test-v8-file-size/a.js":["cc0579eda025ac6d18f3914d42ba60abe2b1a8e7",1064,2168]}
```
以文件名为 key, 对应 invalidationKey, 在 buffer 文件中的起始位，和结束位。

而 buffer 文件，则存储的是所有 code cache 的二进制文件。

以上。

## 总结

* 工具类应用使用此包会加速构建速度。
* 开发，甚至是了解需要对 node 的运行，v8 周边有深入了解。

## issue

有问题，来 [github](https://github.com/flyyang/blog/issues/13) 一起讨论。

##  参考

* [https://nodejs.org/api/vm.html](https://nodejs.org/api/vm.html)
* [https://v8.dev/blog/code-caching](https://v8.dev/blog/code-caching)