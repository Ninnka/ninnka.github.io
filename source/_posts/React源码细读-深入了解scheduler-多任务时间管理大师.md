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

# Scheduler 如何执行

# Scheduler 任务中断与任务恢复

# 参考

[postmessage & scheduler](https://www.yuque.com/docs/share/8c167e39-1f5e-4c6d-8004-e57cf3851751)
