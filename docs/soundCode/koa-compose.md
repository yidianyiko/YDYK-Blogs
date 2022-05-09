---
date: 2021-11-08
title: 【源码第四期】理解koa-compose
tags:
  - 源码
describe: 浅析 koa-compose 中间件原理
---

> 
> 非常感谢若川大佬的阅读源码活动，原文：<https://juejin.cn/post/7005375860509245471>

### 1. 阅读前准备



在阅读之前有简单的了解过koa,以及中间件的执行过程，并且也在上一期活动中通过CO源码的阅读，加深了next()函数的执行原理。今天看看koa-compoes做了一些什么操作。

### 2. 学习目标

本次阅读学习主要有以下几个目标：

-   阅读源码，了解代码意义
-   对比co源码，解决了什么问题




### 3.开始阅读

koa-compose项目主要分为两部分：

-   jest的test测试用例部分
-   index.js koa-compose主文件

直接进入index.js 开读！

这里只有短短不到50行的代码，compose函数结构上看，通过闭包的形式完成了整个功能

具体代码如下：

这里分为两部分：

-   第一部分校验参数
-   第二部分接收参数并返回Promise

```
function compose (middleware) {
  if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
  }

  /**
   * @param {Object} context
   * @return {Promise}
   * @api public
   */

  return function (context, next) {
    // last called middleware #
    let index = -1
    return dispatch(0)
    function dispatch (i) {
    }
  }
}
```

第一部分很好理解，首先对接收到的参数进行两次判断：

-   如果这个参数不是数组，则抛出错误
-   如果这个数组参数中的每一项都不是函数时，则抛出错误

<!---->

-   如果这个参数是数组，且数组中的每一项都是函数时，则继续往下执行dispatch函数

```
function dispatch (i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i]
      if (i === middleware.length) fn = next
      if (!fn) return Promise.resolve()
      try {
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)))
      } catch (err) {
        return Promise.reject(err)
      }
    }
```

dispatch函数做了以下这些事：

-   首先判断是否存在next()多次调用的情况，如果存在则会直接报错
-   接着陆续取出middlewatre中的第1、2、3...个函数并开始执行

<!---->

-   如果检测到为最后一个时，next为undefined，此时fn不是函数，直接返回resolve
-   如果不是最后一个，则也会返回一个resolve，但resolve内容是取到下一个middleware中的函数

<!---->

-   当执行到next()函数时，会重新return进入dispatch函数内部，继续执行下一个函数
-   直到最后一个函数时，fn为undefined，直接resolve

一句话总结：嵌套调用Promise.resolve，往里深入，执行完毕后，从底部再重新往上执行

于是我简化了一下compose，这样能更方便理解：

```
const [t1, t2, t3] = stack;
const testC = function(context){
    await Promise.resolve(
      t1(context, function next(){
        return Promise.resolve(
          t2(context, function next(){
              return Promise.resolve(
                  t3(context, function next(){
                    return Promise.resolve();
                  })
              )
          })
        )
    })
  );
};
```

### 4.总结

虽然koa-compose源码不多，但设计非常精巧，一个

`return Promise.resolve(fn(context, dispatch.bind(null, i + 1)))`

就将所有的中间件串联了起来

我也能加深对中间件的理解，以前只知道怎么使用，现在可以知道这其中的原理。

#### 4.1 与co的对比

相同点：

co与koa-compose都同样通过使用promise完成了串联和调用。

不同点：

co传入的是一个`generator`，koa-compose传入的是一个普通函数

co通过promise自动执行`generator`的next函数，koa-compose则是通过一个闭包，将promise嵌套在内部并执行
