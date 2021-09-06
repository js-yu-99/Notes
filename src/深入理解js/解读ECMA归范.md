# 解读ECMA规范

>ECMAScript是基于对象的：基本语言和主机设施是有对象提供的，而ECMAScript程序是通信对象的集群。

JS中的数据类型：

+ 原始数据类型
  + number:  NaN（不是有效数字）、Infinity（无穷大的值）
  + string: 基于单引号、双引号、反引号包起来的都是字符串
  + boolean: true/false
  + null
  + symbol: 唯一值
  + undefined
  + bigint
+ 对象类型
  + 标准普通对象： Object
  + 标准特殊对象：  Array、Regexp、Date、Error、Math、ArrayBuffer、DateView......
  + 非标准特殊对象： Number、String、Boolean、BigInt......
  + 可调用对象： （实现了call方法）function

> + isNaN(value)  不论value是什么类型，默认隐式转换为数字类型Number，再校验是否是有效数字，如果是有效数字返回false，不是有效数字返回true（isNaN('NaN')  // true）
> + Object.is(NaN, NaN) 不会隐式转换  (Object.is(NaN, 'NaN')  // false)

> Symbol的作用：
>
> + 对象的唯一属性
> + 宏观管理标识：保证标志唯一性（vuex、redux）
>
> **只能通过Object.getOwnPropertySymbols获取对象中的Symbol**


