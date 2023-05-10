# Fiber协调引擎

## 什么是Fiber协调引擎

> React组件会渲染出一棵元素树，每次有props、state等数据变动时，组件会渲染出新的元素树，React框架会与之前的树做Diffing对比，将元素的变动最终体现在浏览器页面的DOM中。这一过程就称为**协调（Reconciliation）**。
>
> > 在React的早期版本，协调是一个**同步过程**，这意味着当虚拟DOM足够复杂，或者元素渲染时产生的各种计算足够中，协调过程本身就可能超过16ms，严重的会导致页面卡顿。从React v16开始，协调从之前的同步改成了**异步过程**，这主要得益于新的**Fiber协调引擎**。

Fiber协调引擎做的事情基本上贯穿了React应用的整个生命周期，包括但不限于：

- 创建各类FiberNode并组建Fiber树；

- 调度并执行各类工作(Work)，如渲染函数组件、挂载或是更新Hooks、实例化或更新类组件；

- 比对新旧Fiber，触发DOM变更；

- 获取context数据；

- 错误处理；

- 性能监控；

## Fiber中的重要概念和模型

在协调过程中存在着各种动作，如调用生命周期方法或Hooks，这在Fiber协调引擎中被称作是**工作(Work)**。Fiber中最基本的模型是**FiberNode，用于描述一个组件需要做的或者已完成的工作，每个组件可能对应一个或多个FiberNode**。这与一个组件渲染可能会产生一个或多个React元素是一致的。

实际上，每个FiberNode的数据都来自于元素树中的一个元素，元素与FiberNode是意义对应的。与元素树不同的是，元素树每次渲染都会被重建，而FiberNode会被复用，FiberNode的属性会被更新。

下面是FiberNode的数据结构：

```typescript
type Fiber = {
    // ---- Fiber类型 ----
    
    /** 工作类型，枚举值包括：函数组件、类组件、HTML元素、Fragment等  */
    tag: WorkTag,
    /** 就是子元素列表用的key属性 */
    key: null | string,
    /** 对于React元素ReactElement.type属性 */
    elementType: any,
    /** 函数组件对于的函数或类组件对应的类，HTML元素对应的元素标签名如div */
    type: any,
    
    // ---- Fiber Tree树形结构
    
    /** 指向父FiberNode的指针 */
    return: Fiber | null,
    /** 指向子FiberNode的指针 */
    child: Fiber | null,
    /** 指向平级FiberNode的指针 */
    sibing: Fiber | null,
    
    // ---- Fiber数据 ----
    
    /** 经本次渲染更新的props值 */
    pendingProps: any,
    /** 上一次渲染的props值 */
    memoizedProps: any,
    /** 上一次渲染的state值，或是本次更新中的state值 */
    memoizedState: any,
    /** 各种state更新、回调、副作用回调和DOm更新的队列 */
    updateQueue: mixed,
    /** 为类组件保存对实例对象的引用，或为HTML元素保存对真实DOM的引用 */
    stateNode: any,
    
    // ---- Effect副作用 ----
    
    /** 副作用种类的位域，可同时标记多种副作用，如Placement、Update、Callback等 */
    flags: Flags,
    /** 指向下一个鞠永副作用的Fiber的引用，在React 18中已被弃用 */
    nextEffect: Fiber | null,
    
    // ---- 异步性/并发性 ----
    
    /** 当前Fiber与成对的进行中Fiber的双向引用 */
    alternate: Fiber | null,
    /** 标记Lane车道模型中车道的位域，表示调度的优先级 */
    lanes: Lanes,
}
```

还有与Hooks相关的模型，这包括了Hook和Effect

```typescript
type Hook = {
    memoizedState: any,
    baseState: any,
    baseQueue: Update<any, any> | null,
    queue: any,
    next: Hook | null,
}

type Effect = {
    tag: HookFlags,
    create: () => (() => void) | void,
    destroy: (() => void) | void,
    deps: Array | null,
    next: Effect,
}
```

此外，还有Dispatcher。基本每个Hook都有mount和update两个dispatcher，如useEffect有mountEffect和updateEffect；少数Hooks还有额外的rerender的dispatcher，如useState有rerenderState。

## 协调引擎是怎样的？

当第一次渲染，React元素树被创建出来后，Fiber协调引擎会从HostRoot这个特殊的元素开始，遍历元素树，创建对应的FiberNode。

FiberNode与FiberNode之间，并没有按照传统的parent-children方式建立树形结构。而是在父节点和它的第一个子节点间，利用child和return属性建立了双向链表。节点与它的平级节点间，利用sibling属性建立了单向链表，同时平级节点的return属性，也都被设置成和单向链表起点的节点return一样的值引用。

这样做的好处是，可以再协调引擎进行工作的过程中，**避免递归遍历Fiber树，而仅仅用两层循环来完成深度优先遍历**，这个用于遍历Fiber树的循环被称作workLoop。

以下是workLoop的示意代码，为了便于理解，对源码中的performUnitOfWork、beginWork、completeUnitOfWork、completeWork做了合并和简化，就是里面的workWork:

```js
let workInProgress;

function workLoop() {
    while (workInProgress && !shouldYield()) {
        const child = workWork(workInProgress);
        if (child) {
            workInProgress = child;
            continue;
        }
        
        let completedWork = workInProgress;
        do {
         if (completedWork.sibling) {
                workInProgress = completedWork.sibling;
                break;
            }
            completedWork = completedWork.return;
        } while (completedWork);
    }
}
```

更狠的一点是，这个循环**随时可以跑，随时可以停**。这意味着workLoop既可以同步跑，也可以异步跑，当workLoop发现进行中的Fiber工作耗时过长时，可以根据一个shouldYield()标记决定是否暂停工作，释放计算资源给更紧急的任务，等完成后再恢复工作。

### 渲染阶段

当组件内更新state或有context更新时，React会进入**渲染阶段（Render Phase）**。这一阶段是异步的，Fiber协调引擎会启动workLoop，从Fiber树的根部开始遍历，快速跳过已处理的节点；对有变化的节点，引擎会为**Current（当前）节点**克隆一个**WorkInProgress（进行中）节点**，将这两个FiberNode的alternate属性分别指向对方，并把更新都记录在WorkInProgress节点上。

你可以理解成同时存在两棵Fiber树，**一棵Current树，对应着目前已经渲染到页面上的内容；另一棵是WorkInProgress树，记录着即将发生的修改。**

函数组件的Hooks也是在渲染阶段执行的。除了useContext，Hooks在挂载后，都会形成一个由Hook.next属性连接的单向链表，而这个链表会挂在FiberNode.memoizedState属性上。

在此基础上，useEffect这样会产生副作用的Hooks，会额外创建于Hook对象一一对应的Effect对象，赋值给Hook.memoizedState属性。此外，也会在FiberNode.updateQueue属性上，维护一个由Effect.next属性连接的单向链表，并把这个Effect对象加入到链表末尾。

### 提交阶段

当Fiber树所有节点都完成工作后，WorkInProgress节点会被改称为**FinishedWork(已完成)节点**，WorkInProgress树也会被改称为**FinishedWork树**。这时React会进入提交阶段（Commit Phase），这一阶段主要是同步执行的。Fiber协调引擎会把FinishedWork节点上记录的所有修改，按一定顺序提交并体现在页面上。

提交阶段又分成如下3个先后同步执行的子阶段：

- **变更前（Before Mutation）子阶段**。这个阶段会调用类组件的getSnapshotBeforeUpdate方法。

- **变更（Mutation）子阶段**。这个子阶段会更新真实DOM树。
  - 递归提交与删除相关的副作用，包括移除ref、移除真实DOM、执行类组件的componentWillUnmount。
  - 递归提交添加、重新排序真实DOM等副作用。
  - 依次执行FiberNode上useLayoutEffect的清除函数。
  - 引擎用FinishedWork树替换Current树，供下次渲染阶段使用。

- **布局（Layout）子阶段**。这个子阶段真实DOM树已经完成了变更，会调用useLayoutEffect的副作用回调函数，和类组件的componentDidMount方法。

在提交阶段中，引擎还会多次异步或同步调用flushPassiveEffects()。这个函数会先后两轮按深度优先遍历Fiber树上的每个节点：

- 第一轮：如果节点的updateQueue链表中有待执行的、由useEffect定义的副作用，则顺序执行他们的**清除函数**。
- 第二轮：如果节点的updateQueue链表中有待执行的、由useEffect定义的副作用，则顺序执行它们的**副作用回调函数**，并保存清除函数，供下一轮提交阶段执行。

这个flushPassiveEffects()函数真正的执行时机，是在上述提交阶段的**三个同步子阶段之后，下一次渲染阶段之前**。引擎会保证在下一次渲染之前，执行完所有待执行的副作用。

你也许会好奇，协调引擎的Diffing算法在哪里？其实从渲染到提交阶段，到处都在利用memoizedProps和memoizedState与新的props、state做比较，以减少不必要的工作，进而提高性能。

> 文章来源：极客时间-《现代 React Web 开发实战》-宋一玮
