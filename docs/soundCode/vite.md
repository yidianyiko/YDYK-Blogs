---
date: 2022-02-21
title: 【源码计划第九期】浅析vite
tags:
  - 源码
describe: 学习原生ESModule
---

> 对vite的第一印象：快，非常快

本着对vite的兴趣，在阅读完[玩具vite](https://github.com/vuejs/vue-dev-server)源码后，对vite的原理有了一个简单的理解：

-   使用浏览器原生ESModule加载项目文件
-   把需要加载的文件，转译成浏览器看得懂的js文件


-   使用缓存机制，提升HMR速度

### 一、什么是原生ESModule？

#### 1.解释

这是一个可以让HTML加载script标签时，使用ESModule的方式直接进行加载，例如：

```
<script type="module">
	import main from './main.js'
</script>
```

#### 2.为什么会快？

浏览器在加载页面遇到原生ES模块时，会通过发送请求的方式导入模块。页面引入模块但未被加载时，这些模块将不会被导入

### 二、这个工具做了什么事情？

#### 1.开发背景

在编写vue项目时，我们会需要经历以下步骤：

-   引入vue，将vue挂载在html的一个节点中

```
import Vue from 'vue'
import App from './test.vue'

new Vue({
  render: h => h(App)
}).$mount('#app')
```

对于原生ESModule来说，import文件时，需要提供文件完整的URL路径，不能进行简写。


-   编写vue文件

```
<template>
  ...
</template>

<script>
export default {
  ...
}
</script>

<style scoped>
...
</style>
```

对于浏览器来说，不支持解析vue文件，这样也就导致无法加载页面

#### 工具需要解决的事情

-   在不改变编写习惯的前提下，改变文件加载路径
-   对vue文件进行转译

### 三、实现过程

在这个小工具中通过使用加载中间件拦截请求的方式进行文件的实时编译，这一文件存放在`../bin/vue-dev-server.js`中

```
const express = require('express')
const { vueMiddleware } = require('../middleware')

const app = express()
const root = process.cwd();

// 最重要的一步
app.use(vueMiddleware())

app.use(express.static(root))

app.listen(3000, () => {
  console.log('server running at http://localhost:3000')
})
```


最重要的中间件文件被放在了`../middleware`中

#### 1.解析主逻辑

在这个中间件中，分别对`.vue`、`.js`、包含`__modules`三种类型的文件进行了不同的处理

```
return async (req, res, next) => {
    if (req.path.endsWith('.vue')) {      
      // ...code 处理vue文件
      }
      
      send(res, out.code, 'application/javascript')
    } else if (req.path.endsWith('.js')) {
       // ...code 处理js文件
      }

      send(res, out, 'application/javascript')
    } else if (req.path.startsWith('/__modules/')) {
      // ...code 处理第三方包
      }

      send(res, out, 'application/javascript')
    } else {
      next()
    }
  }
```


#### 1.1解析vue文件

对于浏览器请求到vue的文件时，会被中间件拦截下来，执行`bundlesSFC()`逻辑。

这部分主要是将.vue中的`template`、`script`、`css`实时编译成浏览器可以被正常加载的代码。这部分主要涉及到vue组件编译的代码，会在后续进行深入研究。

```
if (req.path.endsWith('.vue')) {      
  const key = parseUrl(req).pathname
  let out = await tryCache(key)

  if (!out) {
    // Bundle Single-File Component
    const result = await bundleSFC(req)
    out = result
    cacheData(key, out, result.updateTime)
  }

  send(res, out.code, 'application/javascript')
} 

async function bundleSFC (req) {
  const { filepath, source, updateTime } = await readSource(req)
  const descriptorResult = compiler.compileToDescriptor(filepath, source)
  const assembledResult = vueCompiler.assemble(compiler, filepath, {
    ...descriptorResult,
    script: injectSourceMapToScript(descriptorResult.script),
    styles: injectSourceMapsToStyles(descriptorResult.styles)
  })
  return { ...assembledResult, updateTime }
}
```

#### 1.2 解析js文件

这部分主要是获取js文件地址，并通过`transformModuleImports()`方法，将js中文件的引入方式全部转化为ESModule的引入方式。

```
else if (req.path.endsWith('.js')) {
  const key = parseUrl(req).pathname
  let out = await tryCache(key)

  if (!out) {
    // transform import statements
    const result = await readSource(req)
    out = transformModuleImports(result.source)
    cacheData(key, out, result.updateTime)
  }

  send(res, out, 'application/javascript')
}


// 这里用了第三方库recast，是一个将文件编译成AST树的插件

function transformModuleImports(code) {
  const ast = recast.parse(code)
  recast.types.visit(ast, {
    visitImportDeclaration(path) {
      const source = path.node.source.value
      // 同时会对js中出现引入方式为非完整URL路径、且是npm包的代码进行转义成'__modules/xxx'
      if (!/^./?/.test(source) && isPkg(source)) {
        path.node.source = recast.types.builders.literal(`/__modules/${source}`)
      }
      this.traverse(path)
    }
  })
  return recast.print(ast).code
}
```


#### 1.3 解析转义后的__modules

在处理完需要转义的js包路径后，会对这部分的文件进行加载，主要是通过封装好的`loadPkg()`进行操作

```
else if (req.path.startsWith('/__modules/')) {
  const key = parseUrl(req).pathname
  const pkg = req.path.replace(/^/__modules//, '')

  let out = await tryCache(key, false) // Do not outdate modules
  if (!out) {
    out = (await loadPkg(pkg)).toString()
    cacheData(key, out, false) // Do not outdate modules
  }

  send(res, out, 'application/javascript')
}

// 这部分逻辑中，尤大只支持加载vue
async function loadPkg(pkg) {
  if (pkg === 'vue') {
    const dir = path.dirname(require.resolve('vue'))
    const filepath = path.join(dir, 'vue.esm.browser.js')
    return readFile(filepath)
  }
  else {
    // TODO
    // check if the package has a browser es module that can be used
    // otherwise bundle it with rollup on the fly?
    throw new Error('npm imports support are not ready yet.')
  }
}
```

通过这部分的转义，最初的`import Vue from 'vue'`最后通过转义，加载的文件变成了vue中的`vue.esm.browser.js`文件，达到加载vue的目的

#### 2.实时编译中变得更快

这部分通过使用LRU缓存库，通过对加载后的文件进行缓存，在加载文件时进行文件对比，从而决定是否更新文件

```
// 初始化LRU缓存
const LRU = require('lru-cache')
	cache = new LRU({
  max: 500,
  length: function (n, key) { return n * 2 + key.length }
})

// 判断文件是否需要缓存
async function tryCache (key, checkUpdateTime = true) {
  // 首先检查缓存中是否有该文件
  const data = cache.get(key)

  if (checkUpdateTime) {
    // 是否有该缓存时间
    const cacheUpdateTime = time[key]
    // 创建一个文件更新时间并进行对比
    const fileUpdateTime = (await stat(path.resolve(root, key.replace(/^//, '')))).mtime.getTime()
    if (cacheUpdateTime < fileUpdateTime) return null
  }

  return data
}

// 在对文件进行处理完后，都会将文件加入缓存中
function cacheData (key, data, updateTime) {
  	// 调用包中的方法，判断文件是否有变化
    const old = cache.peek(key)
    if (old != data) 
      cache.set(key, data)
      if (updateTime) time[key] = updateTime
      return true
    } else return false
  }
```

### 四、总结

通过对这个源码的阅读，首先对vite的“急速”体验，有了基础的理解。当然vite的代码会比当前包更复杂，处理更多的情况。

在阅读完后，后续还会对vue和vite进行更深入的了解，包括：

-   LRU缓存
-   SFC编译解析


-   JS文件解析成AST树
