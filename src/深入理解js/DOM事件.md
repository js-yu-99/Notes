## 什么是事件？

事件是浏览器赋予元素的默认行为，不论我们是否为其绑定方法，当某些行为触发的时候，相关的时间都会被触发执行。

https://developer.mozilla.org/zh-CN/docs/Web/Events

+ 鼠标事件
  + click：点击事件（PC）、单击事件（移动端，300ms内没有发生第二次点击）
  + dblclick：双击事件
  + ......

+ 键盘事件
+ 手指事件
+ ......



## 什么是事件绑定？

给元素默认的事件行为绑定方法，这样可以在行为触发的时候，执行这个方法。

```js
document.body.onClick = function () {}
// 给body的点击事件行为绑定方法。
```

+ DOM0事件绑定
  + [元素].on[事件] = [函数]
  + 移除绑定：[元素].on[事件]  = null 或 其他非函数值
  + 原理：每一个DOM元素对象的私有属性上都有很多类似于“onxxx”的私有属性，我们给这些代表事件的私有属性赋值。
  + 如果没有对应事件的私有属性值，则无法基于这种方法实现事件绑定
  + 只能给当前元素的某个事件行为绑定一个方法。
  + 好处是执行效率快，开发者使用方便。
+ DOM2事件绑定
  + 语法：[元素].addEventListener([事件], [回调函数], [捕获/冒泡])
  + 移除：[元素].removeEventListener([事件], [回调函数], [捕获/冒泡])，但是需要参数和绑定的时候一样（主要是回调函数）
  + 原理
    + 绑定的方法一般不是匿名函数，主要目的是方便移除事件绑定的时候使用
    + 每一个DOM元素都会基于原型链的查找机制，查找到EventTarget.prototype上的addEventListener等方法，基于这些方法实现事件的绑定和移除。
    + 采用事件池机制，可以给某个事件绑定多个不同的方法。
    + 凡是浏览器支持的事件行为，都可以基于这种模式完成事件的绑定和移除





## 事件对象

给当前元素的某个事件行为绑定方法，当事件行为触发，方法执行的时候，不仅会把方法执行，而且还会给方法默认传递一个实参，这个实参就是事件对象。

是用来储存当前事件操作及触发的相关信息的，浏览器本身记录的，记录的是当前这次操作信息，和在哪个函数中无关。（同一类事件类型的事件对象是同一个）

+ 鼠标事件对象 MouseEvent
  + clentX、clentY  鼠标触发点距离当前窗口左上角的x、y轴坐标
  + pageX、pageY  鼠标触发点距离body 左上角的x、y轴坐标
  + type  事件类型
  + target/srcElement 获取当前事件源（当前操作的元素）
  + path 传播路径
  + ev.perventDefault() / ev.returnValue = false 阻止默认行为
  + ev.stopPropagation() / ev.cancelBubble = true 阻止冒泡传播
+ 键盘事件对象 KeyboardEvent
  + Which / keyCode 获取按下的键盘码
  + altKey  是否按下alt键（组合键）
  + ctrlKey  是否按下alt键（组合键）
  + shiftKey  是否按下alt键（组合键）
  + key / code 存储键盘名字
+ TouchEvent  手指事件对象
  + changedTouches / targetTouches / touches 都是用来记录手指信息的。
  + changedTouches 手指按下、移动、离开屏幕  都存储了对应的信息，哪怕离开屏幕后，存储的也是最后一次手指在屏幕汇总的信息；获取的结果都是一个TouchList集合，记录每一根手指的信息；



# 默认行为

+ 鼠标右键菜单
+ a标签跳转
+ 部分浏览器会记录输入内容
+ 键盘按下会输入内容



禁用右键菜单

```js
window.oncontextmenu = function (ev) {
	ev.preventDefault();
}
```



# 事件传播

+ 阶段一：捕获阶段（CAPTURING_PHASE）
  + 从最外层元素一直向里面逐级查找，直到找到事件源为止
  + 目的是为了冒泡阶段的传播提供路径（event.path）
  + window -> document -> html -> body -> ... -> 事件源

+ 阶段二：目标阶段（AT_TARGET）
  + 触发事件源的相关事件行为

+ 阶段三：冒泡阶段（BUBBLING_PHASE）
  + 按照捕获阶段收集的传播路径，不仅仅当前时间源的相关事件行为被触发，而且从内到外，其祖先所有元素的相关事件行为也都会被处罚



addEventListener(事件，方法，false/true)

最后一个参数默认是false：控制方法是在冒泡阶段触发执行的，如设置为true 可以控制在捕获阶段触发执行



事件代理 可以提高60%左右的性能
