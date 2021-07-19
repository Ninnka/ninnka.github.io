---
title: React源码细读-Reconciliation In Concurrent Mode
date: 2021-07-18 15:24:14
categories:
	- Web
	- JavaScript
	- React
	- Reconciler
tags:
  - 前端
	- React
	- JavaScript
	- Reconciler
	- Reconcilition
	- ConcurrentMode
---

# Concurrent Mode

官方对于 `Concurrent Mode`，用了一个有趣的比喻，把 `Concurrent Mode` 隐喻为版本控制，版本控制大家都了解，平常开发项目时，大家会在同一个项目的不同分支中修改内容，比如修改同一个文件

假设现在没有版本控制，为了避免冲突，A同学在修改a文件时，B、C、D等其他同学就得等着A同学修改完然后释放文件，这种情况下会阻塞B，C，D等其他同学的开发进度

而 `Concurrent Mode` 则提供了一个类似分支的概念，把A，B，C等同学比作 `React` 中的 `render 任务`，浏览器的渲染进程，IO进程等等，当`React` 中的 `render任务` 执行时，可以不阻塞浏览器中的其他进程

其实严格来说，`Concurrent Mode` 比作版本控制也不是那么妥当，毕竟场景不是一模一样，只是希望大家能理解到其中的含义即可

个人理解的话，`React` 中的 `Concurrent Mode` 是指在 `Reconciler` 中处理 `long task` 时，可以不阻塞浏览器中的其他进程，并且 `React` 中的 `render 任务` 具有各自的优先级，任务可以通过**过时间分片** + **优先级调度**的方式在**执行**和**暂停**之间切换状态

# Legacy & Concurrent & Blocking

`React v17` 中提供了三种可选的方式来创建 `React` 应用，分别是 `Legacy Mode`，`Concurrent Mode`，`Blocking Moode`

`Legacy Mode` 是我们熟悉的传统模式，目前是通过 `ReactDOM.render` 来触发这个模式，`Legacy Mode` 下大家都熟悉，`Reconcile Fiber` 流程一步到位，不可中断，当页面中存在大量组件 `render` 时会导致时间过长，阻塞浏览器的渲染进程，页面响应速度慢，交互卡顿明显

`Concurrent Mode` 目的是为了能让渲染流程变的可中断，也就是我们现在常听到的 `Interruptible Rendering` - **可中断式渲染** 这一概念

`React` 计划在 v18 中正式默认启用 `Concurrent Mode`，但是让大型的 `React App` 一口气升级上来难免会遇到一些问题，主要体现在 `React` 现在的生态中会有些组件还在使用一些不安全的生命周期，一些老旧的 `React Library` 可能没办法跟 `Concurrent Mode` 兼容运行

为了能更加平滑过渡到 `Concurrent Mode`，新增了 `Blocking Mode`，目前 `React` v17 中则是采用了 `Blocking Mode` 的过渡模式，可以通过 `ReactDOM.createBlockingRoot` 开启过渡模式，在 `Blocking Mode` 期间，减少 `unsafe_liftcycle`，`String Refs`，`Legacy Context`，`findDOMNode` 等不稳定或者不兼容 api 的使用，等待 `React` 的生态逐步跟进后，再尝试使用 `React` v18 的新特性

![](https://img.ninnka.top/1626508076589.png)

[关于Blocking Mode特性的参考链接](https://reactjs.org/docs/concurrent-mode-adoption.html#feature-comparison)

如果你想提前体验 `Concurrent Mode`，可以通过 `ReactDOM.createRoot(rootNode).render(<App />)` 的方式主动开启**并发模式**

**不同模式下开启的方式可以参考这个（出自官方文档）**

> `Legacy Mode`: `ReactDOM.render(<App />, rootNode)`
> `Blocking Mode`: `ReactDOM.createBlockingRoot(rootNode).render(<App />)`
> `Concurrent Mode`: `ReactDOM.createRoot(rootNode).render(<App />)`

# Reconcile Fiber

**此文基于 `react v17.0.2` 分析，[仓库传送门](https://github.com/facebook/react)**

在前几个篇章中，解读过 `Scheduler` - **调度器** 的工作方式，`Scheduler` 通过**时间分片**控制了每个任务的最大执行时间，给任务设置不同的过期时间，分为 `timerQueue` 和 `taskQueue`，通过 `MessageChannel` 来手动调度 `taskQueue` 中每个任务的执行

`Scheduler` 的好处就在于，每个任务在有限的时间内完成部门或全部工作，每帧内的剩余时间可以留给浏览器做渲染或者页面的响应

`React` 通过 `Scheduler` 实现任务的中断与恢复，但是，仅此就够了吗？

假设其中一个任务是个 `while` 循环，每次循环都需要处理一下组件的 `render` 函数，遍历的节点有十万个，在这个任务结束之前，`Scheduler` 没法做下一步的剩余时间计算，在这个 `long task` 执行的期间，页面如果有交互产生的话，一般会出现我们常说的“掉帧”现象，因为计算资源目前被 JavaScript 的事件占据，浏览器需要等待资源释放才能够处理UI

`Concurrent Mode` 作为 `React` 发展方向上的一个重要环节，又提出了怎样的解决方案呢？

`Concurrent Mode` 的出现无非是为了解决这两大类的问题：

1. 受限于 `CPU` 的更新，比如 `render` 函数调用时的创建 `component dom`
2. 受限于 `IO` 的更新，比如：`fetching data from network`

小伙伴肯定会好奇，为啥 `Concurrent Mode` 能解决这些呢？再说这两个算啥问题？🤣

我们回到前面的说的那个 `long task` 任务处理问题，比如十万个节点执行 `render`，那肯定会耗费不少时间，而 `render` 就是创建 `component dom` 的过程

如果这个 `long task` 可以实现前面所说的“并发模式”的话，通过**过时间分片** + **优先级调度**的方式在**执行**和**暂停**之间切换状态，就像 `Scheduler` 那样，就可以解决第一类问题

`Concurrent Mode` 通过分割 `Reconcile` 中的任务，在适当的时机释放 `CPU` 支援，让浏览器的渲染进程可以更迅速的响应

第一类问题在开启 `Concurrent Mode` 后就会得到优化，而第二类问题通常是涉及到UI交互上的体验优化

比如这样一个场景：比如点击某个按钮后，展示loading，同时从后端拉取数据，但是这个拉取数据的过程很快，展示的loading可能会一闪而过，这个相信大家都常见🤣

又或者是这种场景：有一个下拉框，选中某个选项后，拉取接口数据，但是由于接口响应慢或者网络不顺畅，导致数据加载时间较长，此时你切换了另外一个选项，这两个选项最后几乎同一时间响应，你会看到，页面显示变成第一选项的结果然后又迅速切换到了最新的结果🤣

上面的场景中，无论是哪种，loading和结果的展示几乎完全取决于 `网络IO` 的速度，那么有没有一种办法可以平衡一下体验呢？

在 `Concurrent Mode` 下，`React` 提供了两个新的功能用于解决这个 `网络 IO` 下的 UI 交互问题

* `Suspense`
* `useTransition`

感兴趣的小伙伴可以看看官方文档提供的demo [Suspense with useTransition](https://codesandbox.io/s/jovial-lalande-26yep)（这里不对 `Suspense` 和 `useTransition` 做深入探讨，后续会单独开一个篇章分析）

这么听起来 `Concurrent Mode` 好像是挺厉害的🤣，那么我们来看看 `Concurrent Mode` 下的 `Reconciler` 是如何工作的

在分析之前，我先抛出几个疑问

* `Reconciler` 是如何利用 `Scheduler` 进行任务中断与恢复的
* `Reconciler` 遍历 `Fiber Tree` 时是怎么中断任务的，中断后为什么可以恢复到上次循环中断的位置
* 遍历 `Fiber Tree` 途中如果中断与恢复的中途出现更高优先级的任务，该如何处理
* 有些 willxxx 生命周期，比如 `componentWillReceiveProps`，为什么会执行两次

对这几个问题肯定有小伙伴会感到疑惑，没关系，这次带着问题盘他

<div align="center">
<img width=160 src="https://img.ninnka.top/1626596402374.png"/>
</div>

## performConcurrentWorkOnRoot

`performConcurrentWorkOnRoot` 是 `Scheduler` 开启调度时实际执行的任务，简单回顾一下 `ensureRootIsScheduled`

```js
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  let newCallbackNode;
  if (newCallbackPriority === SyncLanePriority) {
    // ....
  } else if (newCallbackPriority === SyncBatchedLanePriority) {
    // ....
  } else {
    const schedulerPriorityLevel = lanePriorityToSchedulerPriority(
      newCallbackPriority,
    );
    // 这里调度的任务就是 performConcurrentWorkOnRoot
    newCallbackNode = scheduleCallback(
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root),
    );
  }
  // 这里把 react lane priority 保存到了 root.callbackPriority
  root.callbackPriority = newCallbackPriority;
  // 这里把 Scheduler 返回的 Task 保存到了 root.callbackNode
  root.callbackNode = newCallbackNode;
}
```

只看部分核心代码，在调度的某个分支中，会传入 `performConcurrentWorkOnRoot` 函数作为需要调度的任务

在 `Scheduler` - **调度器** 开始调度 `Task` 后，会进入我们常说的 `Reconciler` - **协调器** 中，`performConcurrentWorkOnRoot` 就是 `Reconciler` 执行的第一步

```js
function performConcurrentWorkOnRoot(root) {
  // 进入 Reconciler 后，清除 currentEvent 的一些信息
  currentEventTime = NoTimestamp;
  currentEventWipLanes = NoLanes;
  currentEventPendingLanes = NoLanes;

  // 取出正在执行的 scheduler task
  const originalCallbackNode = root.callbackNode;
  // 遍历执行回调，比如 useEffect，setState 的回调
  const didFlushPassiveEffects = flushPassiveEffects();
  if (didFlushPassiveEffects) {
    // 执行完回调后有可能 scheduler 中正在执行的 task 已经发生了变化
    // 需要做一个判断是否需要继续执行这个任务
    if (root.callbackNode !== originalCallbackNode) {
      return null;
    } else {
      // 这个 else 现在没啥意义，可能只是留个占位吧，说不定后面会调整
    }
  }

  // 获取当前需要执行的 `lane/lanes`，用最高优先级的 `lane` 作为任务执行的优先级标准
  // 同时计算 lane 对应的 priority
  let lanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );

  // 开始 Reconciler 的 render 阶段
  // exitStatus 用来表示 render 流程的结果状态
  let exitStatus = renderRootConcurrent(root, lanes);
  // 省略一部分代码 ... 这里的中间代码会处理 exitStatus 在不同状态下的相关流程，我们先忽略
}
```

我们先来看 `performConcurrentWorkOnRoot` 的删减后的前半部分

* 进入 `Reconciler` 后，清除 `currentEvent` 的一些信息，留给下一次进入的事件
* 获取当前需要执行的 `lane/lanes`，用最高优先级的 `lane` 作为任务执行的优先级标准，同时计算 lane 对应的 priority（这个步骤很重要）
* 开始 `Reconciler` 的 `render` 阶段，`exitStatus` 用来表示 `render` 流程的结果状态

`renderRootConcurrent` 从函数名上看，似乎已经发现了什么，但是这里按下不表，大家先舒口气，我们继续分析 `performConcurrentWorkOnRoot` 的下半部分

```js
function performConcurrentWorkOnRoot(root) {
  // 省略一部分代码 ... 执行 renderRootConcurrent 前的一些准备

  // 开始 Reconciler 的 render 阶段
  // exitStatus 用来表示 render 流程的结果状态
  let exitStatus = renderRootConcurrent(root, lanes);

  if (
    includesSomeLane(
      workInProgressRootIncludedLanes,
      workInProgressRootUpdatedLanes,
    )
  ) {
    // ... 处理边缘场景
    // 在 rendering 阶段中，如果发生了新的 update，但是这个 update 是把一些隐藏的组件重新展示出来
    // 但是新的 update 的 lane 已经在当前的 rendering 阶段中被标记过了（执行过了），所以需要重头开始
    prepareFreshStack(root, NoLanes);
    // 看到这段注释，想必大家会很疑惑，不着急，我们现在还不需要它
  } else if (exitStatus !== RootIncomplete) {
    // 如果退出状态不等于未完成，那么说明有可能是一下几种
    // 1、已完成
    // 2、有报错被捕获
    // 3、有报错未被捕获，严重级别的错误
    // 这些流程先忽略，避免干扰
  }

  ensureRootIsScheduled(root, now());
  if (root.callbackNode === originalCallbackNode) {
    // 如果当前执行的 scheduler task 未发生变化，还是最初在执行的那个 task
    // 那么返回一个新的函数，注意这一步，很关键
    return performConcurrentWorkOnRoot.bind(null, root);
  }
  // 如果当前执行的 scheduler task 已经发生了变化或者被取消了，返回 null
  // 这一步没有返回函数，也很关键
  return null;
}
```

上半部分已 `render` 流程的结果为结束点，下半部分主要是处理 `exitStatus` 在不同场景下的流程

* 先处理边缘场景，在 rendering 阶段中，如果发生了新的 update，但是这个 update 是把一些隐藏的组件重新展示出来，但是新的 update 的 lane 已经在当前的 rendering 阶段中被标记过了（执行过了），所以需要重头开始
* 如果退出状态不等于未完成，那么说明有可能是一下几种：1、已完成；2、有报错被捕获；3、有报错未被捕获，严重级别的错误，这些分支中的处理这里不展开分析
* 最后的处理的是状态为 `RootIncomplete` 情况下的流程，这一步分为两个步骤，这两个步骤都很关键
* 1：执行 `ensureRootIsScheduled` 函数
* 2：如果当前执行的 scheduler task 未发生变化，还是最初在执行的那个 task，则返回 `performConcurrentWorkOnRoot` 函数，并绑定 `root` 参数；如果不相同，则返回 null

函数最后的那两个步骤，有一个分支流程下返回了函数，而且就是 `performConcurrentWorkOnRoot`

回想下前面抛出的第一个疑问：**`Reconciler` 是如何利用 `Scheduler` 进行任务中断与恢复的**，现在有了些思绪，那么 `performConcurrentWorkOnRoot` 具体是如何做到的呢？

<!-- 此外，从 `performConcurrentWorkOnRoot` 的代码不难看出，`Reconciler` 中与 `render` 阶段的事关重要的几个部分

* `renderRootConcurrent`
* `prepareFreshStack`
* `ensureRootIsScheduled`
* `getNextLanes` -->

### `Reconciler` 是如何利用 `Scheduler` 进行任务中断与恢复的

回顾 `Scheduler` 中恢复任务的条件

```js
const continuationCallback = callback(didUserCallbackTimeout);
currentTime = getCurrentTime();
if (typeof continuationCallback === 'function') {
  // 这里是真正的恢复任务，等待下一轮循环时执行
  currentTask.callback = continuationCallback;
  // ....
}
```

执行的任务必须返回一个函数，下一轮循环时才会恢复执行，`performConcurrentWorkOnRoot` 作为被执行的任务，整个函数只有在这个条件下才返回了函数，关键在于 `root.callbackNode` 是否会被修改

```js
ensureRootIsScheduled(root, now())
if (root.callbackNode === originalCallbackNode) {
  // 如果当前执行的 scheduler task 未发生变化，还是最初在执行的那个 task
  // 那么返回一个新的函数，注意这一步，很关键
  return performConcurrentWorkOnRoot.bind(null, root);
}
```

但是反观 `performConcurrentWorkOnRoot`，并没有对 `root.callbackNode` 做修改，那么关键应该在这

```js
ensureRootIsScheduled(root, now())
```

显然应该把关注点放在 `ensureRootIsScheduled`，稍微回想下 `Scheduler`，在进入调度流程前，也会经过 `ensureRootIsScheduled`，我们之前只关注了 `ensureRootIsScheduled` 是如何进入 `Scheduler` 的，这次需要分析进入 `Scheduler` 前的一些场景处理

```js
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  // 当前正在执行中的 scheduler task
  const existingCallbackNode = root.callbackNode;

  // 看看哪些 lane 已经超时了，标记到 root.expiredLanes
  markStarvedLanesAsExpired(root, currentTime);

  // 获取当前需要执行的 `lane/lanes`，用最高优先级的 `lane` 作为任务执行的优先级标准
  // 同时计算 lane 对应的 priority
  const nextLanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );
  // nextLanes 对应的 priority
  const newCallbackPriority = returnNextLanesPriority();

  if (nextLanes === NoLanes) {
    // 没有需要执行的 lanes / lane
    if (existingCallbackNode !== null) {
      // 取消当前的 scheduler task
      // 并清除保存在 root 上的一些信息
      cancelCallback(existingCallbackNode);
      root.callbackNode = null;
      root.callbackPriority = NoLanePriority;
    }
    return;
  }

  // 如果已经有执行中的任务，可以判断是否复用执行中的任务
  if (existingCallbackNode !== null) {
    const existingCallbackPriority = root.callbackPriority;
    if (existingCallbackPriority === newCallbackPriority) {
      // 任务优先级没有发生变化，则不需要走后续的 scheduler 分配任务流程
      return;
    }
    // 取消当前的 task
    cancelCallback(existingCallbackNode);
  }
  // 上面的流程中如果没有return，意味着一定会取消当前执行中的任务

  // 处理 newCallbackNode 在不同 lane 下的情况
  let newCallbackNode;

  // 如果上述流程都没走到，那么一定会进入下面进入 schedulerCallback 的流程
  // ... newCallbackNode = xxxx;

  // 进入 schedulerCallback 的流程意味着 callbackPriority，callbackNode 会被覆盖
  root.callbackPriority = newCallbackPriority;
  root.callbackNode = newCallbackNode;
}
```

// TODO

如果没有出现更高优先级的 lane priority，任务 task 就不会取消，也就是 root.callbackNode 和 root.callbackPriority 都没有发生变化

那么 performConcurrentWorkOnRoot 会返回 performConcurrentWorkOnRoot.bind(null, root)，scheduler->workloop 会保存这个返回的回调，在下次调度重新执行



### `Reconciler` 遍历 `Fiber Tree` 时是怎么中断任务的，中断后为什么可以恢复到上次循环中断的位置

遍历过程中是从 workInProgress 这个 fiber node 开始的，workInProgress 初始化时的方式是 clone 一份 root.current

```js
workInProgressRoot = root;
workInProgress = createWorkInProgress(root.current, null);
```

当 workLoopConcurrent 恢复执行时，会从上次停止的 workInProgress 开始 

#### 如果中断与恢复的中途出现更高优先级的 lane priority，该如何处理

前面有提到过**如果没有出现更高优先级的 lane priority，任务 task 就不会取消**

那么这个**更高优先级的lane**是怎么检测出来的呢

`performConcurrentWorkOnRoot` 中执行的循环退出时，都会执行 `ensureRootIsScheduled`

`ensureRootIsScheduled` 具体是怎么做的呢

### ensureRootIsScheduled 

用于标记当前 scheduler 中执行的任务以及对应的 lane priority
* root.callbackPriority
* root.callbackNode

`ensureRootIsScheduled` 会通过 `getNextLanes` 来获取最高优先级的 lane，如果刚刚的任务是中断的，那么就把任务的lane priority 和新获取的最高优先级的 lane priority 对比

一样的话就退出，走前面 **“如果没有出现更高优先级的 lane priority，任务 task 就不会取消”** 的流程

如果不一样，那就把中断的任务取消掉，开启一个新的调度，这个调度就是以前提到的 `scheduler`


**那么疑问来了重新调度的任务会继续从停止的任务开始吗？**

**或者说该不该从停止的地方恢复？**

我们来看看 `performConcurrentWorkOnRoot` 中 `reconcile` 的下一步骤 `renderRootConcurrent`

### renderRootConcurrent

* 判断是否需要重置 `workInProgress` 相关状态
* 进入 `workLoopConcurrent`
* 根据 `workInProgress` 状态返回当前的 `reconcile 处理状态`


#### prepareFreshStack

前面有提到过遍历过程中是从 workInProgress 这个 fiber node 开始的，workInProgress 初始化时的方式是 clone 一份 root.current

在进入 `reconcile` 阶段后，到达 `workLoopCurrent` 之前，会先初始化 `workInProgress`

初始化的过程很简单

```js
root.finishedWork = null;
root.finishedLanes = NoLanes;
// ...
workInProgressRoot = root;
workInProgress = createWorkInProgress(root.current, null);
workInProgressRootRenderLanes = subtreeRenderLanes = workInProgressRootIncludedLanes = lanes;
workInProgressRootExitStatus = RootIncomplete;
workInProgressRootFatalError = null;
workInProgressRootSkippedLanes = NoLanes;
workInProgressRootUpdatedLanes = NoLanes;
workInProgressRootPingedLanes = NoLanes;
```


#### 有些 willxxx 的生命周期为什么会执行两次


`prepareFreshStack`



#### getNextLanes



#### markRootUpdated

scheduleUpdateOnFiber 时会通过 markRootUpdated 标记当前 update.lane 到 root.pendingLanes



### workLoopConcurrent



### beginWork 流程



### performUnitOfWork 流程



### markUpdateLaneFromFiberToRoot

函数返回一个 fiber node




## Render 与 Commit


### finishConcurrentRender


#### commitRoot
