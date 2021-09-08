# 解读ECMA规范

>ECMAScript是基于对象的：基本语言和主机设施是有对象提供的，而ECMAScript程序是通信对象的集群。

### JS中的数据类型

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



### 数据类型检测

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

### 数据类型转换

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

