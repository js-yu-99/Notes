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
+ Array.isAray(valye) 检测值是否为数组

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

+ `Object,is()`



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
cosnole.log(a);
```

**堆内存和栈内存都存在于运行内存中 (CPU)**

**ECStack (Execution Context Stack) **执行环境栈（栈内存）

- 供代码执行
- 存储原始值和变量

**EC (Execution Contextl)** 执行上下文  |  **ECG** 全局执行上下文

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
+ 存储的内容：函数体中的代码当做字符串先存储起来。
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
+ 根据情况，决定当前形成的私有上下文是否会出站释放
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

