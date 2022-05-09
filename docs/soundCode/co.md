---
date: 2021-11-06
title: 【源码第三期】简单理解co源码及实现
tags:
  - 源码
describe: 浅析 co源码
---

>
> 更多的源码阅读文章更新在自己的语雀：https://www.yuque.com/shouhu-pelkv/nnq9kp
>
> 非常感谢若川大佬的阅读源码活动，原文：<https://juejin.cn/post/6844904088220467213#heading-11>
### 1. 什么是CO




之前完全没有听说过CO是什么，简单的了解到它是利用了`ES6的generator和promise特性`实现了异步流程控制的一个库。简单的实现流程就是co在内部将`generator`对象的`next()`方法自动执行，实现`generator`函数的运行，并返回最终运行的结果。




### 2. 阅读前准备




#### 2.1 简单了解generator

虽然`generator`函数是一个ES6提供的异步解决方案，但由于自己之前从来没有写过，所以需要“复习”下这块内容。




##### 2.1.1 generator的声明




`generator`函数声明时与普通函数不一样的地方主要有两个点：

-   function 与 函数名中间必须加一个* 号
-   定义内部状态时使用yield声明
-   function 与 函数名中间必须加一个* 号
-   定义内部状态时使用yield声明

```
function* testGenerator() {
	yield 'test';
  yield 'Gelx';
  
  return 'break';
}

let run = testGenerator();
```

##### 2.1.2 generator的使用

使用 `generator`函数时，函数并不会马上执行，它会指向内部状态的指针对象。如果要执行时，必须调用遍历器对象的`next`方法，使得指针移向下一个状态。每次调用next方法，内部指针就从函数头部或上一次停下来的地方开始执行，直到遇到下一个yield表达式（或return语句）为止。

更详细的generator函数介绍可以参考[Generator函数的语法](https://es6.ruanyifeng.com/#docs/generator)

### 3. 开始阅读

#### 3.1 阅读前疑问

在简单的了解了co的作用以及`generator`的使用后，有以下几个疑问：

-   怎么样实现依次调用next()方法
-   如何将**yield**后边运算异步结果返回给对应的变量

<!---->

-   co自身怎么返回generator函数最后的return值

#### 3.2 开始阅读



##### 3.2.1 源码地址

co的源码地址为：<https://github.com/tj/co>

#### 3.2.2 如何使用



在项目的`Readme.md`中可以找到co的使用方法

```
co(function* () {
  var result = yield Promise.resolve(true);
  return result;
}).then(function (value) {
  console.log(value);
}, function (err) {
  console.error(err.stack);
});
```

用法也比较简单，调用co函数，里面接收了一个`generator`函数，同时这个函数中返回了一个Promise。

#### 3.2.3 源码阅读

```
function co(gen) {
  var ctx = this;
  // 剔除gen对象，保留其他的入参
  var args = slice.call(arguments, 1);

 // 返回一个最外围的Promise
  return new Promise(function(resolve, reject) {
    // 判断gen是不是函数，以及判断gen.next是不是函数，不是函数则直接reslove
    if (typeof gen === 'function') gen = gen.apply(ctx, args);
    if (!gen || typeof gen.next !== 'function') return resolve(gen);
		// 执行第一次next
    onFulfilled();
    // 先执行gen.next，获取参数
    function onFulfilled(res) {
      var ret;
      try {
        ret = gen.next(res);
      } catch (e) {
        return reject(e);
      }
      // 将执行返回的结果传入next函数
      next(ret);
      return null;
    }
		
		// 对gen.throw
    function onRejected(err) {
      var ret;
      try {
        ret = gen.throw(err);
      } catch (e) {
        return reject(e);
      }
      next(ret);
    }

		// co中做的next执行
    function next(ret) {
      // 如果gen已经执行完毕，则直接返回结果
      if (ret.done) return resolve(ret.value);
      // 如果没有执行完成，就封装成Promise继续执行
      var value = toPromise.call(ctx, ret.value);
      // 判断value是否是Promise,如果是就继续执行，不是就Rejected
      if (value && isPromise(value)) return value.then(onFulfilled, onRejected);
      return onRejected(new TypeError('You may only yield a function, promise, generator, array, or object, '
        + 'but the following object was passed: "' + String(ret.value) + '"'));
    }
  });
}
```

通过对源码的简单阅读，可以总结出co执行的过程如下：

-   进入最外层promise
-   判断传入的gen、gen.next是否为函数，是则继续执行

<!---->

-   进入onFulfilled，并记录下`generator`函数执行的第一个yield返回的参数，并将该参数传入next函数
-   如果传入next函数的done为true，则返回最外层的promise的resolve

<!---->

-   如果传入next函数的done为false，则返回value，并判断该value是否可以转为内部promise对象，如果无法转移，则抛出错误，返回最外层promise的reject
-   如果可以转化为promise对象，则执行内部promise，通过`.then(onFulfilled, onRejected)`开始执行

<!---->

-   继续在onFulfilled、onRejected内部继续调用next函数，执行yield
-   当所有的yield执行返回完毕后，将最后的return值返回给最外层promise的resolve

<!---->

-   结束调用

#### 3.2.4 next函数

在整个co执行的流程中，最重要的就是next函数，而next函数中最重要的一步则是转化为内部Promise对象，这部分的源码如下：

```
function toPromise(obj) {
    // 确保obj有意义
    if (!obj) return obj;
    // 若是Promise对象，则直接返回
    if (isPromise(obj)) return obj;
    // 若是generator函数或者generator对象，则递归调用co函数
    if (isGeneratorFunction(obj) || isGenerator(obj)) return co.call(this, obj);
    // 若是函数，则转化promise类函数后再返回
    if ('function' == typeof obj) return thunkToPromise.call(this, obj);
    // 若是数组，转化promise数组后返回
    if (Array.isArray(obj)) return arrayToPromise.call(this, obj);
    // 若是对象，转化promise对象后返回
    if (isObject(obj)) return objectToPromise.call(this, obj);
    return obj;
}
```

### 4. 实现简单co

\


#### 4.1 实现简单模拟请求

首先先实现通过调用Promise获取参数

```
function request(time) {
	return new Promise(resolve => {
  	setTimeout(()=> {
    	resolve({name:"瘦虎"})
    },time)
  })
}


// 用yield获取请求到的值

function* getData() {
	yield request()
}
```

#### 4.2 实现简单co

整个co核心功能有两点：

-   参数传递
-   generator.next 自动执行

```
function co(gen) {
	1. 获取参数
  let ctx = this;
  const args = Array.propotype.slice.call(arguments, 1);
  gen = gen.apply(ctx,args);
	return new Promise((resolve,reject) => {
  	onFulfilled()
    
    function onFulfilled(res) {
    	let ret = gen.next(res);
      next(ret);
    }
    
    function next(ret) {
    	if(ret.done) return resolve(ret.value);
      let promise = ret.value
      
      promise && promise.then(onFulfilled);
    }
  })
}

// 最后效果
co(function* getData() {
	let res = yield request();
  console.log(result)
})
```

###

### 总结

这次阅读co花了很长的时间，原因在于除了需要了解co的整套执行逻辑外，还需要了解`generator`的使用方式、特性、执行规律，这样才能更好的理解为什么会有co函数。虽然源码不多，但看懂整套逻辑后，突然通透的感觉还是很舒服的，希望后续的阅读，我可以提升自己的阅读效率，更高效的完成学习。