# ReactDOM.Render链路分析

```typescript
// react-dom/src/client/ReactDOMRoot.js

ReactDOMHydrationRoot.prototype.render = ReactDOMRoot.prototype.render = function(
  children: ReactNodeList,
): void {
  // this为ReactDOM.createRoot()返回的ReactDOMRoot实例。_internalRoot指向fiberRoot(FiberRootNode)
  const root = this._internalRoot;
  
  // children为<App />编译后的ReactElement对象
  /*
  	{
  		"$$typeof": Symbol(react.element),
  		"key": null,
  		"props": {},
  		"ref": null,
  		"type": f App(),
        "_owner": null,
        "_store": {}
	}
  */
  updateContainer(children, root, null, null);
};
```

# updateContainer

`updateContainer`函数位于`react-reconciler`包中, 它串联了`react-dom`与`react-reconciler`. 其最后调用了`scheduleUpdateOnFiber`.

`scheduleUpdateOnFiber`是`输入`阶段的入口函数.

在`ReactFiberWorkLoop.js`中, 承接输入的函数只有`scheduleUpdateOnFiber`. 在`react-reconciler`对外暴露的 api 函数中, 只要涉及到需要改变 fiber 的操作(无论是`首次渲染`或`后续更新`操作), 最后都会间接调用`scheduleUpdateOnFiber`, 所以`scheduleUpdateOnFiber`函数是输入链路中的`必经之路`.

```typescript
// shared/ReactTypes
export type ReactNode =
  | React$Element<any>
  | ReactPortal
  | ReactText
  | ReactFragment
  | ReactProvider<any>
  | ReactConsumer<any>;
export type ReactEmpty = null | void | boolean;
export type ReactNodeList = ReactEmpty | React$Node;

// react-reconciler/src/ReactFiberReconciler.old
import type {ReactNodeList} from 'shared/ReactTypes';
export function updateContainer(
  element: ReactNodeList, 
  container: OpaqueRoot, // fiberRoot(FiberRootNode)
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): Lane {
  // current树即rootFiber(FiberNode)
  const current = container.current;
  const eventTime = requestEventTime();
  // 获取fiber更新优先级(车道) -- react-reconciler/src/ReactFiberWorkLoop.old.js
  const lane = requestUpdateLane(current);

  // 获取context对象，默认为空对象，赋给fiberRoot
  const context = getContextForSubtree(parentComponent);
  if (container.context === null) {
    container.context = context;
  } else {
    container.pendingContext = context;
  }

  // 创建update(Update)，后续会由当前fiber的updateQueue.shared.pending指向
  const update = createUpdate(eventTime, lane);
  // 为DevTools添加属性 Caution: React DevTools currently depends on this property being called "element".
  update.payload = {element};
  
  // 回调函数
  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    update.callback = callback;
  }
  
  // update入队，返回FiberRootNode
  const root = enqueueUpdate(current, update, lane);
  if (root !== null) {
    // 进入scheduleUpdateOnFiber函数 -> react-reconciler/src/ReactFiberWorkLoop.old.js
    scheduleUpdateOnFiber(root, current, lane, eventTime);
    entangleTransitions(root, current, lane);
  }

  return lane;
}
```

## getContextForSubtree

```typescript
// react-reconciler/src/ReactFiberReconciler.old

function getContextForSubtree(
  parentComponent: ?React$Component<any, any>,
): Object {
  if (!parentComponent) {
    return emptyContextObject;
  }

  const fiber = getInstance(parentComponent);
  const parentContext = findCurrentUnmaskedContext(fiber);

  if (fiber.tag === ClassComponent) {
    const Component = fiber.type;
    if (isLegacyContextProvider(Component)) {
      return processChildContext(fiber, Component, parentContext);
    }
  }

  return parentContext;
}
```

## createUpdate

```typescript
// react-reconciler/src/ReactFiberClassUpdateQueue.old.js

export function createUpdate(eventTime: number, lane: Lane): Update<*> {
  const update: Update<*> = {
    eventTime,
    lane,

    tag: UpdateState,
    payload: null,
    callback: null,

    next: null,
  };
  return update;
}
```

## UpdateQueue

```typescript
// react-reconciler/src/ReactFiberClassUpdateQueue.old.js

export type Update<State> = {|
  eventTime: number, // 发起update事件的时间(17.0.2中作为临时字段, 即将移出)
  lane: Lane, // update所属的优先级

  tag: 0 | 1 | 2 | 3, // 表示update种类, 共 4 种. UpdateState,ReplaceState,ForceUpdate,CaptureUpdate
  payload: any, // 载荷, 根据场景可以设置成一个回调函数或者对象
  callback: (() => mixed) | null, // 回调函数. commit完成之后会调用.

  next: Update<State> | null, // 指向链表中的下一个, 由于UpdateQueue是一个环形链表, 最后一个update.next指向第一个update对象
|};

// =============== UpdateQueue ==============
type SharedQueue<State> = {|
  pending: Update<State> | null, // 指向即将输入的update队列. 在class组件中调用setState()之后, 会将新的 update 对象添加到这个队列中来.
  interleaved: Update<State> | null,
  lanes: Lanes,
};

export type UpdateQueue<State> = {|
  baseState: State,
  firstBaseUpdate: Update<State> | null,
  lastBaseUpdate: Update<State> | null,
  shared: SharedQueue<State>,
  effects: Array<Update<State>> | null,
|};
```

![Fiber与updateQueue的数据结构和引用关系](C:\Users\AniviaH\Desktop\React源码\ReactDOM.createRoot.render\Fiber与updateQueue的数据结构和引用关系.png)

## enqueueUpdate

```typescript
// react-reconciler/src/ReactFiberClassUpdateQueue.old.js

export function enqueueUpdate<State>(
  fiber: Fiber,
  update: Update<State>,
  lane: Lane,
): FiberRoot | null {
  const updateQueue = fiber.updateQueue;
  if (updateQueue === null) {
    // Only occurs if the fiber has been unmounted.
    return null;
  }

  const sharedQueue: SharedQueue<State> = (updateQueue: any).shared;

  // 判断是否是render phase update
  if (isUnsafeClassRenderPhaseUpdate(fiber)) {
    // This is an unsafe render phase update. Add directly to the update
    // queue so we can process it immediately during the current render.
    const pending = sharedQueue.pending;
    if (pending === null) {
      // This is the first update. Create a circular list.
      update.next = update;
    } else {
      update.next = pending.next;
      pending.next = update;
    }
    sharedQueue.pending = update;

    // Update the childLanes even though we're most likely already rendering
    // this fiber. This is for backwards compatibility in the case where you
    // update a different component during render phase than the one that is
    // currently renderings (a pattern that is accompanied by a warning).
    return unsafe_markUpdateLaneFromFiberToRoot(fiber, lane);
  } else {
    // 正常进入这里
    return enqueueConcurrentClassUpdate(fiber, sharedQueue, update, lane);
  }
}

// react-reconciler/src/ReactFiberWorkLoop.old.js
export function isUnsafeClassRenderPhaseUpdate(fiber: Fiber) {
  // Check if this is a render phase update. Only called by class components,
  // which special (deprecated) behavior for UNSAFE_componentWillReceive props.
  return (
    // TODO: Remove outdated deferRenderPhaseUpdateToNextBatch experiment. We
    // decided not to enable it.
    (!deferRenderPhaseUpdateToNextBatch ||
      (fiber.mode & ConcurrentMode) === NoMode) &&
    (executionContext & RenderContext) !== NoContext
  );
}
```

### enqueueConcurrentClassUpdater

```typescript
// react-reconciler/src/ReactFiberConcurrentUpdates.old.js

export function enqueueConcurrentClassUpdate<State>(
  fiber: Fiber,
  queue: ClassQueue<State>, // SharedQueue的别名
  update: ClassUpdate<State>, // Update的别名
  lane: Lane,
) {
  const interleaved = queue.interleaved;
  if (interleaved === null) {
    // This is the first update. Create a circular list.
    update.next = update;
    // At the end of the current render, this queue's interleaved updates will
    // be transferred to the pending queue.
    pushConcurrentUpdateQueue(queue);
  } else {
    update.next = interleaved.next;
    interleaved.next = update;
  }
  queue.interleaved = update;

  return markUpdateLaneFromFiberToRoot(fiber, lane);
}

let concurrentQueues: Array<
  HookQueue<any, any> | ClassQueue<any>,
> | null = null;
export function pushConcurrentUpdateQueue(
  queue: HookQueue<any, any> | ClassQueue<any>, // HookQueue: UpdateQueue的别名；ClassQueue: SharedQueue的别名
) {
  if (concurrentQueues === null) {
    concurrentQueues = [queue];
  } else {
    concurrentQueues.push(queue);
  }
}

function markUpdateLaneFromFiberToRoot(
  sourceFiber: Fiber,
  lane: Lane,
): FiberRoot | null {
  // Update the source fiber's lanes
  sourceFiber.lanes = mergeLanes(sourceFiber.lanes, lane);
  let alternate = sourceFiber.alternate;
  
  // Walk the parent path to the root and update the child lanes.
  // 向父级路径遍历直到rootFiber,更新childLans属性
  let node = sourceFiber;
  let parent = sourceFiber.return;
  while (parent !== null) {
    parent.childLanes = mergeLanes(parent.childLanes, lane);
    alternate = parent.alternate;
    
    node = parent;
    parent = parent.return;
  }
  if (node.tag === HostRoot) {
    // rootFiber 返回FiberRootNode
    const root: FiberRoot = node.stateNode;
    return root;
  } else {
    return null;
  }
}
```

# scheduleUpdateOnFiber

```typescript
// react-reconciler/src/ReactFiberWorkLoop.old.js

export function scheduleUpdateOnFiber(
  root: FiberRoot,
  fiber: Fiber,
  lane: Lane,
  eventTime: number,
) {
  // Mark that the root has a pending update.
  markRootUpdated(root, lane, eventTime);

  if (
    (executionContext & RenderContext) !== NoLanes &&
    root === workInProgressRoot
  ) {
    // ...
  } else {
    // 进入这里
    // This is a normal update, scheduled from outside the render phase. For
    // example, during an input event.
      
    // ...

    // 进入ensureRootIsScheduled 
    ensureRootIsScheduled(root, eventTime);
    if (
      lane === SyncLane &&
      executionContext === NoContext &&
      (fiber.mode & ConcurrentMode) === NoMode &&
      // Treat `act` as if it's inside `batchedUpdates`, even in legacy mode.
      !(__DEV__ && ReactCurrentActQueue.isBatchingLegacy)
    ) {
      // Flush the synchronous work now, unless we're already working or inside
      // a batch. This is intentionally inside scheduleUpdateOnFiber instead of
      // scheduleCallbackForFiber to preserve the ability to schedule a callback
      // without immediately flushing it. We only do this for user-initiated
      // updates, to preserve historical behavior of legacy mode.
      resetRenderTimer();
      flushSyncCallbacksOnlyInLegacyMode();
    }
  }
}
```

## markRootUpdated

```typescript
// react-reconciler/src/ReactFiberConcurrentUpdates.old.js
export function markRootUpdated(
  root: FiberRoot,
  updateLane: Lane,
  eventTime: number,
) {
  root.pendingLanes |= updateLane;

  if (updateLane !== IdleLane) {
    root.suspendedLanes = NoLanes;
    root.pingedLanes = NoLanes;
  }

  const eventTimes = root.eventTimes;
  const index = laneToIndex(updateLane);
  
  eventTimes[index] = eventTime;
}
```

## ensureRootIsScheduled

```typescript
// react-reconciler/src/ReactFiberWorkLoop.old.js

// Use this function to schedule a task for a root. There's only one task per
// root; if a task was already scheduled, we'll check to make sure the priority
// of the existing task is the same as the priority of the next level that the
// root has work on. This function is called on every update, and right before
// exiting a task.

import {
  // Aliased because `act` will override and push to an internal queue
  scheduleCallback as Scheduler_scheduleCallback,
} from './Scheduler';

// 注册调度任务
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  const existingCallbackNode = root.callbackNode;

  // Check if any lanes are being starved by other work. If so, mark them as
  // expired so we know to work on those next.
  markStarvedLanesAsExpired(root, currentTime);

  // Determine the next lanes to work on, and their priority.
  const nextLanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );

  // We use the highest priority lane to represent the priority of the callback.
  const newCallbackPriority = getHighestPriorityLane(nextLanes);

  // Schedule a new callback.
  let newCallbackNode;
  // 判断同步异步
  if (newCallbackPriority === SyncLane) {
    // 同步
    // Special case: Sync React callbacks are scheduled on a special
    // internal queue
    if (root.tag === LegacyRoot) {
      if (__DEV__ && ReactCurrentActQueue.isBatchingLegacy !== null) {
        ReactCurrentActQueue.didScheduleLegacyUpdate = true;
      }
      scheduleLegacySyncCallback(performSyncWorkOnRoot.bind(null, root));
    } else {
      scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root));
    }
    if (supportsMicrotasks) {
      // Flush the queue in a microtask.
      if (__DEV__ && ReactCurrentActQueue.current !== null) {
        // Inside `act`, use our internal `act` queue so that these get flushed
        // at the end of the current scope even when using the sync version
        // of `act`.
        ReactCurrentActQueue.current.push(flushSyncCallbacks);
      } else {
        scheduleMicrotask(() => {
          // In Safari, appending an iframe forces microtasks to run.
          // https://github.com/facebook/react/issues/22459
          // We don't support running callbacks in the middle of render
          // or commit so we need to check against that.
          if (
            (executionContext & (RenderContext | CommitContext)) ===
            NoContext
          ) {
            // Note that this would still prematurely flush the callbacks
            // if this happens outside render or commit phase (e.g. in an event).
            flushSyncCallbacks();
          }
        });
      }
    } else {
      // Flush the queue in an Immediate task.
      scheduleCallback(ImmediateSchedulerPriority, flushSyncCallbacks);
    }
    newCallbackNode = null;
  } else {
    // 异步
    let schedulerPriorityLevel;
    switch (lanesToEventPriority(nextLanes)) { // 车道转为事件优先级
      case DiscreteEventPriority:
        schedulerPriorityLevel = ImmediateSchedulerPriority;
        break;
      case ContinuousEventPriority:
        schedulerPriorityLevel = UserBlockingSchedulerPriority;
        break;
      case DefaultEventPriority:
        schedulerPriorityLevel = NormalSchedulerPriority;
        break;
      case IdleEventPriority:
        schedulerPriorityLevel = IdleSchedulerPriority;
        break;
      default:
        schedulerPriorityLevel = NormalSchedulerPriority;
        break;
    }
    newCallbackNode = scheduleCallback(
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root),
    );
  }

  root.callbackPriority = newCallbackPriority;
  root.callbackNode = newCallbackNode;
}

function scheduleCallback(priorityLevel, callback) {
  if (__DEV__) {
    // If we're currently inside an `act` scope, bypass Scheduler and push to
    // the `act` queue instead.
    const actQueue = ReactCurrentActQueue.current;
    if (actQueue !== null) {
      actQueue.push(callback);
      return fakeActCallbackNode;
    } else {
      return Scheduler_scheduleCallback(priorityLevel, callback);
    }
  } else {
    // In production, always call Scheduler. This function will be stripped out.
    // 注册调度任务，交由scheduler包进行任务调度循环
    // Scheduler_scheduleCallback 从 ./Scheduler.js导入，其再从 scheduler/src/forks/Scheduler导入，对应函数为 unstable_scheduleCallback
    // 参数二为任务回调函数，在调度任务循环执行时就会执行传入的回调，回到react-reconciler包逻辑 performConcurrentWorkOnRoot
    return Scheduler_scheduleCallback(priorityLevel, callback);
  }
}
```

# unstable_scheduleCallback(scheduler包)

```typescript
// scheduler/src/forks/Scheduler.js

import {push, pop, peek} from '../SchedulerMinHeap';

// Max 31 bit integer. The max integer size in V8 for 32-bit systems.
// Math.pow(2, 30) - 1
// 0b111111111111111111111111111111
var maxSigned31BitInt = 1073741823;

// Times out immediately
var IMMEDIATE_PRIORITY_TIMEOUT = -1;
// Eventually times out
var USER_BLOCKING_PRIORITY_TIMEOUT = 250;
var NORMAL_PRIORITY_TIMEOUT = 5000;
var LOW_PRIORITY_TIMEOUT = 10000;
// Never times out
var IDLE_PRIORITY_TIMEOUT = maxSigned31BitInt;

// Tasks are stored on a min heap
var taskQueue = [];
var timerQueue = [];

// Incrementing id counter. Used to maintain insertion order.
var taskIdCounter = 1;

// This is set while performing work, to prevent re-entrance.
var isPerformingWork = false;

var isHostCallbackScheduled = false;
var isHostTimeoutScheduled = false;

function unstable_scheduleCallback(priorityLevel, callback, options) {
  var currentTime = getCurrentTime();

  var startTime;
  if (typeof options === 'object' && options !== null) {
    var delay = options.delay;
    if (typeof delay === 'number' && delay > 0) {
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
  } else {
    startTime = currentTime;
  }

  var timeout;
  switch (priorityLevel) {
    case ImmediatePriority: // 1
      timeout = IMMEDIATE_PRIORITY_TIMEOUT; // -1
      break;
    case UserBlockingPriority: // 2
      timeout = USER_BLOCKING_PRIORITY_TIMEOUT; // 250
      break;
    case IdlePriority: // 5
      timeout = IDLE_PRIORITY_TIMEOUT; // 1073741823
      break;
    case LowPriority: // 4
      timeout = LOW_PRIORITY_TIMEOUT; // 10000
      break;
    case NormalPriority: // 3
    default:
      timeout = NORMAL_PRIORITY_TIMEOUT; // 5000
      break;
  }

  var expirationTime = startTime + timeout;

  // task中没有next属性, 它不是一个链表, 其顺序是通过堆排序来实现的(小顶堆数组, 始终保证数组中的第一个task对象优先级最高).
  var newTask = {
    id: taskIdCounter++, // 唯一标识
    callback, // task 最核心的字段, 指向react-reconciler包所提供的回调函数.
    priorityLevel, // 优先级
    startTime, // 一个时间戳,代表 task 的开始时间(创建时间 + 延时时间).
    expirationTime, // 过期时间.
    sortIndex: -1, // 控制 task 在队列中的次序, 值越小的越靠前.
  };

  if (startTime > currentTime) {
    // This is a delayed task.
    newTask.sortIndex = startTime;
    push(timerQueue, newTask);
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      // All tasks are delayed, and this is the task with the earliest delay.
      if (isHostTimeoutScheduled) {
        // Cancel an existing timeout.
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      // Schedule a timeout.
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    // 正常进入else分支
    newTask.sortIndex = expirationTime;
    // task推入taskQueue小顶堆(数组，每次push或pop会重组内部顺序，始终保证第一个元素优先级最高)
    // 在任务调度循环(workLoop)中会不断取出taskQueue中的任务，并执行任务回调
    push(taskQueue, newTask);
    
    // Schedule a host callback, if needed. If we're already performing work,
    // wait until the next time we yield.
    if (!isHostCallbackScheduled && !isPerformingWork) { // isHostCallbackScheduled未被设置 && isPerformingWork未被设置
      // 标记isHostCallbackScheduled为true
      isHostCallbackScheduled = true;
      // 请求异步回调-内部通过初始化出来的异步方式函数触发异步事件
      // 传入的flushWork会被暂存到 scheduledHostCallback，在异步回调中调用它
      requestHostCallback(flushWork);
    }
  }

  return newTask;
}
```

## ScheduleMinHeap

![task-heap](C:\Users\AniviaH\Desktop\React源码\ReactDOM.createRoot.render\task-heap.png)

```typescript
// scheduler/src/SchedulerMinHeap.js

type Node = {
    id: number,
    sortIndex: number
}
type Heap = Array<Node>

export function push(heap: Heap, node: Node): void {
    const index = heap.length
    heap.push(node)
    shiftUp(heap, node, index)
}

export function peek(heap: Heap): Node | null {
    return heap.length === 0 ? null : heap[0]
}

export function pop(heap: Heap): Node | null {
    if (heap.length === 0) {
        return null
    }
    const first = heap[0]
    const last = heap.pop()
    if (last !== first) {
        heap[0] = last
        shiftDown(heap, last, 0)
    }
    return first
}

function shiftUp (heap, node, i) {
    let index = i
    while (index > 0) {
        /*
        	无符号右移运算符（>>>）（零填充右移）将左操作数计算为无符号数，并将该数字的二进制表示形式移位为右操作数指定的位数，取模 32。
        	向右移动的多余位将被丢弃，零位从左移入。其符号位变为 0，因此结果始终为非负数。与其他按位运算符不同，零填充右移返回一个无符号 32 位整数。
        */
        const parentIndex = (index - 1) >>> 1 (除以2舍去余数 - 除以2并向下取整)
        const parent = heap[parentIndex]
        if (compare(parent, node) > 0) {
            // The parent is larger. Swap positions.
            heap[parentIndex] = node
            heap[index] = parent
            index = parentIndex
        } else {
            // The parent is smaller. Exit.
            return
        }
    }
}

function shiftDown (heap, node, i) {
    let index = i
    const length = heap.length
    const halfLength = length >>> 1
    while (index < halfLength) {
        const leftIndex = (index + 1) * 2 - 1
        const left = heap[leftIndex]
        const rightIndex = leftIndex + 1
        const right = heap[rightIndex]
        
        // If the left or right node is smaller, swap with the smaller of those.
        if (compare(left, node) < 0) {
            if (rightIndex < length && compare(right, left) < 0) {
                heap[index] = right
                heap[rightIndex] = node
                index = rightIndex
            } else {
                heap[index] = left
                heap[leftIndex] = node
                index = leftIndex
            }
        } else {
            // Neither child is smaller. Exit.
        }
    }
}

function compare (a, b) {
    // Compare sort index first, the task id
    const diff = a.sortIndex - b.sortIndex
    return diff !== 0 ? diff : a.id - b.id
}
```

## requestHostCallback -> schedulePerformWorkUntilDeadline(异步事件触发)

```typescript
// scheduler/src/forks/Scheduler.js

let isMessageLoopRunning = false;
let scheduledHostCallback = null;
let taskTimeoutID = -1;

let needsPaint = false;

/*
	异步方式选择
	根据宿主环境的支持，选择合适的任务调度方式
	三种方式进行选择 localSetImmediate、MessageChannel、localSetTimeout
	现代浏览器通常会选择到第二种 MessageChannel 方式
*/
let schedulePerformWorkUntilDeadline;
if (typeof localSetImmediate === 'function') {
  // Node.js and old IE.
  // There's a few reasons for why we prefer setImmediate.
  //
  // Unlike MessageChannel, it doesn't prevent a Node.js process from exiting.
  // (Even though this is a DOM fork of the Scheduler, you could get here
  // with a mix of Node.js 15+, which has a MessageChannel, and jsdom.)
  // https://github.com/facebook/react/issues/20756
  //
  // But also, it runs earlier which is the semantic we want.
  // If other browsers ever implement it, it's better to use it.
  // Although both of these would be inferior to native scheduling.
  schedulePerformWorkUntilDeadline = () => {
    localSetImmediate(performWorkUntilDeadline);
  };
} else if (typeof MessageChannel !== 'undefined') {
  // DOM and Worker environments.
  // We prefer MessageChannel because of the 4ms setTimeout clamping.
  const channel = new MessageChannel();
  const port = channel.port2;
  channel.port1.onmessage = performWorkUntilDeadline; // 异步事件的监听函数(异步入口)
  schedulePerformWorkUntilDeadline = () => {
    port.postMessage(null);
  };
} else {
  // We should only fallback here in non-browser environments.
  schedulePerformWorkUntilDeadline = () => {
    localSetTimeout(performWorkUntilDeadline, 0);
  };
}

function requestHostCallback(callback) {
  // 暂存callback，在异步回调函数performWorkUntilDeadline中，调用该变量
  scheduledHostCallback = callback;
    
  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true;
    // 初始化时生成的MessageChannel实例，通过onmessage监听，并设置回调函数为performWorkUntilDeadline
    // schedulePerformWorkUntilDeadline则通过channel.port.postMessage触发监听的事件
    schedulePerformWorkUntilDeadline();
  }
}
```

## performWorkUntilDeadline--异步事件监听的回调函数(异步入口)

```typescript
// scheduler/src/forks/Scheduler.js

const performWorkUntilDeadline = () => {
  if (scheduledHostCallback !== null) { // scheduledHostCallback -> flushWork（任务调度循环）
    const currentTime = getCurrentTime();
    // Keep track of the start time so we can measure how long the main thread
    // has been blocked.
    startTime = currentTime;
    const hasTimeRemaining = true;

    // If a scheduler task throws, exit the current browser task so the
    // error can be observed.
    //
    // Intentionally not using a try-catch, since that makes some debugging
    // techniques harder. Instead, if `scheduledHostCallback` errors, then
    // `hasMoreWork` will remain true, and we'll continue the work loop.
    let hasMoreWork = true;
    try {
      // 任务调度循环(flushWork)-执行任务回调
      hasMoreWork = scheduledHostCallback(hasTimeRemaining, currentTime);
    } finally {
      // 执行完任务队列还有任务，则在下一次事件中触发和回调
      if (hasMoreWork) {
        // If there's more work, schedule the next message event at the end
        // of the preceding one.
        schedulePerformWorkUntilDeadline();
      } else {
        isMessageLoopRunning = false;
        scheduledHostCallback = null;
      }
    }
  } else {
    isMessageLoopRunning = false;
  }
  // Yielding to the browser will give it a chance to paint, so we can
  // reset this.
  needsPaint = false;
};
```

## flushWork

```typescript
// scheduler/src/forks/Scheduler.js

function flushWork(hasTimeRemaining, initialTime) {
  // We'll need a host callback the next time work is scheduled.
  isHostCallbackScheduled = false; // 重置标记为false
  
  if (isHostTimeoutScheduled) {
    // We scheduled a timeout but it's no longer needed. Cancel it.
    isHostTimeoutScheduled = false;
    cancelHostTimeout();
  }

  // 标记正在执行任务
  isPerformingWork = true;
  const previousPriorityLevel = currentPriorityLevel;
  try {
    if (enableProfiling) {
      try {
        return workLoop(hasTimeRemaining, initialTime);
      } catch (error) {
        if (currentTask !== null) {
          const currentTime = getCurrentTime();
          markTaskErrored(currentTask, currentTime);
          currentTask.isQueued = false;
        }
        throw error;
      }
    } else {
      // 任务调度循环workLoop
      // No catch in prod code path.
      return workLoop(hasTimeRemaining, initialTime);
    }
  } finally {
    currentTask = null;
    currentPriorityLevel = previousPriorityLevel;
    isPerformingWork = false;
    if (enableProfiling) {
      const currentTime = getCurrentTime();
      markSchedulerSuspended(currentTime);
    }
  }
}

function cancelHostTimeout() {
  localClearTimeout(taskTimeoutID);
  taskTimeoutID = -1;
}
```

## workLoop

```typescript
// scheduler/src/forks/Scheduler.js

function workLoop(hasTimeRemaining, initialTime) {
  let currentTime = initialTime;
  advanceTimers(currentTime);
  // 小顶堆头结点-最优先任务
  currentTask = peek(taskQueue);
    
  while (
    currentTask !== null &&
    !(enableSchedulerDebugging && isSchedulerPaused)
  ) {
    // 入队任务的回调
    const callback = currentTask.callback;
    if (typeof callback === 'function') {
      currentTask.callback = null;
      currentPriorityLevel = currentTask.priorityLevel;
      const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;
      
      // 执行调度任务。回到react-reconciler包
      // callback -> performConcurrentWorkOnRoot(renderConcurrentRoot + commitRoot)
      const continuationCallback = callback(didUserCallbackTimeout);
      currentTime = getCurrentTime();
      if (typeof continuationCallback === 'function') {
        // 还返回有新回调，继续加到当前任务去执行
        currentTask.callback = continuationCallback;
      } else {
        // 没有新回调，弹出当前任务
        if (currentTask === peek(taskQueue)) {
          pop(taskQueue);
        }
      }
      advanceTimers(currentTime);
    } else {
      // callback不是函数，直接弹出
      pop(taskQueue);
    }
    // 取下一个任务
    currentTask = peek(taskQueue);
  }
  
  // Return whether there's additional work
  // 返回true代表还有任务，则会在performWorkUntilDeadline执行scheduledHostCallback处得到返回值，还有任务则进行下一次事件触发与回调
  if (currentTask !== null) {
    return true;
  } else {
    const firstTimer = peek(timerQueue);
    if (firstTimer !== null) {
      requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
    }
    return false;
  }
}

function advanceTimers(currentTime) {
  // Check for tasks that are no longer delayed and add them to the queue.
  let timer = peek(timerQueue);
  while (timer !== null) {
    if (timer.callback === null) {
      // Timer was cancelled.
      pop(timerQueue);
    } else if (timer.startTime <= currentTime) {
      // Timer fired. Transfer to the task queue.
      pop(timerQueue);
      timer.sortIndex = timer.expirationTime;
      push(taskQueue, timer);
      if (enableProfiling) {
        markTaskStart(timer, currentTime);
        timer.isQueued = true;
      }
    } else {
      // Remaining timers are pending.
      return;
    }
    timer = peek(timerQueue);
  }
}
```

# react两大工作循环

1. `任务调度循环`

源码位于[`Scheduler.js`](https://github.com/facebook/react/blob/v17.0.2/packages/scheduler/src/Scheduler.js), 它是`react`应用得以运行的保证, 它需要循环调用, 控制所有任务(`task`)的调度.

2. `fiber构造循环`

源码位于[`ReactFiberWorkLoop.js`](https://github.com/facebook/react/blob/v17.0.2/packages/react-reconciler/src/ReactFiberWorkLoop.old.js), 控制 fiber 树的构造, 整个过程是一个[深度优先遍历](https://7kms.github.io/react-illustration-series/algorithm/dfs).

这两个循环对应的 js 源码不同于其他闭包(运行时就是闭包), 其中定义的全局变量, 不仅是该作用域的私有变量, 更用于`控制react应用的执行过程`.

## 区别与联系

1. 区别
   - `任务调度循环`是以`二叉堆`为数据结构(详见[react 算法之堆排序](https://7kms.github.io/react-illustration-series/algorithm/heapsort)), 循环执行`堆`的顶点, 直到`堆`被清空.
   - `任务调度循环`的逻辑偏向宏观, 它调度的是每一个任务(`task`), 而不关心这个任务具体是干什么的(甚至可以将`Scheduler`包脱离`react`使用), 具体任务其实就是执行回调函数`performSyncWorkOnRoot`或`performConcurrentWorkOnRoot`.
   - `fiber构造循环`是以`树`为数据结构, 从上至下执行深度优先遍历(详见[react 算法之深度优先遍历](https://7kms.github.io/react-illustration-series/algorithm/dfs)).
   - `fiber构造循环`的逻辑偏向具体实现, 它只是任务(`task`)的一部分(如`performSyncWorkOnRoot`包括: `fiber`树的构造, `DOM`渲染, 调度检测), 只负责`fiber`树的构造.
2. 联系
   - `fiber构造循环`是`任务调度循环`中的任务(`task`)的一部分. 它们是从属关系, 每个任务都会重新构造一个`fiber`树.

## 主干逻辑

两大循环的分工可以总结为: 大循环(任务调度循环)负责调度`task`, 小循环(fiber 构造循环)负责实现`task` .

react 运行的主干逻辑, 即将`输入转换为输出`的核心步骤, 实际上就是围绕这两大工作循环进行展开.

结合上文的宏观概览图(展示核心包之间的调用关系), 可以将 react 运行的主干逻辑进行概括:

1. 输入: 将每一次更新(如: 新增, 删除, 修改节点之后)视为一次`更新需求`(目的是要更新`DOM`节点).
2. 注册调度任务: `react-reconciler`收到`更新需求`之后, 并不会立即构造`fiber树`, 而是去调度中心`scheduler`注册一个新任务`task`, 即把`更新需求`转换成一个`task`.
3. 执行调度任务(输出): 调度中心 `scheduler`通过`任务调度循环`来执行`task`(`task`的执行过程又回到了`react-reconciler`包中).
   - `fiber构造循环`是`task`的实现环节之一, 循环完成之后会构造出最新的 fiber 树.
   - `commitRoot`是`task`的实现环节之二, 把最新的 fiber 树最终渲染到页面上, `task`完成.

主干逻辑就是`输入到输出`这一条链路, 为了更好的性能(如`批量更新`, `可中断渲染`等功能), `react`在输入到输出的链路上做了很多优化策略, 比如本文讲述的`任务调度循环`和`fiber构造循环`相互配合就可以实现`可中断渲染`.

--引用自github 图解React-[两大工作循环](https://7kms.github.io/react-illustration-series/main/workloop)

![two-workloop](C:\Users\AniviaH\Desktop\React源码\ReactDOM.createRoot.render\two-workloop.png)