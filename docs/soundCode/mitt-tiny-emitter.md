---
date: 2021-11-30
title: 【源码计划第七期】mitt、tiny-emitter 发布订阅
tags:
  - 源码
describe: 认识发布订阅模式
---

### 1.阅读前准备

这两个库都是在vue中非常熟悉、使用频率很高的事件传递方法之一，而Vue3.x版本移除了内置的$on、$off的方法，改用使用第三方库进行该方法的调用

阅读前先温习一下正常的使用方法：

#### 1.1 使用方法

首先 **$on和$emit的事件必须在一个公共实例上才会被触发** 。

在vue项目开发时，经常会遇到兄弟、爷孙组件的传值情况，此时通常会使用Vue实例作为中央事件处理总线，将事件放置在全局中触发

```
vue.prototype.$bus = new Vue();

this.$bus.$emit('test','gelx')

// mounted中接受函数
mounted() {
	this.$bus.$on('test',info => {
    console.log(info)
  })
}
```



#### 1.2 简单了解Map

源码中大量使用的Map的方法，由于之前没有接触过Map，因此做一个小小的预习

##### 1.2.1 什么是map

Map对象保存键/值 对，是键/值对的集合

##### 1.2.2与Object对象的不同点



-   Map的键可以是任意值
-   Map的键值是有顺序的，遍历时会按插入的顺序返回键值


-   通过size属性可以获取Map的键值对个数
-   Map可以进行迭代

##### 1.2.3常用的属性、方法

-   属性


-   -   size 获取键值对数量


-   方法


-   -   clear() 清空map对象
    -   delete(key) 移除某个指定元素


-   -   get(key) 返回某个指定元素
    -   has(key) 检测是否存在某个元素

-   -   set(key,value) 添加至Map对象的key 键和对应的value 值

### 2.开始阅读



今天主要有两部分源码

第一部分mitt: <https://github.com/developit/mitt>

第二部分tiny-emitter: <https://github.com/scottcorgan/tiny-emitter>

学习的目标就是了解和学习发布订阅者模式

### 3.源码正文

#### 3.1 miit源码阅读

首先开始阅读mitt源码

具体源码目录如下，主要有两部分：源码主体、test测试用例

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f21b58adf35a494b95d456d71c61d1df~tplv-k3u1fbpfcp-zoom-1.image)

第一件事，`tsc src/index.ts`

希望自己可以早点学会ts，进行ts源码阅读……

##### 3.1.1 源码主体

整个源码函数主要实现了四个功能：

1：接收一个外来参数，或直接新建一个Map对象

2：实现on函数

3：实现off函数

4：实现emit函数

```
function mitt(all) {
    all = all || new Map();
    return {
        all: all,
        on: function (type, handler) {
            // code ...
        },
        off: function (type, handler) {
            // code ...
        },
        emit: function (type, evt) {
            // code ...
        }
    };
}
```

##### 3.1.2 on()，它是一个订阅者订阅事件的函数

```
on: function (type, handler) {
  // 获取指定type对应的handler
  var handlers = all.get(type);
  // 如果存在，就为已有的处理上push一个新的handler
  if (handlers) {
    handlers.push(handler);
  }
  // 如果不存在，就新增一个type，且对应的处理由数组存放，为了方便添加多个处理
  else {
    all.set(type, [handler]);
  }
},
```

##### 3.1.3 off()，它是一个订阅者删除订阅事件的函数

```
off: function (type, handler) {
  // 同样获取指定type的handler
  var handlers = all.get(type);
  // 如果传入了handler
  if (handlers) {
    // 且这个type存在时
    if (handler) {
      // 会在type中寻找与传入的handler相对应的handler
      // 如果存在，就会直接返回在数组中的位置，进行删除
      // 如果不存在，就会直接返回原数组，这里下面会继续分析
      handlers.splice(handlers.indexOf(handler) >>> 0, 1);
    }
    // 如果没有传入handler
    // 默认当前type下全部清空
    else {
      all.set(type, []);
    }
  }
},
```

这部分出现了一个 `>>> 0` 的表达式，进一步了解这个表达式的意义

js中>>表示有符号位移，>>>表示无符号位移

-   有符号位移表示 右边移去一个0，左边补充一个1
-   无符号位移表示 右边移去一个0，左边补充一个0

当需要操作正数的时候，是不会有差别的

但操作负数的时候，就会有非常大的差别

负数在二进制中的表达是以补码的形式存在，也就是正常的二进制码的反码再加+

负数进行无符号的位移时，会在左边补充一个0，变成了正数，也就是目前数组的长度，删除一个最大长度上的第一位（空值），所以这部分会将内容全部返回

##### 3.1.4 emit()，它是一个发布者发布事件的函数

```
emit: function (type, evt) {
  // 获取指定type对应的handlers
  var handlers = all.get(type);
  // 如果存在
  if (handlers) {
    // 浅拷贝handlers，并循环对所有handler执行参数为evt的方法
    handlers
      .slice()
      .map(function (handler) {
      handler(evt);
    });
  }
  // 如果不存在
  // 找全局上绑定的handlers
  handlers = all.get('*');
  // 如果全局上纯在handlers
  // 依次进行执行，并传递当前触发的type
  if (handlers) {
    handlers
      .slice()
      .map(function (handler) {
      handler(type, evt);
    });
  }
}
```

这部分的emit仅可传递一个参数，如果需要传递多个参数，需要以对象或数组的形式传入



#### 3.2.1 tiny-emitter 部分



这个工具库实现的功能与mitt大同小异，同样实现了`$on $once $off $emit`的功能，但具体的使用上会有一些小的差异

##### 3.2.2 使用方式

先看看官方文档给出的使用案例：

```
var Emitter = require('tiny-emitter');
var emitter = new Emitter();

emitter.on('some-event', function (arg1, arg2, arg3) {
 //
});

emitter.emit('some-event', 'arg1 value', 'arg2 value', 'arg3 value');
```

从这里就可以看到与miit一个比较大的不同是：emit可以传递多个参数

##### 3.2.3 源码阅读

\


首先依旧是`tsc index.d.ts`

这个库是定义了一个空的函数，通过在它的原型上定义了4种方法：

-   on()
-   once()

<!---->

-   emit()
-   off()

```
function E () {
  // Keep this empty so it's easier to inherit from
  // (via https://github.com/lipsmack from https://github.com/scottcorgan/tiny-emitter/issues/3)
}

E.prototype = {
  on: function (name, callback, ctx) {
    //code...
  },

  once: function (name, callback, ctx) {
    //code...
  },

  emit: function (name) {
    // code...
  },

  off: function (name, callback) {
    // code...
  }
};
```

##### 3.2.4 on() 订阅事件

```
on: function (name, callback, ctx) {
  	// 判断是否存在e,如果不存在，就是用空对象 
  	// mitt是用的是Map
    var e = this.e || (this.e = {});
  	// 并判断传入的name是否存在，用数组的方式存储
    (e[name] || (e[name] = [])).push({
      fn: callback,
      ctx: ctx
    });
		// 返回this，保持上下文依旧可以调用
    return this;
  },
```

##### once() 只执行一次订阅时间

```
once: function (name, callback, ctx) {
    var self = this;
    function listener () {
      // 执行完成后调用off注销事件
      self.off(name, listener);
      callback.apply(ctx, arguments);
    };

    listener._ = callback
    return this.on(name, listener, ctx);
  },
```

##### emit() 发布事件

```
emit: function (name) {
  // 创建一个data数组，并获取参数，这里可以拿到多个参数
  var data = [].slice.call(arguments, 1);
  var evtArr = ((this.e || (this.e = {}))[name] || []).slice();
  var i = 0;
  var len = evtArr.length;

  for (i; i < len; i++) {
    // 遍历执行回调函数，并接收传入的data参数
    evtArr[i].fn.apply(evtArr[i].ctx, data);
  }

  return this;
},
```

##### off() 取消事件绑定

```
off: function (name, callback) {
    var e = this.e || (this.e = {});
  	// 获取e上name中对应的所有的回调函数
    var evts = e[name];
    var liveEvents = [];
		// 如果off传入了回调函数
    if (evts && callback) {
      // 会遍历name中所有的函数
      for (var i = 0, len = evts.length; i < len; i++) {
        	// 如果存在与传入的回调函数不同的函数，就push进数组中
        if (evts[i].fn !== callback && evts[i].fn._ !== callback)
          liveEvents.push(evts[i]);
      }
    }
  	// 最后如果这个数组存在内容，就保留继续执行，如果不存在，就整个删掉
    (liveEvents.length)
      ? e[name] = liveEvents
      : delete e[name];

    return this;
  }
```

### 4.总结



通过阅读这两个开源库也学到了：

-   Map的具体用法，与Object的异同点
-   发布订阅的设计模式


-   事件传递的原理，也能在实际工作中使用时，更清楚为什么要这么使用