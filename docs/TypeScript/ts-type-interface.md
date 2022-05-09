---
date: 2021-11-24
title: 【TypeScript第一期】基础类型与接口
tags:
  - TypyScript
describe: 学习 TS基础类型与接口 
---

### 1. 食用Typescript



在使用Typescript之前，我只知道它是一个可以支持类型检查的JavaScript超集，从今天开始决定重新好好认识Typescript，为之后接触React和阅读其他开源库源码做准备。

### 2.类型

ts支持着与js几乎相同的数据类型，并且在js上还支持了其他的类型

#### 2.1 基础类型

```
// 布尔
let isBoolean:boolean = true;

// 数字
let isNum:number = 6;

//　字符串
let isStr:string = 'gelx'

// 数组
// 这里有两种写法，一种是直接在元素后加[]
// 另一种是使用泛型(?)后面会继续了解这是啥

// 第一种写法
let isNumArr: number[] = [1,2,3]
let isStrArr: string[] = ['g','e','l','x']

// 第二种写法
let isArr:Array<number> = [1,2,3]

// null && undefined
// 他们是所有类型的子集，可以赋值在任何类型上
let n:null = null
let u:undefined = undefined

// Object

let obj:object = {x:1}
```

#### 2.2 拓展类型

##### 2.2.1 元组

表示一个已知数量和类型的数组，数量和类型不受限制

```
let makeTup:[string, number, string]

x = ['gelx', 18, 'cm']
```

##### 2.2.2 枚举

为一组数值赋予意义，可以自定义值的数值

```
enum Name {xm,xh,xb}

let name:Name = Name.xm
```

ts这个写法，转换为js之后是这样的

```
var Name;
(function (Name) {
    Name[Name["xm"] = 0] = "xm";
    Name[Name["xh"] = 1] = "xh";
    Name[Name["xb"] = 2] = "xb";
})(Name || (Name = {}));
var n = Name.xm
```

其实就是定义了一个变量，用一个自执行的函数，往这个变量里添加对应参数相关的序号和内容，所以在官方文档中，枚举中的元素可以改变顺序

```
enum Name {xm=1 ,xh,xb}

let name:Name = Name.xm

let numb:string = Name[2] 
console.log(numb)  // xh
```

##### 2.2.3 万物皆可any

any表示可以是任何类型，在我们不清楚这个变量的类型时候，可以采用any进行标记，这可以使得这个变量可以传入所有的类型

```
let isAny: any

isAny = 'string' // ok
isAny = false // ok
```


##### 2.2.4 void

void跟any相反，它表示没有任何类型，一般用于没有返回值的函数声明中

```
function tVoid(): void {
	console.log('gelx')
}
```

##### 2.2.5 never

这个类型表示永不存在的类型，比如抛出异常的函数、不会有返回值的函数

```
function nError(msg:string): never {
	throw new Error(msg)
}
```

#### 2.3 类型断言

在进行对某一个any类型的属性赋值或对其进行参数、内容查询的时候，我们可以在赋值时对属性进行断言

```
let anyValue: any = "maybe is string"

// 这里有两种写法，
// 尖括号写法 不可用于JSX
let stringL: number =(<string>anyValue).length

// as写法，可用于JSX

let strL:number = (anyValue as string).length
```

### 3. 接口

定义： 对象类型的描述，并且还可以对类的一部分行为进行抽象



#### 3.1 简单使用

最简单的使用：直接定义接口内部的参数。此时若添加的参数比定义的少、多，都不被允许，并抛出错误

```
interface testInter {
	name: string;
  age: num
}

// 被允许的情况
let gelx: testInter = {
	name: 'gelx';
  age: 22;
}

// 不被允许的情况：少参数
let gelxError1: testInter = {
	name: 'gelx'
}

// 不被允许的情况：多参数
let gelxError2: testInter = {
	name: 'gelx';
  age: 24;
  toll: 183
}
```

****

#### 3.2 可选属性

如果不确定参数是否是需要的，可以通过设置`?`设置可选参数

但这依旧不能添加未定义的参数

```
interface testChoosable {
	name: string;
  age?: num
}

// 被允许的情况
let gelx: testChoosable  = {
	 name: 'gelx'
}

let gelx: testChoosable = {
	name: 'gelx',
  age: 23;
}


// 不被允许的情况
let gelx: testChoosable = {
	name: 'gelx',
  type: 'people'
}
```

#### 3.3 任意属性

如果希望接口可以有任何属性，可以使用`[propName: any]:any`定义一个任意属性取任意类型的值

**如果定义了任意属性，确定属性和可选属性的类型都必须是它的类型子集**

****

```
interface person {
	name: string;
  age?: number;
  [propsName: any]:any
}

let gelx : Preson = {
	name: 'gelx',
  age: 22,
  sex: 'man'
}
```

#### 3.4 只读属性

可以理解为与`const`类似，当我们想定义一些只能在创建时赋值的字段时，可以使用`readonly`定义只读属性

```
interface readOnlyTest {
	readonli id: number;
	name: string;
	age: number
  [propName:any] :any
}

let gelx: readOnlyTest = {
	name: 'gelx',
  age: 22
}

// 此时会报错
gelx.id = 2021
```

这里的只读属性的约束，**存在于第一次给对象赋值时，而不是第一次给只读属性赋值**
