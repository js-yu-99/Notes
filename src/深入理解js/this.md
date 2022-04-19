## this使用情况

+ 事件绑定

+ 函数执行（普通函数执行、对象成员访问、匿名函数、回调函数......）
+ 构造函数
+ 箭头函数（生成器函数generator）
+ 基于call、apply、bind强制修改this指向

this：函数执行的主体（谁执行的函数）

***全局上下文中的this：window***

***块级上下文中没有自己的this，所有的this都是继承上级上下文中的this***

### 事件绑定

+ DOM0: xxx.onxxx = function () {}
+ DOM2: xxx.addEventListener('xxx', function() {})  /  xxx.addachEvent('onxxx', function (){})

给当前元素的某个事件行为绑定方法（此时是创建方法，方法没执行）。当事件行为触发，浏览器会把绑定的函数执行，此时函数中的this是当前元素对象本身。



### 函数执行

+ 普通函数执行：看函数执行前是否有”点“操作符，有”点“的话”点“前面是谁this就是谁，没有"点"this是window（undefined）
+ 匿名函数
  + 函数表达式：等同于普通函数或者事件绑定等机制
  + 自执行函数：this一般都是window（undefined）
  + 回调函数：把一个函数作为实参，传递给另外一个执行的函数，this一般都是window（undefined），除非有特殊处理
+ 括号表达式：小括号中包含多项，这样也只取最后一项，但是this受到影响（一般都是window）

```js
function fn() {
	console.log(this);
}
let obj = {
  name: 'wang',
  fn: fn
};
fn(); // this -> window
obj.fn(); // this -> obj

(obj.fn)(); // this -> obj
(10, obj.fn)(); // this -> window  括号表达式

(function () {
  console.log(this)
})() // 自执行函数 this -> window

// 回调函数
function foo(callback) {
  callback();
}
foo(obj.fn); // this -> window

var arr = [1, 2, 3];
arr.forEach(function () {
  console.log(this); // this -> obj
}, obj);
```

```js
var x = 3, obj = {x: 5};
obj.fn = (function () {
	this.x *= ++x; // this -> window
	return function (y) {
		this.x *= (++x) + y;
		console.log(x);
	}
})();
var fn = obj.fn;
obj.fn(6);
fn(4);
console.log(obj.x, x);
/**
13
234
95 234
*/
```





---------



# 珠峰 - this

## 关于this的几种情况

+ 给当前元素的某个事件行为绑定方法，方法中的this是当前元素本身「排除IE低版本」。
+ 方法执行，看方法前面是否有“点”，有“点”点前面是谁，this就是谁，没有“点”，this是window“「严格情况下是undefined」。
  + 自执行函数中的this一般是window / undefined。
  + 回调函数中的this一般是window / undefined。某些方法中会做一些特殊的处理。
  + 括号表达式
  + ...
+ 构造函数执行，构造函数体中的this是当前类的实例。
+ 箭头函数中没有this「类似的还有块级上下文」，所以无论怎么去修改，都没有作用。一定是其所在的上级上下文中的this
+ call / apply / bind 这三个方法可以用来强制改变函数中的this指向。



### call

```js
fn.call(); // this -> window
fn.call(null) // this -> window / null  (严格模式)
```

