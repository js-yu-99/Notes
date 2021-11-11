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

#### 3.componentWillMount

在react组件加载完之前立即执行，此时还不能访问到真实的dom结构.

#### 4.componentWillReceiveProps

函数的执行是在更新组件阶段，该生命周期执行驱动是因为父组件更新带来的 props 修改，但是只要父组件触发 render 函数，调用 React.createElement 方法，那么 props 就会被重新创建，生命周期 componentWillReceiveProps 就会执行了。这就解释了即使 props 没变，该生命周期也会执行。

- componentWillReceiveProps 可以用来监听父组件是否执行 render 。
- componentWillReceiveProps 可以用来接受 props 改变，组件可以根据props改变，来决定是否更新 state ，因为可以访问到 this ， 所以可以在异步成功回调(接口请求数据)改变 state 。这个是 getDerivedStateFromProps 不能实现的。

#### 5.componentWillUpdate

在更新发生前被立即调用

#### 6.render

所谓 render 函数，就是 jsx 的各个元素被 React.createElement 创建成 React element 对象的形式。一次 render 的过程，就是创建 React.element 元素的过程。

#### 7.getSnapshotBeforeUpdate

> **getSnapshotBeforeUpdate**(prevProps,preState){}

两个参数：

- prevProps更新前的props ；
- preState更新前的state；

**获取更新前的快照**，可以进一步理解为 获取更新前 DOM 的状态

作用：

- getSnapshotBeforeUpdate 这个生命周期意义就是配合componentDidUpdate 一起使用，计算形成一个 snapShot 传递给 componentDidUpdate 。保存一次更新前的信息。

#### 8.componentDidUpdate

> **componentDidUpdate**(prevProps, prevState, snapshot) {}

三个参数：

- prevProps 更新之前的 props ；
- prevState 更新之前的 state ；
- snapshot 为 getSnapshotBeforeUpdate 返回的快照，可以是更新前的 DOM 信息。

#### 9.componentDidMount

componentDidMount 生命周期执行时机和 componentDidUpdate 一样，一个是在**初始化**，一个是**组件更新**。此时 DOM 已经创建完。

作用：

- 可以做一些关于 DOM 操作，比如基于 DOM 的事件监听器。

#### 10.shouldComponentUpdate

> **shouldComponentUpdate**(newProps,newState,nextContext){}

shouldComponentUpdate 三个参数，第一个参数新的 props ，第二个参数新的 state ，第三个参数新的 context 。

```js
shouldComponentUpdate(newProps,newState){
    if(newProps.a !== this.props.a ){ /* props中a属性发生变化 渲染组件 */
        return true
    }else if(newState.b !== this.props.b ){ /* state 中b属性发生变化 渲染组件 */
        return true
    }else{ /* 否则组件不渲染 */
        return false
    }
}
```

- 这个生命周期，一般用于性能优化，shouldComponentUpdate 返回值决定是否重新渲染的类组件。需要重点关注的是第二个参数 newState ，如果有 getDerivedStateFromProps 生命周期 ，它的返回值将合并到 newState ，供 shouldComponentUpdate 使用。

#### 11.componentWillUnmount

componentWillUnmount 是组件销毁阶段唯一执行的生命周期，主要做一些收尾工作，比如清除一些可能造成内存泄漏的定时器，延时器，或者是一些事件监听器。

### 函数组件生命周期替代方案

#### 1 useEffect 和 useLayoutEffect

**useEffect**

```js
useEffect(()=>{
    return destory
},dep)
```

useEffect 第一个参数 callback, 返回的 destory ， destory 作为下一次callback执行之前调用，用于清除上一次 callback 产生的副作用。

第二个参数作为依赖项，是一个数组，可以有多个依赖项，依赖项改变，执行上一次callback 返回的 destory ，和执行新的 effect 第一个参数 callback 。

```js
useEffect(() => {
  console.log(1);
  return () => {
    console.log(2);
  };
});
useEffect(() => {
  console.log(3);
  return () => {
    console.log(4);
  };
});
// 2 4 1 3
```

对于 useEffect 执行， React 处理逻辑是采用异步调用 ，对于每一个 effect 的 callback， React 会向 `setTimeout`回调函数一样，放入任务队列，等到主线程任务完成，DOM 更新，js 执行完成，视图绘制完毕，才执行。所以 effect 回调函数不会阻塞浏览器绘制视图。

**useLayoutEffect**

useLayoutEffect 和 useEffect 不同的地方是采用了同步执行，那么和useEffect有什么区别呢？

- 首先 useLayoutEffect 是在DOM 绘制之前，这样可以方便修改 DOM ，这样浏览器只会绘制一次，如果修改 DOM 布局放在 useEffect ，那 useEffect 执行是在浏览器绘制视图之后，接下来又改 DOM ，就可能会导致浏览器再次回流和重绘。而且由于两次绘制，视图上可能会造成闪现突兀的效果。
- useLayoutEffect callback 中代码执行会阻塞浏览器绘制。

**一句话概括如何选择 useEffect 和 useLayoutEffect ：修改 DOM ，改变布局就用 useLayoutEffect ，其他情况就用 useEffect 。**



#### 2 componentDidMount 替代方案

```js
React.useEffect(()=>{
    /* 请求数据 ， 事件监听 ， 操纵dom */
},[])  /* dep = [] */
```

这样当前 effect 没有任何依赖项，也就只有初始化执行一次。

- `componentDidMount()`完全等价于`useLayoutEffect( fn , [ ] )`，但是不等价于`useEffect( fn , [ ] )`。

#### 3 componentWillUnmount 替代方案

```js
React.useEffect(()=>{
  /* 请求数据 ， 事件监听 ， 操纵dom ， 增加定时器，延时器 */
  return function componentWillUnmount(){
    /* 解除事件监听器 ，清除定时器，延时器 */
  }
},[])/* dep = [] */
```

在 componentDidMount 的前提下，useEffect 第一个函数的返回函数，可以作为 componentWillUnmount 使用。

#### 4 componentWillReceiveProps 代替方案

说 useEffect 代替 componentWillReceiveProps 着实有点牵强。

- 首先因为二者的执行阶段根本不同，一个是在render阶段，一个是在commit阶段。
- 其次 **useEffect 会初始化执行一次**，但是 componentWillReceiveProps 只有组件更新 props 变化的时候才会执行。

```js
React.useEffect(()=>{
    console.log('props变化：componentWillReceiveProps')
},[ props ])
```

此时依赖项就是 props，props 变化，执行此时的 useEffect 钩子。

```js
React.useEffect(()=>{
    console.log('props中number变化：componentWillReceiveProps')
},[ props.number ]) /* 当前仅当 props中number变化，执行当前effect钩子 */
```

useEffect 还可以针对 props 的某一个属性进行追踪。此时的依赖项为 props 的追踪属性。如上述代码，只有 props 中 number 变化，执行 effect 。

#### 5 componentDidUpdate 替代方案

useEffect 和 componentDidUpdate 在执行时期虽然有点差别，useEffect 是异步执行，componentDidUpdate 是同步执行 ，但都是在 commit 阶段 。但是向上面所说 useEffect 会默认执行一次，而 componentDidUpdate 只有在组件更新完成后执行。

```js
React.useEffect(()=>{
    console.log('组件更新完成：componentDidUpdate ')     
}) /* 没有 dep 依赖项 */
```

注意此时useEffect没有第二个参数。

没有第二个参数，那么每一次执行函数组件，都会执行该 effect。



## Ref

对于 Ref ，应该分成两个部分去分析，第一个部分是 **Ref 对象的创建**，第二个部分是 **React 本身对Ref的处理**。两者不要混为一谈，所谓 Ref 对象的创建，就是通过 React.createRef 或者 React.useRef 来创建一个 Ref 原始对象。而 React 对 Ref 处理，主要指的是对于标签中 ref 属性，React 是如何处理以及 React 转发 Ref 。

### Ref对象创建

**什么是 ref 对象**，所谓 ref 对象就是用 `createRef` 或者 `useRef` 创建出来的对象，一个标准的 ref 对象应该是如下的样子：

```js
{
    current:null , // current指向ref对象获取到的实际内容，可以是dom元素，组件实例，或者其他。
}
```

**①类组件React.createRef**

```jsx
import React from 'react';

export class RefDemo extends React.PureComponent<any, any> {
    public refDom = React.createRef();
    public componentDidMount(): void {
        console.log(this.refDom.current); // <div>
    }

    public render(): React.ReactNode {
        return (
            <div ref={ this.refDom }>
                reco
            </div>
        );
    }
}
```

```js
function createRef() {
  var refObject = {
    current: null
  };
  return refObject;
}
```

createRef 只做了一件事，就是创建了一个对象，对象上的 current 属性，用于保存通过 ref 获取的 DOM 元素，组件实例等。 createRef 一般用于类组件创建 Ref 对象，可以将 Ref 对象绑定在类组件实例上，这样更方便后续操作 Ref。

注意：不要在函数组件中使用 createRef，否则会造成 Ref 对象内容丢失等情况。

**②函数组件 useRef**

第二种方式就是函数组件创建 Ref ，可以用 hooks 中的 useRef 来达到同样的效果。

```jsx
const RefDemo = (props) => {
    const refDom = useRef();
    useEffect(() => {
        console.log(refDom.current); // <div>
    });
    return (
        <div ref={ refDom }>
            reco
        </div>
    );
};
```

useRef 底层逻辑是和 createRef 差不多，就是 ref 保存位置不相同，类组件有一个实例 instance 能够维护像 ref 这种信息，但是由于函数组件每次更新都是一次新的开始，所有变量重新声明，所以 useRef 不能像 createRef 把 ref 对象直接暴露出去，如果这样每一次函数组件执行就会重新声明 Ref，此时 ref 就会随着函数组件执行被重置，这就解释了在函数组件中为什么不能用 createRef 的原因。

为了解决这个问题，hooks 和函数组件对应的 fiber 对象建立起关联，将 useRef 产生的 ref 对象挂到函数组件对应的 fiber 上，函数组件每次执行，只要组件不被销毁，函数组件对应的 fiber 对象一直存在，所以 ref 等信息就会被保存下来。



### React对Ref属性的处理-标记ref

- **① Ref属性是一个字符串。**

  ```jsx
  class Demo extends React.Component{
      public componentDidMount(){
         console.log(this.refs); // {currentDom: XX, currentComInstance: XX}
      }
      public render() {
        return <div>
          <div ref="currentDom" >字符串模式获取元素或组件</div>
          <Children ref="currentComInstance" />
      </div>
      }
  }
  ```

- **② Ref 属性是一个函数。**

  ```jsx
  class Demo extends React.Component{
      public currentDom = null
      public currentComponentInstance = null
      public componentDidMount(){
          console.log(this.currentDom)
          console.log(this.currentComponentInstance)
      }
      public render() {
        return <div>
          		<div ref={(node)=> this.currentDom = node }  >Ref模式获取元素或组件</div>
          		<Children ref={(node) => this.currentComponentInstance = node  }  />
      		</div>
      }
  }
  ```

+ **③ Ref属性是一个ref对象。**

  ```jsx
  class Demo extends React.Component{
      public currentDom = React.createRef(null)
      public currentComponentInstance = React.createRef(null)
      public componentDidMount(){
          console.log(this.currentDom)
          console.log(this.currentComponentInstance)
      }
      public render() {
        return <div>
           <div ref={ this.currentDom }  >Ref对象模式获取元素或组件</div>
          <Children ref={ this.currentComponentInstance }  />
    	 </div>
      }
  }
  ```

  

### ref高阶用法

#### 1 forwardRef 转发 Ref

forwardRef 的初衷就是解决 ref 不能跨层级捕获和传递的问题。 forwardRef 接受了父级元素标记的 ref 信息，并把它转发下去，使得子组件可以通过 props 来接受到上一层级或者是更上层级的ref

**① 场景一：跨层级获取**

比如想要通过标记子组件 ref ，来获取孙组件的某一 DOM 元素，或者是组件实例。

> 场景：想要在 GrandFather 组件通过标记 ref ，来获取孙组件 Son 的组件实例。

```jsx
// 孙组件
function Son (props){
    const { grandRef } = props
    return <div>
        <div> i am alien </div>
        <span ref={grandRef} >这个是想要获取元素</span>
    </div>
}
// 父组件
class Father extends React.Component{
    constructor(props){
        super(props)
    }
    render(){
        return <div>
            <Son grandRef={this.props.grandRef}  />
        </div>
    }
}
const NewFather = React.forwardRef((props,ref)=> <Father grandRef={ref}  {...props} />)
// 爷组件
class GrandFather extends React.Component{
    constructor(props){
        super(props)
    }
    node = null 
    componentDidMount(){
        console.log(this.node) // span #text 这个是想要获取元素
    }
    render(){
        return <div>
            <NewFather ref={(node)=> this.node = node } />
        </div>
    }
}
```

**② 场景二:合并转发ref**

通过 forwardRef 转发的 ref 不要理解为只能用来直接获取组件实例，DOM 元素，也可以用来传递合并之后的自定义的 ref 

> 场景：想通过Home绑定ref，来获取子组件Index的实例index，dom元素button，以及孙组件Form的实例

```jsx
// 表单组件
class Form extends React.Component{
    render(){
       return <div>{...}</div>
    }
}
// index 组件
class Index extends React.Component{ 
    componentDidMount(){
        const { forwardRef } = this.props
        forwardRef.current={
            form:this.form,      // 给form组件实例 ，绑定给 ref form属性 
            index:this,          // 给index组件实例 ，绑定给 ref index属性 
            button:this.button,  // 给button dom 元素，绑定给 ref button属性 
        }
    }
    form = null
    button = null
    render(){
        return <div   > 
          <button ref={(button)=> this.button = button }  >点击</button>
          <Form  ref={(form) => this.form = form }  />  
      </div>
    }
}
const ForwardRefIndex = React.forwardRef(( props,ref )=><Index  {...props} forwardRef={ref}  />)
// home 组件
export default function Home(){
    const ref = useRef(null)
     useEffect(()=>{
         console.log(ref.current)
     },[])
    return <ForwardRefIndex ref={ref} />
}
```

**③ 场景三：高阶组件转发**

如果通过高阶组件包裹一个原始类组件，就会产生一个问题，如果高阶组件 HOC 没有处理 ref ，那么由于高阶组件本身会返回一个新组件，所以当使用 HOC 包装后组件的时候，标记的 ref 会指向 HOC 返回的组件，而并不是 HOC 包裹的原始类组件，为了解决这个问题，forwardRef 可以对 HOC 做一层处理。

```jsx
function HOC(Component){
  class Wrap extends React.Component{
     render(){
        const { forwardedRef ,...otherprops  } = this.props
        return <Component ref={forwardedRef}  {...otherprops}  />
     }
  }
  return  React.forwardRef((props,ref)=> <Wrap forwardedRef={ref} {...props} /> ) 
}
class Index extends React.Component{
  render(){
    return <div>hello,world</div>
  }
}
const HocIndex =  HOC(Index)
export default ()=>{
  const node = useRef(null)
  useEffect(()=>{
    console.log(node.current)  /* Index 组件实例  */ 
  },[])
  return <div><HocIndex ref={node}  /></div>
}
```

#### 2 ref实现组件通信

**① 类组件 ref**

对于类组件可以通过 ref 直接获取组件实例，实现组件通信。

```jsx
export class Item extends React.PureComponent {
    constructor() {
        super();
        this.state = {
            age: 20
        };
    }
    public render(): React.ReactNode {
        return (
            <div>
                { this.state.age }
            </div>
        );
    }
}

export class RefDemo extends React.PureComponent<any, any> {
    public refDom = React.createRef();

    @Bind
    public setAge() {
        this.refDom.current.setState({
            age: this.refDom.current.state.age + 20
        });
    }

    public render(): React.ReactNode {
        console.log('render'); // 只在父组件执行一次
        return (
            <div>
                reco
                <Item ref={ this.refDom } />
                <button onClick={ () => {
                    this.setAge();
                } }>修改age</button>
            </div>
        );
    }
}
```

**② 函数组件 forwardRef + useImperativeHandle**

对于函数组件，本身是没有实例的，但是 React Hooks 提供了，useImperativeHandle 一方面第一个参数接受父组件传递的 ref 对象，另一方面第二个参数是一个函数，函数返回值，作为 ref 对象获取的内容。

useImperativeHandle 接受三个参数：

+ 第一个参数 ref : 接受 forWardRef 传递过来的 ref 。

+ 第二个参数 createHandle ：处理函数，返回值作为暴露给父组件的 ref 对象。

+ 第三个参数 deps :依赖项 deps，依赖项更改形成新的 ref 对象。

![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1633696235583-cb0b77c2-d7b9-4ea1-9a1b-7eee64abc4fe.png)



```
// 子组件
const Son = (props, ref) => {
    const inputRef = useRef(null);
    const [ inputValue , setInputValue ] = useState('');
    useImperativeHandle(ref, () => {
        const handleRefs = {
            onFocus() {              /* 声明方法用于聚焦input框 */
                inputRef.current.focus();
            },
            onChangeValue(value) {   /* 声明方法用于改变input的值 */
                setInputValue(value);
            }
        };
        return handleRefs;
    }, []);
    return <div>
        <input placeholder="请输入内容"  ref={inputRef}  value={inputValue} />
    </div>;
};

const ForwarSon = React.forwardRef(Son);

export class RefDemo extends React.PureComponent<any, any> {
    public refDom = React.createRef();

    public handerClick() {
        const { onFocus , onChangeValue } = this.refDom.current;
        onFocus(); // 让子组件的输入框获取焦点
        onChangeValue('let us learn React!'); // 让子组件input
    }

    public render(): React.ReactNode {
        console.log('render');
        return (
            <div>
                reco
                <ForwarSon ref={ this.refDom } />
                <button onClick={ () => {
                    this.handerClick();
                } }>修改age</button>
            </div>
        );
    }
}
```

**3 函数组件缓存数据**

函数组件每一次 render ，函数上下文会重新执行，那么有一种情况就是，在执行一些事件方法改变数据或者保存新数据的时候，有没有必要更新视图，有没有必要把数据放到 state 中。如果视图层更新不依赖想要改变的数据，那么 state 改变带来的更新效果就是多余的。这时候更新无疑是一种性能上的浪费。

这种情况下，useRef 就派上用场了，useRef 可以创建出一个 ref 原始对象，只要组件没有销毁，ref 对象就一直存在，那么完全可以把一些不依赖于视图更新的数据储存到 ref 对象中。这样做的好处有两个：

- 第一个能够直接修改数据，不会造成函数组件冗余的更新作用。
- 第二个 useRef 保存数据，如果有 useEffect ，useMemo 引用 ref 对象中的数据，无须将 ref 对象添加成 dep 依赖项，因为 useRef 始终指向一个内存空间，**所以这样一点好处是可以随时访问到变化后的值。**

### ref原理揭秘

用回调函数方式处理 Ref ，**如果点击一次按钮，会打印几次 console.log ？**

```jsx
export class RefDemo extends React.PureComponent<any, any> {
    public refDom = React.createRef();
    public node = null;

    constructor() {
        super();
        this.state = {
            num: 0
        };
    }

    public render(): React.ReactNode {
        return (
            <div>
                <div ref={ (ref) => {
                    this.node = ref;
                    console.log('此时的参数是什么：', this.node );
                } } >121212</div>
                <div>{ this.state.num }</div>
                <button onClick={() => this.setState({ num: this.state.num + 1  }) } >点击</button>
            </div>
        );
    }
}
```

多次点击后的打印：第一次打印为 null ，第二次才是 div

![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1633697074728-c5581e15-90b1-4190-be16-2350b221a585.png)



#### ref 执行时机和处理逻辑

对于整个 Ref 的处理，都是在 commit 阶段发生的。 commit 阶段会进行真正的 Dom 操作，此时 ref 就是用来获取真实的 DOM 以及组件实例的，所以需要 commit 阶段处理。

但是对于 Ref 处理函数，React 底层用两个方法处理：**commitDetachRef** 和 **commitAttachRef** ，上述两次 console.log 一次为 null，一次为div 就是分别调用了上述的方法。

这两次一次在 DOM 更新之前，一次在 DOM 更新之后。



- 第一阶段：一次更新中，在 commit 的 mutation 阶段, 执行commitDetachRef，commitDetachRef 会清空之前ref值，使其重置为 null。
- 第二阶段：DOM 更新阶段，这个阶段会根据不同的 effect 标签，真实的操作 DOM 。

- 第三阶段：layout 阶段，在更新真实元素节点之后，此时需要更新 ref 。

```js
function commitDetachRef(current: Fiber) {
  const currentRef = current.ref;
  if (currentRef !== null) {
    if (typeof currentRef === 'function') { /* function 和 字符串获取方式。 */
      currentRef(null); 
    } else {   /* Ref对象获取方式 */
      currentRef.current = null;
    }
  }
}

function commitAttachRef(finishedWork: Fiber) {
  const ref = finishedWork.ref;
  if (ref !== null) {
    const instance = finishedWork.stateNode;
    let instanceToUse;
    switch (finishedWork.tag) {
      case HostComponent: //元素节点 获取元素
        instanceToUse = getPublicInstance(instance);
        break;
      default:  // 类组件直接使用实例
        instanceToUse = instance;
    }
    if (typeof ref === 'function') {
      ref(instanceToUse);  //* function 和 字符串获取方式。 */
    } else {
      ref.current = instanceToUse; /* ref对象方式 */
    }
  }
}
```

这一阶段，主要判断 ref 获取的是组件还是 DOM 元素标签，如果 DOM 元素，就会获取更新之后最新的 DOM 元素。上面流程中讲了三种获取 ref 的方式。 如果是字符串 ref="node" 或是 函数式 ref={(node)=> this.node = node } 会执行 ref 函数，重置新的 ref 。

如果是 ref 对象方式。

```js
node = React.createRef() <div ref={ node } ></div> 
```

会更新 ref 对象的 current 属性。达到更新 ref 对象的目的。

#### Ref 的处理特性

React 被 ref 标记的 fiber，那么每一次 fiber 更新都会调用 **commitDetachRef** 和 **commitAttachRef** 更新 Ref 吗 ？

**答案是否定的，只有在 ref 更新的时候，才会调用如上方法更新 ref ，究其原因还要从如上两个方法的执行时期说起**

**更新ref**

**commitDetachRef** **调用时机**

```js
function commitMutationEffects(){
     if (effectTag & Ref) {
      const current = nextEffect.alternate;
      if (current !== null) {
        commitDetachRef(current);
      }
    }
}
```

**commitAttachRef** **调用时机**

```js
function commitLayoutEffects(){
     if (effectTag & Ref) {
      commitAttachRef(nextEffect);
    }
}
```

从上可以清晰的看到只有含有 Ref tag 的时候，才会执行更新 ref。

```js
function markRef(current: Fiber | null, workInProgress: Fiber) {
  const ref = workInProgress.ref;
  if (
    (current === null && ref !== null) ||      // 初始化的时候
    (current !== null && current.ref !== ref)  // ref 指向发生改变
  ) {
    workInProgress.effectTag |= Ref;
  }
}
```

首先 markRef 方法执行在两种情况下：

- 第一种就是类组件的更新过程中。
- 第二种就是更新 HostComponent 的时候。



markRef 会在以下两种情况下给 effectTag 标记 Ref，只有标记了 Ref tag 才会有后续的 commitAttachRef 和 commitDetachRef 流程。（ current 为当前调和的 fiber 节点 ）

- 第一种 current === null && ref !== null：就是在 fiber 初始化的时候，第一次 ref 处理的时候，是一定要标记 Ref 的。
- 第二种 current !== null && current.ref !== ref：就是 fiber 更新的时候，但是 ref 对象的指向变了。

用回调函数方式处理 Ref ，**如果点击一次按钮，会打印几次 console.log？**这个问题中，为什么每次都会打印ref？

因为每一次更新的时候，都给ref赋值了新的函数，那么markRef中就会判断成 current.ref !== ref，所以就会重新打 Ref 标签，那么在 commit 阶段，就会更新 ref 执行 ref 回调函数了。

**卸载 ref**

```js
this.state.isShow && <div ref={()=>this.node = node} >元素节点</div>
```

在一次更新的时候，改变 isShow 属性，使之由 true 变成了 false， 那么 div 元素会被卸载，那么 ref 会怎么处理呢？

被卸载的 fiber 会被打成 Deletion effect tag ，然后在 commit 阶段会进行 commitDeletion 流程。对于有 ref 标记的 ClassComponent （类组件） 和 HostComponent （元素），会统一走 safelyDetachRef 流程，这个方法就是用来卸载 ref。改变 isShow 属性，使之由 true 变成了 false， 那么 div 元素会被卸载，那么 ref 会怎么处理呢？

```js
function safelyDetachRef(current) {
  const ref = current.ref;
  if (ref !== null) {
    if (typeof ref === 'function') {  // 函数式 ｜ 字符串
        ref(null)
    } else {
      ref.current = null;  // ref 对象
    }
  }
}
```

- 对于字符串 ref="dom" 和函数类型 ref={(node)=> this.node = node } 的 ref，会执行传入 null 置空 ref 。
- 对于 ref 对象类型，会清空 ref 对象上的 current 属性。

借此完成卸载 ref 流程。



## Context（新版）

提供上下文模式是为了解决 props 传递的问题。

**提供者永远要在消费者上层**

```
const ThemeContext = React.createContext(null) //
const ThemeProvider = ThemeContext.Provider  //提供者
const ThemeConsumer = ThemeContext.Consumer // 订阅消费者
```

createContext 接受一个参数，作为初始化 context 的内容，返回一个context 对象，Context 对象上的 Provider 作为提供者，Context 对象上的 Consumer 作为消费者。



### 新版本提供者

```
const ThemeProvider = ThemeContext.Provider  //提供者
export default function ProviderDemo(){
    const [ contextValue , setContextValue ] = React.useState({  color:'#ccc', background:'pink' })
    return <div>
        <ThemeProvider value={ contextValue } > 
            <Son />
        </ThemeProvider>
    </div>
}
```

provider 作用有两个：

- value 属性传递 context，供给 Consumer 使用。
- value 属性改变，ThemeProvider 会让消费 Provider value 的组件重新渲染。



### 新版本消费者

对于新版本想要获取 context 的消费者，React 提供了3种形式

#### 类组件之contextType 方式

React v16.6 提供了 contextType 静态属性，用来获取上面 Provider 提供的 value 属性。

```
const ThemeContext = React.createContext(null)
// 类组件 - contextType 方式
class ConsumerDemo extends React.Component{
   render(){
       const { color,background } = this.context
       return <div style={{ color,background } } >消费者</div> 
   }
}
ConsumerDemo.contextType = ThemeContext

const Son = ()=> <ConsumerDemo />
```

- 类组件的静态属性上的 contextType 属性，指向需要获取的 context（ demo 中的 ThemeContext ），就可以方便获取到最近一层 Provider 提供的 contextValue 值。
- 这种方式只适用于类组件。



#### 函数组件之 useContext 方式

React hooks 提供了 useContext。

```
const ThemeContext = React.createContext(null)
// 函数组件 - useContext方式
function ConsumerDemo(){
    const  contextValue = React.useContext(ThemeContext) /*  */
    const { color,background } = contextValue
    return <div style={{ color,background } } >消费者</div> 
}
const Son = ()=> <ConsumerDemo />
```

useContext 接受一个参数，就是想要获取的 context ，返回一个 value 值，就是最近的 provider 提供 contextValue 值。



#### 订阅者之 Consumer 方式

```
const ThemeConsumer = ThemeContext.Consumer // 订阅消费者

function ConsumerDemo(props){
    const { color,background } = props
    return <div style={{ color,background } } >消费者</div> 
}
const Son = () => (
    <ThemeConsumer>
       { /* 将 context 内容转化成 props  */ }
       { (contextValue)=> <ConsumerDemo  {...contextValue}  /> }
    </ThemeConsumer>
) 
```

- Consumer 订阅者采取 render props 方式，接受最近一层 provider 中value 属性，作为 render props 函数的参数，可以将参数取出来，作为 props 混入 ConsumerDemo 组件，说白了就是 context 变成了 props。



### 动态context

在 Provider 里 value 的改变，会使引用**contextType **,**useContext**消费该 context 的组件重新 render ，同样会使 Consumer 的 children 函数重新执行，与前两种方式不同的是 Consumer 方式，当 context 内容改变的时候，不会让引用 Consumer 的父组件重新更新。

**如何阻止 Provider value 改变造成的 children 不必要的渲染？**

使用React.memo，pureComponent对子组件props进行浅比较处理。

> **const** Son = React.memo(()=> <ConsumerDemo />)  



### 其他的API

+ #### displayName

  + context 对象接受一个名为 `displayName` 的 property，类型为字符串。React DevTools 使用该字符串来确定 context 要显示的内容。

    ```js
    const MyContext = React.createContext(/* 初始化内容 */);
    MyContext.displayName = 'MyDisplayName';
    
    <MyContext.Provider> // "MyDisplayName.Provider" 在 DevTools 中
    <MyContext.Consumer> // "MyDisplayName.Consumer" 在 DevTools 中
    ```



### Demo

```jsx
import React, {useContext} from 'react';
import { HomeOutlined, SettingFilled, SmileOutlined, SyncOutlined, LoadingOutlined } from '@ant-design/icons';

const ThemeContext = React.createContext(null); // 主题颜色Context
const theme = { // 主题颜色
    dark: { color: '#1890ff' , background: '#1890ff', border: '1px solid blue' , type: 'dark',  },
    light: { color: '#fc4838' , background: '#fc4838', border: '1px solid pink' , type: 'light'  }
};

/* input输入框 - useContext 模式 */
const Input = (props) => {
    const  { color, border } = useContext(ThemeContext);
    const { label , placeholder } = props;
    return <div>
        <label style={{ color }} >{ label }</label>
        <input className="input" placeholder={placeholder}  style={{ border }} />
    </div>;
};

/* 容器组件 -  Consumer模式 */
const Box = (props) => {
    return <ThemeContext.Consumer>
        { (themeContextValue) => {
            const { border, color } = themeContextValue;
            return <div className="context_box" style={{ border, color }} >
                { props.children }
            </div>;
        } }
    </ThemeContext.Consumer>;
};

const Checkbox = (props) => {
    const { label, name, onChange } = props;
    const { type, color } = React.useContext(ThemeContext);
    return <div className="checkbox"  onClick={onChange} >
        <label htmlFor="name" > {label} </label>
        <input type="checkbox" id={name} value={type} name={name} checked={ type === name }  style={{ color } } />
    </div>;
};

// contextType 模式
class App extends React.PureComponent {
    public static contextType = ThemeContext;
    public render() {
        const { border, setTheme, color, background} = this.context;
        return <div className="context_app" style={{ border , color }}  >
            <div className="context_change_theme"   >
                <span> 选择主题： </span>
                <Checkbox label="light"  name="light" onChange={ () => setTheme(theme.light) }  />
                <Checkbox label="dark" name="dark"  onChange={ () => setTheme(theme.dark) }   />
            </div>
            <div className="box_content" >
                <Box >
                    <Input label="姓名："  placeholder="请输入姓名"  />
                    <Input label="age："  placeholder="请输入年龄"  />
                    <button className="searchbtn" style={ { background } } >确定</button>
                    <button className="concellbtn" style={ { color } } >取消</button>
                </Box>
                <Box >
                    <HomeOutlined  twoToneColor={ color } />
                    <SettingFilled twoToneColor={ color }  />
                    <SmileOutlined twoToneColor={ color }  />
                    <SyncOutlined spin  twoToneColor={ color }  />
                    <SmileOutlined twoToneColor={ color }  rotate={180} />
                    <LoadingOutlined twoToneColor={ color }   />
                </Box>
                <Box >
                    <div className="person_des" style={{ color: '#fff' , background }}  >
                        I am alien  <br/>
                        let us learn React!
                    </div>
                </Box>
            </div>
        </div>;
    }
}

export default function() {
    const [ themeContextValue , setThemeContext ] = React.useState(theme.dark);
    /* 传递颜色主题 和 改变主题的方法 */
    return <ThemeContext.Provider value={ { ...themeContextValue, setTheme: setThemeContext  } } >
        <App/>
    </ThemeContext.Provider>;
}

```

![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1635746192698-f341abb6-8bcb-4d5b-bbd8-a41f9fae24ba.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1635746210234-ed4651f3-5f5d-4289-96d3-9fb903425580.png)



## 渲染控制

### render阶段

render的作用是根据一次更新中产生的新状态值，通过React.createElement，替换成新的状态，得到新的React element对象，新的element对象上，保存了最新状态值。createElement会产生一个全新的props。

接下来，React会调和由render函数产生children，将子代element变成fiber，将props变成pendingProps，至此当前组件更新变比。然后如果children是组件，会继续重复上一步，直到全部fiber调和完毕。完成render阶段。



### React控制render的方法

- 第一种就是从父组件直接隔断子组件的渲染，经典的就是 memo，缓存 element 对象。
- 第二种就是组件从自身来控制是否 render ，比如：PureComponent ，shouldComponentUpdate 。

#### useMemo

```js
const cacheSomething = useMemo(create,deps)
```

- `create`：第一个参数为一个函数，函数的返回值作为缓存值，如上 demo 中把 Children 对应的 element 对象，缓存起来。
- `deps`： 第二个参数为一个数组，存放当前 useMemo 的依赖项，在函数组件下一次执行的时候，会对比 deps 依赖项里面的状态，是否有改变，如果有改变重新执行 create ，得到新的缓存值。
- `cacheSomething`：返回值，执行 create 的返回值。如果 deps 中有依赖项改变，返回的重新执行 create 产生的值，否则取上一次缓存值。

**useMemo原理：**

useMemo 会记录上一次执行 create 的返回值，并把它绑定在函数组件对应的 fiber 对象上，只要组件不销毁，缓存值就一直存在，但是 deps 中如果有一项改变，就会重新执行 create ，返回值作为新的值记录到 fiber 对象上。

**useMemo应用场景：**

- 可以缓存 element 对象，从而达到按条件渲染组件，优化性能的作用。
- 如果组件中不期望每次 render 都重新计算一些值,可以利用 useMemo 把它缓存起来。
- 可以把函数和属性缓存起来，作为 PureComponent 的绑定方法，或者配合其他Hooks一起使用。



#### PureComponent

- 对于 props ，PureComponent 会浅比较 props 是否发生改变，再决定是否渲染组件，所以只有点击 numberA 才会促使组件重新渲染。
- 对于 state ，如上也会浅比较处理，当上述触发 ‘ state 相同情况’ 按钮时，组件没有渲染。
- 浅比较只会比较基础数据类型，对于引用类型，比如 demo 中 state 的 obj ，单纯的改变 obj 下属性是不会促使组件更新的，因为浅比较两次 obj 还是指向同一个内存空间，想要解决这个问题也容易，浅拷贝就可以解决。这样就是重新创建了一个 obj ，所以浅比较会不相等，组件就会更新了。

pureComponent原型链上存在属性isPureReactComponent

```js
function checkShouldComponentUpdate(){
     if (typeof instance.shouldComponentUpdate === 'function') {
         return instance.shouldComponentUpdate(newProps,newState,nextContext)  /* shouldComponentUpdate 逻辑 */
     } 
    if (ctor.prototype && ctor.prototype.isPureReactComponent) {
        return  !shallowEqual(oldProps, newProps) || !shallowEqual(oldState, newState)
    }
}
```

- isPureReactComponent 就是判断当前组件是不是纯组件的，如果是 PureComponent 会浅比较 props 和 state 是否相等。
- 还有一点值得注意的就是 shouldComponentUpdate 的权重，会大于 PureComponent。

shallowEqual 浅比较流程：

- 第一步，首先会直接比较新老 props 或者新老 state 是否相等。如果相等那么不更新组件。
- 第二步，判断新老 state 或者 props ，有不是对象，或者为 null 的，那么直接返回 false ，更新组件。
- 第三步，通过 Object.keys 将新老 props 或者新老 state 的属性名 key 变成数组，判断数组的长度是否相等，如果不相等，证明有属性增加或者减少，那么更新组件。
- 第四步，遍历老 props 或者老 state ，判断对应的新 props 或新 state ，有没有与之对应并且相等的（这个相等是浅比较），如果有一个不对应或者不相等，那么直接返回 false ，更新组件。

**PureComponent注意事项**

1. 避免使用箭头函数。不要给是 PureComponent 子组件绑定箭头函数，因为父组件每一次 render ，如果是箭头函数绑定的话，都会重新生成一个新的箭头函数， PureComponent 对比新老 props 时候，因为是新的函数，所以会判断不想等，而让组件直接渲染，PureComponent 作用终会失效。
2. PureComponent 的父组件是函数组件的情况，绑定函数要用 useCallback 或者 useMemo 处理。这种情况还是很容易发生的，就是在用 class + function 组件开发项目的时候，如果父组件是函数，子组件是 PureComponent ，那么绑定函数要小心，因为函数组件每一次执行，如果不处理，还会声明一个新的函数，所以 PureComponent 对比同样会失效，



#### shouldComponentUpdate

shouldComponentUpdate 可以根据传入的新的 props 和 state ，或者 newContext 来确定是否更新组件



#### React.memo

React.memo 可作为一种容器化的控制渲染方案，可以对比 props 变化，来决定是否渲染组件，首先先来看一下 memo 的基本用法。React.memo 接受两个参数，第一个参数 Component 原始组件本身，第二个参数 compare 是一个函数，可以根据一次更新中 props 是否相同决定原始组件是否重新渲染。

memo的几个特点是：

- React.memo: 第二个参数 返回 true 组件不渲染 ， 返回 false 组件重新渲染。和 shouldComponentUpdate 相反，shouldComponentUpdate : 返回 true 组件渲染 ， 返回 false 组件不渲染。
- memo 当二个参数 compare 不存在时，会用**浅比较原则**处理 props ，相当于仅比较 props 版本的 pureComponent 。
- memo 同样适合类组件和函数组件。

被 memo 包裹的组件，element 会被打成 `REACT_MEMO_TYPE` 类型的 element 标签，在 element 变成 fiber 的时候， fiber 会被标记成 MemoComponent 的类型。

- memo 主要逻辑是

  

  - 通过 memo 第二个参数，判断是否执行更新，如果没有那么第二个参数，那么以浅比较 props 为 diff 规则。如果相等，当前 fiber 完成工作，停止向下调和节点，所以被包裹的组件即将不更新。
  - memo 可以理解为包了一层的高阶组件，它的阻断更新机制，是通过控制下一级 children ，也就是 memo 包装的组件，是否继续调和渲染，来达到目的的。

  

  #### 打破渲染限制

  - 1 forceUpdate。类组件更新如果调用的是 forceUpdate 而不是 setState ，会跳过 PureComponent 的浅比较和 shouldComponentUpdate 自定义比较。其原理是组件中调用 forceUpdate 时候，全局会开启一个 hasForceUpdate 的开关。当组件更新的时候，检查这个开关是否打开，如果打开，就直接跳过 shouldUpdate 。
  - 2 context穿透，上述的几种方式，都不能本质上阻断 context 改变，而带来的渲染穿透，所以开发者在使用 Context 要格外小心，既然选择了消费 context ，就要承担 context 改变，带来的更新作用。

  #### 渲染控制流程图

  

  ![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1636348038819-c8f95cc8-7ef7-41ac-b871-dac0bc0e1550.png)

  

  #### 什么时候需要注意控制渲染

  - 第一种情况数据可视化的模块组件（展示了大量的数据），这种情况比较小心因为一次更新，可能伴随大量的 diff ，数据量越大也就越浪费性能，所以对于数据展示模块组件，有必要采取 memo ， shouldComponentUpdate 等方案控制自身组件渲染。
  - 第二种情况含有大量表单的页面，React 一般会采用受控组件的模式去管理表单数据层，表单数据层完全托管于 props 或是 state ，而用户操作表单往往是频繁的，需要频繁改变数据层，所以很有可能让整个页面组件高频率 render 。

  - 第三种情况就是越是靠近 app root 根组件越值得注意，根组件渲染会波及到整个组件树重新 render ，子组件 render ，一是浪费性能，二是可能执行 useEffect ，componentWillReceiveProps 等钩子，造成意想不到的情况发生。



## 渲染调优

### Suspense 异步渲染

Suspense 是 React 提出的一种同步的代码来实现异步操作的方案。Suspense 让组件‘等待’异步操作，异步请求结束后在进行组件的渲染，也就是所谓的异步渲染，但是这个功能目前还在实验阶段，相信不久这种异步渲染的方式就能和大家见面了。

**Suspense 用法**

Suspense 是组件，有一个 fallback 属性，用来代替当 Suspense 处于 loading 状态下渲染的内容，Suspense 的 children 就是异步组件。多个异步组件可以用 Suspense 嵌套使用。

```
// 子组件
function UserInfo() {
  // 获取用户数据信息，然后再渲染组件。
  const user = getUserInfo();
  return <h1>{user.name}</h1>;
}
// 父组件
export default function Index(){
    return <Suspense fallback={<h1>Loading...</h1>}>
        <UserInfo/>
    </Suspense>
}
```

- Suspense 包裹异步渲染组件 UserInfo ，当 UserInfo 处于数据加载状态下，展示 Suspense 中 fallback 的内容。

传统模式：挂载组件-> 请求数据 -> 再渲染组件。
异步模式：请求数据-> 渲染组件。

异步渲染相比传统数据交互相比好处：

- 不再需要 componentDidMount 或 useEffect 配合做数据交互，也不会因为数据交互后，改变 state 而产生的二次更新作用。
- 代码逻辑更简单，清晰。



**Suspense原理：** 

Suspense 在执行内部可以通过 try{}catch{} 方式捕获异常，这个异常通常是一个 Promise ，可以在这个 Promise 中进行数据请求工作，Suspense 内部会处理这个 Promise ，Promise 结束后，Suspense 会再一次重新 render 把数据渲染出来，达到异步渲染的效果。

![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1636363679271-eaf35ccb-5484-4a69-ac9e-97ce43ae51e8.png)



### React.lazy 动态加载（懒加载）

React.lazy 接受一个函数，这个函数需要动态调用 import() 。它必须返回一个 Promise ，该 Promise 需要 resolve 一个 default export 的 React 组件。

```
const LazyComponent = React.lazy(() => import('./test.js'))

export default function Index(){
   return <Suspense fallback={<div>loading...</div>} >
       <LazyComponent />
   </Suspense>
}
```

- 用 React.lazy 动态引入 test.js 里面的组件，配合 Suspense 实现动态加载组件效果。**这样很利于代码分割，不会让初始化的时候加载大量的文件。**



**React.lazy原理：**

lazy 内部模拟一个 promiseA 规范场景。完全可以理解 React.lazy 用 Promise 模拟了一个请求数据的过程，但是请求的结果不是数据，而是一个动态的组件。下一次渲染就直接渲染这个组件，所以是 React.lazy 利用 Suspense **接收 Promise ，执行 Promise ，然后再渲染**这个特性做到动态加载的。

```
function lazy(ctor){
    return {
         $$typeof: REACT_LAZY_TYPE,
         _payload:{
            _status: -1,  //初始化状态
            _result: ctor,
         },
         _init:function(payload){
             if(payload._status===-1){ /* 第一次执行会走这里  */
                const ctor = payload._result;
                const thenable = ctor();
                payload._status = Pending;
                payload._result = thenable;
                thenable.then((moduleObject)=>{
                    const defaultExport = moduleObject.default;
                    resolved._status = Resolved; // 1 成功状态
                    resolved._result = defaultExport;/* defaultExport 为我们动态加载的组件本身  */ 
                })
             }
            if(payload._status === Resolved){ // 成功状态
                return payload._result;
            }
            else {  //第一次会抛出Promise异常给Suspense
                throw payload._result; 
            }
         }
    }
}
```

整个流程是，React.lazy 包裹的组件会标记 REACT_LAZY_TYPE 类型的 element，在调和阶段会变成 LazyComponent 类型的 fiber ，React 对 LazyComponent 会有单独的处理逻辑：

- 第一次渲染首先会执行 init 方法，里面会执行 lazy 的第一个函数，得到一个Promise，绑定 Promise.then 成功回调，回调里得到将要渲染组件 defaultExport ，当第二个 if 判断的时候，因为此时状态不是 Resolved ，所以会走 else ，抛出异常 Promise，抛出异常会让当前渲染终止。
- 这个异常 Promise 会被 Suspense 捕获到，Suspense 会处理 Promise ，Promise 执行成功回调得到 defaultExport（将想要渲染组件），然后 Susponse 发起第二次渲染，第二次 init 方法已经是 Resolved 成功状态，那么直接返回 result 也就是真正渲染的组件。这时候就可以正常渲染组件了。

![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1636364074927-1fa7ffee-c258-47e7-92f3-27fe67c8f8f7.png)

### 渲染错误边界

#### componentDidCatch

componentDidCatch 可以捕获异常，它接受两个参数：

- 1 error —— 抛出的错误。
- 2 info —— 带有 componentStack key 的对象，其中包含有关组件引发错误的栈信息。

componentDidCatch 中可以再次触发 setState，来降级UI渲染，componentDidCatch() 会在commit阶段被调用，因此允许执行副作用。

```
public componentDidCatch(error) {
  this.sendErrorInfo(error);
  this.setState({ hasError: true });
}

public render(): React.ReactNode {
  if (this.state.hasError) {
    // You can render any custom fallback UI
    return <Empty
             image={ Empty.PRESENTED_IMAGE_SIMPLE }
             description={
        <div>抱歉，组件出现未知错误，请点击刷新按钮重新加载。<br/>
          <Button onClick={ this.onClick } type={'primary'} style={{marginTop: '10px'}}>
            刷新
          </Button>
        </div>
      }
             />;
  }
  return <>
    { this.props.children }
    </>;
}
```

componentDidCatch 作用：

- 可以调用 setState 促使组件渲染，并做一些错误拦截功能。
- 监控组件，发生错误，上报错误日志。



#### static getDerivedStateFromError

React更期望用 getDerivedStateFromError 代替 componentDidCatch 用于处理渲染异常的情况。getDerivedStateFromError 是静态方法，内部不能调用 setState。getDerivedStateFromError 返回的值可以合并到 state，作为渲染使用。用 getDerivedStateFromError 解决如上的情况。

```
state={
  hasError:false
}  
static getDerivedStateFromError(){
  return { hasError:true }
}
render(){  
  /* 如上 */
}
```

注意事项： 如果存在 getDerivedStateFromError 生命周期钩子，那么将不需要 componentDidCatch 生命周期再降级 ui。



### 多节点Diff中key的重要性

### diff children流程

**第一步：遍历新 children ，复用 oldFiber**

> react-reconciler/src/ReactChildFiber.js

```js
function reconcileChildrenArray(){
    /* 第一步  */
    for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {  
        if (oldFiber.index > newIdx) {
            nextOldFiber = oldFiber;
            oldFiber = null;
        } else {
            nextOldFiber = oldFiber.sibling;
        }
        const newFiber = updateSlot(returnFiber,oldFiber,newChildren[newIdx],expirationTime,);
        if (newFiber === null) { break }
        // ..一些其他逻辑
        }  
        if (shouldTrackSideEffects) {  // shouldTrackSideEffects 为更新流程。
            if (oldFiber && newFiber.alternate === null) { /* 找到了与新节点对应的fiber，但是不能复用，那么直接删除老节点 */
                deleteChild(returnFiber, oldFiber);
            }
        }
    }
}
```

- 第一步对于 React.createElement 产生新的 child 组成的数组，首先会遍历数组，因为 fiber 对于同一级兄弟节点是用 sibling 指针指向，所以在遍历children 遍历，sibling 指针同时移动，找到与 child 对应的 oldFiber 。
- 然后通过调用 updateSlot ，updateSlot 内部会判断当前的 tag 和 key 是否匹配，如果匹配复用老 fiber 形成新的 fiber ，如果不匹配，返回 null ，此时 newFiber 等于 null 。
- 如果是处于更新流程，找到与新节点对应的老 fiber ，但是不能复用 `alternate === null `，那么会删除老 fiber 。

**第二步：统一删除oldfiber**

```js
if (newIdx === newChildren.length) {
    deleteRemainingChildren(returnFiber, oldFiber);
    return resultingFirstChild;
}
```

- 第二步适用于以下情况，当第一步结束完 `newIdx === newChildren.length` 此时证明所有 newChild 已经全部被遍历完，那么剩下没有遍历 oldFiber 也就没有用了，那么调用 deleteRemainingChildren 统一删除剩余 oldFiber 。

情况一：节点删除

- **oldChild: A B C D**
- **newChild: A B**

A , B 经过第一步遍历复制完成，那么 newChild 遍历完成，此时 C D 已经没有用了，那么统一删除 C D。

**第三步：统一创建newFiber**

```js
if(oldFiber === null){
   for (; newIdx < newChildren.length; newIdx++) {
       const newFiber = createChild(returnFiber,newChildren[newIdx],expirationTime,)
       // ...
   }
}
```

- 第三步适合如下的情况，当经历过第一步，oldFiber 为 null ， 证明 oldFiber 复用完毕，那么如果还有新的 children ，说明都是新的元素，只需要调用 createChild 创建新的 fiber 。

情况二：节点增加

- **oldChild: A B**
- **newChild: A B C D**

A B 经过第一步遍历复制完，oldFiber 没有可以复用的了，那么直接创建 C D。

**第四步：针对发生移动和更复杂的情况**

```js
const existingChildren = mapRemainingChildren(returnFiber, oldFiber);
for (; newIdx < newChildren.length; newIdx++) {
    const newFiber = updateFromMap(existingChildren,returnFiber)
    /* 从mapRemainingChildren删掉已经复用oldFiber */
}
```

- mapRemainingChildren 返回一个 map ，map 里存放剩余的老的 fiber 和对应的 key (或 index )的映射关系。
- 接下来遍历剩下没有处理的 Children ，通过 updateFromMap ，判断 mapRemainingChildren 中有没有可以复用 oldFiber ，如果有，那么复用，如果没有，新创建一个 newFiber 。
- 复用的 oldFiber 会从 mapRemainingChildren 删掉。

情况三：节点位置改变

- **oldChild: A B C D**
- **newChild: A B D C**

如上 A B 在第一步被有效复用，第二步和第三步不符合，直接进行第四步，C D 被完全复用，existingChildren 为空。

**第五步：删除剩余没有复用的oldFiber**

```js
if (shouldTrackSideEffects) {
    /* 移除没有复用到的oldFiber */
    existingChildren.forEach(child => deleteChild(returnFiber, child));
}
```

最后一步，对于没有复用的 oldFiber ，统一删除处理。

情况四：复杂情况(删除 + 新增 + 移动)

- **oldChild: A B C D**
- **newChild: A E D B**

首先 A 节点，在第一步被复用，接下来直接到第四步，遍历 newChild ，E被创建，D B 从 existingChildren 中被复用，existingChildren 还剩一个 C 在第五步会删除 C ，完成整个流程。

### 关于diffChild思考和key的使用

- 1 React diffChild 时间复杂度 O(n^3) 优化到 O(n)。
- 2 React key 最好选择唯一性的id，如上述流程，如果选择 Index 作为 key ，如果元素发生移动，那么从移动节点开始，接下来的 fiber 都不能做得到合理的复用。 index 拼接其他字段也会造成相同的效果。





## 事件原理

**React 为什么要写出一套自己的事件系统？**

首先，对于不同的浏览器，对事件存在不同的兼容性，React 想实现一个兼容全浏览器的框架， 为了实现这个目标就需要创建一个兼容全浏览器的事件系统，以此抹平不同浏览器的差异。

其次，v17 之前 React 事件都是绑定在 document 上，v17 之后 React 把事件绑定在应用对应的容器 container 上，将事件绑定在同一容器统一管理，防止很多事件直接绑定在原生的 DOM 元素上。造成一些不可控的情况。由于不是绑定在真实的 DOM 上，所以 React 需要模拟一套事件流：事件捕获-> 事件源 -> 事件冒泡，也包括重写一下事件源对象 event 。



### 冒泡阶段和捕获阶段

```js
export default function Index(){
    const handleClick=()=>{ console.log('模拟冒泡阶段执行') } 
    const handleClickCapture = ()=>{ console.log('模拟捕获阶段执行') }
    return <div>
        <button onClick={ handleClick  } onClickCapture={ handleClickCapture }  >点击</button>
    </div>
}
```

- 冒泡阶段：开发者正常给 React 绑定的事件比如 onClick，onChange，默认会在模拟冒泡阶段执行。
- 捕获阶段：如果想要在捕获阶段执行可以将事件后面加上 Capture 后缀，比如 onClickCapture，onChangeCapture。



### 阻止冒泡

React 中如果想要阻止事件向上冒泡，可以用 `e.stopPropagation()` 。

```js
export default function Index(){
    const handleClick=(e)=> {
        e.stopPropagation() /* 阻止事件冒泡，handleFatherClick 事件讲不在触发 */
    }
    const handleFatherClick=()=> console.log('冒泡到父级')
    return <div onClick={ handleFatherClick } >
        <div onClick={ handleClick } >点击</div>
    </div>
}
```

- React 阻止冒泡和原生事件中的写法差不多，当如上 handleClick上 阻止冒泡，父级元素的 handleFatherClick 将不再执行，但是底层原理完全不同，接下来会讲到其功能实现。



### 阻止默认行为

React 阻止默认行为和原生的事件也有一些区别。

**原生事件：** `e.preventDefault()` 和 `return false` 可以用来阻止事件默认行为，由于在 React 中给元素的事件并不是真正的事件处理函数。**所以导致 return false 方法在 React 应用中完全失去了作用。**

**React事件** 在React应用中，可以用 e.preventDefault() 阻止事件默认行为，这个方法并非是原生事件的 preventDefault ，由于 React 事件源 e 也是独立组建的，所以 preventDefault 也是单独处理的。



### 事件合成



React 事件系统可分为三个部分：

- 第一个部分是事件合成系统，初始化会注册不同的事件插件。
- 第二个就是在一次渲染过程中，对事件标签中事件的收集，向 container 注册事件。
- 第三个就是一次用户交互，事件触发，到事件执行一系列过程。



___



## 问与答

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

+ 问：当 props 不变的前提下， PureComponent 组件能否阻止 componentWillReceiveProps 执行？
  + 答案是否定的，componentWillReceiveProps 生命周期的执行，和纯组件没有关系，纯组件是在 componentWillReceiveProps 执行之后浅比较 props 是否发生变化。所以 PureComponent 下不会阻止该生命周期的执行。

+ 问：React.useEffect 回调函数 和 componentDidMount / componentDidUpdate 执行时机有什么区别 ？

  答：useEffect 对 React 执行栈来看是异步执行的，而 componentDidMount / componentDidUpdate 是同步执行的，useEffect代码不会阻塞浏览器绘制。在时机上 ，componentDidMount / componentDidUpdate 和 useLayoutEffect 更类似。



+ **问**：context 与 props 和 react-redux 的对比？

  **答**： context解决了：

  1、解决了 props 需要每一层都手动添加 props 的缺陷。2、解决了改变 value ，组件全部重新渲染的缺陷。

  react-redux 就是通过 Provider 模式把 redux 中的 store 注入到组件中的。

