## React理念

官方说明：

> 我们认为，React 是用 JavaScript 构建**`快速响应`**的大型 Web 应用程序的首选方式。它在 Facebook 和 Instagram 上表现优秀。

关键是实现`快速响应`

制约快速响应的因素为：

+ CPU瓶颈
+ IO瓶颈

### CPU瓶颈

JS可以操作DOM，`GUI渲染线程`与`JS线程`是互斥的。所以**JS脚本执行**和**浏览器布局、绘制**不能同时执行。

在每16.6ms时间内，需要完成如下工作：**JS脚本执行 -----  样式布局 ----- 样式绘制**

当JS执行时间过长，超出了16.6ms，这次刷新就没有时间执行**样式布局**和**样式绘制**了。

**React的解决办法：**

> 在浏览器每一帧的时间中，预留一些时间给JS线程，`React`利用这部分时间更新组件（预留的初始时间为5ms）

当预留的时间不够用时，`React`将线程控制权交还给浏览器使其有时间渲染UI，`React`则等待下一帧时间到来继续被中断的工作。

> 这种将长任务分拆到每一帧中，像蚂蚁搬家一样一次执行一小段任务的操作，被称为`时间切片`（time slice）

所以，解决`CPU瓶颈`的关键是实现`时间切片`，而`时间切片`的关键是：将**同步的更新**变为**可中断的异步更新**。

### IO瓶颈

如何在`网络延迟`客观存在的情况下，减少用户对`网络延迟`的感知？

将人机交互研究的结果整合到真实的UI中。为此，`React`实现了**Suspense**功能及配套的`hook`-**useDeferredValue**



## React15架构

React15架构可以分为两层：

- Reconciler（协调器）—— 负责找出变化的组件
- Renderer（渲染器）—— 负责将变化的组件渲染到页面上

### Reconciler（协调器）

在`React`中可以通过`this.setState`、`this.forceUpdate`、`ReactDOM.render`等API触发更新。

每当有更新发生时，**Reconciler**会做如下工作：

- 调用函数组件、或class组件的`render`方法，将返回的JSX转化为虚拟DOM
- 将虚拟DOM和上次更新时的虚拟DOM对比
- 通过对比找出本次更新中变化的虚拟DOM
- 通知**Renderer**将变化的虚拟DOM渲染到页面上

### Renderer（渲染器）

由于`React`支持跨平台，所以不同平台有不同的**Renderer**。前端最熟悉的是负责在浏览器环境渲染的**Renderer** —— ReactDOM。

除此之外，还有：

- **ReactNative**渲染器，渲染App原生组件
- **ReactTest**渲染器，渲染出纯Js对象用于测试
- **ReactArt **渲染器，渲染到Canvas, SVG 或 VML (IE8)

在每次更新发生时，**Renderer**接到**Reconciler**通知，将变化的组件渲染在当前宿主环境。

---

### 架构缺点

React15会通过递归的方式更新子组件，由于递归执行，所以更新一旦开始，中途就无法中断。当层级很深时，递归更新时间超过了16ms，用户交互就会变得卡顿。



## 新的架构（16、17）

React16架构可以分为三层：

- Scheduler（调度器）—— 调度任务的优先级，高优任务优先进入**Reconciler**
- Reconciler（协调器）—— 负责找出变化的组件
- Renderer（渲染器）—— 负责将变化的组件渲染到页面上

### Scheduler（调度器）

既然我们以浏览器是否有剩余时间作为任务中断的标准，那么我们需要一种机制，当浏览器有剩余时间时通知我们。

其实部分浏览器已经实现了这个API，这就是requestIdleCallback。但是由于以下因素，`React`放弃使用：

- 浏览器兼容性
- 触发频率不稳定，受很多因素影响。比如当我们的浏览器切换tab后，之前tab注册的`requestIdleCallback`触发的频率会变得很低

基于以上原因，`React`实现了功能更完备的`requestIdleCallback`polyfill，这就是**Scheduler**。除了在空闲时触发回调的功能外，**Scheduler**还提供了多种调度优先级供任务设置。

### Reconciler（协调器）

更新工作从递归变成了可以中断的循环过程。每次循环都会调用`shouldYield`判断当前是否有剩余时间。

```js
/** @noinline */
function workLoopConcurrent() {
  // Perform work until Scheduler asks us to yield
  while (workInProgress !== null && !shouldYield()) {
    workInProgress = performUnitOfWork(workInProgress);
  }
}
```

在React16中，**Reconciler**与**Renderer**不再是交替工作。当**Scheduler**将任务交给**Reconciler**后，**Reconciler**会为变化的虚拟DOM打上代表增/删/更新的标记，类似这样：

```js
export const Placement = /*             */ 0b0000000000010;
export const Update = /*                */ 0b0000000000100;
export const PlacementAndUpdate = /*    */ 0b0000000000110;
export const Deletion = /*              */ 0b0000000001000;
```

整个**Scheduler**与**Reconciler**的工作都在内存中进行。只有当所有组件都完成**Reconciler**的工作，才会统一交给**Renderer**。

### Renderer（渲染器）

**Renderer**根据**Reconciler**为虚拟DOM打的标记，同步执行对应的DOM操作。



## 代数效应

`代数效应`是`函数式编程`中的一个概念，用于将`副作用`从`函数`调用中分离。

React中的代数效应体现为`hooks`



## Fiber

`Fiber`并不是计算机术语中的新名词，他的中文翻译叫做`纤程`，与进程（Process）、线程（Thread）、协程（Coroutine）同为程序执行过程。

`React Fiber`可以理解为：

`React`内部实现的一套状态更新机制。支持任务不同`优先级`，可中断与恢复，并且恢复后可以复用之前的`中间状态`。

其中每个任务更新单元为`React Element`对应的`Fiber节点`。

### Fiber结构

```js
function FiberNode(
  tag: WorkTag,
  pendingProps: mixed,
  key: null | string,
  mode: TypeOfMode,
) {
  // 作为静态数据结构的属性
  // Fiber对应组件的类型 Function/Class/Host...
  this.tag = tag;
  // key属性
  this.key = key;
  // 大部分情况同type，某些情况不同，比如FunctionComponent使用React.memo包裹
  this.elementType = null;
  // 对于 FunctionComponent，指函数本身，对于ClassComponent，指class，对于HostComponent，指DOM节点tagName
  this.type = null;
  // Fiber对应的真实DOM节点
  this.stateNode = null;

  // 用于连接其他Fiber节点形成Fiber树
  // 指向父级Fiber节点
	this.return = null;
	// 指向子Fiber节点
	this.child = null;
	// 指向右边第一个兄弟Fiber节点
	this.sibling = null;
  this.index = 0;

  this.ref = null;

  // 作为动态的工作单元的属性
  this.pendingProps = pendingProps;
  this.memoizedProps = null;
  this.updateQueue = null;
  this.memoizedState = null;
  this.dependencies = null;

  this.mode = mode;
	// 保存本次更新会造成的DOM操作
  this.effectTag = NoEffect;
  this.nextEffect = null;

  this.firstEffect = null;
  this.lastEffect = null;

  // 调度优先级相关
  this.lanes = NoLanes;
  this.childLanes = NoLanes;

  // 指向该fiber在另一次更新时对应的fiber
  this.alternate = null;
}
```

### 双缓存

> **在内存中构建并直接替换**的技术

`React`使用“双缓存”来完成`Fiber树`的构建与替换——对应着`DOM树`的创建与更新。

#### 双缓存fiber树

在`React`中最多会同时存在两棵`Fiber树`。当前屏幕上显示内容对应的`Fiber树`称为`current Fiber树`，正在内存中构建的`Fiber树`称为`workInProgress Fiber树`。

`current Fiber树`中的`Fiber节点`被称为`current fiber`，`workInProgress Fiber树`中的`Fiber节点`被称为`workInProgress fiber`，他们通过`alternate`属性连接。

```js
currentFiber.alternate === workInProgressFiber;
workInProgressFiber.alternate === currentFiber;
```

`React`应用的根节点通过使`current`指针在不同`Fiber树`的`rootFiber`间切换来完成`current Fiber`树指向的切换。

即当`workInProgress Fiber树`构建完成交给`Renderer`渲染在页面上后，应用根节点的`current`指针指向`workInProgress Fiber树`，此时`workInProgress Fiber树`就变为`current Fiber树`。

每次状态更新都会产生新的`workInProgress Fiber树`，通过`current`与`workInProgress`的替换，完成`DOM`更新。

> FiberRootNode -> rootFiber -> ...



## render阶段

### 递-阶段

首先从rootFiber开始向下深度优先遍历。为遍历到的每个Fiber节点调用beginWork方法。

该方法会根据传入的Fiber节点创建子Fiber节点，并将这两个Fiber节点连接起来。

当遍历到叶子节点（即没有子组件的组件）时就会进入“归”阶段。

### 归-阶段

在“归”阶段会调用completeWork处理Fiber节点。

当某个Fiber节点执行完completeWork，如果其存在兄弟Fiber节点（即fiber.sibling !== null），会进入其兄弟Fiber的“递”阶段。

如果不存在兄弟Fiber，会进入父级Fiber的“归”阶段。

“递”和“归”阶段会交错执行直到“归”到rootFiber。至此，render阶段的工作就结束了。

### beginWork

```jsx
ReactDOM.render(
  <Demo />,
  document.getElementById('root')
);

export default class Demo extends React.Component<any, any> {
    public render(): React.ReactNode {
        return (
            <div id="APP">
                <span>nihao</span>
                <p>
                    zhongguo
                </p>
            		wangyu
            </div>
        )
    }
}
```

beginWork执行（深度优先遍历）

> rootFiber(child) <-> (return)-Demo-(child) <-> (return)-div-(child) <-> (return)-span-(child) <-> nihao(return)

completeWork执行

> nihao(return) <-> span-(sibling) <-> p(child) <-> zhongguo(return) -> p -> div -> Demo ...... 

- 当执行到没有子节点的节点时就会调用completeWork，之后依次往上寻找父节点的兄弟节点等操作
- 如果存在兄弟节点，那就是beginWork和completeWork交替执行的过程

- 当节点只有唯一文本节点时，react提供的优化不会为其生成自己的fiber节点

通过current === null ?来区分组件是处于mount还是update。

beginWork的工作可以分为两部分：

- update时：如果current存在，在满足一定条件时可以复用current节点，这样就能克隆current.child作为workInProgress.child，而不需要新建workInProgress.child。
- mount时：除fiberRootNode以外，current === null。会根据fiber.tag不同，创建不同类型的子Fiber节点

```jsx
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  // update时：如果current存在可能存在优化路径，可以复用current（即上一次更新的Fiber节点）
  if (current !== null) {
    // ...省略

    // 复用current
    return bailoutOnAlreadyFinishedWork(
      current,
      workInProgress,
      renderLanes,
    );
  } else {
    didReceiveUpdate = false;
  }

  // mount时：根据tag不同，创建不同的子Fiber节点
  switch (workInProgress.tag) {
    case IndeterminateComponent: 
      // ...省略
    case LazyComponent: 
      // ...省略
    case FunctionComponent: 
      // ...省略
    case ClassComponent: 
      // ...省略
    case HostRoot:
      // ...省略
    case HostComponent:
      // ...省略
    case HostText:
      // ...省略
    // ...省略其他类型
  }
}
```

- current：当前组件对应的Fiber节点在上一次更新时的Fiber节点，即workInProgress.alternate
- workInProgress：当前组件对应的Fiber节点

- renderLanes：优先级相关

#### reconcileChildren

- 对于mount的组件，他会创建新的子Fiber节点
- 对于update的组件，他会将当前组件与该组件在上次更新时对应的Fiber节点比较（也就是俗称的Diff算法），将比较的结果生成新Fiber节点

```jsx
export function reconcileChildren(
  current: Fiber | null,
  workInProgress: Fiber,
  nextChildren: any,
  renderLanes: Lanes
) {
  if (current === null) {
    // 对于mount的组件
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderLanes,
    );
  } else {
    // 对于update的组件
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,
      nextChildren,
      renderLanes,
    );
  }
}
```

> mountChildFibers与reconcileChildFibers这两个方法的逻辑基本一致。唯一的区别是：reconcileChildFibers会为生成的Fiber节点带上effectTag属性，而mountChildFibers不会。



![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1632874736160-235ed54e-c263-4a58-844e-ebe2dc72f82f.png)



#### mount

```js
// mount时：根据tag不同，创建不同的Fiber节点
switch (workInProgress.tag) {
  case IndeterminateComponent: 
    // ...省略
  case LazyComponent: 
    // ...省略
  case FunctionComponent: 
    // ...省略
  case ClassComponent: 
    // ...省略
  case HostRoot:
    // ...省略
  case HostComponent:
    // ...省略
  case HostText:
    // ...省略
  // ...省略其他类型
}
```

#### update

满足如下情况时`didReceiveUpdate === false`（即可以直接复用前一次更新的`子Fiber`，不需要新建`子Fiber`）

1. `oldProps === newProps && workInProgress.type === current.type`，即`props`与`fiber.type`不变
2. `!includesSomeLane(renderLanes, updateLanes)`，即当前`Fiber节点`优先级不够

```js
if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;

    if (
      oldProps !== newProps ||
      hasLegacyContextChanged() ||
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      didReceiveUpdate = true;
    } else if (!includesSomeLane(renderLanes, updateLanes)) {
      didReceiveUpdate = false;
      switch (workInProgress.tag) {
        // 省略处理
      }
      return bailoutOnAlreadyFinishedWork(
        current,
        workInProgress,
        renderLanes,
      );
    } else {
      didReceiveUpdate = false;
    }
  } else {
    didReceiveUpdate = false;
  }
```

#### effectTag

`render阶段`的工作是在内存中进行，当工作结束后会通知`Renderer`需要执行的`DOM`操作。要执行`DOM`操作的具体类型就保存在`fiber.effectTag`中。

```js
// DOM需要插入到页面中
export const Placement = /*                */ 0b00000000000010;
// DOM需要更新
export const Update = /*                   */ 0b00000000000100;
// DOM需要插入到页面中并更新
export const PlacementAndUpdate = /*       */ 0b00000000000110;
// DOM需要删除
export const Deletion = /*                 */ 0b00000000001000;
...
```

> 通过二进制表示`effectTag`，可以方便的使用位操作为`fiber.effectTag`赋值多个`effect`。

那么，如果要通知`Renderer`将`Fiber节点`对应的`DOM节点`插入页面中，需要满足两个条件：

1. `fiber.stateNode`存在，即`Fiber节点`中保存了对应的`DOM节点`
2. `(fiber.effectTag & Placement) !== 0`，即`Fiber节点`存在`Placement effectTag`



### completeWork

```js
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const newProps = workInProgress.pendingProps;

  switch (workInProgress.tag) {
    case IndeterminateComponent:
    case LazyComponent:
    case SimpleMemoComponent:
    case FunctionComponent:
    case ForwardRef:
    case Fragment:
    case Mode:
    case Profiler:
    case ContextConsumer:
    case MemoComponent:
      return null;
    case ClassComponent: {
      // ...省略
      return null;
    }
    case HostRoot: {
      // ...省略
      updateHostContainer(workInProgress);
      return null;
    }
    case HostComponent: {
      // ...省略
      return null;
    }
  // ...省略
```

和`beginWork`一样，我们根据`current === null ?`判断是`mount`还是`update`。

同时针对`HostComponent`，判断`update`时我们还需要考虑`workInProgress.stateNode != null ?`（即该`Fiber节点`是否存在对应的`DOM节点`）

```js
case HostComponent: {
  popHostContext(workInProgress);
  const rootContainerInstance = getRootHostContainer();
  const type = workInProgress.type;

  if (current !== null && workInProgress.stateNode != null) {
    // update的情况
    // ...省略
  } else {
    // mount的情况
    // ...省略
  }
  return null;
}
```

#### mount阶段

- 为`Fiber节点`生成对应的`DOM节点`
- 将子孙`DOM节点`插入刚生成的`DOM节点`中
- 与`update`逻辑中的`updateHostComponent`类似的处理`props`的过程

```js
// mount的情况

// ...省略服务端渲染相关逻辑

const currentHostContext = getHostContext();
// 为fiber创建对应DOM节点
const instance = createInstance(
    type,
    newProps,
    rootContainerInstance,
    currentHostContext,
    workInProgress,
  );
// 将子孙DOM节点插入刚生成的DOM节点中
appendAllChildren(instance, workInProgress, false, false);
// DOM节点赋值给fiber.stateNode
workInProgress.stateNode = instance;

// 与update逻辑中的updateHostComponent类似的处理props的过程
if (
  finalizeInitialChildren(
    instance,
    type,
    newProps,
    rootContainerInstance,
    currentHostContext,
  )
) {
  markUpdate(workInProgress);
}
```

`mount`时只会在`rootFiber`存在`Placement effectTag`。那么`commit阶段`是如何通过一次插入`DOM`操作（对应一个`Placement effectTag`）将整棵`DOM树`插入页面的呢？

原因就在于`completeWork`中的`appendAllChildren`方法。

由于`completeWork`属于“归”阶段调用的函数，每次调用`appendAllChildren`时都会将已生成的子孙`DOM节点`插入当前生成的`DOM节点`下。那么当“归”到`rootFiber`时，我们已经有一个构建好的离屏`DOM树`。

（在`reconcileChildren`的时候，rootFiber是存在alternate的，即rootFiber存在对应的current Fiber，所以rootFiber会走`reconcileChildFibers`的逻辑，所以`shouldTrackSideEffects`等于true会追踪副作用，最后为rootFiber打上`Placement`的`effectTag`，然后将dom一次性插入，提高性能。）

#### update阶段

当`update`时，`Fiber节点`已经存在对应`DOM节点`，所以不需要生成`DOM节点`。需要做的主要是处理`props`，比如：

- `onClick`、`onChange`等回调函数的注册
- 处理`style prop`
- 处理`DANGEROUSLY_SET_INNER_HTML prop`
- 处理`children prop`

最主要的逻辑是调用`updateHostComponent`方法。

```js
if (current !== null && workInProgress.stateNode != null) {
  // update的情况
  updateHostComponent(
    current,
    workInProgress,
    type,
    newProps,
    rootContainerInstance,
  );
}
```

在`updateHostComponent`内部，被处理完的`props`会被赋值给`workInProgress.updateQueue`，并最终会在`commit阶段`被渲染在页面上。

```ts
workInProgress.updateQueue = (updatePayload: any);
```

其中`updatePayload`为数组形式，他的偶数索引的值为变化的`prop key`，奇数索引的值为变化的`prop value`。

#### effectList

作为`DOM`操作的依据，`commit阶段`需要找到所有有`effectTag`的`Fiber节点`并依次执行`effectTag`对应操作。难道需要在`commit阶段`再遍历一次`Fiber树`寻找`effectTag !== null`的`Fiber节点`么？？？

> 为了解决这个问题，在`completeWork`的上层函数`completeUnitOfWork`中，每个执行完`completeWork`且存在`effectTag`的`Fiber节点`会被保存在一条被称为`effectList`的单向链表中。
>
> `effectList`中第一个`Fiber节点`保存在`fiber.firstEffect`，最后一个元素保存在`fiber.lastEffect`。
>
> 类似`appendAllChildren`，在“归”阶段，所有有`effectTag`的`Fiber节点`都会被追加在`effectList`中，最终形成一条以`rootFiber.firstEffect`为起点的单向链表。
>
> 这样，在`commit阶段`只需要遍历`effectList`就能执行所有`effect`了。

![img](https://cdn.nlark.com/yuque/0/2021/png/21510703/1633739447486-482e22dd-6673-4aad-b421-0bf1f9587e22.png)

## commit阶段

`Renderer`工作的阶段被称为`commit`阶段。`commit`阶段可以分为三个子阶段：

- before mutation阶段（执行`DOM`操作前）
- mutation阶段（执行`DOM`操作）
- layout阶段（执行`DOM`操作后）

```js
// 保存之前的优先级，以同步优先级执行，执行完毕后恢复之前优先级
const previousLanePriority = getCurrentUpdateLanePriority();
setCurrentUpdateLanePriority(SyncLanePriority);

// 将当前上下文标记为CommitContext，作为commit阶段的标志
const prevExecutionContext = executionContext;
executionContext |= CommitContext;

// 处理focus状态
focusedInstanceHandle = prepareForCommit(root.containerInfo);
shouldFireAfterActiveInstanceBlur = false;

// beforeMutation阶段的主函数
commitBeforeMutationEffects(finishedWork);

focusedInstanceHandle = null;
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



