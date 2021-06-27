---
title: React源码细读-深入了解scheduler"多任务时间管理大师"
date: 2021-06-19 16:43:43
categories:
	- Web
	- JavaScript
	- React
	- Scheduler
tags:
	- React
	- JavaScript
	- Scheduler
---

# Scheduler 多任务时间管理大师

在 `react` 提出 `Fiber` 之前，复杂耗时任务会阻塞页面的渲染，降低页面的响应速度，为了缓解耗时任务与渲染等其他进程之间资源争夺的情况，`react` 增加了 `scheduler` 这一模块，通过 `scheduler` 更好的调度多任务，控制任务的执行顺序

`scheduler` 通过划分任务优先级, 时间切片, 任务中断、任务恢复等机制来保证高优任务的执行，只占用每一帧中尽可能短的时间用于处理任务，把主要资源在有必要的情况下交还给其他线程，以此提高页面响应速度，提升用户体验

能做到的这些不得不说是“多任务时间管理大师”了😁

**此文基于 `react v17.0.2` `scheduler v0.20.0` 分析，[仓库传送门](https://github.com/facebook/react)**

# React & Scheduler 缠缠绵绵

前面提到 `scheduler` 可以帮助 `react` 更好的去调度和控制多任务的执行，那么 `react` 是怎么把交给 `scheduler` 调度和控制执行的呢？

**这里不对react到scheduler的流程做深入解读，仅用于交代从react到scheduler的流程过渡**

话不多说，老规矩，我们先来看一个简单的 demo

```js
import * as React from "react";

class A extends React.PureComponent {
  state = {
    val: 1
  };

	onClickBtn = () => {
    this.setState({ val: 2 });
    this.setState({ val: 3 });
    this.setState({ val: 4 });
    setTimeout(() => {
      this.setState({ val: 5 });
      this.setState({ val: 6 });
    });
  };

  render() {
    const { val } = this.state;
    return (
      <p>
        <button onClick={this.onClickBtn}>click</button>
        {val}
      </p>
    );
  }
}

export default function App() {
  return (
    <A />
  );
}
```

案例的视图很简单，就只有一个按钮和一个值，当点击 click 时，会触发 `setState`，视图的值也会发生变化

<!-- more -->

![](https://tva1.sinaimg.cn/large/008i3skNgy1groqdmp4p4j312s0t4ad2.jpg)

了解 `react setState`  机制的朋友都知道，`react18` 之前的批量更新是区分场景的，这里的前三个 `setState` 因为处于事件回调函数的同步调用中，所以在触发 `setState` 时会进入 `enqueueUpdate` 函数

![](https://tva1.sinaimg.cn/large/008i3skNgy1gror2vjud5j30u00z9aiz.jpg)

## enqueueUpdate 函数

简化一下 `enqueueUpdate` 函数后可以知道，它本质就做了一件简单的事：把当前创建的更新对象 `Update` 存入当前 `Fiber` 实例的 `updateQueue.shared` 队列中，`updateQueue.shared` 已链表结构存在，通过 `next` 指针链接下一个 `Update` 对象

```js
export function enqueueUpdate<State>(
  fiber: Fiber,
  update: Update<State>,
  lane: Lane,
) {
  const updateQueue = fiber.updateQueue;
  const sharedQueue: SharedQueue<State> = (updateQueue: any).shared;

  if (isInterleavedUpdate(fiber, lane)) {
    // ....
  } else {
    const pending = sharedQueue.pending;
    if (pending === null) {
      // This is the first update. Create a circular list.
      update.next = update;
    } else {
      update.next = pending.next;
      pending.next = update;
    }
    sharedQueue.pending = update;
  }
}
```

`enqueueUpdate` 操作了一个很关键的对象-`Update`，这个在后续跟 `Scheduler` 的交互中有很深的关联，我们先看看它具体是什么

## Update 对象

```js
// @flow
export type Update<State> = {|
	// eventTime 是临时属性，后续会通过在 root 节点存储一个 transition 映射到 event-time 的 map 来替他这个属性
  eventTime: number,
  lane: Lane,

  tag: 0 | 1 | 2 | 3,
  payload: any,
  callback: (() => mixed) | null,

  next: Update<State> | null,
|};
```

从 `Update` 的数据结构可以简单了解到，`Update` 是用来描述当前的对象操作的一些基本信息：时间（eventTime）、优先级（lane）、回调函数（callback）、下一个更新对象（next）等

![](https://tva1.sinaimg.cn/large/008i3skNgy1grorehzctkj30zq0l0gq1.jpg)

需要注意这里的用来表示优先级的属性 `lane`

## Lane 多车道优先级模型

`Lane` 模型是 `react` 为了解决 `ExpirationTime` 模型导致的低优先级任务 `长时间等待/饿死` 的问题

目前 `react` 中共有 31 种不同的 `lane` 值，其中区分了不同的 `lane 车道` 与 `lanes 车道组`

```js
export type Lanes = number;
export type Lane = number;

export const TotalLanes = 31;

export const NoLanes: Lanes = /*                        */ 0b0000000000000000000000000000000;
export const NoLane: Lane = /*                          */ 0b0000000000000000000000000000000;

export const SyncLane: Lane = /*                        */ 0b0000000000000000000000000000001;

export const InputContinuousHydrationLane: Lane = /*    */ 0b0000000000000000000000000000010;
export const InputContinuousLane: Lanes = /*            */ 0b0000000000000000000000000000100;

export const DefaultHydrationLane: Lane = /*            */ 0b0000000000000000000000000001000;
export const DefaultLane: Lanes = /*                    */ 0b0000000000000000000000000010000;

const TransitionHydrationLane: Lane = /*                */ 0b0000000000000000000000000100000;
const TransitionLanes: Lanes = /*                       */ 0b0000000001111111111111111000000;
const TransitionLane1: Lane = /*                        */ 0b0000000000000000000000001000000;
const TransitionLane2: Lane = /*                        */ 0b0000000000000000000000010000000;
const TransitionLane3: Lane = /*                        */ 0b0000000000000000000000100000000;
const TransitionLane4: Lane = /*                        */ 0b0000000000000000000001000000000;
const TransitionLane5: Lane = /*                        */ 0b0000000000000000000010000000000;
const TransitionLane6: Lane = /*                        */ 0b0000000000000000000100000000000;
const TransitionLane7: Lane = /*                        */ 0b0000000000000000001000000000000;
const TransitionLane8: Lane = /*                        */ 0b0000000000000000010000000000000;
const TransitionLane9: Lane = /*                        */ 0b0000000000000000100000000000000;
const TransitionLane10: Lane = /*                       */ 0b0000000000000001000000000000000;
const TransitionLane11: Lane = /*                       */ 0b0000000000000010000000000000000;
const TransitionLane12: Lane = /*                       */ 0b0000000000000100000000000000000;
const TransitionLane13: Lane = /*                       */ 0b0000000000001000000000000000000;
const TransitionLane14: Lane = /*                       */ 0b0000000000010000000000000000000;
const TransitionLane15: Lane = /*                       */ 0b0000000000100000000000000000000;
const TransitionLane16: Lane = /*                       */ 0b0000000001000000000000000000000;

const RetryLanes: Lanes = /*                            */ 0b0000111110000000000000000000000;
const RetryLane1: Lane = /*                             */ 0b0000000010000000000000000000000;
const RetryLane2: Lane = /*                             */ 0b0000000100000000000000000000000;
const RetryLane3: Lane = /*                             */ 0b0000001000000000000000000000000;
const RetryLane4: Lane = /*                             */ 0b0000010000000000000000000000000;
const RetryLane5: Lane = /*                             */ 0b0000100000000000000000000000000;

export const SomeRetryLane: Lane = RetryLane1;

export const SelectiveHydrationLane: Lane = /*          */ 0b0001000000000000000000000000000;

const NonIdleLanes = /*                                 */ 0b0001111111111111111111111111111;

export const IdleHydrationLane: Lane = /*               */ 0b0010000000000000000000000000000;
export const IdleLane: Lanes = /*                       */ 0b0100000000000000000000000000000;

export const OffscreenLane: Lane = /*                   */ 0b1000000000000000000000000000000;
```

这里先不展开说明，后续会另开一篇深入探讨

我们接着 `enqueueUpdate` 分析，在执行完 `enqueueUpdate` 后，紧接着到了 `scheduleUpdateOnFiber`，这个是 `react` 与 `scheduler` 接触的起点

## scheduleUpdateOnFiber 函数

老规则，先来分析简化后的主要逻辑

```js
export function scheduleUpdateOnFiber(
  fiber: Fiber,
  lane: Lane,
  eventTime: number,
) {
  // 给根节点标记有更新
  markRootUpdated(root, lane, eventTime);
  // 在当前 Fiber 实例的 lanes 和所有父节点的 childLanes 中添加当前 Update.lane
	const root = markUpdateLaneFromFiberToRoot(fiber, lane);

  // 获取当前任务的 react 优先级
  const priorityLevel = getCurrentPriorityLevel();

  if (lane === SyncLane) {
    if (
      // ... 边缘场景处理（这里不管）
    ) {
      // ... 边缘场景处理（这里不管）
    } else {
      ensureRootIsScheduled(root, eventTime);
      schedulePendingInteractions(root, lane);
      // ... 边缘场景处理（这里不管）
    }
  } else {
		// 这一段判断很长，主要是针对 `离散事件` 导致的更新和高优先级任务做了特殊处理
		// 比如说用户输入或者用户点击等交互事件，如果这些事件生成新的 update
		// 那么需要把当前根节保存到 rootsWithPendingDiscreteUpdates（Set结构）中
    if (
      (executionContext & DiscreteEventContext) !== NoContext &&
      (priorityLevel === UserBlockingSchedulerPriority ||
        priorityLevel === ImmediateSchedulerPriority)
    ) {
      if (rootsWithPendingDiscreteUpdates === null) {
        rootsWithPendingDiscreteUpdates = new Set([root]);
      } else {
        rootsWithPendingDiscreteUpdates.add(root);
      }
    }
    ensureRootIsScheduled(root, eventTime);
    schedulePendingInteractions(root, lane);
  }
  // 保存最近更新的节点
  mostRecentlyUpdatedRoot = root;
}
```

从简化后的代码不难看出，`scheduleUpdateOnFiber` 主要做了这几件事

1. 区分 `SyncLane` 等级和其他 `Lane` 等级做不同的处理
2. 无论是哪种 `Lane`，最后都执行 `ensureRootIsScheduled` 函数

而 `ensureRootIsScheduled` 就是进入调度领域的最后一步

## ensureRootIsScheduled 函数

`ensureRootIsScheduled` 函数是 `react` 把任务交由 `scheduler` 调度的最有一步

`ensureRootIsScheduled` 同样有处理分支场景和边缘场景，还有最高 `Lane` 优先级的获取处理，这里暂且不深入

虽然这里的代码很长，内部调用的函数又多还不好理解，看的让人一脸懵逼😳

为了避免思路被吸引开，这里我们只需记住两个地方 `调度入口1` 和 `调度入口3`，其他有必要的部分只会稍微提一下

```js
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  // ...
	// 获取最高 lane 等级的新任务的 lane 值
  // 同时设置全局变量 return_highestLanePriority
	// return_highestLanePriority 对应的是 最高 lane 等级的优先级
	// return_highestLanePriority 可以通过 returnNextLanesPriority 函数返回
  const nextLanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );
	// newCallbackPriority 实际就是 return_highestLanePriority
	// 也就是 nextLanes lane 等级对应的 lane 优先级
  const newCallbackPriority = returnNextLanesPriority();
  // ...
  // 开启一个新的调度
  let newCallbackNode;
  if (newCallbackPriority === SyncLanePriority) {
		// 调度入口1
    newCallbackNode = scheduleSyncCallback(
      performSyncWorkOnRoot.bind(null, root),
    );
  } else if (newCallbackPriority === SyncBatchedLanePriority) {
		// 调度入口2
    // ... 边缘场景（这里不管）
  } else {
    const schedulerPriorityLevel = lanePriorityToSchedulerPriority(
      newCallbackPriority,
    );
		// 调度入口 3
    newCallbackNode = scheduleCallback(
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root),
    );
  }

  root.callbackPriority = newCallbackPriority;
  root.callbackNode = newCallbackNode;
}
```

我们下来看看 `ensureRootIsScheduled` 做了哪些事

1. `nextLanes` 获取最高 lane 等级新任务的 lanes 值
2. `newCallbackPriority` 获取最高 lane 等级的优先级
3. 根据 `newCallbackPriority` 进入不同的调度入口，比如 `调度入口1` 和 `调度入口3`

### lanePriorityToSchedulerPriority

这里需要额外提一下 `lanePriorityToSchedulerPriority`，这里的函数名实际上与做的事情有些出入，这里的 `Scheduler Priority` 实际是指的 `react` 内部的 `React Scheduler Priority`，不是 `Schduler Priority`

不是你的错觉，这里看起来确实很绕😭

说回正题，这个函数本质就是把 `Lane` 模型的优先级机制转化成 `React Schduler` 的优先级机制

代码本体就是如此简单，通过一长串 `switch case`，把 `Lane` 优先级转为 `React Scheduler` 优先级

```js
export function lanePriorityToSchedulerPriority(
  lanePriority: LanePriority,
): ReactPriorityLevel {
  switch (lanePriority) {
    case SyncLanePriority:
    case SyncBatchedLanePriority:
      return ImmediateSchedulerPriority;
    case InputDiscreteHydrationLanePriority:
    case InputDiscreteLanePriority:
    case InputContinuousHydrationLanePriority:
    case InputContinuousLanePriority:
      return UserBlockingSchedulerPriority;
    case DefaultHydrationLanePriority:
    case DefaultLanePriority:
    case TransitionHydrationPriority:
    case TransitionPriority:
    case SelectiveHydrationLanePriority:
    case RetryLanePriority:
      return NormalSchedulerPriority;
    case IdleHydrationLanePriority:
    case IdleLanePriority:
    case OffscreenLanePriority:
      return IdleSchedulerPriority;
    case NoLanePriority:
      return NoSchedulerPriority;
    default:
      invariant(
        false,
        'Invalid update priority: %s. This is a bug in React.',
        lanePriority,
      );
  }
}
```

刚刚提到了 `React Scheduler Priority` 和 `Schduler Priority`，这里可以不用纠结，在 `react` 中会对他们的关系做一个映射，接着往下看，后面会提到

### scheduleSyncCallback & scheduleCallback 调度入口

这两个调度入口的本质其实都是调用 `Scheduler_scheduleCallback` 函数，启动调度流程

回想上一小节中提到的 `React Scheduler Priority` 和 `Schduler Priority`，在真正进入 `Scheduler` 流程之前，会通过 `reactPriorityToSchedulerPriority` 函数做转换


```js
export function scheduleCallback(
  reactPriorityLevel: ReactPriorityLevel,
  callback: SchedulerCallback,
  options: SchedulerCallbackOptions | void | null,
) {
  // `React Scheduler Priority` 转换为 `Schduler Priority`
  const priorityLevel = reactPriorityToSchedulerPriority(reactPriorityLevel);
	// 真正的调度开始
  return Scheduler_scheduleCallback(priorityLevel, callback, options);
}
```

即然有 `scheduleCallback`，那么为什么还需要 `scheduleSyncCallback`？


<div align="center">
<img width=200 src="https://inews.gtimg.com/newsapp_bt/0/11865093918/1000"/>
</div>

`scheduleCallback` 和 `scheduleSyncCallback` 不是既生瑜何生亮的关系，它们有不一样的流程

不同的点在于，`scheduleCallback` 需要将 `React Scheduler` 优先级转为 `Scheduler` 优先级

此外，`scheduleSyncCallback` 会把 `callback` 推入 `syncQueue` 队列保存起来，在后续执行 `flushSyncCallbackQueue` 时使用，这里的 `syncQueue` 是 `react` 内部的任务队列，同时，在进入 `scheduler` 流程时把 `Scheduler Priority` 设置为了最高等级，简单来说就是需要立即执行的任务

```js
export function scheduleSyncCallback(callback: SchedulerCallback) {
  if (syncQueue === null) {
    syncQueue = [callback];
		// 真正的调度开始
    immediateQueueCallbackNode = Scheduler_scheduleCallback(
      Scheduler_ImmediatePriority,
      flushSyncCallbackQueueImpl,
    );
  } else {
    syncQueue.push(callback);
  }
  return fakeCallbackNode;
}
```

`syncQueue` 会通过 `flushSyncCallbackQueueImpl` 遍历，然后逐个执行其中的任务，这个流程是在 `react` 内部进行

这听起来好像有点调度的问道？是的，其实对于 `SyncLane` 的任务，`react` 会对其先做一层任务队列封装，再把处理任务队列的函数作为任务抛给 `Scheduler`

这是 `scheduleSyncCallback` 相对于 `scheduleCallback` 最大的不同点

### flushSyncCallbackQueueImpl

在 `scheduleSyncCallback` 中提到过，`flushSyncCallbackQueueImpl` 用来遍历 `syncQueue`，执行 `callback`，实际可以理解为它是一个同步任务调度器，不同于 `Scheduler_scheduleCallback`，`flushSyncCallbackQueueImpl` 利用的是 `Scheduler` 提供的 `unstable_runWithPriority` 函数来进行任务调度

函数内部的细节不比过于深究，我们只需要知道它做了这样一件事：

遍历 `syncQueue`，利用的是 `Scheduler` 提供的 `unstable_runWithPriority` 函数来执行 `callback`

```js
function flushSyncCallbackQueueImpl() {
  if (!isFlushingSyncQueue && syncQueue !== null) {
    isFlushingSyncQueue = true;
    let i = 0;
    if (decoupleUpdatePriorityFromScheduler) {
      // ... 这里是react的新特性流程，目前不过走这里
    } else {
      try {
        const isSync = true;
        const queue = syncQueue;
        // 通过 runWithPriority 来执行 单个任务
        runWithPriority(ImmediatePriority, () => {
          for (; i < queue.length; i++) {
            let callback = queue[i];
            do {
              callback = callback(isSync);
            } while (callback !== null);
          }
        });
        syncQueue = null;
      } catch (error) {
        // 发生报错时，保留剩余的任务队列
        if (syncQueue !== null) {
          syncQueue = syncQueue.slice(i + 1);
        }
        // 通过 scheduler 进行任务恢复
        Scheduler_scheduleCallback(
          Scheduler_ImmediatePriority,
          flushSyncCallbackQueue,
        );
        // 同时抛出错误
        throw error;
      } finally {
        isFlushingSyncQueue = false;
      }
    }
    return true;
  } else {
    return false;
  }
}

```

### reactPriorityToSchedulerPriority

`reactPriorityToSchedulerPriority` 的作用就是把 `React` 中的优先级转换为 `Scheduler` 中的优先级

`React` 中的优先级主要以下几种

```js
export const ImmediatePriority: ReactPriorityLevel = 99;
export const UserBlockingPriority: ReactPriorityLevel = 98;
export const NormalPriority: ReactPriorityLevel = 97;
export const LowPriority: ReactPriorityLevel = 96;
export const IdlePriority: ReactPriorityLevel = 95;
// React-only.
export const NoPriority: ReactPriorityLevel = 90;
```

```js
function reactPriorityToSchedulerPriority(reactPriorityLevel) {
  switch (reactPriorityLevel) {
    case ImmediatePriority:
      return Scheduler_ImmediatePriority;
    case UserBlockingPriority:
      return Scheduler_UserBlockingPriority;
    case NormalPriority:
      return Scheduler_NormalPriority;
    case LowPriority:
      return Scheduler_LowPriority;
    case IdlePriority:
      return Scheduler_IdlePriority;
    default:
      invariant(false, 'Unknown priority level.');
  }
}
```

## 总结

上述过程只是简单描述了 `react` 到 `scheduler` 的过渡流程，通过删减分支流程，只梳理出其中的主流程，可以有这样一个大致流程图

![](https://img.ninnka.top/1624714200073-react%20to%20scheduler.png)


# Scheduler 如何调度

## 写在前面

为了便于理解 `Scheduler` 如何调度，需要先了解几个基本概念

在 `Scheduler` 中, 任务被分在了两个不同的队列中：

* 待调度的队列，也叫未过期队列 `timerQueue`
* 调度中的队列，也就任务队列 `taskQueue`

每个 `task` 都有一个 `expirationTime` “过期时间”，`expirationTime` 由 `startTime` 和 `timeout` 组成，`startTime` 是安排调度的开始时间

这两种队列怎么区分的呢？

* 如果 `startTime` “开始时间” > `currentTime` ”当前时间“，那么任务没有过期，任务”推入“ `timerQueue`
* 如果 `currentTime` ”当前时间“ >= `startTime` “开始时间”，那么任务已过期，任务“推入” `taskQueue`

伪代码大概长这样

```js
// currentTime 当前时间
// expirationTime 过期时间
if (currentTime >= startTime) {
  // 已过期
  push(taskQueue, task);
} else {
  // 未过期
  push(timerQueue, task);
}
```

虽然都在用 `Queue` “队列” 来命名变量，但它们实际并不是大家普遍认识的那个链表线性数据结构，这里留个悬念，后面我们在解密

<div align="center">
<img width=200 src="https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcSSIAJCP4Ryh-uLXYPudrUzmsENrWx_hv3CQQ&usqp=CAU"/>
</div>

## unstable_scheduleCallback

回到正题，前面说到 `react` 中任务或者任务队列最后会通过调用 `Scheduler_scheduleCallback` 来开启调度，那么 `Scheduler_scheduleCallback` 是怎么实现的呢？

`Scheduler_scheduleCallback` 本身就已经相当简洁易懂了，从上至下的逻辑没有断层，语义通畅

`Scheduler_scheduleCallback` 负责调度任务的创建和分配，调度的启动

```js
function unstable_scheduleCallback(priorityLevel, callback, options) {
  var currentTime = getCurrentTime();

  // -------------- 获取 “开始时间” startTime
  var startTime;
  if (typeof options === 'object' && options !== null) {
    // delay 是主动设置的任务延期时长
    var delay = options.delay;
    if (typeof delay === 'number' && delay > 0) {
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
  } else {
    startTime = currentTime;
  }
  // --------------

  // -------------- 根据不同的优先级设置 “超时时间” timeout
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
  // --------------

  // 设置截止时间
  var expirationTime = startTime + timeout;

  // 创建新的 task 对象
  var newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    // 保存了开始时间
    startTime,
    // 保存了截止时间
    expirationTime,
    sortIndex: -1,
  };

  if (startTime > currentTime) {
    // 开始时间大于当前时间，任务未过期
    newTask.sortIndex = startTime;
    // 推入 timerQueue
    push(timerQueue, newTask);
    // ------ 检查是否有执行中的 hostTimtout
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      if (isHostTimeoutScheduled) {
        // 有的话取消掉
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      // 重新启动 hostTimeout
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    // 任务已过期
    newTask.sortIndex = expirationTime;
    // 推入 taskQueue
    push(taskQueue, newTask);
    // 开始启动调度
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    }
  }
  // 返回当前 task 的引用，外部会获取这个然后挂载到 root 节点上
  return newTask;
}
```

总结一下，`Scheduler_scheduleCallback` 也就做了这几件事：

* 获取 “开始时间” startTime
* 根据不同的优先级设置 “超时时间” timeout
* 设置截止时间
* 创建新的 task 对象
* 如果任务未过期，把任务推入 `timerQueue`，检查是否有执行中的 `hostTimeout`，有的话取消掉，重新开启一个 `hostTimeout`
* 如果任务已过期，把任务推入 `taskQueue`，开始启动调度 `requestHostCallback(flushWork)`，`flushWork` 是真正需要执行的函数

![](https://img.ninnka.top/1624778324759-Scheduler_scheduleCallback.png)

细心的小伙伴应该发现了，`startTime` 用来判断任务是否过期，那么 `expirationTime` 存在的意义是啥？

<div align="center">
<img width=200 src="https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcT5h6kz3l6iPiGihIwmM1RNOLXP5uIib7Sc4c3IOHnZL3KLpaWOLmQ73BnP32lN_cFoqNU&usqp=CAU"/>
</div>

## expirationTime 过期时间

上面提到了 `startTime` 是用于判断任务是否已过期，讲道理似乎不需要 `expirationTime` 过期时间了

但是 `timerQueue` `taskQueue` 是个队列呀，一个队列里所有任务不可能都有一样的“优先级”吧😏

如果想区分一个队列里不同任务的优先级，给他们排个序，那要怎么办呢，关键就在 `expirationTime` 中了

`expirationTime` 用于标识一个任务具体的过期时间，当前任务在1分钟后过期跟10分钟后过期其实本质上都没有什么区别，因为都还没有过期，但是关键在于10分钟后过期的情况，可以把当前任务稍微放一放，把资源先给其他任务执行

这个就是 `expirationTime` 存在的理由

## push(timerQueue, newTask) & push(taskQueue, newTask)

任务不管是否过期，都会通过一个 `push` 方法”推入队列“中

这个 `push` 咋看一下很像数组中常用的 `Array.prototype.push`，但由于 `timerQueue` 和 `taskQueue` 的特殊结构，`push` 并不是简单推入而已

`push` 的代码也是相当简洁，当然并不是它没做什么事，只是对函数拆分的非常细

`push` 函数所在的文件是 `react/packages/scheduler/src/SchedulerMinHeap.js`

看到文件名，若有所思？

各位还记得前面挖到坑吗，`timerQueue` 和 `taskQueue` 的特殊结构

精通数据结构与算法的小伙伴们肯定已经发现了，两个队列都是用 `最小堆` 实现的

<div align="center">
<img width=120 src="https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTkJRmbec2P6hZKvoynSu96H8qi1mBI_0ysKlnOESWXHhM9KFk8vq2cPcc7oH9lMZHYxmY&usqp=CAU"/>
</div>

> 最小堆，是一种经过排序的完全二叉树，其中任一非终端节点的数据值均不大于其左子节点和右子节点的值。 ----百度百科。

```js
export function push(heap: Heap, node: Node): void {
  const index = heap.length;
  heap.push(node);
  siftUp(heap, node, index);
}
```

这里的 `heap` 是利用数组实现的

```js
type Heap = Array<Node>;
type Node = {|
  id: number,
  sortIndex: number,
|};
```

`最小堆` 有两个比较关键的操作，上浮和下层

通过代码可以看出，新增的节点时push到数组最末尾的，要构成最小堆，就需要判断新增的节点是否满足“数据值均不大于其左子节点和右子节点的值“这一条件

如果不满足，则需要把新增的节点上浮，这个过程在 `siftUp` 中

```js
function siftUp(heap, node, i) {
  let index = i;
  while (true) {
    // 这里的通过无符号右移一位来获取父节点的index
    const parentIndex = (index - 1) >>> 1;
    const parent = heap[parentIndex];
    if (parent !== undefined && compare(parent, node) > 0) {
      // 父节点大于子节点
      heap[parentIndex] = node;
      heap[index] = parent;
      index = parentIndex;
    } else {
      // 父节点小于子节点，退出
      return;
    }
  }
}
```

由于本文不详细讨论数据结构与算法，后面有机会再单独开一篇章分析 `最小堆` 😁

![](https://img.ninnka.top/1624780291235-pushQueue%28minHeap%29.png)


### timeout

不同优先级的任务有不同的超时时间，对于需要立即执行的任务呢？也会有超时时间吗？

`Scheduler` 中的超时时间设计的比较特别

```js
// Math.pow(2, 30) - 1
// 0b111111111111111111111111111111
var maxSigned31BitInt = 1073741823;

// 立即执行
var IMMEDIATE_PRIORITY_TIMEOUT = -1;
// 常用超时时间
var USER_BLOCKING_PRIORITY_TIMEOUT = 250;
// 普通优先级
var NORMAL_PRIORITY_TIMEOUT = 5000;
// 低优先级
var LOW_PRIORITY_TIMEOUT = 10000;
// 永不超时
var IDLE_PRIORITY_TIMEOUT = maxSigned31BitInt;
```

对于需要立即执行的任务，超时时间是-1，通过-1可以使 `expirationTime` 尽可能小，使得任务优先级更高，在 `taskQueue` 中的排序也会靠前（通过最小堆 `siftUp`）
## requestHostTimeout & cancelHostTimeout

在 `Scheduler_schedulerCallback` 中，未过期的任务流程中出现了这两个奇怪的函数，相信第一眼看上去肯定是懵逼😳的，因为这函数名着实让人猜不透😂

猜不透就不猜了，我们来看看它们都做了啥

**requestHostTimeout**
```js
requestHostTimeout = function(callback, ms) {
  taskTimeoutID = setTimeout(() => {
    callback(getCurrentTime());
  }, ms);
};
```

**cancelHostTimeout**
```js
cancelHostTimeout = function() {
  clearTimeout(taskTimeoutID);
  taskTimeoutID = -1;
};
```

好家伙，这两个基友合着就是一个 `setTimeout` 控制器

<div align="center">
<img width=180 style="background: #fff" src="https://p3.pstatp.com/origin/pgc-image/3772c86f9b8a47b0b1844fbafb3ae8ca"/>
</div>

看来重点不在这个 `setTimeout` 上，回头看看 `Scheduler_schedulerCallback`

```js
requestHostTimeout(handleTimeout, startTime - currentTime);
```

第二个是 `timeout` 时长，第一个应该就是回调函数了，嫣然回首，那人却在灯火阑珊处

好家伙，重点原来是你 `handleTimeout`

## handleTimeout

这函数也是短小精悍，到这里虽然绕了个远路，不过不要紧，我们先好好交流♂

```js
function handleTimeout(currentTime) {
  isHostTimeoutScheduled = false;
  advanceTimers(currentTime);
  
  // 判断是否已启动调度任务回调
  if (!isHostCallbackScheduled) {
    if (peek(taskQueue) !== null) {
      // 当前 调度中的队列 不为空
      isHostCallbackScheduled = true;
      // 启动调度任务回调
      requestHostCallback(flushWork);
    } else {
      // 如果 当前 调度中的队列 为空
      // 获取待调度的任务中的第一个
      const firstTimer = peek(timerQueue);
      if (firstTimer !== null) {
        // 计算待调度的任务距离现在还有多久才会开始进入调度，并设置为timeout参数
        // 重新执行 handleTimeout
        requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
      }
    }
  }
}
```

我们先来分析代码看看它做了啥

* 执行 `advanceTimers`
* 判断是否已启动调度任务回调
* 如果当前 调度中的队列 不为空并且有调度中的任务，那么启动调度任务回调
* 如果当前 调度中的队列 为空，获取待调度的任务中的第一个，计算待调度的任务距离现在还有多久才会开始进入调度，并设置为timeout参数，重新执行 `handleTimeout`

咋看一下，我们好像把 `handleTimeout` 分析完了🤔，但是还存在不少疑点

* `advanceTimers` 是什么
* 又见到了 `requestHostCallback`，它具体做了什么
* 为什么 ”计算待调度的任务距离现在还有多久才会开始进入调度，并设置为 timeout 参数“，然后 ”重新执行 handleTimeout“

综合以上几个疑点，访问这个函数最最最关键的流程在于

```js
if (peek(taskQueue) !== null) {
  // 当前 调度中的队列 不为空
  isHostCallbackScheduled = true;
  // 启动调度任务回调
  requestHostCallback(flushWork);
}
```

做了这么多骚操作似乎都是为了能进入到这个流程里，回头看看 `else` 中的流程

```js
else {
  // 如果 当前 调度中的队列 为空
  // 获取待调度的任务中的第一个
  const firstTimer = peek(timerQueue);
  if (firstTimer !== null) {
    // 计算待调度的任务距离现在还有多久才会开始进入调度，并设置为timeout参数
    // 重新执行 handleTimeout
    requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
  }
}
```

为什么 ”计算待调度的任务距离现在还有多久才会开始进入调度，并设置为 timeout 参数“，然后 ”重新执行 handleTimeout“，`handleTimeout` 就能进入到 `if` 的流程中了呢？

`if` 和 `else` 都没找到答案，显然我们应该先瞅瞅 `advanceTimers` 是何方神圣

## advanceTimers

`advanceTimers` 从函数名上理解的话，大概是把 `timers` 提前的意思

提前 `timers`？使 `timers` 提前运行？使 `timers` 待调度的任务提前执行？

似乎有点那味了，我们还是来看看代码

```js
function advanceTimers(currentTime) {
  // 获取第一个待调度的任务
  let timer = peek(timerQueue);
  while (timer !== null) {
    // 如果任务不为空
    if (timer.callback === null) {
      // 没有 callback 说明已经执行完了或者是个空任务，直接忽略
      pop(timerQueue);
    } else if (timer.startTime <= currentTime) {
      // 任务超时了，需要执行了
      pop(timerQueue);
      // 标记排序用的标识
      timer.sortIndex = timer.expirationTime;
      // 把 timer 移动到 taskQueue
      push(taskQueue, timer);
      // ....
    } else {
      return;
    }
    // 如果出现有任务被移动或者移除的情况，检查下一个 timer
    timer = peek(timerQueue);
  }
}
```

总结一下做了啥事：

* 获取第一个待调度的任务，并且任务不为空
* 判断是否有callback，没有 callback 说明已经执行完了或者是个空任务，直接忽略
* 判断任务是否超时，超时的情况从 `timerQueue` 取出来，`push` 到 `taskQueue`，并更新排序标记
* 如果出现有任务被移动或者移除的情况，检查下一个 timer

虽说 `advanceTimers` 是个 `while` 循环，但是触发条件必须在 `taskQueue` 为空的时候

综上所述，`advanceTimers` 其实是任务分配器，用于”把不需要再等待调度的任务从 `timerQueue` 移动到 `taskQueue`“（这也更提不提前没啥关系呀，确实没啥关系）

把 `advanceTimers` 和 `handleTimeout` 的结合起来重新思考下，它们的流程大概是这样的

![](https://img.ninnka.top/1624786716030-handleTimeout.png)

![](https://img.ninnka.top/1624786804192-advanceTimers.png)

## requestHostCallback

结束了 `hostTimeout` 和 `timers` 的恩恩怨怨，我们回过头来分析下 `requestHostCallback`

`requestHostCallback` 是调度的第一步：注册任务，并通知调用

```js
requestHostCallback = function(callback) {
  scheduledHostCallback = callback;
  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true;
    port.postMessage(null);
  }
};
```

代码很简单，我们先分析记录一下

* 保存任务到 `scheduledHostCallback`
* 标记 `isMessageLoopRunning`，标记消息轮询开始
* `port.postMessage(null)` 发送一个消息通知

这个流程中的关键应该在于 `scheduledHostCallback` 和 `port.postMessage(null)`

但是 `scheduledHostCallback` 用在哪里？`port.postMessage(null)` 中的 `port` 是什么，消息发给谁，是为了做什么？

带着问题我们继续往下看

`requestHostCallback` 在 `react/packages/scheduler/src/forks/SchedulerHostConfig.default.js` 中

观察文件中代码，可以发现，这个文件初次执行时，会初始化 `requestHostTimeout` `cancelHostTimeout` `requestHostCallback` `performWorkUntilDeadline`   `forceFrameRate` 等函数

`requestHostTimeout` `cancelHostTimeout` 现在应该都有了解了

但是 `requestHostCallback` `performWorkUntilDeadline`  `forceFrameRate` `shouldYieldToHost` 就很陌生

别担心，直觉告诉我，刚刚问题的答案很可能在 `performWorkUntilDeadline` 里

## performWorkUntilDeadline

从函数名上分析大概可以知道，这个函数会用来处理任务中的 callback，直到任务超过最大可执行时长

```js
const channel = new MessageChannel();
const port = channel.port2;
channel.port1.onmessage = performWorkUntilDeadline;
```

```js
const performWorkUntilDeadline = () => {
  if (scheduledHostCallback !== null) {
    const currentTime = getCurrentTime();
    // 截止时间点，在当前时间上加 yieldInterval
    // yieldInterval 可以理解为最大可执行时长，也就是常说的时间切片，每片5ms
    deadline = currentTime + yieldInterval;
    // 是否有剩余时间
    const hasTimeRemaining = true;
    try {
      // 调用了 scheduledHostCallback，并保存返回结果
      const hasMoreWork = scheduledHostCallback(
        hasTimeRemaining,
        currentTime,
      );
      if (!hasMoreWork) {
        // 如果 scheduledHostCallback 返回 false，那么任务结束
        isMessageLoopRunning = false;
        scheduledHostCallback = null;
      } else {
        // 如果 scheduledHostCallback 返回不为 false，那么发送消息，重新调度执行
        port.postMessage(null);
      }
    } catch (error) {
      // 如果 scheduledHostCallback 返回不为 false，那么发送消息，重新调度执行，并抛出错误
      port.postMessage(null);
      throw error;
    }
  } else {
    isMessageLoopRunning = false;
  }
  needsPaint = false;
};
```

果不其然，`performWorkUntilDeadline` 中控制了 `scheduledHostCallback` 的执行

分析总结一下：这里分为两部分

第一部分：
* 创建一个 `MessageChannel` 实例
* 为 `port1.onmessage` 注册 `performWorkUntilDeadline`

第二部分：
* 设置截止时间点，在当前时间上加 `yieldInterval`，`yieldInterval` 可以理解为最大可执行时长，也就是常说的时间切片，每片5ms
* 调用了 scheduledHostCallback，并保存返回结果
* 如果 scheduledHostCallback 返回 false，那么任务结束
* 如果 scheduledHostCallback 返回不为 false，那么发送消息，重新调度执行
* 如果 scheduledHostCallback 返回不为 false，那么发送消息，重新调度执行，并抛出错误
* `isMessageLoopRunning` 置为 false，标记消息轮询结束

细心的小伙伴应该发现了，`port1.onmessage = performWorkUntilDeadline`，在 `performWorkUntilDeadline` 中调用 `port.postMessage(null)`，不是会触发 `performWorkUntilDeadline` 的执行吗？？？

是的，没错！这就是实现恢复执行的第一步，到此还不算恢复中断任务，先留个坑接着往下看

## MessageChannel

大家肯定听说过 `requestIdleCallback`，`requestAnimationFrame`，但是这两个 api 的不稳定让 `Scheduler` 放弃了它们，最终利用 `MessageChannel` 来人为控制调度频率，这个调度频率可以理解为每个任务的可执行最大时长

说到这里可能大家对 `MessageChannel` 还是不了解，可以回想下 `iframe`，与父页面通信时，通常会使用 `postMessage`，它们的兼容性是真的好

![](https://img.ninnka.top/1624794783571.png)

而且用起来也简单

```js
var channel = new MessageChannel();
function handleMessage(e) {
  alert(e.data);
}
channel.port1.onmessage = handleMessage;
channel.port2.postMessage('hello react scheduler~');
```

![](https://img.ninnka.top/1624795445360.png)

`MessageChannel` 的任务是 `macrotask`，优先级要比 `Promise` 低

![](https://img.ninnka.top/1624795518834.png)

这个 `MessageChannel` 同样可以使用 `postMessage` 在两个端口之间实现通信，具体可以自行查阅[MDN-MessageChannel](https://developer.mozilla.org/zh-CN/docs/Web/API/MessageChannel/MessageChannel)

## yieldInterval & forceFrameRate

终于到大家都熟知的”时间切片“了😄，每个任务的可执行最大时长默认设置为 5ms，每帧16ms总时长，任务执行占5ms，配合 `MessageChannel` 后粒度控制比起原生的 `requestIdleCallback`，`requestAnimationFrame` 要稳定的多了。

```js
// 默认为 5ms
let yieldInterval = 5;
```

前面提到了 `yieldInterval` 默认为 5ms 是对于 60Hz 刷新率的显示器，这个可能还不错，但是对于刷新率底的显示器，可能就是那么合理了

所以，`Scheduler` 内部会自行设置 `yieldInterval` 的方法，当然也提供了入口让外部设置

```js
// 可以自行设置 fps，范围在 0 ~ 125
forceFrameRate = function(fps) {
  if (fps < 0 || fps > 125) {
    console['error'](
      'forceFrameRate takes a positive int between 0 and 125, ' +
        'forcing frame rates higher than 125 fps is not supported',
    );
    return;
  }
  if (fps > 0) {
    // 0 ~ 125 之间的刷新率可以自动计算
    yieldInterval = Math.floor(1000 / fps);
  } else {
    // reset the framerate
    yieldInterval = 5;
  }
};
```

到此关于 `Scheduler` 的调度流程都已经结束了

![](https://img.ninnka.top/1624796966538-performWorkUntilDeadline%20%26%20Messagechannel.png)

`Scheduler` 实际是分为 `任务调度` 和 `任务执行` 两个部分的，前面留下的”恢复中断任务“的坑需要在执行中探讨

# Scheduler 如何执行

`Scheduler` 中负责执行的角色其实在前面已经提到了

在 `unstable_scheduleCallback` 中提到过 `requestHostCallback(flushWork)`，`flushWork` 才是真正负责执行任务的 **执行者**

```js
function flushWork(hasTimeRemaining, initialTime) {
  isHostCallbackScheduled = false;
  if (isHostTimeoutScheduled) {
    // 要开始执行了，不需要再等待 待调度任务 进入调度队列了，直接取消掉
    isHostTimeoutScheduled = false;
    cancelHostTimeout();
  }
  // 标记在执行中了
  isPerformingWork = true;
  // 保存当前的优先级
  const previousPriorityLevel = currentPriorityLevel;
  try {
    // ... 这里有些开发环境的性能收集代码，忽略即可
    // 交给小弟 workLoop 去做任务中断与恢复了
    return workLoop(hasTimeRemaining, initialTime);
  } finally {
    // 标记当前无任务执行
    currentTask = null;
    // 恢复优先级
    currentPriorityLevel = previousPriorityLevel;
    // 标记执行结束
    isPerformingWork = false;
    // ... 这里有些开发环境的性能收集代码，忽略即可
  }
}
```

分析总计一下 `flushWork` 做了啥：

* 取消掉 `hostTimeout`，因为要开始执行了，不需要再等待 待调度任务 进入调度队列了
* 标记在执行中了 `isPerformingWork = true`
* 保存当前的优先级
* 交给小弟 `workLoop` 去做任务中断与恢复了
* 执行结束后，标记当前无任务执行，恢复优先级，标记执行结束 `isPerformingWork = false`

`flushWork` 代码还是挺简单的，因为负责的事情都交给小弟 `workLoop` 去干了

我们常说的任务恢复与中断都在小弟 `workLoop` 中执行

## workLoop 任务中断与任务恢复

虽说刚刚提到 `flushWork` 是执行者，但是很多脏活累活都是 `workLoop` 在做，比如老生常谈的 `任务中断与任务恢复`

<div align="center">
<img width=180 style="background: #fff" src="https://img.ninnka.top/1624803015163-a7s52-ifhqj.png"/>
</div>

我们看看 `workLoop` 具体是怎么做的，代码还挺长，下面会做详细解读

```js
function workLoop(hasTimeRemaining, initialTime) {
  let currentTime = initialTime;
  // 熟悉的 advanceTimers，先把过期的任务从 timerQueue 捞出来丢到 taskQueue 打包一块执行了
  advanceTimers(currentTime);
  // 获取优先级最高的任务
  currentTask = peek(taskQueue);
  // 循环任务队列
  while (
    currentTask !== null &&
    !(enableSchedulerDebugging && isSchedulerPaused)
  ) {
    if (
      currentTask.expirationTime > currentTime &&
      (!hasTimeRemaining || shouldYieldToHost())
    ) {
      // 如果没有剩余时间或者该停止了就退出循环
      break;
    }
    const callback = currentTask.callback;
    if (typeof callback === 'function') {
      // 只有 callback 为函数时才会被识别为有效的任务
      currentTask.callback = null;
      // 设置执行任务的优先级，回想下 flushWork中的恢复优先级，关键就在这
      currentPriorityLevel = currentTask.priorityLevel;
      const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;
      // 。。。
      const continuationCallback = callback(didUserCallbackTimeout);
      currentTime = getCurrentTime();
      if (typeof continuationCallback === 'function') {
        // 这里是真正的恢复任务，等待下一轮循环时执行
        currentTask.callback = continuationCallback;
        // ....
      } else {
        // ... 不需要恢复任务了，标识当前任务已执行完，把任务从队列中移除掉
        // 因为被中断的任务是
        if (currentTask === peek(taskQueue)) {
          pop(taskQueue);
        }
      }
      // 熟悉的 advanceTimers，先把过期的任务从 timerQueue 捞出来丢到 taskQueue 打包一块执行了
      advanceTimers(currentTime);
    } else {
      // callback 为空，不是有效的任务或者已经执行完了，直接移除掉
      pop(taskQueue);
    }
    // 获取最高优先级的任务（不一定是下一个任务）
    currentTask = peek(taskQueue);
  }
  
  if (currentTask !== null) {
    // 还有任务说明调度被暂停了，返回true标明需要恢复任务
    return true;
  } else {
    const firstTimer = peek(timerQueue);
    if (firstTimer !== null) {
      // 任务都跑完了，又到了熟悉的 requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime)
      requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
    }
    // 返回false意味着当前任务都执行完了，不需要恢复
    return false;
  }
}
```

相信”代码太长，不想看“的各位已经直接翻到这里了

老规矩，总结分析一波：

* 在循环 `taskQueue` 之前，先通过 `advanceTimers` 把过期的任务从 `timerQueue` 捞出来丢到 `taskQueue` 打包一块执行了
* 获取优先级最高的任务作为第一个处理的任务
* 进入循环，在执行任务前，先看看还有没有时间
* 如果没有时间，跳出循环，返回 true，标明需要恢复任务
* 如果有时间，正常执行任务，并保存任务的返回值
* 如果返回值是函数，说明任务执行时长不够了，需要恢复
* 如果返回值不是函数，说明已经执行完了，从队列中移除当前任务
* 每个任务执行后（不一定执行完），都通过 `advanceTimers` 把过期的任务从 `timerQueue` 捞出来丢到 `taskQueue`，因为在执行过程中有可能部分任务也过期了

结合 `flushWork` 和 `workLoop` 来看，流程大概是这样的

![](https://img.ninnka.top/1624801928271-flushWork%20%26%20workLoop.png)


## shouldYieldToHost

执行的流程基本已结束，但有一个还需要提一嘴的函数 `shouldYieldToHost`

这个函数用来判断是否需要等待

```js
const scheduling = navigator.scheduling;
shouldYieldToHost = function() {
  const currentTime = getCurrentTime();
  if (currentTime >= deadline) {
    // 任务执行已超出时间分片的允许范围
    // 判断一下浏览器的渲染进程是否在工作中，是否有用户交互
    if (needsPaint || scheduling.isInputPending()) {
      // 如果有，就认为当前任务需要停止了
      return true;
    }
    // 如果浏览器很空闲，那么再给些时间执行任务
    return currentTime >= maxYieldInterval;
  } else {
    // 时间还充足，不需要停下
    return false;
  }
};
```

函数主要通过比较当前时间和任务执行截止时间，如果 `currentTime >= deadline` 那么任务超出时间分片的允许范围，需要暂停

比较惊喜的是，这个函数用到了一个新的web api，`navigator.scheduling`

这个新的 api 就很有意思了，它是 facebook 对浏览器贡献的第一个 api [isinputpending-api](https://engineering.fb.com/2019/04/22/developer-tools/isinputpending-api/)

![](https://habrastorage.org/webt/ot/me/yt/otmeytc2idaafvqyj9c3dcekdw4.jpeg)

感兴趣的小伙伴可以自行查阅

# 总结

到此，对 `Scheduler` 解读就暂告一段落了，主要是对主流程分支的代码做了一次解读，其实分支流程中还有些等待发现的奥秘。目前基于 `react v17.0.2` `scheduler v0.20.0` 分析，以后源码若有更新会尽早同步

潇潇洒洒 35300+ 字，希望看到这里的小伙伴给个赞👍🏻