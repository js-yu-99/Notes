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

