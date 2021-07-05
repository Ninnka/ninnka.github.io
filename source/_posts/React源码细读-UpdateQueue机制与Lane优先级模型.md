---
title: React源码细读-UpdateQueue机制与Lane优先级模型
date: 2021-07-03 18:56:30
categories:
	- Web
	- JavaScript
	- React
	- Lane
tags:
	- React
	- JavaScript
	- Lane
	- UpdateQueue
---

# React UpdateQueue

使用过 `React` 的小伙伴肯定也都知道 `React` 在事件回调或生命周期回调中会对同步任务内的更新做批量处理

那么这里的批量更新又是通过什么机制去实现的呢？

> **此文基于 `react v17.0.2` 分析，[仓库传送门](https://github.com/facebook/react)**

## UpdateQueue 的创建

### Update 的结构

每次更新都新创建一个 `Update` 对象，这个 `Update` 对象的结构如下

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

`Update` 对象创建后，会通过 `enqueueUpdate` 函数，挂载到 `Fiber` 实例的 `updateQueue` 队列中


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

### UpdateQueue 的结构

```js
export type UpdateQueue<State> = {|
  baseState: State,
  firstBaseUpdate: Update<State> | null,
  lastBaseUpdate: Update<State> | null,
  shared: SharedQueue<State>,
  effects: Array<Update<State>> | null,
|};
```

谈到队列，普遍印象中的队列应该是先进先出，先进来排前面，按照进入的时间顺序排列

但是 `React UpdateQueue` 的结构比较，`pending` 指向的是最后一个 `Update` 对象，第一个进入 `Update` 对象排在第一个，后面依次按照进入的顺序排列

<!-- more -->

假设有这样一串代码，点击按钮后，会依次执行四次 `setState`

```js
class A extends React.PureComponent {
  state = {
    val: 1
  };

  onClickBtn = () => {
    this.setState({ val: 2 });
    this.setState({ val: 3 });
    this.setState({ val: 4 });
    this.setState({ val: 5 });
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
```

最后会得到一个怎样的 `UpdateQueue` 呢？我们一步一步分析

当第一个 `setState` 执行后，可以看到生成的 `Update` 对象结构以及数据是这样的

![](https://tva1.sinaimg.cn/large/008i3skNgy1grorehzctkj30zq0l0gq1.jpg)

经过 `enqueueUpdate` 处理后

![](https://img.ninnka.top/1625371355182.png)

![](https://img.ninnka.top/1625371473204.png)

这里的链表其实是一个环形结构，`pending` 永远指向最新的 `update`，最新的 `update` 排在第一个位置

这么说可能会很绕，我们来看看关键代码

```js
const pending = sharedQueue.pending;
if (pending === null) {
  // 第一个 update 进入队列，创建一个环形结构
  update.next = update;
} else {
  // 最新的 update 的下一个指向第一个进入的 update
  update.next = pending.next;
  // 最新的 update 插入到第一个位置中
  pending.next = update;
}
// pending 指向最新的 update
sharedQueue.pending = update;
```

上面的例子中，当我们执行完四个 `setState` 后

```js
this.setState({ val: 2 });
this.setState({ val: 3 });
this.setState({ val: 4 });
this.setState({ val: 5 });
```

会得到这样一个 `UpdateQueue`

![](https://img.ninnka.top/1625372015491-%E6%9B%B4%E6%96%B0%E9%98%9F%E5%88%97.png)

从 debugger 中的变量可以更加清晰的看到

![](https://img.ninnka.top/1625378527601.png)

有小伙伴可能会问，为什么要做成环形链表？普通的一样可以解决问题

做成环形链表可以只需要利用一个指针，便能找到第一个进入的节点和最后一个进入的节点，更加方便的找到最后一个 `Update` 对象，同时插入新的 `Update` 对象也非常方便。如果使用普通的线性链表，想跟环形一样插入和查找都方便的话，就需要同时记录第一个和最后一个节点的位置，维护成本相较于环形肯定是更高了

## UpdateQueue 状态管理

上一节中提到了 `UpdateQueue` 的数据结构，其中有三个用于状态管理的数据节点

```js
export type UpdateQueue<State> = {|
  baseState: State,
  firstBaseUpdate: Update<State> | null,
  lastBaseUpdate: Update<State> | null,
  // ... 其他节点
|};
```

这几个 `state` 分别存储了在创建和更新阶段时的不同状态值

先来看看 `UpdateQueue` 的创建

```js
export function initializeUpdateQueue<State>(fiber: Fiber): void {
  const queue: UpdateQueue<State> = {
    baseState: fiber.memoizedState,
    firstBaseUpdate: null,
    lastBaseUpdate: null,
    shared: {
      pending: null,
    },
    effects: null,
  };
  fiber.updateQueue = queue;
}
```

`initializeUpdateQueue` 用于初始化 `UpdateQueue`，我们这次主要关心 `baseState` `firstBaseUpdate` `lastBaseUpdate` 的创建

* `baseState`: 前一次更新计算得出的状态，比如：创建时是声明的初始值 `state`，更新时是最后得到的 `state`（除去因优先级不够导致被忽略的 `Update`）
* `firstBaseUpdate` 更新阶段中由于优先级不够导致被忽略的第一个 `Update` 对象
* `lastBaseUpdate` 更新阶段中由于优先级不够导致被忽略的最后一个 `Update` 对象

`baseState` 应该很好理解，但是 `firstBaseUpdate` `lastBaseUpdate` 又是什么鬼

<div align="center">
<img width=120 src="https://img.ninnka.top/1625385692857.png"/>
</div>

对于 `firstBaseUpdate` `lastBaseUpdate`，有个很关键的因素，那就是优先级，这两个属性可以理解为一个队列的起点指针和结尾指针，这个队列表示的是“低优先级 `UpdateQueue`”

这么说可能有个基本的概念了，我们再回想 `Update` 对象的基本结构，里面有个很重要的属性：`lane`

```js
// @flow
export type Update<State> = {|
  // ... 其他属性
  lane: Lane,
  tag: 0 | 1 | 2 | 3,
  // ... 其他属性
|};
```

为了方便理解，我们更新一下 `UpdateQueue` 结构的展示效果

![](https://img.ninnka.top/1625382865726-updateQueue%20with%20lane.png)

我们给不同的 `Update` 对象添加了一个假设的 `lane` 值

到这里大家应该能够对 `UpdateQueue` 的基本结构和创建过程有一个比较清晰的概念了

但是我们仍有两个疑问待探讨

* 关于 `UpdateQueue` 的更新过程
* `Lane 优先级机制` 是什么

我们先来看看 `UpdateQueue 更新机制`

## UpdateQueue 更新机制

前面描述了 `UpdateQueue` 是如何创建的，以及 `UpdateQueue` 中存在哪些关键的属性

其中关于 `firstBaseUpdate` `lastBaseUpdate`，需要在 `processUpdateQueue` 中慢慢品味

### processUpdateQueue

这个函数的名字很好猜，顾名思义：处理 `UpdateQueue`

在 `react` 的更新流程中，如果是 `ClassComponent` 或者 `HostRoot`，会走到 `processUpdateQueue` 函数内，函数比较长，为了避免陷入各种细节中，我们先来做一个简单的思考：

现在有一个链表，里面存储着需要更新的数据，每个更新都有自己的优先级，我们给函数传入一个优先级区间 `renderLanes`，**高于或等于这个优先级区间的更新可以执行，不满足优先级条件的需要暂存起来放到下一次处理流程中**

因为是个环形链表，我们肯定需要遍历进行处理，遍历需要一个停止的点，那么具体是什么呢？先提供两个选择

1. 判断到当前 `Update` 为 `pending` 时指针停止
2. 把环形链表解开，从头到位遍历即可

开始循环链表前，把延迟执行的 `update` 一起拼接起来遍历

开始遍历后，如果 `update` 满足条件，就计算新的 `state`；如果出现 `update` 不满足条件

对于更新流程我们做了上述一个简单的思考，带着疑问和思考我们来看看简化后的代码

```js
export function processUpdateQueue<State>(
  workInProgress: Fiber,
  props: any,
  instance: any,
  renderLanes: Lanes,
): void {
  // 队列的别名
  const queue: UpdateQueue<State> = (workInProgress.updateQueue: any);
  // ...

  // 先存了 firstBaseUpdate 和 lastBaseUpdate
  let firstBaseUpdate = queue.firstBaseUpdate;
  let lastBaseUpdate = queue.lastBaseUpdate;

  let pendingQueue = queue.shared.pending;
  if (pendingQueue !== null) {
    // 如果队列不为空，则把 fiber 中的 queue pending 指针置位空
    queue.shared.pending = null;
    // 记录第一个和最后一个 Update，同时把环形链表的环拆开
    const lastPendingUpdate = pendingQueue;
    const firstPendingUpdate = lastPendingUpdate.next;
    lastPendingUpdate.next = null;
    if (lastBaseUpdate === null) {
      // 如果 lastBaseUpdate 为空，firstBaseUpdate 和 lastBaseUpdate 就被赋值为 queue 的起点和终点
      firstBaseUpdate = firstPendingUpdate;
    } else {
      // 如果 lastBaseUpdate 不为空，那么会拼接到 queue 的前面
      lastBaseUpdate.next = firstPendingUpdate;
    }
    lastBaseUpdate = lastPendingUpdate;

    // 上述的操作其实是把 queue 环形队列解环
    // 然后把 firstBaseUpdate 和 lastBaseUpdate 构成的队列拼接到 queue 前面
    // 最后构成一个大的线性 UpdateQueue
    // firstBaseUpdate 和 lastBaseUpdate 分别作为队列的起点和终点

    const current = workInProgress.alternate;
    if (current !== null) {
      // 执行类似上述的操作，把 workInProgress 节点中的 queue 同步到 current 节点
    }
  }

  if (firstBaseUpdate !== null) {
    // 新的 state
    let newState = queue.baseState;
    // 新的 lane 优先级
    let newLanes = NoLanes;

    let newBaseState = null;
    let newFirstBaseUpdate = null;
    let newLastBaseUpdate = null;

    // 第一个 update
    let update = firstBaseUpdate;

    // 队列的循环处理
    do {
      const updateLane = update.lane;
      const updateEventTime = update.eventTime;
      if (!isSubsetOfLanes(renderLanes, updateLane)) {
        // 这里通过 lane 的工具函数判断出当前 update 的 lane 不满足更新优先级条件
        // 下面是把不满足更新条件的 update 用链表存了起来
        const clone: Update<State> = {
          eventTime: updateEventTime,
          lane: updateLane,

          tag: update.tag,
          payload: update.payload,
          callback: update.callback,

          next: null,
        };
        // newFirstBaseUpdate 和 newLastBaseUpdate 是队列的起点和终点
        if (newLastBaseUpdate === null) {
          newFirstBaseUpdate = newLastBaseUpdate = clone;
          // 这里有个很特殊的处理，如果出现有update被延迟执行，
          // 那么会把当前已经计算好的 newState 先做一次保存
          newBaseState = newState;
        } else {
          newLastBaseUpdate = newLastBaseUpdate.next = clone;
        }
        // 更新 update 的 lane，往里塞入了满足条件的优先级，这样下次遍历到时才能执行
        newLanes = mergeLanes(newLanes, updateLane);
      } else {
        // 满足更新条件的 update
        if (newLastBaseUpdate !== null) {
          // 注意这里的分支，如果前面出现过有 update 被推迟
          // 那么后面所有任务都必须进入到被延迟的队列中
          const clone: Update<State> = {
            eventTime: updateEventTime,
            // 这里通过设置 NoLane 保证了下一次检查时一定会通过上面的 isSubsetOfLanes
            lane: NoLane,

            tag: update.tag,
            payload: update.payload,
            callback: update.callback,

            next: null,
          };
          newLastBaseUpdate = newLastBaseUpdate.next = clone;
        }

        // 计算新的 state
        newState = getStateFromUpdate(
          workInProgress,
          queue,
          update,
          newState,
          props,
          instance,
        );
        // 保存 setState 的 callback，就是第二个参数
        const callback = update.callback;
        if (callback !== null) {
          // 标记当前 fiber 有 callback
          workInProgress.flags |= Callback;
          const effects = queue.effects;
          if (effects === null) {
            queue.effects = [update];
          } else {
            effects.push(update);
          }
        }
      }
      // 下一个 update 对象
      update = update.next;
      if (update === null) {
        pendingQueue = queue.shared.pending;
        if (pendingQueue === null) {
          // 循环处理结束
          break;
        } else {
          // 当前的 queue 处理完后，需要检查一下 queue.shared.pending 是否有更新
          // 然后把剩余的 update 都推入到 lastBaseUpdate
          // ... 这里涉及到链表拆环的代码，与上面的拆环没有太大差异
        }
      }
    } while (true);

    if (newLastBaseUpdate === null) {
      // 这里有一个很关键的判断，只有在没出现update被延迟的情况下
      // 才会把整个队列的计算结果赋值给 newBaseState
      newBaseState = newState;
    }
    
    // newBaseState 就是新的 baseState，也就是后面界面展示时用到的 fiber.memoizedState
    queue.baseState = ((newBaseState: any): State);

    // 保存被延迟的 update
    queue.firstBaseUpdate = newFirstBaseUpdate;
    queue.lastBaseUpdate = newLastBaseUpdate;

    // 标记哪些区间的update被延迟了
    markSkippedUpdateLanes(newLanes);
    workInProgress.lanes = newLanes;

    // 只更新 workInProgress（新节点）的 memoizedState
    workInProgress.memoizedState = newState;
  }
}
```

TL;DR😂

因为太长没看代码的小伙伴估计已经翻到这里了

那么先总结一波，`processUpdateQueue` 函数做了这几件事：

* 先存了 `queue.shared.pending` `firstBaseUpdate` 和 `lastBaseUpdate` 的别名变量
* 如果 `pendingQueue` 不为空，记录第一个和最后一个 `Update`，同时把环形链表的环拆开；然后把 `firstBaseUpdate` 和 `lastBaseUpdate` 构成的队列拼接到 `queue` 前面，最后构成一个大的线性 `UpdateQueue`，`firstBaseUpdate` 和 `lastBaseUpdate` 分别作为队列的起点和终点
* 把 workInProgress 节点中的 queue 同步到 current 节点
* 循环遍历前面得到的线性更新队列，起点是 `firstBaseUpdate`
* 通过 `lane` 的工具函数判断出当前 `update` 的 `lane` 是否满足更新队列的优先级
* 如果不满足更新优先级条件，把不满足更新条件的 `update` 用链表存了起来，`newFirstBaseUpdate` 和 `newLastBaseUpdate` 是队列的起点和终点，这里有个很特殊的处理，如果出现有 `update` 被延迟执行，那么会把当前已经计算好的 `newState` 先做一次保存，然后更新 `update` 的 `lane`，往里塞入了满足条件的优先级，这样下次遍历到时才能执行
* 如果满足更新优先级条件，注意这里的分支，首先判断前面如果出现过有 update 被推迟那么后面所有任务都必须进入到被延迟的队列中，并且被推迟的 `Update` 对象的 `lane` 会被设置为 `NoLane` 等级了
* 然后通过 `getStateFromUpdate` 计算新的 `state`，存放在 `newState`；然后保存 `setState` 的 `callback`，就是第二个参数，接着标记当前 `fiber` 有 `callback`，存放在 `flags` 字段中
* 遍历到下一个 `Update` 对象
* 开始处理下一个 `Update` 对象前，查一下 queue.shared.pending 是否有更新，然后把剩余的 update 都推入到 lastBaseUpdate（这里涉及到链表拆环的代码，与上面的拆环没有太大差异）
* 这里有一个很关键的判断，只有在没出现update被延迟的情况下，才会把整个队列的计算结果 `newState` 赋值给 `queue.baseState`
* 保存被延迟的 `update`，标记哪些区间的 `update` 被延迟了，只更新 `workInProgress`（新节点）的 `memoizedState`

一口气梳理完发现就是有不少细节在里头，文字果然还是TL;DR😂

那么我们还是按老规矩看看图

![](https://img.ninnka.top/1625417693795-processUpdateQueue.png)

### getStateFromUpdate

在 `processUpdateQueue` 的流程中可以看到，计算 `newState` 时会通过 `getStateFromUpdate` 从 `Update` 对象计算出新的 `state`

对于不同的更新标识 `update.tag` 会做不同的处理，我们这次主要了解 `UpdateState` 这个流程，其他几个流程会做个简单介绍

先看看简化后的代码

```js
function getStateFromUpdate<State>(
  workInProgress: Fiber,
  queue: UpdateQueue<State>,
  update: Update<State>,
  prevState: State,
  nextProps: any,
  instance: any,
): any {
  // 根据不同的更新标识来做处理
  switch (update.tag) {
    // state 替换
    case ReplaceState: {
      const payload = update.payload;
      if (typeof payload === 'function') {
        // ... 
        const nextState = payload.call(instance, prevState, nextProps);
        // ...
        return nextState;
      }
      return payload;
    }
    // update 捕获
    case CaptureUpdate: {
      workInProgress.flags =
        (workInProgress.flags & ~ShouldCapture) | DidCapture;
    }
    // setState 的更新 state 基本流程
    case UpdateState: {
      const payload = update.payload;
      let partialState;
      if (typeof payload === 'function') {
        // ...
        // setState 接收的如果是函数
        partialState = payload.call(instance, prevState, nextProps);
        // ...
      } else {
        // 新的局部 state 对象，用于接下来的对象合并
        partialState = payload;
      }
      if (partialState === null || partialState === undefined) {
        // null 和 undefined 会直接忽略掉
        return prevState;
      }
      // 合并新旧 state
      return Object.assign({}, prevState, partialState);
    }
    // 熟悉的 this.forceUpdate
    case ForceUpdate: {
      hasForceUpdate = true;
      return prevState;
    }
  }
  return prevState;
}
```

简单总结一下 `getStateFromUpdate` 做的事情：

* `UpdateState` 是 `setState` 的更新 `state` 基本流程，这一步中主要是判断当前 `partialState` 是否函数，如果是函数则调用后获取返回值，否则直接把 `partialState` 作为 `newState` 的一部分，最后与原来的 `state` 做一次合并后返回
* `ReplaceState` 大致流程与 `UpdateState` 相似，但是最后不会做合并，而是直接把结果作为新的 `state` 返回
* `ForceUpdate` 大家应该都熟悉，这个阶段不会做 `state` 计算，只是把 `hasForceUpdate` 标记为 `true`，后续会用到
* `CaptureUpdate` 这个大家应该都没有怎么关注过，这个比较特别，在整个 `react` 的生命周期内，只有在渲染报错被 `componentDidCatch` 捕获之后，走 `getDerivedStateFromError` 流程，创建一个新的 `Update` 对象，这个对象会收集当前报错的一些信息，并更新到组件状态中

## Lane 优先级机制

不久前我们介绍过 `Scheduler` 的调度和执行机制，`Scheduler` 中根据一套内置的优先级规则，把任务分成了 `timer 待调度` 和 `task 调度中` 两部分

但这次提到的 `Lane` 机制是 `React` 内部中的一套优先级机制

`React` 在早期提出了任务优先级的概念，优先级最早是用 `expirationTime`，通过 `expirationTime` 来做任务调度，由于 `expirationTime` 的一些缺陷，`React` 又提出了新的 `Lane` 优先级模型

`Lane` 模型是 `react` 为了解决 `ExpirationTime` 模型导致的低优先级任务 `长时间等待/饿死` 的问题

目前 `react` 中共有 31 种不同的 `lane` 值，其中区分了不同的 `lane 车道` 与 `lanes 车道组`

我们想象一下赛车场，假设这个赛车场有31条车道，每5条车道组成一个车道区间，数字靠前的车道区间起点靠前，可以比别的车道更快出发，车道区间内的车道优先级一致，只是数字区分了不同的车道，比如：1，2，3，4，5是一个车道区间，分别在5个不同车道，一块出发

画了一个简易图的便于理解

![](https://img.ninnka.top/1625473060724-lanes%E8%BD%A6%E9%81%93%E5%8C%BA%E9%97%B4%E6%A8%A1%E5%9E%8B-%E8%B5%9B%E9%81%93%E5%9B%BE%20%281%29.png)

为了便于查找，这里把所有 `Lane/Lanes` 都列举了，感兴趣的可以仔细研究看看

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

### Lane 的计算

既然 `Lane` 可以用作优先级比较，那么这个比较过程是怎样的呢？

一般来说，比较不就是大于小于等于吗？比如两个 `lane` 值通过 `lane1 > lane2` 的方式得到结果

但是回想一下，刚刚不是才提到过 `Lane` 车道模型是有区间性质的，比如 `TransitionLanes` 有 16 个车道，从 `TransitionLane1` 到 `TransitionLane16`

这个需求如果是通过简单的 `>/=/<` 显然很难处理这种需求了

细心的小伙伴肯定已经发现，上面列举出的 `Lane` 都是二进制表示的

在 `processUpdateQueue` 函数中有用到两个比较关键的 `Lane` 操作函数

* `isSubsetOfLanes`
* `mergeLanes`

```js
export function isSubsetOfLanes(set: Lanes, subset: Lanes | Lane) {
  return (set & subset) === subset;
}

export function mergeLanes(a: Lanes | Lane, b: Lanes | Lane): Lanes {
  return a | b;
}
```

`isSubsetOfLanes` 用于判断当前的 `lane` 是否处在目标 `lanes` 上，如果满足这个条件，可以认为这个 `lane` 在对应的车道区间中

比如：有个 `lane` 是 `TransitionLane1`，当这个 `lane` 满足 `TransitionLanes` 的条件时，就做对应的更新流程。由于 `TransitionLanes` 是个范围区间，我们需要知道 `lane` 是否在 `TransitionLanes` 中，首先需要对他们进行 `&` 操作，然后把结果与 `TransitionLane1` 比较；因为如果 `TransitionLanes` 包含 `TransitionLane1`，那么 `&` 后的结果一定是 `TransitionLane1` 自身，那么最后肯定与 `TransitionLane1` 相等

`mergeLanes` 则是把目标 `lane` 或者 `lanes` 加入到当前的 `lane/lanes` 中

比如：当前有个 `lane` 是 `InputContinuousHydrationLane`，如果想让当前的 `lane` 满足 `InputContinuousLane` 的条件，我们可以在保持原本属性不变的情况，新增一个 `lane`，通过 `|` 操作可以合并两个 `lane/lanes`

此外，`react` 还提供了 `removeLanes` `includesSomeLane` 等常用的 `lane` 操作函数，原理就是基本的位运算，感兴趣的小伙伴可以看看源码

```js
// 文件路径 react/packages/react-reconciler/src/ReactFiberLane.js
export function includesSomeLane(a: Lanes | Lane, b: Lanes | Lane) {
  return (a & b) !== NoLanes;
}
// ...
export function removeLanes(set: Lanes, subset: Lanes | Lane): Lanes {
  return set & ~subset;
}

export function laneToLanes(lane: Lane): Lanes {
  return lane;
}
```

# 顺带说两句

看到这里的小伙伴应该对 `UpdateQueue` 的批处理机制有一定的了解了，同时对 `react` 的 `Lane` 优先级模型也有了初步的认识

对于 `Lane` 优先级模型这里先做了一些简单的介绍，后续会对 `Lane` 优先级模型和 `expirationTime` 模型做一些更加具体的对比分析

