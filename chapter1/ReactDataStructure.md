<!--
 * @Description: In User Settings Edit
 * @Author: your name
 * @Date: 2019-07-28 00:28:22
 * @LastEditTime: 2019-09-05 13:31:30
 * @LastEditors: Please set LastEditors
 -->


## Fiber
React团队重写了React 的核心算法---reconciliation,一般将之前的算法叫stack reconciliation，现在的叫fiber reconciliation。


#### Stack reconciliation
算法的工作流程和函数的调用过程类似，react在进行组件渲染时，从setState开始到渲染完成整个过程是同步的，无法暂停，知道整个过程完成，如果组件比较大，js的执行会占用主线程过多的时间，会导致丢帧。打个比方， 假如我现在要更新1000个组件，每个组件平均花时间1ms，那么在1s内，浏览器的整个线程都被阻塞了，这时候用户在input上的任何操作都不会有反应，等到更新完毕，界面上突的一下就显示了原来用户的输入，这个体验是非常差的。

#### Fiber reconciliation
Fiber的特点

- 暂停工作，并在之后可以返回再次开始；
- 可以为不同类型的工作设置优先级；
- 复用之前已经完成的工作；
- 中止已经不再需要的工作。

React将任务分成小片，在一小片段的时间内运行这些分片任务，让主线程做优先级更高的事情，如果有任何待处理的事情，就回来完成工作,**fiber即为一个分片任务**

用一张经典的图来概括就是这样

![结构](../images/fiber01.png)


一个Fiber就是一个工作单元， React 的一个核心概念是 UI 是数据的投影 ，组件的本质可以看作输入数据，输出UI的描述信息（虚拟DOM树），即：


```
ui = f(data)
```

也就是说，渲染一个 React app，其实是在调用一个函数，函数本身会调用其它函数，形成调用栈，递归调用导致的调用栈我们本身无法控制， 只能一次执行完成。而 Fiber 就是为了解决这个痛点，可以去按需要打断调用栈，手动控制 stack frame——就这点来说，Fiber 可以理解为 virtual stack frame。



## Fiber的结构

普通的js对象 ,它包含有关组件的输入和输出的信息。
下面是fiber一些重要的属性。



key | type |desc
---|---|---
tag | Number | FiberNode的类型,如HostRoot,HostComponent，HostText等有21种
key | string | ReactElement里面的key
elementType | ReactElement.type | 我们调用`createElement`的第一个参数
type | function/class | 异步组件resolved之后返回的内容，一般是`function`或者`class`
stateNode | （FiberRoot，DomElement，ReactComponentInstance）|FiberNode会通过stateNode绑定一些其他的对象，例如FiberNode对应的Dom、FiberRoot、ReactComponent实例
return |	FiberNode或null	| 表示父级 FiberNode
child | FiberNode或null	 |表示第一个子 FiberNode
sibling |	FiberNode或null	| 表示紧紧相邻的下一个兄弟 FiberNode
alternate | FiberNode或null	 |Fiber调度算法采取了双缓冲池算法，current的alternate指向workInProgerss ,而workInProgress的alternate指向current，在创建workInProgerss的时候会尽量重用current.alternate
pendingProps |	Object	|表示新的props,来自element的props
memoizedProps | Object | 表示上一次render时的props
memoizedState |	Object | 表示处理后的新state
updateQueue | UpdateQueue | 更新队列，队列内放着即将要发生的变更状态
effectTag | Number | 16进制的数字，可以理解为通过一个字段标识n个动作，如Placement、Update、Deletion、Callback……所以源码中看到很多 &=
firstEffect | FiberNode，null|与副作用操作遍历流程相关 当前节点下，第一个需要处理的side effect
nextEffect | FiberNode，null|表示下一个将要处理的side effect
lastEffect | FiberNode，null|表示最后一个将要处理的side effect
pendingWorkPriority | number | 工作的优先级，数字越大，优先级越大
expirationTime | number | 代表任务在未来的哪个时间点应该被完成
childExpirationTime | number | 快速确定子树中是否有不在等待的变化



## EffectTag

当state和props变化时，会引起视图的重新渲染，这个过程叫做side effect，而side effect分为很多种情况，具体要执行哪种effect，在react中是通过effectTag属性记录的，在源码中以二进制的形式保存，因此可以记录多种操作。
```
export type SideEffectTag = number;

// Don't change these two values. They're used by React Dev Tools.
export const NoEffect = /*              */ 0b000000000000;
export const PerformedWork = /*         */ 0b000000000001;

// You can change the rest (and add more).
export const Placement = /*             */ 0b000000000010;
export const Update = /*                */ 0b000000000100;
export const PlacementAndUpdate = /*    */ 0b000000000110;
export const Deletion = /*              */ 0b000000001000;
export const ContentReset = /*          */ 0b000000010000;
export const Callback = /*              */ 0b000000100000;
export const DidCapture = /*            */ 0b000001000000;
export const Ref = /*                   */ 0b000010000000;
export const Snapshot = /*              */ 0b000100000000;
export const Passive = /*               */ 0b001000000000;

// Passive & Update & Callback & Ref & Snapshot
export const LifecycleEffectMask = /*   */ 0b001110100100;

// Union of all host effects
export const HostEffectMask = /*        */ 0b001111111111;

export const Incomplete = /*            */ 0b010000000000;
export const ShouldCapture = /*         */ 0b100000000000;
```

- NoEffect：一般作为effectTag的初始值，或者用于effectTag的比较判断，表示NoWork
- PerformedWork：由react devtools读取，NoEffect和PerformedWork都不会被committed，当创建effcet list（后面会介绍）时，会跳过NoEffect和PerformedWork
- Placement：向树中插入新的子节点，对应的状态为MOUNTING，当执行commitPlacement函数完成插入后，清除该标志位
- Update：当props、state、context发生变化，或者forceUpdate时，会标记为Update，检查到标记后，执行commitUpdate函数进行属性更新，与其相关的生命周期函数为componentDidMount和componentDidUpdate
- Deletion：标记将要卸载的节点，检查到标记后，执行commitDeletion函数对组件进行卸载，在节点树中删除对应对节点，与其相关的生命周期函数为componentWillUnmount
- ContentReset 当从文本域节点切换到非文本域或空节点时，打上此标记，将文本内容进行重置，文本域节点包括textarea、option、noscript、string、number和直接在标签中写入的__html。当检测到标记后，执行commitResetTextContent函数将对应节点的text清空
- Callback：当setState、forceUpdate有callback函数，或者在Commit阶段捕获到错误时，会更新update.callback，并标记Callback，随后检测到标记后会触发commitLifeCycles函数，根据不同到组件类型进行不同的commit
- DidCapture：针对于懒加载的React.Suspense（SuspenseComponent）组件提供的标志位，DidCapture位置位表示要渲染的组件被挂起，进而先渲染fallback的内容
- ShouldCapture：标记是否需要将节点挂起，一般捕获边界错误或者超时会置位，随后用于判断是否进行DidCapture
- Ref：当节点中存在属性ref时，会进行markRef当标记，随后会在commitAllLifeCycles阶段执行commitAttachRef触发相应当ref回调函数
- Snapshot：在渲染更新之前，当前后当props或state发生变化时，触发getSnapshotBeforeUpdate生命周期钩子




Hooks出来之后新增了HookEffectTag，用于标记ReactHook的effect的tag，后面再作介绍

```

export const NoEffect = /*             */ 0b00000000; 0
export const UnmountSnapshot = /*      */ 0b00000010; 2
export const UnmountMutation = /*      */ 0b00000100; 4
export const MountMutation = /*        */ 0b00001000; 8
export const UnmountLayout = /*        */ 0b00010000; 16
export const MountLayout = /*          */ 0b00100000; 32
export const MountPassive = /*         */ 0b01000000; 64
export const UnmountPassive = /*       */ 0b10000000; 128

```


## FiberRoot

```
type BaseFiberRootProperties = {|
  // The type of root (legacy, batched, concurrent, etc.)
  tag: RootTag,

  //root节点，render方法接收的第二个参数
  containerInfo: any,
  // 只有在持久更新中会用到，也就是不支持增量更新的平台，react-dom不会用到
  pendingChildren: any,
  // 当前应用对应的Fiber对象，是Root Fiber
  current: Fiber,

  pingCache:
    | WeakMap<Thenable, Set<ExpirationTime>>
    | Map<Thenable, Set<ExpirationTime>>
    | null,

  finishedExpirationTime: ExpirationTime,
  // 已经完成的并准备进行commit的 work-in-progress HostRoot .
  finishedWork: Fiber | null,
  // 通过setTimeout设置的返回内容，如果被新的任务代替，用来取消pending状态的timeout
  timeoutHandle: TimeoutHandle | NoTimeout,
  // 顶层context对象，只有主动调用`renderSubtreeIntoContainer`时才会有用
  context: Object | null,
  pendingContext: Object | null,
  // 确定是否应该在初始挂载时进行hydrate
  +hydrate: boolean,
  // 批量更新列表，此列表指示是否应延迟提交，还包含完成回调。
  firstBatch: Batch | null,
  // Scheduler.scheduleCallback返回的节点
  callbackNode: *,
  // 和root相关的回调到期时间
  callbackExpirationTime: ExpirationTime,
  // 树中最早的挂起到期时间
  firstPendingTime: ExpirationTime,
  // 树中最新的挂起到期时间
  lastPendingTime: ExpirationTime,
  // The time at which a suspended component pinged the root to render again
  pingTime: ExpirationTime,
|};
```

## ReactWorkTag

```
export const FunctionComponent = 0;
export const ClassComponent = 1;
export const IndeterminateComponent = 2; // Before we know whether it is function or class
export const HostRoot = 3; // Root of a host tree. Could be nested inside another node.
export const HostPortal = 4; // A subtree. Could be an entry point to a different renderer.
export const HostComponent = 5;
export const HostText = 6;
export const Fragment = 7;
export const Mode = 8;
export const ContextConsumer = 9;
export const ContextProvider = 10;
export const ForwardRef = 11;
export const Profiler = 12;
export const SuspenseComponent = 13;
export const MemoComponent = 14;
export const SimpleMemoComponent = 15;
export const LazyComponent = 16;
export const IncompleteClassComponent = 17;
export const DehydratedSuspenseComponent = 18;
export const EventComponent = 19;
export const EventTarget = 20;
export const SuspenseListComponent = 21;
```

## Update & UpdateQueue

```
export type Update<State> = {
    //更新的过期时间
  expirationTime: ExpirationTime,
  suspenseConfig: null | SuspenseConfig,
    // export const UpdateState = 0;
    // export const ReplaceState = 1;
    // export const ForceUpdate = 2;
    // export const CaptureUpdate = 3;
  tag: 0 | 1 | 2 | 3,
    //更新内容，比如`setState`接收的第一个参数
  payload: any,
    // 对应的回调，`setState`，`render`都有
  callback: (() => mixed) | null,
    // 指向下一个更新
  next: Update<State> | null,
  // 指向下一个`side effect`
  nextEffect: Update<State> | null,
};

export type UpdateQueue<State> = {
    // 每次操作完更新之后的`state`
  baseState: State,
    // 队列中的第一个`Update`
  firstUpdate: Update<State> | null,
    // 队列中的最后一个`Update`
  lastUpdate: Update<State> | null,
    // 第一个捕获类型的`Update`
  firstCapturedUpdate: Update<State> | null,
    // 最后一个捕获类型的`Update`
  lastCapturedUpdate: Update<State> | null,
    // 第一个`side effect`
  firstEffect: Update<State> | null,
    // 最后一个`side effect`
  lastEffect: Update<State> | null,
    // 第一个和最后一个捕获产生的`side effect`
  firstCapturedEffect: Update<State> | null,
  lastCapturedEffect: Update<State> | null,
};
```
在React中有很多单链表的数据结构，包括上面提到的effect和UpdateQueue。

updateQueue是更新队列，他是一个单向链表，用于更新state，并重绘组件，firstUpdate和lastUpdate分别指向链表的头部和尾部，存储这update对象。以HostRoot为例，从beginWork开始，直到遍历完整个UpdateQueue链表，获取新的状态，当新旧状态不一样时，将effectTag的Update置位，进入更新阶段。

## workInProgress 双缓冲池技术

workInProgress tree是reconcile过程中从fiber tree建立的当前进度快照，所有的工作都是在这颗树上进行，用于计算更新，完成reconciliation过程。

```
workInProgress.alternate = current;
current.alternate = workInProgress;
```

他和当前的fiber通过alternate进行关联，在构建workInProgress 时，会取current.alternate，存在则复用，不存在则创建。这样做能够复用内部对象（fiber），节省内存分配、GC的时间开销。


workInProgress tree构造完毕，得到的就是新的fiber tree，当进入commit阶段就把current指向了workInProgress

```
root.current = finishedWork;
```

## tag
用于标记组件类型，具体分类取值如下：

```
var FunctionComponent = 0;
var ClassComponent = 1;
var IndeterminateComponent = 2; // Before we know whether it is function or class
var HostRoot = 3; // Root of a host tree. Could be nested inside another node.
var HostPortal = 4; // A subtree. Could be an entry point to a different renderer.
var HostComponent = 5;
var HostText = 6;
var Fragment = 7;
var Mode = 8;
var ContextConsumer = 9;
var ContextProvider = 10;
var ForwardRef = 11;
var Profiler = 12;
var SuspenseComponent = 13;
var MemoComponent = 14;
var SimpleMemoComponent = 15;
var LazyComponent = 16;
var IncompleteClassComponent = 17;
var DehydratedSuspenseComponent = 18;
```

## mode

mode在创建时进行设置，在创建之后，mode在Fiber的整个生命周期内保持不变，可能的取值：
```
var NoContext = 0; // 同步模式
var ConcurrentMode = 1; // 异步模式
var StrictMode = 2; //严格模式，一般用于开发中，
var ProfileMode = 4; // 分析模式，一般用于开发中
```