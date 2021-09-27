## Component

```js
// 类组件
function Component(props, context, updater) {
  this.props = props;      //绑定props
  this.context = context;  //绑定context
  this.refs = emptyObject; //绑定ref
  this.updater = updater || ReactNoopUpdateQueue; //上面所属的updater 对象
}
/* 绑定setState 方法 */
Component.prototype.setState = function(partialState, callback) {
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
}
/* 绑定forceupdate 方法 */
Component.prototype.forceUpdate = function(callback) {
  this.updater.enqueueForceUpdate(this, callback, 'forceUpdate');
}
```

Component 底层 React 的处理逻辑是，类组件执行构造函数过程中会在实例上绑定 props 和 context ，初始化置空 refs 属性，原型链上绑定setState、forceUpdate 方法。对于 updater，React 在实例化类组件之后会单独绑定 update 对象。

React 一共有 5 种主流的通信方式：

1. props 和 callback 方式
2. ref 方式。
3. React-redux 或 React-mobx 状态管理方式。
4. context 上下文方式。
5. event bus 事件总线。



## Fiber

```js
export const FunctionComponent = 0;       // 函数组件
export const ClassComponent = 1;          // 类组件
export const IndeterminateComponent = 2;  // 初始化的时候不知道是函数组件还是类组件 
export const HostRoot = 3;                // Root Fiber 可以理解为根元素 ， 通过reactDom.render()产生的根元素
export const HostPortal = 4;              // 对应  ReactDOM.createPortal 产生的 Portal 
export const HostComponent = 5;           // dom 元素 比如 <div>
export const HostText = 6;                // 文本节点
export const Fragment = 7;                // 对应 <React.Fragment> 
export const Mode = 8;                    // 对应 <React.StrictMode>   
export const ContextConsumer = 9;         // 对应 <Context.Consumer>
export const ContextProvider = 10;        // 对应 <Context.Provider>
export const ForwardRef = 11;             // 对应 React.ForwardRef
export const Profiler = 12;               // 对应 <Profiler/ >
export const SuspenseComponent = 13;      // 对应 <Suspense>
export const MemoComponent = 14;          // 对应 React.memo 返回的组件
```

fiber 对应关系

- child： 一个由父级 fiber 指向子级 fiber 的指针。
- return：一个子级 fiber 指向父级 fiber 的指针。
- sibiling: 一个 fiber 指向下一个兄弟 fiber 的指针。



## State

**state 到底是同步还是异步的？**

### 类组件中的 state

```js
function batchedEventUpdates(fn,a){
    /* 开启批量更新  */
   isBatchingEventUpdates = true;
  try {
    /* 这里执行了的事件处理函数， 比如在一次点击事件中触发setState,那么它将在这个函数内执行 */
    return batchedEventUpdatesImpl(fn, a, b);
  } finally {
    /* try 里面 return 不会影响 finally 执行  */
    /* 完成一次事件，批量更新  */
    isBatchingEventUpdates = false;
  }
}
```

在 React 事件执行之前通过 isBatchingEventUpdates=true 打开开关，开启事件批量更新，当该事件结束，再通过 isBatchingEventUpdates = false; 关闭开关，然后在 scheduleUpdateOnFiber 中根据这个开关来确定是否进行批量更新。



```jsx
export default class index extends React.Component{
    state = { number:0 }
    handleClick= () => {
          this.setState({ number:this.state.number + 1 },()=>{   console.log( 'callback1', this.state.number)  })
          console.log(this.state.number)
          this.setState({ number:this.state.number + 1 },()=>{   console.log( 'callback2', this.state.number)  })
          console.log(this.state.number)
          this.setState({ number:this.state.number + 1 },()=>{   console.log( 'callback3', this.state.number)  })
          console.log(this.state.number)
    }
    render(){
        return <div>
            { this.state.number }
            <button onClick={ this.handleClick }  >number++</button>
        </div>
    }
} 
```

点击打印：**0, 0, 0, callback1 1 ,callback2 1 ,callback3 1**

![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1632454162174-85bb711f-a414-40f1-9c5f-1aadea17172d.png)



**为什么异步操作里面的批量更新规则会被打破呢?**

```js
setTimeout(()=>{
    this.setState({ number:this.state.number + 1 },()=>{   console.log( 'callback1', this.state.number)  })
    console.log(this.state.number)
    this.setState({ number:this.state.number + 1 },()=>{    console.log( 'callback2', this.state.number)  })
    console.log(this.state.number)
    this.setState({ number:this.state.number + 1 },()=>{   console.log( 'callback3', this.state.number)  })
    console.log(this.state.number)
})
```

打印 ： **callback1 1 , 1, callback2 2 , 2,callback3 3 , 3** 

![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1632454219313-c758a592-c494-4f93-8bca-9d206ec82701.png)

## 

ReactDomFlushSync可以提高某个更新任务的优先级

```js
handerClick=()=>{
    setTimeout(()=>{
        this.setState({ number: 1  })
    })
    this.setState({ number: 2  })
    ReactDOM.flushSync(()=>{
        this.setState({ number: 3  })
    })
    this.setState({ number: 4  })
}
render(){
   console.log(this.state.number) // 3 4 1
   return ...
}
```

- 首先 flushSync this.setState({ number: 3 })设定了一个高优先级的更新，所以 2 和 3 被批量更新到 3 ，所以 3 先被打印。
- 更新为 4。

- 最后更新 setTimeout 中的 number = 1。

**flushSync补充说明**：flushSync 在同步条件下，会合并之前的 setState | useState，可以理解成，如果发现了 flushSync ，就会先执行更新，如果之前有未更新的 setState ｜ useState ，就会一起合并了，所以就解释了如上，2 和 3 被批量更新到 3 ，所以 3 先被打印。

综上所述， React 同一级别**更新优先级**关系是:

flushSync 中的 setState **>** 正常执行上下文中 setState **>** setTimeout ，Promise 中的 setState。

### 函数组件中的state

useState用法

> [ ①state , ②dispatch ] = useState(③initData)

- ① state，目的提供给 UI ，作为渲染视图的数据源。
- ② dispatch 改变 state 的函数，可以理解为推动函数组件渲染的渲染函数。
- ③ initData 有两种情况，第一种情况是非函数，将作为 state 初始化的值。 第二种情况是函数，函数的返回值作为 useState 初始化的值。

```jsx
const [ number , setNumber ] = React.useState(()=>{
       /*  props 中 a = 1 state 为 0-1 随机数 ， a = 2 state 为 1 -10随机数 ， 否则，state 为 1 - 100 随机数   */
       if(props.a === 1) return Math.random() 
       if(props.a === 2) return Math.ceil(Math.random() * 10 )
       return Math.ceil(Math.random() * 100 ) 
});
// 等同于
const [ number , setNumber ] = React.useState(Math.ceil(Math.random() * 100 ));
```

对于 dispatch的参数,也有两种情况：

- 第一种非函数情况，此时将作为新的值，赋予给 state，作为下一次渲染使用;
- 第二种是函数的情况，如果 dispatch 的参数为一个函数，这里可以称它为reducer，reducer 参数，是上一次返回最新的 state，返回值作为新的 state。

```jsx
const [ number , setNumbsr ] = React.useState(0)
const handleClick=()=>{
   // 函数使用 变为4 因为每次函数的参数都是最新的state值
   setNumber((state)=> state + 2)  // state - > 0 + 2 = 2
   setNumber((state)=> state + 2)  // state - > 2 + 2 = 4
  
   // 非函数 最后number 变为2
   setNumber(number + 2); 
   setNumber(number + 2);
  
   // 组合使用 number变为2 setNumber(number + 2) 因为使用的number 还是0，不是setNumber((state)=> state + 2)返回的最新的值
   setNumber((state)=> state + 2);
   setNumber(number + 2);
   
}
```



## props

### props是什么？

对于在 React 应用中写的子组件，无论是函数组件 `FunComponent`，还是类组件 `ClassComponent` ，父组件绑定在它们标签里的属性/方法，最终会变成 props 传递给它们。但是这也不是绝对的，对于一些特殊的属性，比如说 ref 或者 key ，React 会在底层做一些额外的处理。

### 监听props改变

+ 类组件中：`getDerivedStateFromProps`（旧版componentWillReceiveProps）

+ 函数组件中：`useEffect`

### 简单实现Form表单

```tsx
import React from 'react';
import { Button, Input } from 'antd';
import css from './index.less';
import { Bind } from 'lodash-decorators/bind';

interface IProps {
    label: string;
    name: string;
}

class FormItem extends React.Component<IProps, any> {
    public render(): React.ReactNode {
        const { children, name, handleChange, value, label  } = this.props;
        const _onChange = this.props.children.props.onChange;
        const onChange = (e) => {
            _onChange && _onChange(e);
            handleChange(name, e.target.value);
        };
        return (
            <div className={ css.formItem } >
                <span className={ css.label } >{ label }:</span>
                {
                    React.isValidElement(children)
                        ? React.cloneElement(children, { onChange , value })
                        : null
                }
            </div>
        );
    }
}

FormItem.displayName = 'formItem';

interface IProps {
    onFinish?: (values: any) => void;
}

interface IStates {
    formData: any;
}

class Form extends React.Component<IProps, IStates> {
    public item = FormItem;

    constructor() {
        super();
        this.state = {
            formData: {}
        };
    }
    /* 获取重置表单数据 */
    @Bind
    public resetForm() {
        const { formData } = this.state;
        Object.keys(formData).forEach(item => {
            formData[item] = '';
        });
        this.setState({
            formData
        });
    }
    /* 设置表单数据层 */
    @Bind
    public setValue(name, value) {
        this.setState({
            formData: {
                ...this.state.formData,
                [name]: value
            }
        });
    }

    public render(): React.ReactNode {
        const renderChildren = [];
        const { onFinish } = this.props;
        React.Children.forEach(this.props.children, (child, index) => {
            if (child.type.displayName === 'formItem') {
                const { name } = child.props;
                const Children = React.cloneElement(child, {
                    key: name,
                    handleChange: this.setValue,
                    value: this.state.formData[name] || ''
                }, child.props.children);
                renderChildren.push(Children);
            } else if (child.props.htmlType === 'submit' && child.type === Button) {
                const Children = React.cloneElement(child, {
                    onClick: () => {
                        onFinish(this.state.formData);
                    },
                });
                renderChildren.push(Children);
            } else {
              // ...
            }
        });
        return renderChildren;
    }
}
Form.displayName = 'form';

export class CustomForm extends React.Component<any, any> {
    public onChange(e) {
        console.log(e.target.value); // 组件本身的事件
    }
    @Bind
    public onFinish(values) {
        console.log(values);
    }
    public render(): React.ReactNode {
        return (
            <div>
                <Form
                    onFinish={ this.onFinish }
                >
                    <FormItem label={ '姓名' } name={ 'user' }>
                        <Input onChange={ this.onChange } />
                    </FormItem>
                    <FormItem label={ '邮箱' } name={ 'email' }>
                        <Input />
                    </FormItem>
                    <Button htmlType={ 'submit' }>
                        提交
                    </Button>
                </Form>
            </div>
        );
    }
}

```







## diff算法

### 单节点diff

![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1632358405887-61698573-2c2c-4471-9372-e85d87b95cfa.png)

- key和type相同表示可以复用节点
- key不同直接标记删除节点，然后新建节点

- key相同type不同，标记删除该节点和兄弟节点，然后新创建节点



```js
function reconcileSingleElement(
  returnFiber: Fiber, // diff节点的父节点
  currentFirstChild: Fiber | null, // current fiber
  element: ReactElement // 最新的jsx对象
): Fiber {
  const key = element.key;
  let child = currentFirstChild;
  
  // 首先判断是否存在对应DOM节点
  while (child !== null) {
    // 上一次更新存在DOM节点，接下来判断是否可复用

    // 首先比较key是否相同
    if (child.key === key) {

      // key相同，接下来比较type是否相同

      switch (child.tag) {
        // ...省略case
        
        default: {
          if (child.elementType === element.type) {
            var _existing3 = useFiber(child, element.props);
            // ...
            // type相同则表示可以复用
            // 返回复用的fiber
            return _existing3;
          }
          
          // type不同则跳出switch
          break;
        }
      }
      // 代码执行到这里代表：key相同但是type不同
      // 将该fiber及其兄弟fiber标记为删除
      deleteRemainingChildren(returnFiber, child);
      break;
    } else {
      // key不同，将该fiber标记为删除
      deleteChild(returnFiber, child);
    }
    child = child.sibling;
  }

  // 创建新Fiber，并返回 ...省略
}
```

### 多节点diff

`多Diff`的整体逻辑会经历两轮遍历：

第一轮遍历：处理`更新`的节点。

第二轮遍历：处理剩下的不属于`更新`的节点。

#### 第一轮遍历

第一轮遍历步骤如下：

1. `let i = 0`，遍历`newChildren`，将`newChildren[i]`与`oldFiber`比较，判断`DOM节点`是否可复用。
2. 如果可复用，`i++`，继续比较`newChildren[i]`与`oldFiber.sibling`，可以复用则继续遍历。
3. 如果不可复用，分两种情况：
   1. `key`不同导致不可复用，立即跳出整个遍历，**第一轮遍历结束。**
   2.   `key`相同`type`不同导致不可复用，会将`oldFiber`标记为`DELETION`，并继续遍历

4. 如果`newChildren`遍历完（即`i === newChildren.length - 1`）或者`oldFiber`遍历完（即`oldFiber.sibling === null`），跳出遍历，**第一轮遍历结束。**

当遍历结束后，会有两种结果：

**步骤3跳出的遍历**

此时`newChildren`没有遍历完，`oldFiber`也没有遍历完。

```jsx
// 之前
<li key="0">0</li>
<li key="1">1</li>
<li key="2">2</li>
            
// 之后
<li key="0">0</li>
<li key="2">1</li>
<li key="1">2</li>
```

第一个节点可复用，遍历到`key === 2`的节点发现`key`改变，不可复用，跳出遍历，等待第二轮遍历处理。

此时`oldFiber`剩下`key === 1`、`key === 2`未遍历，`newChildren`剩下`key === 2`、`key === 1`未遍历。

**步骤4跳出的遍历**

可能`newChildren`遍历完，或`oldFiber`遍历完，或他们同时遍历完。

```jsx
// 之前
<li key="0" className="a">0</li>
<li key="1" className="b">1</li>
            
// 之后 情况1 —— newChildren与oldFiber都遍历完
<li key="0" className="aa">0</li>
<li key="1" className="bb">1</li>
            
// 之后 情况2 —— newChildren没遍历完，oldFiber遍历完
// newChildren剩下 key==="2" 未遍历
<li key="0" className="aa">0</li>
<li key="1" className="bb">1</li>
<li key="2" className="cc">2</li>
            
// 之后 情况3 —— newChildren遍历完，oldFiber没遍历完
// oldFiber剩下 key==="1" 未遍历
<li key="0" className="aa">0</li>
```



#### 第二轮遍历

对于第一轮遍历的结果，我们分别讨论：

`newChildren`与`oldFiber`同时遍历完

那就是最理想的情况：只需在第一轮遍历进行组件更新。此时`Diff`结束。

---

`newChildren`没遍历完，`oldFiber`遍历完

已有的`DOM节点`都复用了，这时还有新加入的节点，意味着本次更新有新节点插入，我们只需要遍历剩下的`newChildren`为生成的`workInProgress fiber`依次标记`Placement`。

---

`newChildren`遍历完，`oldFiber`没遍历完

意味着本次更新比之前的节点数量少，有节点被删除了。所以需要遍历剩下的`oldFiber`，依次标记`Deletion`。

---

`newChildren`与`oldFiber`都没遍历完

这意味着有节点在这次更新中改变了位置。

由于有节点改变了位置，所以不能再用位置索引`i`对比前后的节点，我们需要使用`key`将同一个节点在两次更新中对应。

为了快速的找到`key`对应的`oldFiber`，将所有还未处理的`oldFiber`存入以`key`为key，`oldFiber`为value的`Map`中。

```javascript
const existingChildren = mapRemainingChildren(returnFiber, oldFiber);
```

接下来遍历剩余的`newChildren`，通过`newChildren[i].key`就能在`existingChildren`中找到`key`相同的`oldFiber`。

**标记节点是否移动**

既然我们的目标是寻找移动的节点，那么我们需要明确：节点是否移动是以什么为参照物？

参照物是：最后一个可复用的节点在`oldFiber`中的位置索引（用变量`lastPlacedIndex`表示）。

由于本次更新中节点是按`newChildren`的顺序排列。在遍历`newChildren`过程中，每个`遍历到的可复用节点`一定是当前遍历到的`所有可复用节点`中**最靠右的那个**，即一定在`lastPlacedIndex`对应的`可复用的节点`在本次更新中位置的后面。

那么只需要比较`遍历到的可复用节点`在上次更新时是否也在`lastPlacedIndex`对应的`oldFiber`后面，就能知道两次更新中这两个节点的相对位置改变没有。

用变量`oldIndex`表示`遍历到的可复用节点`在`oldFiber`中的位置索引。如果`oldIndex < lastPlacedIndex`，代表本次更新该节点需要向右移动。

`lastPlacedIndex`初始为`0`，每遍历一个可复用的节点，如果`oldIndex >= lastPlacedIndex`，则`lastPlacedIndex = oldIndex`。



## 面试题

+ 问：老版本的 React 中，为什么写 jsx 的文件要默认引入 React?

  **答：因为 jsx 在被 babel 编译后，写的 jsx 会变成上述 React.createElement 形式，所以需要引入 React，防止找不到 React 引起报错。**

+ 问：React.createElement 和 React.cloneElement 到底有什么区别呢?

  **答: 一个是用来创建 element 。另一个是用来修改 element，并返回一个新的 React.element 对象。**

+ 问：如果没有在 constructor 的 super 函数中传递 props，那么接下来 constructor 执行上下文中就获取不到 props ，这是为什么呢？

  答：绑定 props 是在父类 Component 构造函数中，执行 super 等于执行 Component 函数，此时 props 没有作为第一个参数传给 super() ，在 Component 中就会找不到 props 参数，从而变成 undefined ，在接下来 constructor 代码中打印 props 为 undefined 



+ 类组件中的 `setState` 和函数组件中的 `useState` 有什么异同？
  + 相同点
    + 首先从原理角度出发，setState和 useState 更新视图，底层都调用了 scheduleUpdateOnFiber 方法，而且事件驱动情况下都有批量更新规则。
  + 不同点
    + 在不是 pureComponent 组件模式下， setState 不会浅比较两次 state 的值，只要调用 setState，在没有其他优化手段的前提下，就会执行更新。但是 useState 中的 dispatchAction 会默认比较两次 state 是否相同，然后决定是否更新组件。
    + setState 有专门监听 state 变化的回调函数 callback，可以获取最新state；但是在函数组件中，只能通过 useEffect 来执行 state 变化引起的副作用。
    + setState 在底层处理逻辑上主要是和老 state 进行合并处理，而 useState 更倾向于重新赋值。
