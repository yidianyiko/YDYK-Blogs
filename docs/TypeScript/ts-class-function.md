---
date: 2021-11-30
title: 【TypeScript第二期】类、函数、泛型、枚举
tags:
  - TypyScript
describe: 学习 TS类、函数、泛型、枚举
---

### 1.类

TS中有以下这些的用法：

-   public 公有，可以在任何地方被访问，默认属性
-   private 私有，不能被外部访问

-   protected 受保护的，在子类中是允许被访问的
-   readonly

-   参数属性
-   抽象类

-   类的接口

#### 1.1 public

这是一个默认的修饰符，具体写法：

它可以被所有人访问和继承

```
class people {
	public name: string;
  public constructor(theName:string){
  	this.name = theName;
  }
	public move(age: number) {
  	console.log(`${this.name} age is ${age}`)
  }
}
```

#### 1.2 private

当构造函数的修饰符定义为`private`时，该类不允许被继承或实例化

```
class people {
	private name: string;
  public constructor(theName:string){
  	this.name = theName;
  }
	public move(age: number) {
  	console.log(`${this.name} age is ${age}`)
  }
}

new people('boy').name // error报错 是私有的，不允许访问
```

#### 1.3 protected

我理解这是介于`public`与`private`之间的一个属性，在子类中可以访问、继承，但不能直接被实例化

```
class prople {
    protected name: string;
    protected constructor(theName: string) {
      this.name = theName; 
    }
}

class boy extends people {
    private like: string;

    constructor(name: string, like: string) {
        super(name);
        this.like = like;
    }

    public getLike() {
        return `Hello, my name is ${this.name} and I like ${this.like}.`;
    }
}

let gelx = new boy("gelx", "drink");
let girl = new people("red"); // 错误: 'people' 的构造函数是被保护的.
```



#### 1.4 readonly

设置为只读属性，只读属性必须在声明时或构造函数初始化时声明参数，否则都会报错

```
class boy {
    readonly name: string;
    readonly age: number = 18;
    constructor (theName: string) {
        this.name = theName;
    }
}

let gelx = new boy('gelx')
gelx.age = 23; // error age是只读的
```

#### 1.5 参数属性

以上四种属性设置时，可以使用参数属性提升代码简洁

简化一下上面的例子

```
class boy {
  readonly age: number = 18;
	public constructor (public name: string) {
  
  }
}
```

直接在构造方法中传入一个属性，TS会直接将其翻译成，定义一个属性-传入一个属性-属性复制

#### 1.6 抽象类

声明类前添加一个`abstract`表示该类为抽象类

抽象类不能被实例化

```
abstract class people {
  public constructor(public name: string) {}
  public abstract sayHi();
}

let boy = new people('gelx'); // error！不能被实例化
```

抽象类中的抽象方法必须被子类实现

```
abstract class people {
  public name;
  public constructor(name) {
    this.name = name;
  }
  public abstract sayHi();
}

class boy extends people {
	pubilic sayHi() {
  	console.log(`my name is ${this.name}`)
  }
}
```

#### 1.7 类的接口使用

类定义会创建两个东西：类的**实例类型**和一个**构造函数**。 因为类可以创建出实例类型，所以可以在允许使用接口的地方使用类。

```
class Point {
    x: number;
    y: number;
}

interface Point3d extends Point {
    z: number;
}

let point3d: Point3d = {x: 1, y: 2, z: 3};
```

###

### 2. 函数

TS中函数有一些不一样的用法：

-   表达式
-   用接口定义函数

-   可选参数、参数默认值、剩余参数
-   重载

#### 2.1 表达式

一个正常的有输入输出的TS函数表达式：

```
function add(x:number, y:number):number {
	return x+y;
}
add(2,3)
```

如果定义一个参数为函数表达式：

```
let count = function (x:number, y:number):number {
	return x+y;
}
```

当然这种写法，左边的参数无法得到校验，因此通过以下写法，使得左边参数得到校验：

```
let count(x:number, y:number) => number = function (x:number, y:number):number {
	return x+y;
}
```

此时所用到的`=>` 与ES6中的`=>`不是同一个意思

TS类型定义中的`=>`表示函数的定义，左边为输入的类型，右边为输出的类型

#### 2.2 用接口定义函数

定义输入类型时，可以通过接口的方式定义一个输入类型的集合

```
interface addFunc {
	(x: number, y: number):boolean;
}

let count: addFunc => number = function (x:number, y:number):number {
	return x+y;
}
```

#### 2.3 可选参数、参数默认值、剩余参数



##### 2.3.1 可选参数

函数中同样可以使用`？`表示改参数是否可选。



**但可选参数后不允许再出现必选参数**

****

```
function consoleName(fName: string, lName?: string) {
	if (lName) {
        return fName + ' ' + lName;
    } else {
        return fName;
    }
}

let gelx = consoleName('g','x')
let gelx = consoleName('g')
```

****

##### 2.3.2 参数默认值



同样的，可以在传入参数时，设置一个默认值

```
function consoleName(fName: string = 'gelx', lName: string) {
	if (lName) {
        return fName + ' ' + lName;
    } else {
        return fName;
    }
}

let gelx = consoleName('g','x')
let gelx = consoleName('g')
```



##### 2.3.3 剩余参数



在ES6中，如果不知道具体会传递多少参数，可以使用`arguments`获取到所有传入的参数，同样在TypeScript中，可以把这些所有参数聚集在一个变量中，并以数组的方式存储。

**使用时仍需要注意，聚集的参数同样也只能被放在最后使用**

```
function name(fName: string , ...mName: stringp[]) {
	return `${fName} ${mName.join("")}`
}
```

#### 2.4 重载

允许一个函数接受不同数量或类型的参数时，作出不同的处理。当我们需要用到集合类型时，会比较体现出重载的意义

假如此时需要实现一个内容翻转的功能，可以对数字和字符串进行翻转

```
function reverse(x: number | string): number | string | void {
    if (typeof x === 'number') {
        return Number(x.toString().split('').reverse().join(''));
    } else if (typeof x === 'string') {
        return x.split('').reverse().join('');
    }
}
```

这么写不能够精准表达输入为数字时，输出也为数字

所以可以通过重复定义函数，并在最后一次进行函数的实现

```
function reverse(x: number): number;
function reverse(x: string): string;
function reverse(x: number | string): number | string | void {
    if (typeof x === 'number') {
        return Number(x.toString().split('').reverse().join(''));
    } else if (typeof x === 'string') {
        return x.split('').reverse().join('');
    }
}
```

### 3. 泛型

泛型为了解决当我们在定义函数、接口、类时，对不确定类型的传入参数，进行类型指定和保护，确保定义、返回值的类型是统一的。

#### 3.1 基础写法

对于一个参数的写法，是下面这样的基础写法

```
function test<T>(name: T):T {
	return name
}
```

也可以通过定义参数的方式声明泛型函数

```
let myTest: <T>(name: T) => T = test
```

\


当需要定义多个类型时，同样也可以定义多个泛型：

```
function testArr<T,U>(arr: [T,U]):[U,T] {
	return [arr[1], arr[0]]
}
```

#### 3.2 泛型约束

可以通过创建一个接口描述泛型的约束条件，并通过`extends`继承方式来实现约束

```
interface Lengthwise {
	length: number
}
function cLength<T extends Lengthwise>(num: T): T {
	console.log(num.length)
}

cLength({length:10, value: 3})
```

由于使用了`extends`约束泛型`T`必须符合接口 Lengthwise 的形状，所以传入的参数必须包含 `length` 属性

同样可以实现多个参数的相互约束

```
function getProperty(obj: T, key: K) {
    return obj[key];
}

let x = { a: 1, b: 2, c: 3, d: 4 };
getProperty(x, "a")
```

#### 3.3 泛型类

同样泛型可以在类中进行定义

```
class GenericNumber<T> {
    zeroValue: T;
    add: (x: T, y: T) => T;
}

let myGenericNumber = new GenericNumber<number>();
myGenericNumber.zeroValue = 0;
myGenericNumber.add = function(x, y) { return x + y; };
```

###

###

### 4.枚举

枚举又被分为四类：

-   数字枚举
-   字符串枚举

<!---->

-   常数枚举
-   外部枚举

#### 4.1 数字枚举

通过使用`enum`定义一个枚举

```
enum Direction {
    Up,
    Down,
    Left,
    Right
}
```

访问枚举时，可以通过枚举的属性、名字来访问枚举类型

##### 4.2 字符串枚举

```
enum Direction {
    Up = "UP",
    Down = "DOWN",
    Left = "LEFT",
    Right = "RIGHT",
}
```

\


#### 4.3 常数枚举

使用`const enum`代表常数枚举，常数枚举与普通枚举的区别是，它会在编译阶段被删除，并且不能包含计算成员

```
const enum Directions {
    Up,
    Down,
    Left,
    Right
}
```

####

#### 4.4 外部枚举

使用`declare`定义外部枚举，它只会用于编译时的检查，编译结果中会被删除

```
declare enum Directions {
    Up,
    Down,
    Left,
    Right
}
```