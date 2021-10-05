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



## 生命周期

React 有两个重要阶段，`render` 阶段和 `commit` 阶段，React 在render阶段会深度遍历 React fiber 树，目的就是发现不同( diff )，不同的地方就是接下来需要更新的地方，对于变化的组件，就会执行 render 函数。在一次调和过程完毕之后，就到了commit 阶段，commit 阶段会创建修改真实的 DOM 节点。

```js
/* workloop React 处理类组件的主要功能方法 */
function updateClassComponent(){
    let shouldUpdate
    const instance = workInProgress.stateNode // stateNode 是 fiber 指向 类组件实例的指针。
     if (instance === null) { // instance 为组件实例,如果组件实例不存在，证明该类组件没有被挂载过，那么会走初始化流程
        constructClassInstance(workInProgress, Component, nextProps); // 组件实例将在这个方法中被new。
        mountClassInstance(  workInProgress,Component, nextProps,renderExpirationTime ); //初始化挂载组件流程
        shouldUpdate = true; // shouldUpdate 标识用来证明 组件是否需要更新。
     }else{  
        shouldUpdate = updateClassInstance(current, workInProgress, Component, nextProps, renderExpirationTime) // 更新组件流程
     }
     if(shouldUpdate){
        nextChildren = instance.render(); /* 执行render函数 ，得到子节点 */
        reconcileChildren(current,workInProgress,nextChildren,renderExpirationTime) /* 继续调和子节点 */
     }
}
```

几个重要概念：

- ① instance 类组件对应实例。
- ② workInProgress 树，当前正在render的 fiber 树 ，一次更新中，React 会自上而下深度遍历子代 fiber ，如果遍历到一个 fiber ，会把当前 fiber 指向 workInProgress。

- ③ current 树，在初始化更新中，current = null ，在第一次 fiber render之后，会将 workInProgress 树赋值给 current 树。React 来用workInProgress 和 current 来确保一次更新中，快速构建，并且状态不丢失。
- ④ Component 就是项目中的 class 组件。

- ⑤ nextProps 作为组件在一次更新中新的 props 。
- ⑥ renderExpirationTime 作为下一次渲染的过期时间。



在组件实例上可以通过` _reactInternals` 属性来访问组件对应的 fiber 对象。在 fiber 对象上，可以通过 stateNode 来访问当前 fiber 对应的组件实例。

![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1633424460156-70755565-5e77-4d2b-989d-1ed02228e447.png)

### 初始化阶段

**① constructor 执行**

在 mount 阶段，首先执行的 constructClassInstance 函数，用来实例化 React 组件。在实例化组件之后，会调用 mountClassInstance 组件初始化。

```js
function mountClassInstance(workInProgress,ctor,newProps,renderExpirationTime){
    const instance = workInProgress.stateNode;
     const getDerivedStateFromProps = ctor.getDerivedStateFromProps;
  if (typeof getDerivedStateFromProps === 'function') { /* ctor 就是我们写的类组件，获取类组件的静态防范 */
     const partialState = getDerivedStateFromProps(nextProps, prevState); /* 这个时候执行 getDerivedStateFromProps 生命周期 ，得到将合并的state */
     const memoizedState = partialState === null || partialState === undefined ? prevState : Object.assign({}, prevState, partialState); // 合并state
     workInProgress.memoizedState = memoizedState;
     instance.state = workInProgress.memoizedState; /* 将state 赋值给我们实例上，instance.state  就是我们在组件中 this.state获取的state*/
  }
  if(typeof ctor.getDerivedStateFromProps !== 'function' &&   typeof instance.getSnapshotBeforeUpdate !== 'function' && typeof instance.componentWillMount === 'function' ){
      instance.componentWillMount(); /* 当 getDerivedStateFromProps 和 getSnapshotBeforeUpdate 不存在的时候 ，执行 componentWillMount*/
  }
}
```

**② getDerivedStateFromProps 执行**

在初始化阶段，getDerivedStateFromProps 是第二个执行的生命周期，值得注意的是它是从 ctor 类上直接绑定的静态方法，传入 props ，state 。 返回值将和之前的 state 合并，作为新的 state ，传递给组件实例使用。

```js
static getDerivedStateFromProps(nextProps, prevState) {
    const {type} = nextProps;
    // 当传入的type发生变化的时候，更新state
    if (type !== prevState.type) {
        return {
            type,
        };
    }
    // 否则，对于state不进行任何操作
    return null;
}
```

这个生命周期函数是为了替代`componentWillReceiveProps`存在的

**③ componentWillMount 执行**

如果存在 getDerivedStateFromProps 和 getSnapshotBeforeUpdate 就不会执行生命周期componentWillMount。

**④ render 函数执行**

到此为止 mountClassInstancec 函数完成，但是上面 updateClassComponent 函数， 在执行完 mountClassInstancec 后，执行了 render 渲染函数，形成了 children ， 接下来 React 调用 reconcileChildren 方法深度调和 children 。

**⑤componentDidMount执行**

一旦 React 调和完所有的 fiber 节点，就会到 commit 阶段，在组件初始化 commit 阶段，会调用 componentDidMount 生命周期。

```js
function commitLifeCycles(finishedRoot,current,finishedWork){
     switch (finishedWork.tag){                             /* fiber tag 在第一节讲了不同fiber类型 */
        case ClassComponent: {                              /* 如果是 类组件 类型 */
             const instance = finishedWork.stateNode        /* 类实例 */
             if(current === null){                          /* 类组件第一次调和渲染 */
                instance.componentDidMount() 
             }else{                                         /* 类组件更新 */
                instance.componentDidUpdate(prevProps,prevState，instance.__reactInternalSnapshotBeforeUpdate); 
             }
        }
     }
}
```

 componentDidMount 执行时机 和 componentDidUpdate 执行时机是相同的 ，只不过一个是针对初始化，一个是针对组件再更新。到此初始化阶段，生命周期执行完毕。

执行顺序：constructor -> getDerivedStateFromProps / componentWillMount -> render -> componentDidMount

![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1633425911011-d0607d4a-a943-456e-b505-bd5c6e4fa4bf.png)

### 更新阶段

```
function updateClassInstance(current,workInProgress,ctor,newProps,renderExpirationTime){
    const instance = workInProgress.stateNode; // 类组件实例
    const hasNewLifecycles =  typeof ctor.getDerivedStateFromProps === 'function'  // 判断是否具有 getDerivedStateFromProps 生命周期
    if(!hasNewLifecycles && typeof instance.componentWillReceiveProps === 'function' ){
         if (oldProps !== newProps || oldContext !== nextContext) {     // 浅比较 props 不相等
            instance.componentWillReceiveProps(newProps, nextContext);  // 执行生命周期 componentWillReceiveProps 
         }
    }
    let newState = (instance.state = oldState);
    if (typeof getDerivedStateFromProps === 'function') {
        ctor.getDerivedStateFromProps(nextProps,prevState)  /* 执行生命周期getDerivedStateFromProps  ，逻辑和mounted类似 ，合并state  */
        newState = workInProgress.memoizedState;
    }   
    let shouldUpdate = true
    if(typeof instance.shouldComponentUpdate === 'function' ){ /* 执行生命周期 shouldComponentUpdate 返回值决定是否执行render ，调和子节点 */
        shouldUpdate = instance.shouldComponentUpdate(newProps,newState,nextContext,);
    }
    if(shouldUpdate){
        if (typeof instance.componentWillUpdate === 'function') {
            instance.componentWillUpdate(); /* 执行生命周期 componentWillUpdate  */
        }
    }
    return shouldUpdate
}
```

**①执行生命周期 componentWillReceiveProps**

首先判断 getDerivedStateFromProps 生命周期是否存在，如果不存在就执行componentWillReceiveProps生命周期。传入该生命周期两个参数，分别是 newProps 和 nextContext 。

**②执行生命周期 getDerivedStateFromProps**

接下来执行生命周期getDerivedStateFromProps， 返回的值用于合并state，生成新的state。

**③执行生命周期 shouldComponentUpdate**

接下来执行生命周期shouldComponentUpdate，传入新的 props ，新的 state ，和新的 context ，返回值决定是否继续执行 render 函数，调和子节点。这里应该注意一个问题，getDerivedStateFromProps 的返回值可以作为新的 state ，传递给 shouldComponentUpdate 。

**④执行生命周期 componentWillUpdate**

接下来执行生命周期 componentWillUpdate。updateClassInstance 方法到此执行完毕了。

**⑤执行 render 函数**

接下来会执行 render 函数，得到最新的 React element 元素。然后继续调和子节点。

**⑥执行 getSnapshotBeforeUpdate**

getSnapshotBeforeUpdate 的执行也是在 commit 阶段，commit 阶段细分为 before Mutation( DOM 修改前)，Mutation ( DOM 修改)，Layout( DOM 修改后) 三个阶段，getSnapshotBeforeUpdate 发生在before Mutation 阶段，生命周期的返回值，将作为第三个参数 __reactInternalSnapshotBeforeUpdate 传递给 componentDidUpdate 。

**⑦执行 componentDidUpdate**

接下来执行生命周期 componentDidUpdate ，此时 DOM 已经修改完成。可以操作修改之后的 DOM 。到此为止更新阶段的生命周期执行完毕。

![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1633428636068-25d10889-3c96-4586-a9de-845cb78080da.png)



更新阶段对应的生命周期的执行顺序：

componentWillReceiveProps( props 改变) / getDerivedStateFromProp -> shouldComponentUpdate -> componentWillUpdate -> render -> getSnapshotBeforeUpdate -> componentDidUpdate

### 销毁阶段

**①执行生命周期 componentWillUnmount**

在一次调和更新中，如果发现元素被移除，就会打对应的 Deletion 标签 ，然后在 commit 阶段就会调用 componentWillUnmount 生命周期，接下来统一卸载组件以及 DOM 元素。

componentWillUnmount 生命周期，接下来统一卸载组件以及 DOM 元素。

![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1633429917241-218d40ce-94de-4944-9c1d-2b04e5ae2a2c.png)



### React 各阶段生命周期能做些什么

#### 1 constructor

React 在不同时期抛出不同的生命周期钩子，也就意味这这些生命周期钩子的使命。上面讲过 constructor 在类组件创建实例时调用，而且初始化的时候执行一次，所以可以在 constructor 做一些初始化的工作。

```js
constructor(props){
    super(props)        // 执行 super ，别忘了传递props,才能在接下来的上下文中，获取到props。
    this.state={       //① 可以用来初始化state，比如可以用来获取路由中的
        name:'alien'
    }
    this.handleClick = this.handleClick.bind(this) /* ② 绑定 this */
    this.handleInputChange = debounce(this.handleInputChange , 500) /* ③ 绑定防抖函数，防抖 500 毫秒 */
    const _render = this.render
    this.render = function(){
        return _render.bind(this)  /* ④ 劫持修改类组件上的一些生命周期 */
    }
}
/* 点击事件 */
handleClick(){ /* ... */ }
/* 表单输入 */
handleInputChange(){ /* ... */ }
```

constructor 作用：

- 初始化 state ，比如可以用来截取路由中的参数，赋值给 state 。
- 对类组件的事件做一些处理，比如绑定 this ， 节流，防抖等。

- 对类组件进行一些必要生命周期的劫持，渲染劫持，这个功能更适合反向继承的HOC ，在 HOC 环节，会详细讲解反向继承这种模式。

#### 2 getDerivedStateFromProps

> getDerivedStateFromProps(nextProps,prevState) 

两个参数：

- nextProps 父组件新传递的 props ;
- prevState 组件在此次更新前的 state 。

getDerivedStateFromProps 方法作为类的静态属性方法执行，内部是访问不到 this 的，它更趋向于纯函数，从源码中就能够体会到 React 对该生命周期定义为取缔 componentWillMount 和 componentWillReceiveProps 。

如果把 getDerivedStateFromProps 英文分解 get ｜ Derived | State ｜ From ｜ Props 翻译  **得到 派生的 state 从 props 中** ，正如它的名字一样，这个生命周期用于，在初始化和更新阶段，接受父组件的 props 数据， 可以对 props 进行格式化，过滤等操作，返回值将作为新的 state 合并到 state 中，供给视图渲染层消费。

从源码中可以看到，只要组件更新，就会执行 getDerivedStateFromProps，不管是 props 改变，还是 setState ，或是 forceUpdate 。

```jsx
static getDerivedStateFromProps(newProps){
    const { type } = newProps
    switch(type){
        case 'fruit' : 
        return { list:['苹果','香蕉','葡萄' ] } /* ① 接受 props 变化 ， 返回值将作为新的 state ，用于 渲染 或 传递给s houldComponentUpdate */
        case 'vegetables':
        return { list:['菠菜','西红柿','土豆']}
    }
}
render(){
    return <div>{ this.state.list.map((item)=><li key={item} >{ item  }</li>) }</div>
}
```

getDerivedStateFromProps 作用：

- 代替 `componentWillMount` 和 `componentWillReceiveProps`
- 组件初始化或者更新时，将 props 映射到 state。

- 返回值与 state 合并完，可以作为 `shouldComponentUpdate` 第二个参数 newState ，可以判断是否渲染组件。(`getDerivedStateFromProps` 和 `shouldComponentUpdate` 两者没有必然联系)



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



