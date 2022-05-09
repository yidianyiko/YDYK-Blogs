---
date: 2021-11-4
title: 【源码第二期】Vue3-20个工具函数
tags:
  - 源码
describe: 浅析 Vue3-20个工具函数
---

> 第二期源码阅读活动
> 
> 咕咕了一星期的文档
> 
> 依旧很感谢若川大哥组织的源码阅读活动，以下为原文：<https://juejin.cn/post/6994976281053888519>
### 1.可以收获什么?

本次阅读的是Vue3源码中的一小部分内容—— ***工具函数***。一共20个，虽然看起来十分简单，但却可以学到

-   通过build获得一份ts转js的文档
-   通过生成SourceMap，在浏览器中调试代码

<!---->

-   非常基础但很有用的数据类型属性使用方法（尤其是Object）

在开始前，需要准备：

-   目标代码库：`https://github.com/vuejs/vue-next`
-   也许会需要用到的翻译插件（彩云小译），翻译出来的效果如下图


### 2.开始调试

首先可以在`contributing.md`中找到如何参与项目的开发、调试，了解当前项目的目录结构等详细的前期准备操作。

#### 2.1 安装依赖，打包构建

正常情况下，我们可以直接在目录`packages/shared/src/index`中找到我们需要阅读的ts版本的源码文件。

但为了方便阅读，也可以通过`yarn build`进行打包构建，在完成打包构建后，会在`packages/shared/dist/`中发现以下三个文件：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7196d7ad3794403a88588bc081895a68~tplv-k3u1fbpfcp-zoom-1.image)

他们都是打包产物：唯一的区别在于`cjs、esm`的区别：

-   `cjs`是通过require方式加载读取依赖，并且只能在Node中运行，不可在浏览器中运行
-   `esm`是用过import方式加载读取依赖，浏览器和Node都可以读取并加载。

#### 2.2 生成SourceMap调试源码

这步实现起来较为简单，只需要在`package.json` 下的scripts对象中dev声明后追加 `--scourcemap`即可生成，生成的地址会在控制台中输出：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/04ecefe5735d419796e2c76d865efe14~tplv-k3u1fbpfcp-zoom-1.image)

此时随便找一个页面将该文件引入，启动服务之后即可在控制台中进行打点阅读源码的操作。
### 3. 工具🔧函数

本次阅读的重头戏工具函数源码部分。

这次选择阅读ts版本，想通过阅读这部分的源码加深ts的认识。

开干！


#### 3.1 babelParserDefaultPlugins babel默认插件

```
/**
 * List of @babel/parser plugins that are used for template expression
 * transforms and SFC script transforms. By default we enable proposals slated
 * for ES2020. This will need to be updated as the spec moves forward.
 * Full list at https://babeljs.io/docs/en/next/babel-parser#plugins
 */
/*
*用于模板表达式的@babel/parser插件列表转换和SFC脚本转换
*/
export const babelParserDefaultPlugins = [
  'bigInt',
  'optionalChaining',
  'nullishCoalescingOperator'
] as const
```

这部分的`as const`是const断言，作用跟接下来会常出现的`readonly`功能类似，只不过这样使用时可以创建一个完整的readonly对象（会为对象中的每一个属性前都加上readonly），具体文章可以参考[const断言](https://segmentfault.com/a/1190000019239979?utm_source=tag-newest)。


#### 3.2 EMPTY_OBJ 空对象 EMPTY_ARR 空数组 NOOP 空函数 NO 永远返回 false 的函数

```
export const EMPTY_OBJ: { readonly [key: string]: any } = __DEV__
  ? Object.freeze({})
  : {}


export const EMPTY_ARR = __DEV__ ? Object.freeze([]) : []



export const NOOP = () => {}

/**
 * Always return false.
 */
export const NO = () => false
```

这四部分声明都比较简单和短小

首先空对象和空数组会在操作前判断当前所处的环境，是开发环境还是生产环境。

`Object.freeze`是一个冻结对象的操作，被冻结的对象无法被修改或添加新属性。

空函数则是使用该属性时相当于使用了一个空的函数，可以用在判断或return

永远返回false，顾名思义了

#### 3.3 isOn 判断字符串是否以on开头，且on后首字母为大写字母

```
const onRE = /^on[^a-z]/
export const isOn = (key: string) => onRE.test(key)

// ex
isOn('onClick') // true
isOn('onclick') // false
```

这部分涉及到简单的正则表达式：

-   `^`符号在开头，表示为以什么东西开头，在其他地方是指非。同时`$`在结尾时，表示以什么东西结尾。
-   `[^a-z]`就为不是a-z的小写字母

#### 3.4 isModelListener 监听器

```
export const isModelListener = (key: string) => key.startsWith('onUpdate:')
```

这个方法为判断字符串是否以`onUpdate:`为开头

`.startsWith`为ES6新增方法

#### 3.5 extend 合并

```
export const extend = Object.assign
```

其实只是对对象合并的方法进行了一个封装



#### 3.6 remove 去除

```
export const remove = <T>(arr: T[], el: T) => {
  const i = arr.indexOf(el)
  if (i > -1) {
    arr.splice(i, 1)
  }
}
```

传入一个数组和一个元素，判断该元素是否存在数组中，若存在，则删除

Tips: 使用splice虽然可以很方便的进行数组元素的删除，但删除后会将其他的元素位置进行移动，若要处理很庞大的数组项目时，则会造成性能的大量消耗。因此在这部分可以考虑将移除改为置为`NULL`


#### 3.7 hasOwn 判断是否为本身所拥有的属性

```
const hasOwnProperty = Object.prototype.hasOwnProperty
export const hasOwn = (
  val: object,
  key: string | symbol
): key is keyof typeof val => hasOwnProperty.call(val, key)
```

通过对象自带的`hasOwnProperty` API判断当前传入的key是否为obj本身的属性

这部分当中还涉及到我不认识的ts语法 `is keyof typeof`

通过参考同组的纪年小姐姐文档后，发现这是三个ts语法`is`、`keyof`、`typeof`，万能的百度告诉我：

-   is: 类型保护，可以通过is进一步缩小变量的类型
-   keyof: 返回一个类型的所有key组成的联合类型

<!---->

-   typeof: 获取一个变量或者对象的类型




#### 3.8 is… 判断是否为某种类型

```
// 判断是否为数组
export const isArray = Array.isArray

// 判断是否为Map对象
export const isMap = (val: unknown): val is Map<any, any> =>
  toTypeString(val) === '[object Map]'

// 判断是否为Set对象 
export const isSet = (val: unknown): val is Set<any> =>
  toTypeString(val) === '[object Set]'

// 判断是否为Date对象
export const isDate = (val: unknown): val is Date => val instanceof Date

// 判断是否为函数
export const isFunction = (val: unknown): val is Function =>
  typeof val === 'function'

// 判断是否为字符串
export const isString = (val: unknown): val is string => typeof val === 'string'

// 判断是否为Symbol
export const isSymbol = (val: unknown): val is symbol => typeof val === 'symbol'

// 判断是否为对象(不包括null)
export const isObject = (val: unknown): val is Record<any, any> =>
  val !== null && typeof val === 'object'

// 判断是否为Promise
export const isPromise = <T = any>(val: unknown): val is Promise<T> => {
  return isObject(val) && isFunction(val.then) && isFunction(val.catch)
}
```


#### 3.9 toTypeString 对象转字符串，以及附带的判断函数

```
// 对象转字符串
export const objectToString = Object.prototype.toString
export const toTypeString = (value: unknown): string =>
  objectToString.call(value)

// 对象转字符串并截取第八位开始到最后一位
//这个函数的作用为可以判断更多的类型种类
export const toRawType = (value: unknown): string => {
  // extract "RawType" from strings like "[object RawType]"
  return toTypeString(value).slice(8, -1)
}

// 判断是否为纯纯的对象
export const isPlainObject = (val: unknown): val is object =>
  toTypeString(val) === '[object Object]'
```


#### 3.10 isIntegerKey 判断是否为数字型的字符串key

```
export const isIntegerKey = (key: unknown) =>
  isString(key) &&
  key !== 'NaN' &&
  key[0] !== '-' &&
  '' + parseInt(key, 10) === key
```

首先判断key是否为字符串类型，接着排除key不为NaN值，继续排除负值，最后将key通过parseInt方式转换为数字并与通过+隐式转换为字符串，最后与原本的key做对比




#### 3.11 判断是否为保留属性

```
/**
 * Make a map and return a function for checking if a key
 * is in that map.
 * IMPORTANT: all calls of this function must be prefixed with
 * /*#__PURE__*/
 * So that rollup can tree-shake them if necessary.
 */
export function makeMap(
  str: string,
  expectsLowerCase?: boolean
): (key: string) => boolean {
  const map: Record<string, boolean> = Object.create(null)
  const list: Array<string> = str.split(',')
  for (let i = 0; i < list.length; i++) {
    map[list[i]] = true
  }
  return expectsLowerCase ? val => !!map[val.toLowerCase()] : val => !!map[val]


export const isReservedProp = /*#__PURE__*/ makeMap(
  // the leading comma is intentional so empty string "" is also included
  ',key,ref,' +
    'onVnodeBeforeMount,onVnodeMounted,' +
    'onVnodeBeforeUpdate,onVnodeUpdated,' +
    'onVnodeBeforeUnmount,onVnodeUnmounted'
)
```

这里的操作逻辑为，传入一个以逗号分隔的字符串，通过makeMap函数将字符串转化为数组，创建一个空对象，并且循环将这个key赋值进空对象中，最后返回一个包含参数val的值，检查参数val是否存在在字符串中


#### 3.12 cacheStringFunction 缓存字符串函数

```
// 缓存的主函数，实现效果与makeMap类似
const cacheStringFunction = <T extends (str: string) => string>(fn: T): T => {
  const cache: Record<string, string> = Object.create(null)
  return ((str: string) => {
    const hit = cache[str]
    return hit || (cache[str] = fn(str))
  }) as any
}

// 连字符 - 转换为小驼峰 
const camelizeRE = /-(\w)/g
/**
 * @private
 */
export const camelize = cacheStringFunction((str: string): string => {
  return str.replace(camelizeRE, (_, c) => (c ? c.toUpperCase() : ''))
})

// 大写字母转化为 - 连字符
const hyphenateRE = /\B([A-Z])/g
/**
 * @private
 */
export const hyphenate = cacheStringFunction((str: string) =>
  str.replace(hyphenateRE, '-$1').toLowerCase()
)


// 首字母转大写
/**
 * @private
 */
export const capitalize = cacheStringFunction(
  (str: string) => str.charAt(0).toUpperCase() + str.slice(1)
)

// 给输入的字符串前加入on，并将输入的第一个字符串改为大写
/**
 * @private
 */
export const toHandlerKey = cacheStringFunction((str: string) =>
  str ? `on${capitalize(str)}` : ``
)
```


#### 3.14 hasChanged 判断是否有变化

```
// compare whether a value has changed, accounting for NaN.
export const hasChanged = (value: any, oldValue: any): boolean =>
  !Object.is(value, oldValue)
```

Object.is 为输入两个值，返回这两个值是否为同一个值的


#### 3.15 invokeArrayFns 执行数组中的函数

```
export const invokeArrayFns = (fns: Function[], arg?: any) => {
  for (let i = 0; i < fns.length; i++) {
    fns[i](arg)
  }
}
```

可以实现一次执行多个函数的操作


#### 3.16 def 定义一个不可枚举的对象

```
export const def = (obj: object, key: string | symbol, value: any) => {
  Object.defineProperty(obj, key, {
    configurable: true,
    enumerable: false,
    value
  })
}
```

#### 3.17 toNumber 转换为数字

```
export const toNumber = (val: any): any => {
  const n = parseFloat(val)
  return isNaN(n) ? val : n
}
```


#### 3.18 getGlobalThis 全局对象

```
let _globalThis: any
export const getGlobalThis = (): any => {
  return (
    _globalThis ||
    (_globalThis =
      typeof globalThis !== 'undefined'
        ? globalThis
        : typeof self !== 'undefined'
        ? self
        : typeof window !== 'undefined'
        ? window
        : typeof global !== 'undefined'
        ? global
        : {})
  )
}
```

简单来说就是根据不同的环境，采用不同的参数指向全局`this`

大佬的解释：

获取全局 this 指向。

初次执行肯定是 _globalThis 是 undefined。所以会执行后面的赋值语句。

如果存在 globalThis 就用 globalThis。[MDN globalThis](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/globalThis)

如果存在self，就用self。在 Web Worker 中不能访问到 window 对象，但是我们却能通过 self 访问到 Worker 环境中的全局对象。

如果存在window，就用window。

如果存在global，就用global。Node环境下，使用global。

如果都不存在，使用空对象。可能是微信小程序环境下。

下次执行就直接返回 _globalThis，不需要第二次继续判断了。这种写法值得我们学习。



### 4.总结

-   通过这些工具函数，对vue的模板语法解析有了一点点点的了解
-   工具函数除了达到满足开发需求、提升开发效率外，还可以达到优化性能的目的

<!---->

-   学习到了没了解过的TS语法，以及对象的语法