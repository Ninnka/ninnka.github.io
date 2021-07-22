---
title: React源码细读-并发模式下的 Reconciler 与可中断渲染
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
	- reconciliation
	- ConcurrentMode
---

# Concurrent Mode

官方对于 `Concurrent Mode`，用了一个有趣的比喻，把 `Concurrent Mode` 隐喻为版本控制，版本控制大家都了解，平常开发项目时，大家会在同一个项目的不同分支中修改内容，比如修改同一个文件

假设现在没有版本控制，为了避免冲突，A 同学在修改 a 文件时，B、C、D 等其他同学就得等着 A 同学修改完然后释放文件，这种情况下会阻塞 B，C，D 等其他同学的开发进度

而 `Concurrent Mode` 则提供了一个类似分支的概念，把 A，B，C 等同学比作 `React` 中的 `render` 任务，浏览器的渲染进程，IO 进程等等，当`React` 中的 `render` 任务执行时，可以不阻塞浏览器中的其他进程

其实严格来说，`Concurrent Mode` 比作版本控制也不是那么妥当，毕竟场景不是一模一样，只是希望大家能理解到其中的含义即可

个人理解的话，`React` 中的 `Concurrent Mode` 是指在 `Reconciler` 中处理 `long task` 时，可以不阻塞浏览器中的其他进程，并且 `React` 中的 `render 任务` 具有各自的优先级，任务可以通过**过时间分片** + **优先级调度**的方式在**执行**和**暂停**之间切换状态

# Legacy & Concurrent & Blocking

`React v17` 中提供了三种可选的方式来创建 `React` 应用，分别是 `Legacy Mode`，`Concurrent Mode`，`Blocking Moode`

`Legacy Mode` 是我们熟悉的传统模式，目前是通过 `ReactDOM.render` 来触发这个模式，`Legacy Mode` 下大家都熟悉，`Reconcile Fiber` 流程一步到位，不可中断，当页面中存在大量组件 `render` 时会导致时间过长，阻塞浏览器的渲染进程，页面响应速度慢，交互卡顿明显

`Concurrent Mode` 目的是为了能让渲染流程变的可中断，也就是我们现在常听到的 `Interruptible Rendering` - **可中断式渲染** 这一概念

`React` 计划在 v18 中正式默认启用 `Concurrent Mode`，但是让大型的 `React App` 一口气升级上来难免会遇到一些问题，主要体现在 `React` 现在的生态中会有些组件还在使用一些不安全的生命周期，一些老旧的 `React Library` 可能没办法跟 `Concurrent Mode` 兼容运行

为了能更加平滑过渡到 `Concurrent Mode`，新增了 `Blocking Mode`，目前 `React` v17 中则是采用了 `Blocking Mode` 的过渡模式，可以通过 `ReactDOM.createBlockingRoot` 开启过渡模式，在 `Blocking Mode` 期间，减少 `unsafe_liftcycle`，`String Refs`，`Legacy Context`，`findDOMNode` 等不稳定或者不兼容 api 的使用，等待 `React` 的生态逐步跟进后，再尝试使用 `React` v18 的新特性

![](https://img.ninnka.top/1626508076589.png)

[关于 Blocking Mode 特性的参考链接](https://reactjs.org/docs/concurrent-mode-adoption.html#feature-comparison)

<!-- more -->

如果你想提前体验 `Concurrent Mode`，可以通过 `ReactDOM.createRoot(rootNode).render(<App />)` 的方式主动开启**并发模式**

**不同模式下开启的方式可以参考这个（出自官方文档）**

> `Legacy Mode`: `ReactDOM.render(<App />, rootNode)`
>
> `Blocking Mode`: `ReactDOM.createBlockingRoot(rootNode).render(<App />)`
>
> `Concurrent Mode`: `ReactDOM.createRoot(rootNode).render(<App />)`

# Reconcile Fiber

**此文基于 `react v17.0.2` 分析，[仓库传送门](https://github.com/facebook/react)**

在前几个篇章中，解读过 `Scheduler` - **调度器** 的工作方式，`Scheduler` 通过**时间分片**控制了每个任务的最大执行时间，给任务设置不同的过期时间，分为 `timerQueue` 和 `taskQueue`，通过 `MessageChannel` 来手动调度 `taskQueue` 中每个任务的执行

`Scheduler` 的好处就在于，每个任务在有限的时间内完成部门或全部工作，每帧内的剩余时间可以留给浏览器做渲染或者页面的响应

`React` 通过 `Scheduler` 实现任务的中断与恢复，但是，仅此就够了吗？

假设其中一个任务是个 `while` 循环，每次循环都需要处理一下组件的 `render` 函数，遍历的节点有十万个，在这个任务结束之前，`Scheduler` 没法做下一步的剩余时间计算，在这个 `long task` 执行的期间，页面如果有交互产生的话，一般会出现我们常说的“掉帧”现象，因为计算资源目前被 JavaScript 的事件占据，浏览器需要等待资源释放才能够处理 UI

`Concurrent Mode` 作为 `React` 发展方向上的一个重要环节，又提出了怎样的解决方案呢？

`Concurrent Mode` 的出现无非是为了解决这两大类的问题：

1. 受限于 `CPU` 的更新，比如 `render` 函数调用时的创建 `component dom`
2. 受限于 `IO` 的更新，比如：`fetching data from network`

小伙伴肯定会好奇，为啥 `Concurrent Mode` 能解决这些呢？再说这两个算啥问题？🤣

我们回到前面的说的那个 `long task` 任务处理问题，比如十万个节点执行 `render`，那肯定会耗费不少时间，而 `render` 就是创建 `component dom` 的过程

如果这个 `long task` 可以实现前面所说的“并发模式”的话，通过**过时间分片** + **优先级调度**的方式在**执行**和**暂停**之间切换状态，就像 `Scheduler` 那样，就可以解决第一类问题

`Concurrent Mode` 通过分割 `Reconcile` 中的任务，在适当的时机释放 `CPU` 支援，让浏览器的渲染进程可以更迅速的响应

第一类问题在开启 `Concurrent Mode` 后就会得到优化，而第二类问题通常是涉及到 UI 交互上的体验优化

比如这样一个场景：比如点击某个按钮后，展示 loading，同时从后端拉取数据，但是这个拉取数据的过程很快，展示的 loading 可能会一闪而过，这个相信大家都常见 🤣

又或者是这种场景：有一个下拉框，选中某个选项后，拉取接口数据，但是由于接口响应慢或者网络不顺畅，导致数据加载时间较长，此时你切换了另外一个选项，这两个选项最后几乎同一时间响应，你会看到，页面显示变成第一选项的结果然后又迅速切换到了最新的结果 🤣

上面的场景中，无论是哪种，loading 和结果的展示几乎完全取决于 `网络IO` 的速度，那么有没有一种办法可以平衡一下体验呢？

在 `Concurrent Mode` 下，`React` 提供了两个新的功能用于解决这个 `网络 IO` 下的 UI 交互问题

- `Suspense`
- `useTransition`

感兴趣的小伙伴可以看看官方文档提供的 demo [Suspense with useTransition](https://codesandbox.io/s/jovial-lalande-26yep)（这里不对 `Suspense` 和 `useTransition` 做深入探讨，后续会单独开一个篇章分析）

既然 `Concurrent Mode` 能解决这么多问题，那 `Concurrent Mode` 下的 `Reconciler` 是如何工作的：

- `Reconciler` 是如何利用 `Scheduler` 进行任务中断与恢复的
- `Reconciler` 遍历 `Fiber Tree` 时是怎么中断任务的，中断后为什么可以恢复到上次循环中断的位置
- 遍历 `Fiber Tree` 途中如果中断与恢复的中途出现更高优先级的任务，该如何处理
- 有些 willxxx 生命周期，比如 `componentWillReceiveProps`，为什么会执行两次

对这几个问题肯定有小伙伴会感到疑惑，没关系，这次带着问题盘他

<div align="center">
<img width=160 src="https://img.ninnka.top/1626596402374.png"/>
</div>

## `Reconciler` 与 `Scheduler`

`performConcurrentWorkOnRoot` 是 `Scheduler` 开启调度时实际执行的任务，简单回顾一下 `ensureRootIsScheduled`

```js
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  let newCallbackNode;
  if (newCallbackPriority === SyncLanePriority) {
    // ....
  } else if (newCallbackPriority === SyncBatchedLanePriority) {
    // ....
  } else {
    const schedulerPriorityLevel =
      lanePriorityToSchedulerPriority(newCallbackPriority);
    // 这里调度的任务就是 performConcurrentWorkOnRoot
    newCallbackNode = scheduleCallback(
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root)
    );
  }
  // 这里把 react lane priority 保存到了 root.callbackPriority
  root.callbackPriority = newCallbackPriority;
  // 这里把 Scheduler 返回的 Task 保存到了 root.callbackNode
  root.callbackNode = newCallbackNode;
}
```

只看部分核心代码，在调度的某个分支中，会传入 `performConcurrentWorkOnRoot` 函数作为需要调度的任务

在 `Scheduler` - **调度器** 开始调度 `Task` 后，会进入 `Concurrent Mode` 工作流的第一步 `reconciliation`，这一步流程主要是我们常说的 `Reconciler` - **协调器** 在负责

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
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes
  );

  // 开始 Reconciler 的 render 阶段
  // exitStatus 用来表示 render 流程的结果状态
  let exitStatus = renderRootConcurrent(root, lanes);
  // 省略一部分代码 ... 这里的中间代码会处理 exitStatus 在不同状态下的相关流程，我们先忽略
}
```

我们先来看 `performConcurrentWorkOnRoot` 的删减后的前半部分

- 进入 `Reconciler` 后，清除 `currentEvent` 的一些信息，留给下一次进入的事件
- 获取当前需要执行的 `lane/lanes`，用最高优先级的 `lane` 作为任务执行的优先级标准，同时计算 lane 对应的 priority（这个步骤很重要）
- 开始 `Reconciler` 的 `render` 阶段，`exitStatus` 用来表示 `render` 流程的结果状态

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
      workInProgressRootUpdatedLanes
    )
  ) {
    // ... 处理边缘场景
    // 在 rendering 阶段中，如果发生了新的 update，但是这个 update 是把一些隐藏的组件重新展示出来
    // 但是新的 update 的 lane 已经在当前的 rendering 阶段中被标记过了（执行过了），所以需要重头开始
    prepareFreshStack(root, NoLanes);
    // 看到这段注释，想必大家会很疑惑，不着急，我们现在还不需要理解它
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

- 先处理边缘场景，在 rendering 阶段中，如果发生了新的 update，但是这个 update 是把一些隐藏的组件重新展示出来，但是新的 update 的 lane 已经在当前的 rendering 阶段中被标记过了（执行过了），所以需要重头开始
- 如果退出状态不等于未完成，那么说明有可能是一下几种：1、已完成；2、有报错被捕获；3、有报错未被捕获，严重级别的错误，这些分支中的处理这里不展开分析
- 最后的处理的是状态为 `RootIncomplete` 情况下的流程，这一步分为两个步骤，这两个步骤都很关键
- 1：执行 `ensureRootIsScheduled` 函数
- 2：如果当前执行的 scheduler task 未发生变化，还是最初在执行的那个 task，则返回 `performConcurrentWorkOnRoot` 函数，并绑定 `root` 参数；如果不相同，则返回 null

函数最后的那两个步骤，有一个分支流程下返回了函数，而且就是 `performConcurrentWorkOnRoot`

回想下前面抛出的问题：**`Reconciler` 是如何利用 `Scheduler` 进行任务中断与恢复的**，现在或许有了些思绪，那么 `performConcurrentWorkOnRoot` 具体是如何做到的呢？

## `Reconciler` 是如何利用 `Scheduler` 进行任务中断与恢复的

回顾 `Scheduler` 中恢复任务的条件

```js
const continuationCallback = callback(didUserCallbackTimeout);
currentTime = getCurrentTime();
if (typeof continuationCallback === "function") {
  // 这里是真正的恢复任务，等待下一轮循环时执行
  currentTask.callback = continuationCallback;
  // ....
}
```

执行的任务必须返回一个函数，下一轮循环时才会恢复执行，`performConcurrentWorkOnRoot` 作为被执行的任务，整个函数只有在这个条件下才返回了函数，关键在于 `root.callbackNode` 是否会被修改

```js
ensureRootIsScheduled(root, now());
if (root.callbackNode === originalCallbackNode) {
  // 如果当前执行的 scheduler task 未发生变化，还是最初在执行的那个 task
  // 那么返回一个新的函数，注意这一步，很关键
  return performConcurrentWorkOnRoot.bind(null, root);
}
```

但是反观 `performConcurrentWorkOnRoot`，并没有对 `root.callbackNode` 做修改，那么关键应该在这

```js
ensureRootIsScheduled(root, now());
```

显然应该把关注点放在 `ensureRootIsScheduled`，稍微回想下 `Scheduler`，在进入调度流程前，也会经过 `ensureRootIsScheduled`，我们之前只关注了 `ensureRootIsScheduled` 是如何进入 `Scheduler` 的，这次需要分析进入 `Scheduler` 前的一些场景处理

`ensureRootIsScheduled` 在处理 `newCallbackNode` 前做了不少准备，我们来看看具体都做了啥

```js
// 当前正在执行中的 scheduler task
const existingCallbackNode = root.callbackNode;

// 看看哪些 lane 已经超时了，标记到 root.expiredLanes
markStarvedLanesAsExpired(root, currentTime);

// 获取当前需要执行的 `lane/lanes`，用最高优先级的 `lane` 作为任务执行的优先级标准
// 同时计算 lane 对应的 priority
const nextLanes = getNextLanes(
  root,
  root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes
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
```

- 取出当前正在执行中的 `scheduler task`
- 看看哪些 `lane` 已经超时了，`标记(merge)`到 `root.expiredLanes`
- 获取当前需要执行的 `lane/lanes`，用最高优先级的 `lane` 作为任务执行的优先级标准，同时计算 lane 对应的 priority
- 获取 `nextLanes` 对应的 `priority`
- 如果 `nextLanes` 属于 `NoLanes`，那么判断是否有还在执行中的任务，有的话取消当前的 `scheduler task`，并清除保存在 `root` 上的一些信息

流程到这里算是一个分水岭，如果执行完上面的流程后函数退出，说明，任务被取消掉了或者当前无任务了

如果函数继续往下走，那么继续往下看

```js
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

// ... 省略代码 newCallbackNode = schedulerCallback(xxx) / schedulerSyncCallback(xxx)

// 这里把 react lane priority 保存到了 root.callbackPriority
root.callbackPriority = newCallbackPriority;
// 这里把 Scheduler 返回的 Task 保存到了 root.callbackNode
root.callbackNode = newCallbackNode;
```

- 如果已经有执行中的任务，可以判断是否复用执行中的任务，若任务优先级没有发生变化，则不需要走后续的 `Scheduler` 分配任务流程；若优先级发生了变化，则取消当前的 `scheduler task`

函数如果执行了上面这部分的流程后就结束的话，意味着当前执行中的任务是可以复用的

如果函数还未退出，那么说明有两种情况：

1. 没有执行中的任务，下面需要分配一个新的
2. 原有的任务被取消掉了，现在重新分配一个新的

两种情况的处理流程代码是一样的

- 处理 `newCallbackNode` 在不同 lane 下的情况，进入下面 `schedulerCallback` 的流程
- 进入 `schedulerCallback` 的流程意味着 `callbackPriority、callbackNode` 会被覆盖

结合 `ensureRootIsScheduled` 和 `performConcurrentWorkOnRoot` 来看

一般来说，如果没有出现更高优先级的 `lane priority`，任务 `task` 就不会取消，也就是 `root.callbackNode` 和 `root.callbackPriority` 都没有发生变化，若任务处于 `RootInComplete` - **未完成** 状态，那么 `performConcurrentWorkOnRoot` 会返回 `performConcurrentWorkOnRoot.bind(null, root)` 作为恢复任务的关键，`Scheduler` 在执行 `workloop` 流程中会保存这个返回的回调，并重新赋值到 `task.callback` 上，在下次调度时重新执行

简单点说，判断到任务还可以复用的情况，把任务绑定 `root` 参数后重新返回，**利用了任务未执行完且未出现更高优先级的任务就可以继续执行的特点（root 未发生变化）**

## `Reconciler` 遍历 `Fiber Tree` 时是怎么中断任务的，中断后为什么可以恢复到上次循环中断的位置

现在应该都了解 `Reconciler` 是如何利用 `Scheduler` 进行任务中断与恢复的了，但是没有解决前面提到的 `long task` 占用时长的问题呀

<div align="center">
<img width=200 src="https://img.ninnka.top/1626762314226.png"/>
</div>

莫慌，我们稍微看看 `performConcurrentWorkOnRoot` 中的一个关键入口，已经忘记了的小伙伴可以翻回去看看

```js
// 开始 Reconciler 的 render 阶段
// exitStatus 用来表示 render 流程的结果状态
let exitStatus = renderRootConcurrent(root, lanes);
```

`performConcurrentWorkOnRoot` 是个老大哥，它只负责做些统筹工作，细活是交给 `renderRootConcurrent` 去做的，来看看 `renderRootConcurrent` 小弟做了啥

说实话这个函数名太好猜了哈哈，函数流程的关键几乎是一眼就发现了，我们先看看关键部分

```js
function renderRootConcurrent(root: FiberRoot, lanes: Lanes) {
  // ... 省略流程
  do {
    try {
      workLoopConcurrent();
      break;
    } catch (thrownValue) {
      handleError(root, thrownValue);
    }
  } while (true);
  // ... 省略流程
}
```

这里上来就是一个 `do...while` 结构，里面~~又是一个小弟~~执行的是 `workLoopConcurrent`，从函数名就猜到了是要循环做大事情的（这个循环相信大家肯定都愣了一下，我们先抛开这个 `do...while + break` 不管 🤣）

得先说一句，这个函数就这么简单，不是我删删减减

```js
function workLoopConcurrent() {
  // 循环遍历 current fiber tree 直到剩余执行时间不足
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

遍历过程很简单，就是 `workInProgress` 这个 `fiber node` 不为空就执行 `performUnitOfWork`

这里中断循环的两种方法有两种：

1. 整个 `Fiber tree` 遍历 `render` 完成
2. 通过时间分片控制给每个 `fiber node` 分配执行时间

这个时间分片有点眼熟，`Scheduler` 不就有一个吗？是的，这里时间分片跟 `Scheduler` 使用的是同一个函数 `shouldYield`，`shouldYield` 是 `Scheduler` 中 `shouldYieldToHost` 的换名马甲

当循环中断后，`workInProgress` 记录了当前执行中的节点，通过 `Reconciler` 与 `Scheduler` 的任务中断恢复机制，下次进入循环时可以从停止的地方开始

## 如果中断与恢复的途中出现更高优先级的 `lane priority`

前面说到在 `Reconciler` 中，判断到任务还可以复用的情况，把任务绑定 `root` 参数后重新返回，**利用了任务未执行完且未出现更高优先级的任务就可以继续执行的特点（root 未发生变化）**

但是，这里的条件是 **如果没有出现更高优先级的 `lane priority`，`scheduler task` 就不会取消**

如果途中出现了更高优先级的 `lane priority`，那应该如何处理呢？

`ensureRootIsScheduled` 标记了当前 `scheduler` 中执行的任务以及对应的 `lane priority` 到 `root` 节点上

```js
root.callbackPriority = newCallbackPriority;
root.callbackNode = newCallbackNode;
```

`ensureRootIsScheduled` 通过 `getNextLanes` 来获取最高优先级的 lane，如果刚刚的任务是中断的，那么就把任务的 `lane priority` 和新获取的最高优先级的 `lane priority` 对比

优先级一样的情况就走前面 **“如果没有出现更高优先级的 lane priority，scheduler task 就不会取消”** 的流程；如果不一样，那就把中断的任务取消掉，开启一个新的调度

```js
const nextLanes = getNextLanes(
  root,
  root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes
);
// ...
if (existingCallbackPriority === newCallbackPriority) {
  return;
}
cancelCallback(existingCallbackNode);
```

咋看一下，当出现更高优先级的 `lane priority` 时，不过是取消当前任务，重新开启一个调度罢了？

可是，真实情况显然不会这么简单，取消当前任务，重新开启一个调度是第一步，我们接着分析

这里的 `nextLanes` 需要通过 `getNextLanes` 获取，`getNextLanes` 在 `performConcurrentWorkOnRoot` 也有调用过，并且结果作为 `lanes` 参数传递到了 `renderRootConcurrent`

```js
let lanes = getNextLanes(
  root,
  root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes
);
// ...
let exitStatus = renderRootConcurrent(root, lanes);
```

我们来看看 `Reconcile` 中的 `renderRootConcurrent` 利用这个 `lanes` 做了什么

```js
function renderRootConcurrent(root: FiberRoot, lanes: Lanes) {
  // 添加渲染上下文到当前的执行上下文中
  const prevExecutionContext = executionContext;
  executionContext |= RenderContext;
  // 保存当前的 dispatcher
  const prevDispatcher = pushDispatcher();

  // 如果根节点发生了变化，或者当前的 lanes 发生了变化
  if (workInProgressRoot !== root || workInProgressRootRenderLanes !== lanes) {
    // 重置目标更新截止时间
    resetRenderTimer();
    // 准备一个全新的堆栈
    prepareFreshStack(root, lanes);
  }

  // ----- 分割线
}
```

上半部分做的事情比较少，简单分析看看

- 添加渲染上下文到当前的执行上下文中
- 保存当前的 dispatcher
- 如果根节点发生了变化或者当前的 lanes 发生了变化，重置目标更新截止时间，并且准备一个全新的堆栈

`lanes` 参数只在这一部分用到，大概两个地方

1. 根据当前的 `workInProgressRootRenderLanes` 判断 `lanes` 是否发生变化了
2. 若 `lanes` 发生变化了，调用 `prepareFreshStack(root, lanes)`，意思大概是”准备一个全新的堆栈“

不管 `prepareFreshStack` 具体做了啥，先做个猜想，既然当前的 `lanes` 发生了变化，那准备一个全新的堆栈是为了啥？

先留个悬念，我们接着分析下半部分

```js
function renderRootConcurrent(root: FiberRoot, lanes: Lanes) {
  // ----- 分割线

  // workLoopConcurrent
  do {
    try {
      workLoopConcurrent();
      break;
    } catch (thrownValue) {
      handleError(root, thrownValue);
    }
  } while (true);

  // 前面的 workLoopConcurrent 流程中组件 render 阶段会去操作 context 依赖
  // 这里重置 context 依赖
  resetContextDependencies();

  // 还原当前的 dispatcher
  popDispatcher(prevDispatcher);
  // 还原当前的 执行上下文
  executionContext = prevExecutionContext;

  // 判断是否有未处理完的节点
  if (workInProgress !== null) {
    // 还有剩余节点需要处理
    return RootIncomplete;
  } else {
    // 全部处理完，重置 workInProgress 状态
    workInProgressRoot = null;
    workInProgressRootRenderLanes = NoLanes;

    return workInProgressRootExitStatus;
  }
}
```

`renderRootConcurrent` 的后半部分做的事情也比较简单，除了执行 `workLoopConcurrent` 外，就是把前面的一些关键状态做了恢复操作，然后把当前的处理状态做个汇总，返回给 `performConcurrentWorkOnRoot`

- 执行 `workLoopConcurrent`
- 前面的 workLoopConcurrent 流程中组件 render 阶段会去操作 context 依赖，这里重置 context 依赖
- 还原当前的 dispatcher
- 还原当前的 执行上下文
- 判断是否有未处理完的节点，如果还有剩余节点需要处理，返回未完成的状态；如果已完成，重置 workInProgress 状态，返回 `workInProgressRootExitStatus`，这个 `workInProgressRootExitStatus` 的状态可以翻阅上面处理逻辑的部分进行查看

前面分析过 `workLoopConcurrent` 遍历 `Fiber Tree` 时会从 `workInProgress` 节点开始，但是这一套流程下来，并没有直接发现有初始化 `workInProgress` 的地方

大胆猜测一下，这个初始化的流程就在 `prepareFreshStack` 中

```js
function prepareFreshStack(root: FiberRoot, lanes: Lanes) {
  // 重置 finfished 状态
  root.finishedWork = null;
  root.finishedLanes = NoLanes;

  const timeoutHandle = root.timeoutHandle;
  if (timeoutHandle !== noTimeout) {
    // 删除等待中的 suspense 任务，一般环境下，timeoutHandle 就是 setTimeout 返回的 Timeout number
    root.timeoutHandle = noTimeout;
    // 封装了 clearTimeout，就是清除定时器
    cancelTimeout(timeoutHandle);
  }
  // 往上遍历节点释放节点
  if (workInProgress !== null) {
    let interruptedWork = workInProgress.return;
    while (interruptedWork !== null) {
      unwindInterruptedWork(interruptedWork);
      interruptedWork = interruptedWork.return;
    }
  }
  // 清除 workInProgress 相关的状态
  workInProgressRoot = root;
  workInProgress = createWorkInProgress(root.current, null);
  workInProgressRootRenderLanes =
    subtreeRenderLanes =
    workInProgressRootIncludedLanes =
      lanes;
  workInProgressRootExitStatus = RootIncomplete;
  workInProgressRootFatalError = null;
  workInProgressRootSkippedLanes = NoLanes;
  workInProgressRootUpdatedLanes = NoLanes;
  workInProgressRootPingedLanes = NoLanes;
}
```

猜测是对的，`prepareFreshStack` 做了很多 `workInProgress` 状态初始化的工作，初始化的过程也很简单

- 重置 `finfished` 状态
- 删除等待中的 `suspense` 任务（清除定时器），一般环境下，`timeoutHandle` 就是 `setTimeout` 返回的数字
- 往上遍历节点释放节点，这一步是针对 `legacy context` 做的处理
- 清除 `workInProgress` 相关的状态

关键在最后一个步骤：清除 `workInProgress` 相关的状态

这一步重置了 `workInProgressRoot` 节点，初始化了 `workInProgress` 等状态，也就导致了当重新进入 `workLoopConCurrent` 时，`workInProgress` 已不是上次离开时存储的节点，而是根节点 `workInProgressRoot`

这也说明了如果中断与恢复的途中出现更高优先级的 `lane priority`，`workInProgress` 会被重新初始化为根节点，当下次调度开始时会从根节点开始遍历

## 有些 willxxx 的生命周期为什么会执行两次

前面一部分说到遍历 `Fiber Tree` 的途中出现更高优先级的 `lane priority`时，会导致 `workInProgress` 会被重新初始化为根节点

被重置意味着上次遍历执行完 `render` 的节点需要在下次调度时再执行一次，这就导致了部分 `UNSAFE` 的生命周期，如：`componentWillReceiveProps / UNSAFE_componentWillReceiveProps` 在部分场景下可能执行多次

# 写在最后

很抱歉这次图画的少了，后面抽空一个个补上

这次分析只是从大的方向上对并发模式下的 `Reconciler` 做了简单的分析，对于其细节部分的 `diff` 与 `rendering` 过程，下次开个单独的篇幅一起探讨

纯属个人观点，有兴趣的可以一起交流交流 😝
