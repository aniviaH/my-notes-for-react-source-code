# 旧版本的 ReactDom.render 和 新版的root.render的区别

## 旧版本 ReactDom.render

```js
import ReactDOM from 'react-dom'

ReactDOM.render(<App />, document.getElementById('root'))
```

## 新版本 root.render(React18)

```js
import ReactDOM from 'react-dom/client'

const root = ReactDOM.createRoot(document.getElementById('root'))
root.render(<App />)
```

## 两种初始化ReactDOM实例的方式区别

root的一些字段不同(在ensureRootIsScheduled函数内)

- pendingLanes
  旧版本：1
  新版本：16
- tag
  旧版本：0
  新版本：1

函数内部会基于这些字段值判断进行任务调度的方式(以同步还是异步方式)
同步就将 performSyncWorkOnRoot 函数 bind 之后，传入 schedule函数(scheduleSyncCallback 或 scheduleLegacySyncCallback )
异步就将 performConcurrentWorkOnRoot 函数 bind 之后，传入 schedule函数(scheduleCallback)

由调度器调度回调任务

```js
// react-reconciler/src/ReactFiberWorkLoop.old.js

// 注册调度任务
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  const existingCallbackNode = root.callbackNode;

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
```
