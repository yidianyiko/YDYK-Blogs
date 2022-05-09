---
date: 2021-09-14
title: 【源码第一期】Vue-DevTools部分源码阅读
tags:
  - 源码
describe: 浅析 Vue-DevTools--打开组件功能
---
> 很感谢若川大佬组织的源码阅读小组活动
>
> 每天下班后逼自己学习学习
>
> 以下为若川原文：https://juejin.cn/post/6959348263547830280#heading-2

### 什么是Vue-DevTools？
作为一个Vue开发者（不是），自然少不了Chrome中的Vue调试插件。
Vue-DevTools是一个可以在Chrome中进行Vue项目调试的工具，可以帮助开发者在使用Vue开发时，更清楚的了解目前页面中的组件、数据情况。
目前该插件有两个版本，支持Vue3的Beta版本，和支持Vue2的版本。
![界面](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e8264803da804e4a9b0e43833f393cff~tplv-k3u1fbpfcp-watermark.image)
### 要了解什么？
这次主要了解在新版本DevTools中支持了一个新特性：在选择对应的组件后，点击``open-in-editor``的按钮后，即可在编译器中打开对应的组件。

### 实现原理：
主要通过launch-editor-middleware和launch-editor两个库实现了该功能，这两个库又通过调用node的``process``、``child_process``能力，创建一个node的子进程调起编译器打开选中的组件
![组件](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f4a5274aa01d4bfb89259ee898e752c3~tplv-k3u1fbpfcp-watermark.awebp)

### 阅读前准备：
- 在Chrome中准备支持Vue3的最新版本插件（目前最新版本号6.0.0 beta 15）
- ``vue create`` 创建一个vue-cli3项目
- 准备一个编译器

### 开始调试：
> Open in editor在Vue3中是一个开箱即用的功能
>
> 具体如何配置使用：Open component in editor
#### 1.寻找入口，进行调试
##### 1.1寻找入口
根据上述文档的项目引入配置，需要在编译器中搜索``'/__open-in-editor'``，即可在``node_modules`` 中定位到该方法，此时在此处打个点~
![第一个点](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1111a85d7f584419a4c25f3208001a51~tplv-k3u1fbpfcp-watermark.awebp)

再继续进入``launchEditorMiddleware`` 发现这个中间件会调用``launch-editor``进行后续的打开编译器操作，此时可以在调用``launch``函数这行打上一个点~
![第二个点](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a138154cc357433b8a5934e72a036540~tplv-k3u1fbpfcp-watermark.awebp)

##### 1.2启动调试
以Vscode为例：
进入项目的``package.json``，可以看到在``script``属性上有一个“调试”或“debug”的按钮，点击后选择``serve``即可进入调试模式
![启动调试](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d227e2411e854e939247ed183c9169b1~tplv-k3u1fbpfcp-watermark.awebp)

> 在这里我踩了一个小坑（也是因为自己不够谨慎）
> 
> 在npm i完成之后，先npm run serve在8080端口启动了项目，再点击调试
> 
> 这会造成编译器再开启一个进程在8081端口启动项目，这也许会让你在后续调试时发现无法进入断点处
> 
> 此时需要注意调试启动的项目端口是否与浏览器端口一致
接下来就进入到阅读源码部分~
#### 开始阅读：
##### 1.launchEditorMiddleware部分
在项目开始编译时，就会自动进入该部分代码。
> 个人理解在这部分代码中主要做了两件事：
> 
> 1.函数重载，满足不同开发传参需求
> 
> 2.通过node.js获取当前进程所在的位置，为后续打开编译器做准备
```javascript
// serve.js
app.use('/__open-in-editor', launchEditorMiddleware(() => console.log(
  `To specify an editor, specify the EDITOR env variable or ` +
  `add "editor" field to your Vue project config.\n`
)))

//launch-editor-middleware/index.js
module.exports = (specifiedEditor, srcRoot, onErrorCallback) => {
  //这里对传入的第一个参数做一个判断，如果该参数为函数，则将这个参数与错误回调函数的值进行对调
  if (typeof specifiedEditor === 'function') {
      onErrorCallback = specifiedEditor
      specifiedEditor = undefined
    }
	//同样对传入的第二个参数也是做同样的判断
  if (typeof srcRoot === 'function') {
    onErrorCallback = srcRoot
    srcRoot = undefined
  }
	//第二个参数如果传入的是目录，则直接用
  //如果不是则调用node.js中process的能力，获取当前进程所在的位置
  srcRoot = srcRoot || process.cwd()
  return function launchEditorMiddleware (req, res, next) {
    //返回一个中间件
  }
}

```
#### 2 launch-editor部分
##### 2.1执行前路径的判断
F12打开Vue-DevTools调试面板，选择一个组件，点击open-in-editor即可进入断点处
此时，如果切换到Chrome的Network栏时，会发现此时浏览器发送了一个请求：
![请求](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a3df5da4e5524881b03e08bcab0eb11e~tplv-k3u1fbpfcp-watermark.awebp)

结合编译前的``app.use('/__open-in-editor', launchEditorMiddleware(...)``不难知道这是一个中间件的写法，当浏览器发送请求时，就会进入到接下来的代码逻辑中
```javascript
module.exports = (specifiedEditor, srcRoot, onErrorCallback) => {
	// ....省略
  return function launchEditorMiddleware (req, res, next) {
    // 首先会读取路径中的file参数
    const { file } = url.parse(req.url, true).query || {}
    if (!file) {
      res.statusCode = 500
      res.end(`launch-editor-middleware: required query param "file" is missing.`)
    } else {
      // 如果存在该路径，则会执行launch-editor逻辑
      launch(path.resolve(srcRoot, file), specifiedEditor, onErrorCallback)
      res.end()
    }
  }
}
```
##### 2.2执行中最重要的一部分
进入到```launchEditor```函数后，也是该功能最重要的一部分
```javascript
function launchEditor (file, specifiedEditor, onErrorCallback) {
  //2.2.1通过正则匹配的方式读取文件路径、行号、列号的信息并进行返回
  const parsed = parseFile(file)
  let { fileName } = parsed
  const { lineNumber, columnNumber } = parsed
	// 2.2.2调用node.js的方法，以同步的方式检测该路径是否存在，不存在就return结束
  if (!fs.existsSync(fileName)) {
    return
  }
	// 这里同样是一个函数重载的方法
  if (typeof specifiedEditor === 'function') {
    onErrorCallback = specifiedEditor
    specifiedEditor = undefined
  }
	// 2.2.3这里跟错误回调调用了一个方法，比较有意思
  onErrorCallback = wrapErrorCallback(onErrorCallback)

}
```
2.2.3部分，采用了装饰器模式（感谢同组的纪年小姐姐的总结），原理是将要执行的逻辑包裹起来，先执行其他的需要处理的代码，再执行``onErrorCallback``的逻辑。
继续阅读函数~
```javascript
function wrapErrorCallback (cb) {
  return (fileName, errorMessage) => {
    console.log()
    //这里先做了一个错误的输出，同时调用node.js中path的方法，提取出用"/"隔开的path最后一部分内容共
    //并且用了一个chalk库，可以改变控制台输出内容的颜色
    console.log(
      chalk.red('Could not open ' + path.basename(fileName) + ' in the editor.')
    )
    // 此时如果有错误信息时，才会输出错误信息的提示
    if (errorMessage) {
      if (errorMessage[errorMessage.length - 1] !== '.') {
        errorMessage += '.'
      }
      console.log(
        chalk.red('The editor process exited with an error: ' + errorMessage)
      )
    }
    console.log()
    if (cb) cb(fileName, errorMessage)
  }
}
```
若此时在这部分没有报错，则会继续进行接下来的流程。
2.2.4 此时会进入一个很“刺激”的猜测环节
```javascript
//launch-editor/index.js
function launchEditor (file, specifiedEditor, onErrorCallback) {
  ...
	// 此时代码进入猜测函数
  const [editor, ...args] = guessEditor(specifiedEditor)
}

// launch-editor/guess.js
module.exports = function guessEditor (specifiedEditor) {
  // 第一步：判断有没有传入对应的shell命令
  if (specifiedEditor) {
    // 如果传入，利用shell-quote库解析shell命令
    return shellQuote.parse(specifiedEditor)
  }
  // We can find out which editor is currently running by:
  // `ps x` on macOS and Linux
  // `Get-Process` on Windows
  
  // 第二步：猜测环节
  // 上面的三行注释也说明了可以判断当前是在哪个系统环境下运行，从而决定用何种方式启动编译器
  try {
    // 通过node.js中process中标识运行node.js进程的操作系统的方法获取当前的操作系统
    // 因为我的系统是MacOs，直接进入第一个猜测中
    if (process.platform === 'darwin') {
      // 此时调用了同步创建子进程的方法,这里会获取到目前的所有进程
      const output = childProcess.execSync('ps x').toString()
      // COMMON_EDITORS_OSX为一个map表，里面维护着MacOs下支持的编译器，以及对应的字段
      // 通过遍历的方式与当前系统中存在的编译器进行匹配
      const processNames = Object.keys(COMMON_EDITORS_OSX)
      for (let i = 0; i < processNames.length; i++) {
        const processName = processNames[i]
        if (output.indexOf(processName) !== -1) {
          return [COMMON_EDITORS_OSX[processName]]
        }
      }
    }
  // ... 不同平台的我就省略了，原理类似
  // 最后还有一个兜底的方案
  // Last resort, use old skool env vars
  if (process.env.VISUAL) {
    return [process.env.VISUAL]
  } else if (process.env.EDITOR) {
    return [process.env.EDITOR]
  }
  return [null]
}
```
2.2.5 猜测完之后的操作
```javascript
function launchEditor (file, specifiedEditor, onErrorCallback) {
 	// ...
  const [editor, ...args] = guessEditor(specifiedEditor)
  // 如果没有找到，就会报错
  if (!editor) {
    onErrorCallback(fileName, null)
    return
  }
	// 核心部分，根据不同的系统状态，打开调起不同的工具打开编译器
  // childProcess.spawn为异步衍生子进程，并且不会阻塞node.js的事件循环
  if (process.platform === 'win32') {
    // On Windows, launch the editor in a shell because spawn can only
    // launch .exe files.
    _childProcess = childProcess.spawn(
      'cmd.exe',
      ['/C', editor].concat(args),
      { stdio: 'inherit' }
    )
  } else {
    // 因为是MacOs，因此调用Vscode，打开args地址（项目地址），并且子进程将使用父进程的标准输入输出。
    // 这块Node文档参考
    // http://nodejs.cn/api/child_process.html#child_process_child_process_spawn_command_args_options
    // 到这里，对应的组件文件就已经在编译器中被打开了
    _childProcess = childProcess.spawn(editor, args, { stdio: 'inherit' })
  }
  	// 这里是对子进程结束后触发做监听，检测进程退出是否存在异常
  _childProcess.on('exit', function (errorCode) {
    _childProcess = null

    if (errorCode) {
      onErrorCallback(fileName, '(code ' + errorCode + ')')
    }
  })

  _childProcess.on('error', function (error) {
    onErrorCallback(fileName, error.message)
  })
}
```
### 总结
首先小小的表扬一下自己，终于克服了不会读不敢读源码的问题
🎉🎉🎉🎉🎉🎉🎉
以前觉得源码都很难懂，框架也很难了解真正的原理。但是通过这次活动，小小的明白了一个工具中一个小模块的实现方法，很有意思。
也很感谢若川大佬组织这次活动，辛苦了。

这次阅读的过程同时也发现了原来Node可以做很多事情，这也是之前没有了解过的知识点。

### 相关文档和资料：
Vue-DevTools：https://github.com/vuejs/devtools#open-component-in-editor

尤大版本launch-editor：https://github.com/yyx990803/launch-editor

Umijs/launch-editor：https://github.com/umijs/launch-editor


<Comment/>