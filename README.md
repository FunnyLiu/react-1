# 源码分析


## 知识点

### React.memo做了什么？怎么做的？

React.memo(...) 对应的是函数组件，React.PureComponent 对应的是类组件。

React.memo 会返回了一个纯组件 MemodFuncComponent。 我们将在 JSX 标记中渲染此组件。 每当组件中的 props 和 state 发生变化时，React 将检查 上一个 state 和 props 以及下一个 props 和 state 是否相等，如果不相等则函数组件将重新渲染，如果它们相等则函数组件将不会重新渲染。


从实现的角度来看，React.memo方法只是在组件上增加了一个标识位：[笔记内容](https://github.com/FunnyLiu/react-1/blob/readsource/packages/react/src/ReactMemo.js#L27)

``` js
  //相当于带上了一个标识的type
  const elementType = {
    $$typeof: REACT_MEMO_TYPE,
    type,
    compare: compare === undefined ? null : compare,
  };
```

然后在Fiber阶段会针对这个类似额外做适配[笔记内容](https://github.com/FunnyLiu/react-1/blob/readsource/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L3603)：

``` js
    case MemoComponent: {

      //Memo组件会去走这个逻辑
      return updateMemoComponent(
        current,
        workInProgress,
        type,
        resolvedProps,
        updateLanes,
        renderLanes,
      );
    }
```

该方法会进行compare比较[笔记内容](https://github.com/FunnyLiu/react-1/blob/readsource/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L490)：

``` js
    // 如果两次props相等
    if (compare(prevProps, nextProps) && current.ref === workInProgress.ref) {
      //直接返回，不再进行下面的fiber操作。
      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
    }
```

如果比较结果一致，就不进行后续Fiber流程。





### React.lazy做了什么？怎么做的？

它能让你像渲染常规组件一样处理动态引入的组件，配合 webpack 的 Code Splitting ，只有当组件被加载，对应的资源才会导入 ，从而达到懒加载的效果。

``` js
// 不使用 React.lazy
import OtherComponent from './OtherComponent';
// 使用 React.lazy
const OtherComponent = React.lazy(() => import('./OtherComponent'))
```

React.lazy 需要配合 Suspense 组件一起使用，在 Suspense 组件中渲染 React.lazy 异步加载的组件。如果单独使用 React.lazy，React 会给出错误提示。


React.lazy不支持服务端渲染，使用服务端渲染的同学，请绕行至 react-loadable和 loadable-components。

从实现上来看，React.lazy其实是给组件增加了标识位和回调函数[笔记内容](https://github.com/FunnyLiu/react-1/blob/readsource/packages/react/src/ReactLazy.js#L104)。

``` js
//React.Lazy实现
export function lazy<T>(
  ctor: () => Thenable<{default: T, ...}>,
): LazyComponent<T, Payload<T>> {
  const payload: Payload<T> = {
    // We use these fields to store the result.
    //用来标记
    _status: -1,
    _result: ctor,
  };
  //增加标识位
  const lazyType: LazyComponent<T, Payload<T>> = {
    $$typeof: REACT_LAZY_TYPE,
    _payload: payload,
    _init: lazyInitializer,
  };
}
```


然后在fiber的开始阶段，而不是整个项目初始化的阶段。再去执行传入的thenable函数[笔记内容](https://github.com/FunnyLiu/react-1/blob/readsource/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L1278)

``` js
    //如果是lazy组件的话，走单独的流程
    case LazyComponent: {
      const elementType = workInProgress.elementType;
      return mountLazyComponent(
        current,
        workInProgress,
        elementType,
        updateLanes,
        renderLanes,
      );
    }
```





### vue的re-render和React的re-render的区别？

Vue2通过主动记录数据依赖，可以精确到某个组件开始及其子组件的re-render;

Vue3进一步优化到模板中区分静态节点和动态节点，只re-render动态节点；

而React则是每次改变状态后对整个APP进行重新diff并查到需要render的组件，重新执行render。

### this.setState做了什么？

入口在[笔记内容](https://github.com/FunnyLiu/react-1/blob/readsource/packages/react/src/ReactBaseClasses.js#L57)。

而实际上是调用的[笔记内容](https://github.com/FunnyLiu/react-1/blob/readsource/packages/react-reconciler/src/ReactFiberClassComponent.new.js#L205)

接下来进入了任务调度系统：[笔记内容](https://github.com/FunnyLiu/react-1/blob/readsource/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L465)

最后进入渲染整个root的流程：[笔记内容](https://github.com/FunnyLiu/react-1/blob/readsource/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1409)。并调用workLoop进行循环单元更新。




### React.createElement做了什么？

JSX会被编译为React.createElement，让我们看看他做了什么：

[源码实现](https://github.com/FunnyLiu/react-1/blob/readsource/packages/react/src/ReactElement.js#L351)

我们可以看到，React.createElement最终会调用ReactElement方法返回一个包含组件数据的对象，该对象有个参数$$typeof: REACT_ELEMENT_TYPE标记了该对象是个React Element。

换言之，在React中，所有JSX在运行时的返回结果（即React.createElement()的返回值）都是React Element。

### JSX和Fiber什么关系？

JSX是一种描述当前组件内容的数据结构，他不包含组件schedule、reconcile、render所需的相关信息。

比如如下信息就不包括在JSX中：

组件在更新中的优先级

组件的state

组件被打上的用于Renderer的标记

这些内容都包含在Fiber节点中。

所以，在组件mount时，Reconciler根据JSX描述的组件内容生成组件对应的Fiber节点。

在update时，Reconciler将JSX与Fiber节点保存的数据对比，生成组件对应的Fiber节点，并根据对比结果为Fiber节点打上标记。




### React架构怎么划分？

React16架构可以分为三层：

Scheduler（调度器）—— 核心职责只有 1 个, 就是执行回调

把react-reconciler提供的回调函数, 包装到一个任务对象中.

在内部维护一个任务队列, 优先级高的排在最前面.

循环消费任务队列, 直到队列清空.

Reconciler（协调器）—— 负责找出变化的组件，16版本主要是Fiber，15版本是stack。区别在于增加了优先级系统，通过遍历的方式实现可中断的递归，将fiber树的构造过程包装在一个回调函数中, 并将此回调函数传入到scheduler包等待调度.

Renderer（渲染器）—— 负责将变化的组件渲染到页面上，能够将react-reconciler包构造出来的fiber树表现出来, 生成 dom 节点(浏览器中), 生成字符串(ssr)，比如说react-dom、react-native

三者关系：

<img src="https://raw.githubusercontent.com/brizer/graph-bed/master/img/20211109094218.png"/>




### React15和16的Reconciler有什么区别？

Fiber Reconciler是从Stack Reconciler重构而来，通过遍历的方式实现可中断的递归。


### Reconciler主要是做什么的？

此处先归纳一下react-reconciler包的主要作用, 将主要功能分为 4 个方面:

输入: 暴露api函数(如: scheduleUpdateOnFiber), 供给其他包(如react包)调用.

注册调度任务: 与调度中心(scheduler包)交互, 注册调度任务task, 等待任务回调.

执行任务回调: 在内存中构造出fiber树, 同时与与渲染器(react-dom)交互, 在内存中创建出与fiber对应的DOM节点.

输出: 与渲染器(react-dom)交互, 渲染DOM节点.

<img src="https://raw.githubusercontent.com/brizer/graph-bed/master/img/20211109094806.png"/>

图中的1,2,3,4步骤可以反映react-reconciler包从输入到输出的运作流程,这是一个固定流程, 每一次更新都会运行.




### Fiber是什么？什么数据结构？

在React15及以前，Reconciler采用递归的方式创建虚拟DOM，递归过程是不能中断的。如果组件树的层级很深，递归会占用线程很多时间，造成卡顿。

为了解决这个问题，React16将递归的无法中断的更新重构为异步的可中断更新，由于曾经用于递归的虚拟DOM数据结构已经无法满足需要。于是，全新的Fiber架构应运而生。

Fiber包含三层含义：

作为架构来说，之前React15的Reconciler采用递归的方式执行，数据保存在递归调用栈中，所以被称为stack Reconciler。React16的Reconciler基于Fiber节点实现，被称为Fiber Reconciler。

作为静态的数据结构来说，每个Fiber节点对应一个React element，保存了该组件的类型（函数组件/类组件/原生组件...）、对应的DOM节点等信息。

作为动态的工作单元来说，每个Fiber节点保存了本次更新中该组件改变的状态、要执行的工作（需要被删除/被插入页面中/被更新...）。


数据结构如下：

``` js
function FiberNode(
  tag: WorkTag,
  pendingProps: mixed,
  key: null | string,
  mode: TypeOfMode,
) {
  // 作为静态数据结构的属性
  this.tag = tag;
  this.key = key;
  this.elementType = null;
  this.type = null;
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

### Fiber树是怎么工作的？


在React中最多会同时存在两棵Fiber树。当前屏幕上显示内容对应的Fiber树称为current Fiber树，正在内存中构建的Fiber树称为workInProgress Fiber树。

每次状态更新都会产生新的workInProgress Fiber树，通过current与workInProgress的替换，完成DOM更新。


### Fiber、ReactElement、Dom三者关系？

这里我们梳理出ReactElement, Fiber, DOM这 3 种对象的关系

ReactElement 对象(type 定义在shared 包中)

所有采用jsx语法书写的节点, 都会被编译器转换, 最终会以React.createElement(...)的方式, 创建出来一个与之对应的ReactElement对象
fiber 对象(type 类型的定义在ReactInternalTypes.js中)

fiber对象是通过ReactElement对象进行创建的, 多个fiber对象构成了一棵fiber树, fiber树是构造DOM树的数据模型, fiber树的任何改动, 最后都体现到DOM树.
DOM 对象: 文档对象模型

DOM将文档解析为一个由节点和对象（包含属性和方法的对象）组成的结构集合, 也就是常说的DOM树.
JavaScript可以访问和操作存储在 DOM 中的内容, 也就是操作DOM对象, 进而触发 UI 渲染.

<img src="https://raw.githubusercontent.com/brizer/graph-bed/master/img/20211109095133.png"/>

### React组件分哪些阶段和生命周期？

<img src="https://raw.githubusercontent.com/brizer/graph-bed/master/img/20211104161247.png"/>



### React16的的render阶段做了什么事情？

render阶段，根据组件返回的JSX在内存中依次创建Fiber节点并连接在一起构建Fiber树。

“递”阶段

首先从rootFiber开始向下深度优先遍历。为遍历到的每个Fiber节点调用beginWork方法。

该方法会根据传入的Fiber节点创建子Fiber节点，并将这两个Fiber节点连接起来。

当遍历到叶子节点（即没有子组件的组件）时就会进入“归”阶段。

“归”阶段

在“归”阶段会调用completeWork处理Fiber节点。

当某个Fiber节点执行完completeWork，如果其存在兄弟Fiber节点（即fiber.sibling !== null），会进入其兄弟Fiber的“递”阶段。

如果不存在兄弟Fiber，会进入父级Fiber的“归”阶段。

“递”和“归”阶段会交错执行直到“归”到rootFiber。至此，render阶段的工作就结束了。


举个例子：

``` js

function App() {
  return (
    <div>
      i am
      <span>KaSong</span>
    </div>
  )
}

ReactDOM.render(<App />, document.getElementById("root"));
```

生成的Fiber树为：

<img src="https://raw.githubusercontent.com/brizer/graph-bed/master/img/20211104154724.png"/>

其render阶段会执行：

1. rootFiber beginWork
2. App Fiber beginWork
3. div Fiber beginWork
4. "i am" Fiber beginWork
5. "i am" Fiber completeWork
6. span Fiber beginWork
7. span Fiber completeWork
8. div Fiber completeWork
9. App Fiber completeWork
10. rootFiber completeWork



### render阶段的beginWork究竟做了什么？

beginWork的工作是传入当前Fiber节点，创建子Fiber节点。[源码在此](https://github.com/FunnyLiu/react-1/blob/readsource/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L3214)

<img src="https://raw.githubusercontent.com/brizer/graph-bed/master/img/20211104155833.png"/>

### render阶段的completeWork究竟做了什么？

completeWork属于“归”阶段调用的函数，每次调用appendAllChildren时都会将已生成的子孙DOM节点插入当前生成的DOM节点下。那么当“归”到rootFiber时，我们已经有一个构建好的离屏DOM树。


<img src="https://raw.githubusercontent.com/brizer/graph-bed/master/img/20211104160616.png"/>


### commit阶段做了什么？

[源码在此](https://github.com/FunnyLiu/react-1/blob/readsource/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1701)

commit阶段的主要工作（即Renderer的工作流程）分为三部分：

before mutation阶段（执行DOM操作前）

mutation阶段（执行DOM操作）

layout阶段（执行DOM操作后）

在before mutation阶段之前和layout阶段之后还有一些额外工作，涉及到比如useEffect的触发、优先级相关的重置、ref的绑定/解绑。

### commit阶段中的before mutation之前主要做什么？

before mutation之前主要做一些变量赋值，状态重置的工作。


### commit阶段中的layout之后主要做什么?

主要包括三点内容：

1.useEffect相关的处理。


2.性能追踪相关

源码里有很多和interaction相关的变量。他们都和追踪React渲染时间、性能相关，在Profiler API 和DevTools 中使用。


3.在commit阶段会触发一些生命周期钩子（如 componentDidXXX）和hook（如useLayoutEffect、useEffect）。

在这些回调方法中可能触发新的更新，新的更新会开启新的render-commit流程。


### commit阶段中的before mutation阶段主要做什么？

在before mutation阶段，会遍历effectList，依次执行：

1.处理DOM节点渲染/删除后的 autoFocus、blur逻辑

2.调用getSnapshotBeforeUpdate生命周期钩子

3.调度useEffect

其中整个useEffect异步调用分为三步：

before mutation阶段在scheduleCallback中调度flushPassiveEffects

layout阶段之后将effectList赋值给rootWithPendingPassiveEffects

scheduleCallback触发flushPassiveEffects，flushPassiveEffects内部遍历rootWithPendingPassiveEffects

useEffect异步执行的原因主要是防止同步执行时阻塞浏览器渲染。

### commit阶段的mutation阶段主要做什么？


mutation阶段会遍历effectList，依次执行commitMutationEffects。该方法的主要工作为“根据effectTag调用不同的处理函数处理Fiber。

最后将Fiber渲染为Dom。

### commit阶段从layout阶段主要做什么？

该阶段之所以称为layout，因为该阶段的代码都是在DOM渲染完成（mutation阶段完成）后执行的。

该阶段触发的生命周期钩子和hook可以直接访问到已经改变后的DOM，即该阶段是可以参与DOM layout的阶段。

componentWillUnmount会在mutation阶段执行。此时current Fiber树还指向前一次更新的Fiber树，在生命周期钩子内获取的DOM还是更新前的。

componentDidMount和componentDidUpdate会在layout阶段执行。此时current Fiber树已经指向更新后的Fiber树，在生命周期钩子内获取的DOM就是更新后的。



### Diff算法到底是做什么的？

一个DOM节点在某一时刻最多会有4个节点和他相关。

1、current Fiber。如果该DOM节点已在页面中，current Fiber代表该DOM节点对应的Fiber节点。

2、workInProgress Fiber。如果该DOM节点将在本次更新中渲染到页面中，workInProgress Fiber代表该DOM节点对应的Fiber节点。

3、DOM节点本身。

4、JSX对象。即ClassComponent的render方法的返回结果，或FunctionComponent的调用结果。JSX对象中包含描述DOM节点的信息。

Diff算法的本质是对比1和4，生成2。


### Diff算法怎么优化复杂度？


由于Diff操作本身也会带来性能损耗，React文档中提到，即使在最前沿的算法中，将前后两棵树完全比对的算法的复杂程度为 O(n 3 )，其中n是树中元素的数量。

如果在React中使用了该算法，那么展示1000个元素所需要执行的计算量将在十亿的量级范围。这个开销实在是太过高昂。

为了降低算法复杂度，React的diff会预设三个限制：

1、只对同级元素进行Diff。如果一个DOM节点在前后两次更新中跨越了层级，那么React不会尝试复用他。

2、两个不同类型的元素会产生出不同的树。如果元素由div变为p，React会销毁div及其子孙节点，并新建p及其子孙节点。

3、开发者可以通过 key prop来暗示哪些子元素在不同的渲染下能保持稳定。考虑如下例子：

``` js
// 更新前
<div>
  <p key="ka">ka</p>
  <h3 key="song">song</h3>
</div>

// 更新后
<div>
  <h3 key="song">song</h3>
  <p key="ka">ka</p>
</div>

```

如果没有key，React会认为div的第一个子节点由p变为h3，第二个子节点由h3变为p。这符合限制2的设定，会销毁并新建。

但是当我们用key指明了节点前后对应关系后，React知道key === "ka"的p在更新后还存在，所以DOM节点可以复用，只是需要交换下顺序。

这就是React为了应对算法性能瓶颈做出的三条限制

### Diff算法具体怎么实现？

[源码在此](https://github.com/FunnyLiu/react-1/blob/readsource/packages/react-reconciler/src/ReactChildFiber.new.js#L1213)


前面提到，React为了优化复杂度只对同级元素进行Diff。我们可以从同级的节点数量将Diff分为两类：

1、当newChild类型为object、number、string，代表同级只有一个节点

2、当newChild类型为Array，同级有多个节点

针对同级有单个节点的情况：

<img src="https://raw.githubusercontent.com/brizer/graph-bed/master/img/20211104195437.png"/>

React通过先判断key是否相同，如果key相同则判断type是否相同，只有都相同时一个DOM节点才能复用。

针对同级有多个节点的情况：

如果让我设计一个Diff算法，我首先想到的方案是：

判断当前节点的更新属于哪种情况

如果是新增，执行新增逻辑

如果是删除，执行删除逻辑

如果是更新，执行更新逻辑

按这个方案，其实有个隐含的前提——不同操作的优先级是相同的

但是React团队发现，在日常开发中，相较于新增和删除，更新组件发生的频率更高。所以Diff会优先判断当前节点是否属于更新。

由于diff主要是对比Fiber（单链表）和jsx的区别，所以双指针优化无法使用。

基于以上原因，Diff算法的整体逻辑会经历两轮遍历：

第一轮遍历：处理更新的节点。

第二轮遍历：处理剩下的不属于更新的节点。

第一轮遍历步骤如下：

1、let i = 0，遍历newChildren，将newChildren[i]与oldFiber比较，判断DOM节点是否可复用。

2、如果可复用，i++，继续比较newChildren[i]与oldFiber.sibling，可以复用则继续遍历。

3、如果不可复用，分两种情况：

key不同导致不可复用，立即跳出整个遍历，第一轮遍历结束。

key相同type不同导致不可复用，会将oldFiber标记为DELETION，并继续遍历

4、如果newChildren遍历完（即i === newChildren.length - 1）或者oldFiber遍历完（即oldFiber.sibling === null），跳出遍历，第一轮遍历结束。

其他具体参考：https://react.iamkasong.com/diff/multi.html






### React的diff为什么用不了双指针优化？

在我们做数组相关的算法题时，经常使用双指针从数组头和尾同时遍历以提高效率，但是这里却不行。

虽然本次更新的JSX对象 newChildren为数组形式，但是和newChildren中每个组件进行比较的是current fiber，同级的Fiber节点是由sibling指针链接形成的单链表，即不支持双指针遍历。

即 newChildren[0]与fiber比较，newChildren[1]与fiber.sibling比较。

所以无法使用双指针优化。


### React中哪些情况会触发状态更新

ReactDOM.render

this.setState

this.forceUpdate

useState

useReducer

每次状态更新都会创建一个保存更新状态相关内容的对象，我们叫他Update。在render阶段的beginWork中会根据Update计算新的state。

### hooks实现原理

在真实的Hooks中，组件mount时的hook与update时的hook来源于不同的对象，这类对象在源码中被称为dispatcher。

hooks是通过单链表存储的。

``` js
// mount时的Dispatcher
const HooksDispatcherOnMount: Dispatcher = {
  useCallback: mountCallback,
  useContext: readContext,
  useEffect: mountEffect,
  useImperativeHandle: mountImperativeHandle,
  useLayoutEffect: mountLayoutEffect,
  useMemo: mountMemo,
  useReducer: mountReducer,
  useRef: mountRef,
  useState: mountState,
  // ...省略
};

// update时的Dispatcher
const HooksDispatcherOnUpdate: Dispatcher = {
  useCallback: updateCallback,
  useContext: readContext,
  useEffect: updateEffect,
  useImperativeHandle: updateImperativeHandle,
  useLayoutEffect: updateLayoutEffect,
  useMemo: updateMemo,
  useReducer: updateReducer,
  useRef: updateRef,
  useState: updateState,
  // ...省略
};
```

可见，mount时调用的hook和update时调用的hook其实是两个不同的函数。

接下来看看hook的数据结构：

``` js
const hook: Hook = {
  memoizedState: null,

  baseState: null,
  baseQueue: null,
  queue: null,

  next: null,
};
```

不同类型hook的memoizedState保存不同类型数据，具体如下：

useState：对于const [state, updateState] = useState(initialState)，memoizedState保存state的值

useReducer：对于const [state, dispatch] = useReducer(reducer, {});，memoizedState保存state的值

useEffect：memoizedState保存包含useEffect回调函数、依赖项等的链表数据结构effect，你可以在这里 [笔记内容](https://github.com/FunnyLiu/react-1/blob/readsource/packages/react-reconciler/src/ReactFiberHooks.new.js#L1273)看到effect的创建过程。effect链表同时会保存在fiber.updateQueue中

useRef：对于useRef(1)，memoizedState保存{current: 1}

useMemo：对于useMemo(callback, [depA])，memoizedState保存[callback(), depA]

useCallback：对于useCallback(callback, [depA])，memoizedState保存[callback, depA]。与useMemo的区别是，useCallback保存的是callback函数本身，而useMemo保存的是callback函数的执行结果


### hooks为什么用单链表存储而不是数组？

首先说明下，react的hook是单链表的结构，而fre的hook则是数组结构。数组结构和单链表结构可以实现hook。

但是我们在选择数据结构的时候需要考虑场景，hook的场景需要的是顺序访问，不需要随机访问；链表面对插入的场景，复杂度更低；数组会存在爆栈的隐患，而链表不会。

根据132的介绍，hook用数组更好，effectlist和fiber用链表更好，因为hook不需要插入、也不会出现量大到爆栈的情况，而react团队当年是能不用数组就不用数组的政治正确。

### hook为什么不能在条件判断中使用？

hook无论用数组还是链表，都无法解决这个问题，因为hook只初始化一次，但需要执行多次。if-else会干扰初始化的顺序。

而vue3的就没有这个问题，因为vue3不需要反复执行，所以顺序不会发生变化。

### useState和useReducer的实现

本质来说，useState只是预置了reducer的useReducer。

先看看两者用法：

``` js
function App() {
  const [state, dispatch] = useReducer(reducer, {a: 1});

  const [num, updateNum] = useState(0);
  
  return (
    <div>
      <button onClick={() => dispatch({type: 'a'})}>{state.a}</button>  
      <button onClick={() => updateNum(num => num + 1)}>{num}</button>  
    </div>
  )
}
```

#### 声明阶段

mount时，useReducer会调用mountReducer (opens new window)，useState会调用mountState (opens new window)。

我们来简单对比这这两个方法：

``` js
function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  // 创建并返回当前的hook
  const hook = mountWorkInProgressHook();

  // ...赋值初始state

  // 创建queue
  const queue = (hook.queue = {
    pending: null,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: (initialState: any),
  });

  // ...创建dispatch
  return [hook.memoizedState, dispatch];
}

function mountReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  // 创建并返回当前的hook
  const hook = mountWorkInProgressHook();

  // ...赋值初始state

  // 创建queue
  const queue = (hook.queue = {
    pending: null,
    dispatch: null,
    lastRenderedReducer: reducer,
    lastRenderedState: (initialState: any),
  });

  // ...创建dispatch
  return [hook.memoizedState, dispatch];
}
```

basicStateReducer方法如下：

``` js
function basicStateReducer<S>(state: S, action: BasicStateAction<S>): S {
  return typeof action === 'function' ? action(state) : action;
}
```

可见，useState即reducer参数为basicStateReducer的useReducer。


再看看update阶段：

useReducer与useState调用的则是同一个函数updateReducer。

``` js
function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  // 获取当前hook
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;
  
  queue.lastRenderedReducer = reducer;

  // ...同update与updateQueue类似的更新逻辑

  const dispatch: Dispatch<A> = (queue.dispatch: any);
  return [hook.memoizedState, dispatch];
}
```


找到对应的hook，根据update计算该hook的新state并返回。



#### 调用阶段

``` js
function dispatchAction(fiber, queue, action) {

  // ...创建update
  var update = {
    eventTime: eventTime,
    lane: lane,
    suspenseConfig: suspenseConfig,
    action: action,
    eagerReducer: null,
    eagerState: null,
    next: null
  }; 

  // ...将update加入queue.pending
  
  var alternate = fiber.alternate;

  if (fiber === currentlyRenderingFiber$1 || alternate !== null && alternate === currentlyRenderingFiber$1) {
    // render阶段触发的更新
    didScheduleRenderPhaseUpdateDuringThisPass = didScheduleRenderPhaseUpdate = true;
  } else {
    if (fiber.lanes === NoLanes && (alternate === null || alternate.lanes === NoLanes)) {
      // ...fiber的updateQueue为空，优化路径
    }

    scheduleUpdateOnFiber(fiber, lane, eventTime);
  }
}
```

创建update，将update加入queue.pending中，并开启调度。



### useEffect的实现

在v17.0.0中，useEffect的两个阶段会在页面渲染后（layout阶段后）异步执行。

#### 阶段一：销毁函数的执行

useEffect的执行需要保证所有组件在上一次render时的销毁函数必须都执行完后，才能执行任意一个组件的useEffect的回调函数。

这是因为多个组件间可能共用同一个ref。

如果不是按照“全部销毁”再“全部执行”的顺序，那么在某个组件useEffect的销毁函数中修改的ref.current可能影响另一个组件useEffect的回调函数中的同一个ref的current属性。

在useLayoutEffect中也有同样的问题，所以他们都遵循“全部销毁”再“全部执行”的顺序。

在阶段一，会遍历并执行所有useEffect的销毁函数。

``` js
// pendingPassiveHookEffectsUnmount中保存了所有需要执行销毁的useEffect
const unmountEffects = pendingPassiveHookEffectsUnmount;
  pendingPassiveHookEffectsUnmount = [];
  for (let i = 0; i < unmountEffects.length; i += 2) {
    const effect = ((unmountEffects[i]: any): HookEffect);
    const fiber = ((unmountEffects[i + 1]: any): Fiber);
    const destroy = effect.destroy;
    effect.destroy = undefined;

    if (typeof destroy === 'function') {
      // 销毁函数存在则执行
      try {
        destroy();
      } catch (error) {
        captureCommitPhaseError(fiber, error);
      }
    }
  }
```

#### 阶段二：回调函数的执行

与阶段一类似，同样遍历数组，执行对应effect的回调函数。

``` js
// pendingPassiveHookEffectsMount中保存了所有需要执行回调的useEffect
const mountEffects = pendingPassiveHookEffectsMount;
pendingPassiveHookEffectsMount = [];
for (let i = 0; i < mountEffects.length; i += 2) {
  const effect = ((mountEffects[i]: any): HookEffect);
  const fiber = ((mountEffects[i + 1]: any): Fiber);
  
  try {
    const create = effect.create;
   effect.destroy = create();
  } catch (error) {
    captureCommitPhaseError(fiber, error);
  }
}
```

### useRef的实现


事实上，任何需要被"引用"的数据都可以保存在ref中，useRef的出现将这种思想进一步发扬光大。

与其他Hook一样，对于mount与update，useRef对应两个不同dispatcher。

``` js
function mountRef<T>(initialValue: T): {|current: T|} {
  // 获取当前useRef hook
  const hook = mountWorkInProgressHook();
  // 创建ref
  const ref = {current: initialValue};
  hook.memoizedState = ref;
  return ref;
}

function updateRef<T>(initialValue: T): {|current: T|} {
  // 获取当前useRef hook
  const hook = updateWorkInProgressHook();
  // 返回保存的数据
  return hook.memoizedState;
}
```

ref的工作流程:

对于FunctionComponent，useRef负责创建并返回对应的ref。

对于赋值了ref属性的HostComponent与ClassComponent，会在render阶段经历赋值Ref effectTag，在commit阶段执行对应ref操作。


### useMemo和useCallback的实现

mount时

``` js
function mountMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null,
): T {
  // 创建并返回当前hook
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  // 计算value
  const nextValue = nextCreate();
  // 将value与deps保存在hook.memoizedState
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}

function mountCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  // 创建并返回当前hook
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  // 将value与deps保存在hook.memoizedState
  hook.memoizedState = [callback, nextDeps];
  return callback;
}
```

可以看到，与mountCallback这两个唯一的区别是

mountMemo会将回调函数(nextCreate)的执行结果作为value保存

mountCallback会将回调函数作为value保存

update时：

``` js
function updateMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null,
): T {
  // 返回当前hook
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;

  if (prevState !== null) {
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1];
      // 判断update前后value是否变化
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 未变化
        return prevState[0];
      }
    }
  }
  // 变化，重新计算value
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}

function updateCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  // 返回当前hook
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;

  if (prevState !== null) {
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1];
      // 判断update前后value是否变化
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 未变化
        return prevState[0];
      }
    }
  }

  // 变化，将新的callback作为value
  hook.memoizedState = [callback, nextDeps];
  return callback;
}
```

可见，对于update，这两个hook的唯一区别也是是回调函数本身还是回调函数的执行结果作为value

### Concurrent Mode是什么？

Concurrent 模式是一组 React 的新功能，可帮助应用保持响应，并根据用户的设备性能和网速进行适当的调整。

从源码层面讲，Concurrent Mode是一套可控的“多优先级更新架构”。


Concurrent Mode是React过去2年重构Fiber架构的源动力，也是React未来的发展方向。

可以预见，当v17完美支持Concurrent Mode后，v18会迎来一大波基于Concurrent Mode的库。


到要实现Concurrent Mode，最关键的一点是：实现异步可中断的更新。

基于这个前提，React花费2年时间重构完成了Fiber架构。

Fiber架构的意义在于，他将单个组件作为工作单元，使以组件为粒度的“异步可中断的更新”成为可能。

基于这个框架，可以做以下事情：


#### batchedUpdates

如果我们在一次事件回调中触发多次更新，他们会被合并为一次更新进行处理。

如下代码执行只会触发一次更新：

``` js
onClick() {
  this.setState({stateA: 1});
  this.setState({stateB: false});
  this.setState({stateA: 2});
}
```


这种合并多个更新的优化方式被称为batchedUpdates。

batchedUpdates在很早的版本就存在了，不过之前的实现局限很多（脱离当前上下文环境的更新不会被合并）。

在Concurrent Mode中，是以优先级为依据对更新进行合并的，使用范围更广。


#### Suspense
Suspense 可以在组件请求数据时展示一个pending状态。请求成功后渲染数据。

本质上讲Suspense内的组件子树比组件树的其他部分拥有更低的优先级。

#### useDeferredValue

useDeferredValue (opens new window)返回一个延迟响应的值，该值可能“延后”的最长时间为timeoutMs。

例子：
``` js
const deferredValue = useDeferredValue(value, { timeoutMs: 2000 });
```

在useDeferredValue内部会调用useState并触发一次更新。

这次更新的优先级很低，所以当前如果有正在进行中的更新，不会受useDeferredValue产生的更新影响。所以useDeferredValue能够返回延迟的值。

当超过timeoutMs后useDeferredValue产生的更新还没进行（由于优先级太低一直被打断），则会再触发一次高优先级更新。



### Scheduler（调度器）是做什么的？

如果我们同步运行Fiber架构（通过ReactDOM.render），则Fiber架构与重构前并无区别。

但是当我们配合时间切片，就能根据宿主环境性能，为每个工作单元分配一个可运行时间，实现“异步可中断的更新”。

于是，scheduler （调度器）产生了。

Scheduler，他包含两个功能：

时间切片

优先级调度

#### 时间切片的原理

时间切片的本质是模拟实现requestIdleCallback 。

除去“浏览器重排/重绘”，下图是浏览器一帧中可以用于执行JS的时机。

```
一个task(宏任务) -- 队列中全部job(微任务) -- requestAnimationFrame -- 浏览器重排/重绘 -- requestIdleCallback
```

requestIdleCallback是在“浏览器重排/重绘”后如果当前帧还有空余时间时被调用的。

Scheduler的时间切片功能是通过task（宏任务）实现的。

最常见的task当属setTimeout了。但是有个task比setTimeout执行时机更靠前，那就是MessageChannel 。

所以Scheduler将需要被执行的回调函数作为MessageChannel的回调执行。如果当前宿主环境不支持MessageChannel，则使用setTimeout。

在React的render阶段，开启Concurrent Mode时，每次遍历前，都会通过Scheduler提供的shouldYield方法判断是否需要中断遍历，使浏览器有时间渲染：

``` js
function workLoopConcurrent() {
  // Perform work until Scheduler asks us to yield
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

是否中断的依据，最重要的一点便是每个任务的剩余时间是否用完。

在Schdeduler中，为任务分配的初始剩余时间为5ms。随着应用运行，会通过fps动态调整分配给任务的可执行时间。[源码在此](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/scheduler/src/forks/SchedulerHostConfig.default.js#L172-L187)



#### 优先级调度

Scheduler是独立于React的包，所以他的优先级也是独立于React的优先级的。

Scheduler对外暴露了一个方法unstable_runWithPriority。

这个方法接受一个优先级与一个回调函数，在回调函数内部调用获取优先级的方法都会取得第一个参数对应的优先级：


``` js
function unstable_runWithPriority(priorityLevel, eventHandler) {
  switch (priorityLevel) {
    case ImmediatePriority:
    case UserBlockingPriority:
    case NormalPriority:
    case LowPriority:
    case IdlePriority:
      break;
    default:
      priorityLevel = NormalPriority;
  }

  var previousPriorityLevel = currentPriorityLevel;
  currentPriorityLevel = priorityLevel;

  try {
    return eventHandler();
  } finally {
    currentPriorityLevel = previousPriorityLevel;
  }
}
```

可以看到，Scheduler内部存在5种优先级。

在React内部凡是涉及到优先级调度的地方，都会使用unstable_runWithPriority。

比如，我们知道commit阶段是同步执行的。可以看到，commit阶段的起点commitRoot方法的优先级为ImmediateSchedulerPriority。

ImmediateSchedulerPriority即ImmediatePriority的别名，为最高优先级，会立即执行。

#### 优先级的意义

不同优先级意味着什么？不同优先级意味着不同时长的任务过期时间：

``` js
var timeout;
switch (priorityLevel) {
  case ImmediatePriority:
    timeout = IMMEDIATE_PRIORITY_TIMEOUT;
    break;
  case UserBlockingPriority:
    timeout = USER_BLOCKING_PRIORITY_TIMEOUT;
    break;
  case IdlePriority:
    timeout = IDLE_PRIORITY_TIMEOUT;
    break;
  case LowPriority:
    timeout = LOW_PRIORITY_TIMEOUT;
    break;
  case NormalPriority:
  default:
    timeout = NORMAL_PRIORITY_TIMEOUT;
    break;
}

var expirationTime = startTime + timeout;

```

其中：

``` js
// Times out immediately
var IMMEDIATE_PRIORITY_TIMEOUT = -1;
// Eventually times out
var USER_BLOCKING_PRIORITY_TIMEOUT = 250;
var NORMAL_PRIORITY_TIMEOUT = 5000;
var LOW_PRIORITY_TIMEOUT = 10000;
// Never times out
var IDLE_PRIORITY_TIMEOUT = maxSigned31BitInt;

```
可以看到，如果一个任务的优先级是ImmediatePriority，对应IMMEDIATE_PRIORITY_TIMEOUT为-1，那么

``` js
var expirationTime = startTime - 1;

```
则该任务的过期时间比当前时间还短，表示他已经过期了，需要立即被执行。

#### 不同优先级任务的排序

设想一个大型React项目，在某一刻，存在很多不同优先级的任务，对应不同的过期时间。

同时，又因为任务可以被延迟，所以我们可以将这些任务按是否被延迟分为：

已就绪任务

未就绪任务

所以，Scheduler存在两个队列：

timerQueue：保存未就绪任务

taskQueue：保存已就绪任务

每当有新的未就绪的任务被注册，我们将其插入timerQueue并根据开始时间重新排列timerQueue中任务的顺序。

当timerQueue中有任务就绪，即startTime <= currentTime，我们将其取出并加入taskQueue。

取出taskQueue中最早过期的任务并执行他。

为了能在O(1)复杂度找到两个队列中时间最早的那个任务，Scheduler使用小顶堆实现了优先级队列。

