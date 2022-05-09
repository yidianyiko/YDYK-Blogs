---
date: 2021-12-01
title: 【源码计划第八期】create-vue
tags:
  - 源码
describe: 学习cli脚本编写
---

> 继续跟上川哥举办的阅读源码活动~
> 
> 掘金原文：<https://juejin.cn/post/7018344866811740173#heading-4>



### 1. 阅读前准备



这是一个vite+vue3的脚手架，目前还属于比较初版的状态，README也不是很全面，大致浏览源码后，主要有以下几点可以好好学习下：

-   npm init
-   交互式初始化

<!---->

-   根据选择配置生成不同模板
-   模板生成原理



项目地址： <https://github.com/vuejs/create-vue>

这是一个我之前完全没有接触过的项目，文章中间会拓展很多`create-Vue`之外的内容，最终希望自己可以通过阅读源码，产出cli工具，方便在工作中项目开发

### 2. 使用它

使用起来非常简单，只需要 npm init 即可

```
npm init vue@next
```

接着输入项目名、确认是否需要额外的配置项，即可生成一个完整的项目

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f538e0ebb2f745f99bd6f4c1383f3dd2~tplv-k3u1fbpfcp-zoom-1.image)

最终只需要`cd vue-project`、`npm install`、`npm run dev`即可非常急速的打开这个项目

### 3. 阅读源码

首先克隆项目、安装依赖

```
git clone https://github.com/vuejs/create-vue.git
npm install
```

进入package.json看看可执行的脚本

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/75bebe4d2d9b431a9111abfd77fa19fa~tplv-k3u1fbpfcp-zoom-1.image)

看到几个熟悉的脚本：

-   build:将脚手架主文件index.js打包输出成outfile.cjs
-   test:执行测试用例

先从执行test调试起！

#### 3.1 snapshot

执行`npm run test`时，会先执行`npm run pretest` （第三期的知识点：npm钩子函数）

看看snapshot中都做了什么事情~

进入snapshot发现这里做了两件事：

-   调用node的子进程方法中同步进程函数，根据index.js文件中的规则，在playground文件夹下创建不同的模板，方便调用test.js进行测试
-   根据五种拓展属性，进行二进制的排列组合，最终会生成31种+default一共32种组合

这里主要看一下生成模板的方法：

```
// 获取基本路径
const __dirname = path
  .dirname(new URL(import.meta.url).pathname)
  .substring(process.platform === 'win32' ? 1 : 0)


// 这里的bin是执行打包后的index.js文件生成项目模板，方便调试就改成了index.js
// const bin = path.resolve(__dirname, './outfile.cjs')
const bin = path.resolve(__dirname, './index.js')
// 目标文件夹
const playgroundDir = path.resolve(__dirname, './playground/')

function createProjectWithFeatureFlags(flags) {
  // flags会以数组的形式传入，如果出现多个参数时，用-分割
  const projectName = flags.join('-')
  console.log(`Creating project ${projectName}`)
  // 调用子进程
  // 这部分会生成一个类似于 node ./index.js --typescript --force的命令
  // 主要就是调用index.js中的init方法生成模板
  const { status } = spawnSync(
    'node',
    [bin, projectName, ...flags.map((flag) => `--${flag}`), '--force'],
    {
      cwd: playgroundDir,
      stdio: ['pipe', 'pipe', 'inherit']
    }
  )

  if (status !== 0) {
    process.exit(status)
  }
}
```

这里主要的灵魂就是调起子进程，执行文件生成项目模板

接着看看具体是怎么生成模板的

#### 3.2 index.js

这部分主要做了以下几件事：

-   支持feature Flags直接生成模板
-   调用`prompts`进行交互式配置

<!---->

-   根据配置调用render()渲染模板
-   根据配置生成不同README

##### 3.2.1 交互式

首先在script中添加一行`"dev":"node index.js"`运行调试，开始我们的debug

最先进入的会是一段交互式配置，这部分代码将做省略，保留一些调用了工具方法的配置

```
let result = {}

  try {
    result = await prompts(
      [
        {
          name: 'shouldOverwrite',
          // 这里判断了一次改名称是否可写入
          type: () => (canSafelyOverwrite(targetDir) || forceOverwrite ? null : 'confirm'),
          message: () => {
            const dirForPrompt =
              targetDir === '.' ? 'Current directory' : `Target directory "${targetDir}"`

            return `${dirForPrompt} is not empty. Remove existing files and continue?`
          }
        },
        {
          name: 'overwriteChecker',
          type: (prev, values = {}) => {
            if (values.shouldOverwrite === false) {
              throw new Error(red('✖') + ' Operation cancelled')
            }
            return null
          }
        },
        {
          name: 'packageName',
          // 这里判断了一次当前包名称是否合法
          type: () => (isValidPackageName(targetDir) ? null : 'text'),
          message: 'Package name:',
          // 这里调用了转换包名工具
          initial: () => toValidPackageName(targetDir),
          validate: (dir) => isValidPackageName(dir) || 'Invalid package.json name'
        },
      ],
    )
  } catch (cancelled) {
    console.log(cancelled.message)
    process.exit(1)
  }
```

这三个工具函数也比较简单，通过正则或node方法，进行简单的判断

```
// 使用正则判断当前包名是否合法，没有采用validate-npm-package-name（第七期检测包名是否合法的工具），自己实现了检测功能
function isValidPackageName(projectName) {
  return /^(?:@[a-z0-9-*~][a-z0-9-*._~]*/)?[a-z0-9-~][a-z0-9-._~]*$/.test(projectName)
}

// 对传入的包名称进行转写
function toValidPackageName(projectName) {
  return projectName
    .trim()
    .toLowerCase()
    .replace(/\s+/g, '-')
    .replace(/^[._]/, '')
    .replace(/[^a-z0-9-~]+/g, '-')
}

// 使用node的内置方法，判断本地是否存在该名称的文件夹，以及该文件夹下是否有文件
function canSafelyOverwrite(dir) {
  return !fs.existsSync(dir) || fs.readdirSync(dir).length === 0
}
```

在执行完所有的交互配置后，会在文件内部`result`属性中记录最终的配置结果：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b95f5d4163047e7804616c7a7a411f0~tplv-k3u1fbpfcp-zoom-1.image)



##### 3.2.2生成模板

生成完所有配置后，会根据配置进行模板生成，具体生成顺序如下：

-   生成base基础模板
-   根据不同配置需求，继续添加合并模板


-   如果需要typeScript，将所有.js文件改成.ts，并将jsconfig.json改成tsconfig.json
-   如果不需要测试，就把测试相关文件删除


-   生成README文档



第一步！生成模板

```
import renderTemplate from './utils/renderTemplate.js'

// 设置模板文件地址
const templateRoot = path.resolve(__dirname, 'template')

// 生成模板主函数
const render = function render(templateName) {
  // 具体要生成的模板目录下的子目录
  const templateDir = path.resolve(templateRoot, templateName)
  // 传入子目录和项目的地址
  renderTemplate(templateDir, root)
}

// Render base template
render('base')
```

接着看看具体生成模板的流程，同样也分成三部分：

-   递归copy所有的模板文件进项目中
-   如果文件中存在package.json，进行对象合并

<!---->

-   将以`_`开头的文件替换成`.`开头

第一部分：处理文件夹

```
const stats = fs.statSync(src)
// 如果传入的src是文件夹，就递归调用renderTemplate处理文件夹下的每一个文件
if (stats.isDirectory()) {
  // if it's a directory, render its subdirectories and files recursively
  fs.mkdirSync(dest, { recursive: true })
  for (const file of fs.readdirSync(src)) {
    renderTemplate(path.resolve(src, file), path.resolve(dest, file))
  }
  return
}
```

第二部分：处理`package.json`文件：

```
import deepMerge from './deepMerge.js'
import sortDependencies from './sortDependencies.js'
const filename = path.basename(src)

// 如果文件名称为package.json且该文件已经存在
if (filename === 'package.json' && fs.existsSync(dest)) {
  // 存放需要操作的文件
  const existing = JSON.parse(fs.readFileSync(dest))
  // 存放需要拷贝的文件
  const newPackage = JSON.parse(fs.readFileSync(src))
  // 重点！：调用了两个函数，deepMerge用于处理两个json的合并，sortDependencies用于排序
  const pkg = sortDependencies(deepMerge(existing, newPackage))
  fs.writeFileSync(dest, JSON.stringify(pkg, null, 2) + '\n')
  return
}
```

处理文件过程中使用到两个函数：`deepMerge`、`sortDependencies`，分别用于处理合并和排序

继续深入看`deepMerge`，这部分主要是对 对象和数组的处理：

-   处理对象时，通过递归赋值进行处理
-   处理数组时，通过解构合并进行处理

```
const isObject = (val) => val && typeof val === 'object'
const mergeArrayWithDedupe = (a, b) => Array.from(new Set([...a, ...b]))

/**
 * Recursively merge the content of the new object to the existing one
 * @param {Object} target the existing object
 * @param {Object} obj the new object
 */
function deepMerge(target, obj) {
  // 循环传入的template中的模板
  for (const key of Object.keys(obj)) {
    const oldVal = target[key]
    const newVal = obj[key]
		//如果参数的内容是数组
    if (Array.isArray(oldVal) && Array.isArray(newVal)) {
      // 使用解构进行合并
      target[key] = mergeArrayWithDedupe(oldVal, newVal)
    } else if (isObject(oldVal) && isObject(newVal)) {
      // 如果参数内容是对象，就继续递归处理，直到是内容
      target[key] = deepMerge(oldVal, newVal)
    } else {
      // 直接为内容时，直接赋值
      target[key] = newVal
    }
  }

  return target
}
```

合并后继续做排序的处理，这时会进入到`sortDependencies`函数中

```
export default function sortDependencies(packageJson) {
  const sorted = {}
	// 依照数组中元素的顺序，进行遍历排序
  const depTypes = ['dependencies', 'devDependencies', 'peerDependencies', 'optionalDependencies']

  for (const depType of depTypes) {
    if (packageJson[depType]) {
      sorted[depType] = {}

      Object.keys(packageJson[depType])
        .sort()
        .forEach((name) => {
          sorted[depType][name] = packageJson[depType][name]
        })
    }
  }

  return {
    ...packageJson,
    ...sorted
  }
}
```

第三部分，修改文件名

项目中以`.`开头的文件都是一些配置文件，而如果直接在模板中存放这些文件时，可能会造成编译器识别的一些影响，所以在模板中，都以`_`开头存放文件，在生成模板文件时再进行改名

这里比较简单，就是一个node.path的调用

```
if (filename.startsWith('_')) {
  // rename `_file` to `.file`
  dest = path.resolve(path.dirname(dest), filename.replace(/^_/, '.'))
}
```

至此，一个模板已经完全生成！

让我们进入简单的第二步：根据自定义配置继续生成模板

这部分比较简单，从之前的配置信息中，生成调用不同路径下的文件，继续生成模板

```
  // Add configs.
  if (needsJsx) {
    render('config/jsx')
  }
  if (needsRouter) {
    render('config/router')
  }
  if (needsVuex) {
    render('config/vuex')
  }
  if (needsTests) {
    render('config/cypress')
  }
  if (needsTypeScript) {
    render('config/typescript')
  }

  // Render code template.
  // prettier-ignore
  const codeTemplate =
    (needsTypeScript ? 'typescript-' : '') +
    (needsRouter ? 'router' : 'default')
  render(`code/${codeTemplate}`)

  // Render entry file (main.js/ts).
  if (needsVuex && needsRouter) {
    render('entry/vuex-and-router')
  } else if (needsVuex) {
    render('entry/vuex')
  } else if (needsRouter) {
    render('entry/router')
  } else {
    render('entry/default')
  }
```

继续进入第三步：使用`typeScript`

这部分比较简单，递归遍历所有后缀为`.js`的文件，替换成`.ts`，以及将jsconfig.json转换成tsconfig.json（简单粗暴），在这个过程中调用了`preOrderDirectoryTraverse`方法，我们重点细看这部分方法

```
if (needsTypeScript) {
  // rename all `.js` files to `.ts`
  // rename jsconfig.json to tsconfig.json
  preOrderDirectoryTraverse(
    root,
    () => {},
    (filepath) => {
      // 如果后缀为.js
      if (filepath.endsWith('.js')) {
        // 进行替换
        fs.renameSync(filepath, filepath.replace(/.js$/, '.ts'))
        // 同时如果为jsconfig.json
      } else if (path.basename(filepath) === 'jsconfig.json') {
        // 也进行替换
        fs.renameSync(filepath, filepath.replace(/jsconfig.json$/, 'tsconfig.json'))
      }
    }
  )

  // Rename entry in `index.html`
  const indexHtmlPath = path.resolve(root, 'index.html')
  const indexHtmlContent = fs.readFileSync(indexHtmlPath, 'utf8')
  fs.writeFileSync(indexHtmlPath, indexHtmlContent.replace('src/main.js', 'src/main.ts'))
}
```

继续进入`preOrderDirectoryTraverse`

```
// 这里主要是做文件夹和文件的递归
export function preOrderDirectoryTraverse(dir, dirCallback, fileCallback) {
  // 首先读取该路径下所有文件
  for (const filename of fs.readdirSync(dir)) {
    const fullpath = path.resolve(dir, filename)
    // 如果是一个文件夹
    if (fs.lstatSync(fullpath).isDirectory()) {
      // 执行文件夹的回调
      dirCallback(fullpath)
      // in case the dirCallback removes the directory entirely
      // 如果文件夹没有被删除，还存在
      if (fs.existsSync(fullpath)) {
        // 继续递归里面的文件
        preOrderDirectoryTraverse(fullpath, dirCallback, fileCallback)
      }
      continue
    }
    // 执行文件的回调
    fileCallback(fullpath)
  }
}
```

至此，ts的转化也已经完成~

进入第四步：删除测试相关文件

调用`fs.rmdirSync`删除文件夹时，只能删除空文件夹，所以需要通过遍历文件夹以及文件夹内所有的文件，先删除文件，再删除文件夹

这里同样调用了`preOrderDirectoryTraverse`方法，

并且在文件夹回调函数中调用了`emptyDir`方法

```
if (!needsTests) {
  preOrderDirectoryTraverse(
    root,
    (dirpath) => {
      const dirname = path.basename(dirpath)
			// 如果文件夹为测试文件夹
      if (dirname === 'cypress' || dirname === '__tests__') {
        // 调用方法进行删除
        emptyDir(dirpath)
        fs.rmdirSync(dirpath)
      }
    },
    () => {}
  )
}


function emptyDir(dir) {
  // 继续文件夹递归
  postOrderDirectoryTraverse(
    dir,
    (dir) => fs.rmdirSync(dir),
    (file) => fs.unlinkSync(file)
  )
}

export function postOrderDirectoryTraverse(dir, dirCallback, fileCallback) {
  for (const filename of fs.readdirSync(dir)) {
    const fullpath = path.resolve(dir, filename)
    // 如果是文件夹，就继续递归
    if (fs.lstatSync(fullpath).isDirectory()) {
      postOrderDirectoryTraverse(fullpath, dirCallback, fileCallback)
      dirCallback(fullpath)
      continue
    }
    // 如果是文件，调用文件回调函数
    fileCallback(fullpath)
  }
}
```



最后第五步：生成README

```
fs.writeFileSync(
  path.resolve(root, 'README.md'),
  generateReadme({
    projectName: result.projectName || defaultProjectName,
    packageManager,
    needsTypeScript,
    needsTests
  })
)
```



### 4. 总结

这次的cli源码阅读能感知到自己的知识短板，在执行流程能大概读懂外，对于很多封装方法中涉及到的算法、node指令，阅读起来还是比较吃力，需要反复使用debug和查阅node文档进行理解。

结合第6期，可以发现在制作脚手架、开源库时，有很多工具库可以直接使用，加快开发速度，例如`prompts`交互式生成配置，就是一个很不错的库

后续还会反复阅读这个库的源码，学习和借鉴制作cli的流程，开发一个在工作团队内部可以方便快速拉取模板代码库的cli，并总结一篇制作cli工具的文章