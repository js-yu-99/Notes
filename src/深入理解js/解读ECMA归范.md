# 解读ECMA规范

>ECMAScript是基于对象的：基本语言和主机设施是有对象提供的，而ECMAScript程序是通信对象的集群。

## JS中的数据类型

+ 原始数据类型
  + number:  NaN（不是有效数字）、Infinity（无穷大的值）
  + string: 基于单引号、双引号、反引号包起来的都是字符串
  + boolean: true/false
  + null
  + symbol: 唯一值
  + undefined
  + bigint: 大数
+ 对象类型
  + 标准普通对象： Object
  + 标准特殊对象：  Array、Regexp、Date、Error、Math、ArrayBuffer、DateView......
  + 非标准特殊对象： Number、String、Boolean、BigInt......（基于构造函数创建出来的原始值对象类型的格式信息，类型属于对象类型。）
  + 可调用对象： （实现了call方法）function

> + isNaN(value)  不论value是什么类型，默认隐式转换为数字类型Number，再校验是否是有效数字，如果是有效数字返回false，不是有效数字返回true（isNaN('NaN')  // true）
> + Object.is(NaN, NaN) 不会隐式转换  (Object.is(NaN, 'NaN')  // false)

> Symbol的作用：
>
> + 对象的唯一属性
> + 宏观管理标识：保证标志唯一性（vuex、redux）
>
> **只能通过Object.getOwnPropertySymbols获取对象中的Symbol**

> 主要用于大数的计算，打印出大数的计算带有n的后缀 **90071992547409912222n**，加减数字都需要加n  **90071992547409912222n - 1n = 90071992547409912221n**

## 数据类型检测

+ typeof运算符
+ instanceof 检测实例是否属于某个类
+ constructor 获取构造函数
+ Object.prototype.toString.call(value) 检测数据类型
+ Array.isAray(value) 检测值是否为数组

> typeof 的弊端：
>
> + 不能检测null    typeof null  -> object
> + 除可调用对象会返回**function**，其他对象都返回**object**
> + 检测一个未被声明的变量，不会报错，会返回**undefined**
>
> 底层检测类型是按照值存储的二进制进行检测的。对象是 000 开头的，所有null（000000）被检测为**object**

## 数据类型转换

+ 把原始值转为对象：Object(value)
+ 把其他类型转为数字
  + Number(value) （隐式转换）
    + 数学运算、isNaN、==比较
    + 字符串 -> 数字：空字符串转为0、字符串只要出现非数字就会转为NaN
    + 布尔 -> 数字：true变为1，false变为0
    + null -> 0
    + Undefined -> NaN
    + Symbol -> 报错
    + BigInt -> 去掉n后缀
    + 对象 -> 数字：(Symbol.toPrimitive > valueOf > toString)  -> Number
  + parseInt/parseFloat(value)（手动转换）
    + 首先把value变为字符串，从字符串左侧第一个字符开始查找，直到找到一个非有效数字的位置为止。把找到的结果转换为数字，一个都没找到返回NaN。
+ 把其他类型转为字符串
  + 原始值转换直接用引号包起来
  + toString()
  + 字符串/模板字符串 通过加号拼接（+）
+ 把其他类型转为布尔值
  + 只有**0、NaN、null、undefined、空字符串**会转换为false，其他都是true
  + Boolean(value)、!!value、条件判断、逻辑表达式

```js
console.log(!![]); // true
console.log(!!-1); // true
console.log(1 + '1'); // '11'
let n = '10';
console.log({} + n); // 10 运行时将左侧{} 当成是代码块，不参与运算
console.log(n + {}); // '10[object Object]' 字符串拼接

let obj1 = {
  [Symbol.toPrimitive]: function (hint) {
    console.log(hint); // 表示要转换到的原始值的预期类型 'default' | 'string' | 'number'
    return this.x;
  },
  x: 10,
  valueOf: function () {
    return 20;
  }
}
console.log(10 + obj1); // 20  Symbol.toPrimitive > valueOf > toString
let obj2 = {
  [Symbol.toPrimitive](hint) {
    if (hint == "number") {
      return 10;
    }
    if (hint == "string") {
      return "hello";
    }
    return true;
  }
};
console.log(+obj2);     // 10      -- hint 参数值是 "number"
console.log(`${obj2}`); // "hello" -- hint 参数值是 "string"
console.log(obj2 + ""); // "true"  -- hint 参数值是 "default"

console.log(Number('10px'), parseInt('10px')); // NaN, 10
console.log(Number(null), parseInt(null)); // 0, NaN
let result = 100 + true + 21.2 + null + undefined + 'wang' + [] + null + 9 + false;
console.log(result); // NaNwangnull9false
var a = {
  x: 1,
  valueOf: function () {
    return this.x++;
  }
}
if (a == 1 && a == 2 & a==3) {
  console.log('ok');
}
let arr = [27.2, 0, '0013', '14px', 123];
arr = arr.map(parseInt);
console.log(arr); // [27, NaN, 1, 1, 27]
```

> + `i++` 和 `i = i + 1` 以及` i += 1` 三个是否一样？
>   + `i = i + 1` 和` i+=1`是一样的（数字或者字符串）
>   + `i++` 一定返回数字
> + `i++` 和 `++i` 的不同？
>   + `++i` 先i累加1，累加后再进行运算
>   + `i++` 先运算，再累加1
> + 对象隐式转换字符串规则
>   + 先调用对象的Symbol.toPrimitive 属性值，如果没有
>   + 再调用对象的valueOf()获取原始值，如果不是原始值
>   + 再去调用对象的 toString转换为字符串
>
> **Symbol.toPrimitive** 是一个内置的 Symbol 值，它是作为对象的函数值属性存在的，当一个对象转换为对应的原始值时，会调用此函数。 --MDN

---

**js验证两个值是否相等**

+ `==` : 内部会进行隐式类型转换，转为相同类型

  + 对象 == 字符串  对象转为字符串

  + null == undefined   => true

  + NaN == NaN  => false

  + 对象 == 对象   对比内存地址

  + 对象 == 数字、字符串 == 布尔 ...... 都要转为数字，再进行比较

  + ```js
    console.log(![] == false) // true
    console.log([] == false) // true
    ```

+ `===`

+ `Object.is()`



> **parseInt(string, radix)**
>
> + `radix`表示进制，有效范围是 2 - 36之间，不写默认是10进制。如果字符串以`0x` 开始的，默认是16进制
>
> + 如果写0 和不写是一样的
>
> + 把string看做radix进制，从左侧找到所有符合这个进制的字符，遇到不符合的结束查找，把找到的字符转换为数字`十进制`
>
> + 如果radix不在2 - 36之间[排除0]，则返回的是NaN
> + 如果一个数字以零开始，<span style="color: red">浏览器会默认认为其是八进制，并转为十进制</span>
>
> ```js
> // 字符串按照符合进制要求的数字范围进行查找，比如8的范围是 0-7，所以超过7的数字字符就不做拾取
> parseInt('AE', 16); // 174
> parseInt('99', 8); // NaN
> parseInt('22', 2); // NaN
> parseInt('126', 2); // 1
> parseInt('16', 0); // 16
> parseInt('1234.56'); // 1234
> parseInt(0123); // 83
> ```

---

# JS运行机制

js代码运行环境

+ 浏览器 -> webkit内核(V8)、Trident、Gecko、Blink。。。

+ Node -> webkit内核

+ webview ->webkit内核

```js
var a = 12;
var b = a;
b = 13;
cosnole.log(a); // 12
```

**堆内存和栈内存都存在于运行内存中 (CPU)**

**ECStack (Execution Context Stack) **执行环境栈（栈内存）

- 供代码执行
- 存储原始值和变量

**EC (Execution Context)** 执行上下文  |  **ECG** 全局执行上下文

- 区分代码执行的环境
- 全局代码都会在全局上下文中执行
- VO （Varlable Object）变量对象
  - 存储当前上下文声明的变量
  - 栈内存

**等号赋值操作**

+ 创建一个值（原始值类型直接在栈内存中存储起来，对象类型单独开辟一个堆内存空间，用来存储对象中的成员等信息）
+ 声明变量 Declare  （var、function、let、const。。。）把声明的变量存储到当前上下文的`”变量对象 VO/AO“`中，声明的变量存在栈内存中，存在变量提升
+ 让变量和创建的值关联到一起 Defined定义

<span style="color:red; font-weight:bold">计算机中所有的赋值操作都是指针指向操作</span>

**对象类型创建**

+ 在堆内存中开辟一块单独的空间，会产生一个供访问的16进制的地址

+ 把对象中的键值对依次存储到空间当中
+ 把空间地址放到栈中存储，以此来供变量引用

**GO 全局对象（堆内存）**

+ 存放浏览器内置的API

```js
var a = {n: 1};
var b = a;
a.x = a = {n: 2};
console.log(a.x); // undefined
console.log(b); // {n: 1, x: {n: 2}}  成员访问优先级比=赋值高
```



***创建函数***

+ 开辟一个堆内存空间，有一个16进制的地址
+ 存储的内容：函数体中的代码当做字符串先存储起来；当做普通对象也会存一些键值对。
+ 创建函数时，声明了其作用域[[scope]] （创建函数所在的上下文）
+ 堆内存地址放在栈中，供函数名（变量）调用

***函数执行***

+ 形成一个私有的执行上下午`EC(fn)`，创建`AO（Active Object）` 函数中的变量对象
+ 初始化作用域链SCOPE-CHAIN：<EC(fn), EC(G)> 左侧是自己的私有上下文，右侧是创建函数的作用域
+ 初始化this指向
+ 初始化arguments（实参集合）
+ 形参赋值（形参是私有变量）
+ 变量提升
+ 代码执行
+ 根据情况，决定当前形成的私有上下文是否会出栈释放
+ 函数再次执行，所有的操作重新走一遍，函数每一次执行没有直接关系

```js
var x = [12, 23];
function fn(y) {
	y[0] = 100;
	y = [100];
	y[1] = 200;
	console.log(y); // [100, 200]
}
fn(x);
console.log(x); // [100, 23]
```

```js
/**
EC(G)
	VO(G)
		i = 0;
		A = 0X000 [A函数 [[scope]]: EC(G)] // SCOPE-CHAIN
		y = 0x001
		B = 0x003 [B函数 [[scope]]:EC(G)]
*/
var i = 0;
function A() {
  /**
  	EC(A)
  		AO(A)
  			i = 10
  			x = 0x001 [x函数 [[scope]]:EC(A)]
  		作用域链：<EC(A), EC(G)>
  */
	var i = 10;
	function x() {
    /**
    	EC(x)
    		AO(x)
    		作用域链：<EC(x), EC(A)> 函数执行的上级上下文是创建它的作用域【只和在哪创建有关系，和在哪执行没有关系】
    */
		console.log(i); // 获取其上级上下文 EC(A) 中的 i
	}
  return x; // return 0x001;
}
var y = A();
y();
function B() {
  /**
  	EC(B)
  		AO(B)
  			i = 20
  		作用域链：<EC(B), EC(G)>
  */
  var i = 20;
  y();
}
B();
```

```js
let x = 5;
function fn(x) {
  return function (y) {
    console.log(y + (++x));
  }
}
let f = fn(6);
f(7);
fn(8)(9);
f(10);
console.log(x);
// 14 18 18 5
```

**函数执行**

+ 产生一个私有的上下文，然后进栈

+ 当函数执行完，一般情况下，当前形成的上下文都会被栈释放掉（优化栈内存）：上下文被释放，之前存储的私有变量等也会被释放；

+ 但是如果当前上下文中的某些东西（一般都是堆内存），被当前上下文以外的事物所占用，则当前上下文不能出栈释放，函数中声明的所有变量也都被存储起来

`闭包`是一种函数执行的机制，函数执行产生的私有上下文，一方面可以保护内部的私有变量不被污染，另一方面如果不被释放，私有的变量及相关信息也都会保存起来。我们把这种`保护`  + `保存`的机制，称之为`闭包`。（不被释放的上下文称为`闭包`）

**堆内存的释放**

> 如果当前的堆被引用，则不能释放，如果不被引用，浏览器会在空闲的时候释放它

**垃圾回收机制（GC）**

+ 引用计数：被占用一次计数累加1，当取消运用再减去1，当减到零的时候，会把其释放掉
+ 标记清除：被占用后做一个标记，当移除引用时，取消标记，在浏览器空闲的时候，会把所有未被标记的内存回收



```js
let a = 0, b = 0;
function A(a) {
  A = function (b) {
    console.log(a + (b++));
  };
  console.log(a++);
}
A(1); // 1
A(2); // 4
```

# 变量提升

**预解析**：在**`当前上下文`**代码自上而下执行之前，浏览器会把所有带 **`var`** 和 **`function`** 关键字的进行提前的声明或定义

+ 带var的只是提前声明

+ 带function的是提前声明+赋值（定义）

```js
/**
	EC(G)
		变量提升：
			var a;
			fn1 = 0x000; [[scope]]:EC(G)
			var fn2;
			
			a = 10;
			fn2 = 0x001; [[scope]]:EC(G)
*/
console.log(a); // undefined
fn1(); // 'fn1'
var a = 10;
function fn1() { // 函数声明
  /**
  	EC(FN1)
  		作用域链： <EC(FN1), EC(G)>
  		形参赋值：--
  		变量提升：
  			var a;
  			a = 20;
  */
	console.log('fn1'); // 'fn1'
  console.log(a); // undefined
  var a = 20;
  console.log(a); // 20;
}
fn2(); // fn2 is not a function
var fn2 = function () { // 函数表达式
  console.log('fn2');
}
fn2();
// 项目中推荐使用函数表达式的方式创建函数，可以规范函数执行的顺序
var fn = function sum() {
  console.log('sum');
  console.log(sum);
}
// 匿名函数具名化，设置的函数名不能在函数以外使用（并没有在当前上下文中声明这个变量）
// 具名化的名字可以在函数内部的上下文中使用，代表函数本身；默认情况下，其值是不能被修改的；但是可重新声明同名变量，当做私有变量处理

// 老版本浏览器会不管判断直接对函数提升赋值
// 新版本浏览器对于判断体中的函数只声明 不定义
if (1 === 1) {
  function foo() {}
}

// arguments.callee 表示一个函数本身，但是在严格模式下会报错。
// 匿名函数具名化是为了可以在自执行函数内调用其本身
```



# let/const/var 的区别

`let`和`const`都是声明变量，只不过`const`不允许重定向变量的指针，不能重新赋值

```js
let x = 10;
x = 20;
const y = 10;
y = 20; // Uncaught TypeError: Assignment to constant variable. （variable =》变量）
```

## var VS let

+ let不存在变量提升

```js
console.log(x); // Uncaught ReferenceError: Cannot access 'x' before initialization
// 无法在初始化之前访问'x'
let x = 10;
```

**词法解析（AST）**：基于HTTP从服务器拉取回来的JS代码是一段字符串，浏览器首先会按照ECMAScript规则，把字符串变为C++可以识别和解析的一套树结构对象，***所以let一个变量前访问这个变量，词法解析已经知道这个变量会被声明，所以会提示`Cannot access 'x' before initialization` 而不是 `x is not defined`***

+ let不允许重复声明（不论基于什么方式）

  词法解析阶段报的错误，所有代码不会被执行

  ```js
  console.log(1); // 不执行
  var x = 20;
  console.log(x); // Uncaught SyntaxError: Identifier 'x' has already been declared
  let x = 10;
  ```

+ 全局上下文中基于var声明的变量，新版浏览器中不是存放到VO(G)中的，而是直接放到了GO(window)中，基于let声明的变量是存放到VO(G)中的。

  + **全局上下文查找一个变量：**
  + 1.先去VO(G)中是否存在，如果存在就用这个全局变量；
  + 2.如果VO(G)中没有，则再次尝试去GO中找，因为js中的某些操作是可以省略window的，如果有就是获取某个属性值；
  + 3.如果再没有，则直接保存：xxx is not defined

+ 暂时性死区问题

+ 块级作用域（排除函数和对象的大括号）中出现let/const/function，则当前的{}会成为一个块级私有的上下文

  ```js
  /**
  	EC(G)
  		GO -> x: 10
  		VO(G) -> y = 20
  		变量提升：
  			var x;
  			x = 10;
  */
  var x = 20;
  let y = 20;
  if (1 === 1) {
    /**
    	EC(BLOCK)
    		VO(BLOCK) y = 200
    			作用域链：<EC(BLOCK), EC(G)>
    			this: 使用其上级上下文的this
    			变量提升：var不受块级上下文影响
    */
  	var x = 100; // 操作全局的x，var中没有块级上下文 window.x = 100;
  	let y = 200;
  	console.log(x, y); // 100 200
  }
  console.log(x, y); // 100 20
  ```

  ```js
  for (var i = 0;i < 5;i++) {
  	setTimeout(() => {
      console.log(i); // 5 5 5 5 5 
    })
  }
  for (let j = 0;j < 5;j++) { // 产生6个块级上下文 控制循环的父级块上下文，五次循环产生的五个子级块上下文
  	setTimeout(() => {
      console.log(j); // 0 1 2 3 4
    })
  }
  
  let arr = [10, 20, 30];
  let i = 0, len = arr.length;
  for (; i < len; i++) { // 不会产生块级上下文 性能更好
    console.log(i);
  }
  ```




# 变量提升练习题

```js
console.log(foo) // undefined
{
  console.log(foo); // foo
  function foo() {}
  foo = 1;
  console.log(foo); // 1
}
console.log(foo); // foo

/**
在新版本浏览器中
1、如果function出现在除函数、对象的大括号中，则在变量提升阶段，只声明不定义。（即在全局声明变量foo）
2、如果除了函数、对象的大括号中，只要出现let/const/function 关键词，都会产生块级私有上下文，对var无效。
3、function foo() {} 如果变量赋值时在全局和块级上下文中都出现过，那么就会导致一个特殊性
	+ 把这段代码及以前的对foo的操作，都映射给全局一份
	+ 但是之后的代码都只认为是操作块级上下文中的，和全局上下文没有关系
*/

/**
两次function foo() {}
每次都把上面对foo的操作映射给全局上下文一份
*/

console.log(foo) // undefined
{
  console.log(foo); // foo(2)
  function foo() {1}
  foo = 1;
  console.log(foo); // 1
  function foo() {2}
  console.log(foo); // 1
}
console.log(foo); // 1

console.log(foo) // undefined
{
  console.log(foo); // foo(2)
  function foo() {1}
  foo = 1;
  console.log(foo); // 1
  function foo() {2}
  console.log(foo); // 1
  foo = 2;
  console.log(foo); // 2
}
console.log(foo); // 1
```



> 浏览器某机制：如果当前函数使用了ES6中的形参赋值默认值（不论是否生效），并且函数体内有基于let/const/var 声明的变量，则函数在执行的时候，除了形成一个私有的函数上下文，而且还会把函数体{} 当做一个私有的块级上下文，并且块级上下文的上级上下文是私有函数上下文。
>
> 如果函数体中声明的变量和形参变量一直，最开始的时候，会把形参变量的值，同步给同名的私有变量一份。

```js
var x = 1;
function func(x, y = function foo() { x = 2 }) {
  x = 3;
  y();
  console.log(x); // 2
}
func(5);
console.log(x); // 1

var x = 1;
function func(x, y = function foo() { x = 2 }) {
  var x = 3;
  y();
  console.log(x); // 3
}
func(5);
console.log(x); // 1

var x = 1;
function func(x, y = function foo() { x = 2 }) {
  var x;
  y();
  console.log(x); // 5
}
func(5);
console.log(x); // 1
```

