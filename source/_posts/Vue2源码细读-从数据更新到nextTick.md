---
title: Vue2源码细读-从数据更新到nextTick
date: 2021-05-30 21:13:10
categories:
  - Vue2
  - nextTick
tags:
  - Web
	- Vue2
	- 源码
---

## 一个数据更新的demo

先来看一个常见的案例，点击click后会循环1000次，每次都给this.number+1

```vue
<template>
  <div>
    <div>{{number}}</div>
    <div @click="onClick">click</div>
  </div>
</template>
export default {
    data () {
        return {
            number: 0,
        };
    },
    methods: {
        onClick () {
            for(let i = 0; i < 1000; i++) {
                this.number += 1;
            }
        },
    },
}
```

按照我们对vue响应式的理解，number变化后会触发setter函数，进而触发Dept.notify，最后通过Watcher.update()来更新视图，循环1000次会导致视图的更新1000次。

但是连续更新1000次会造成不必要的视图更新，频繁操作DOM的效率也非常低，为了减少布局和渲染，Vue把DOM更新设计为异步更新，每次侦听到数据变化，将开启一个队列，并缓冲在同一事件循环中发生的所有数据变更。如果同一个 watcher 被多次触发，只会被推入到队列中一次。然后在下一个的事件循环tick中，Vue才会真正执行队列中的数据变更，然后页面才会重新渲染。相当于把多个地方的DOM更新放到一个地方一次性全部更新。

## 更新队列 queue

为了避免频繁操作DOM，造成不必要的视图更新，Vue在数据发生变化时，触发setter方法后，setter会把Watcher push到队列queue中。

为此，Vue提供了异步更新的监听接口 —— Vue.nextTick(callback) 或 this.$nextTick(callback) 。当数据发生改变，异步DOM更新完成后，callback回调将被调用。开发者可以在回调中，操作更新后的DOM。

<!-- more -->

```js
// watch.js
  /**
   * Subscriber interface.
   * Will be called when a dependency changes.
   */
  update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      // 上述例子中点击click，修改this.number后，最后会调用对应watcher的 update方法
      queueWatcher(this)
    }
  }
```

```js
/**
 * 把 watcher push到 watcher 队列中
 * 对于 watcher 会做几个判断
 * 1、如果有重复的id，那么就跳过
 * 2、如果还未 flush ，那么正常 push 到队列中
 * 3、如果在 flushing 中，那么按照 id 排序插入到队列中对应的位置
 */
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    // 有重复的id，那么就跳过，不重复就正常 push
    has[id] = true
    if (!flushing) {
      // 还未 flush ，那么正常 push 到队列中
      queue.push(watcher)
    } else {
      // 在 flushing 中，那么按照 id 排序插入到队列中对应的位置
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // 当有 watcher push 到队列中时，把队列置位等待 flush 状态
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      // 在下一个 tick 中遍历队列 run 一遍。
      nextTick(flushSchedulerQueue)
    }
  }
}
```

`nextTick` 中传入了一个新的函数 `flushScheduleerQueue`，这个函数做的事情很简单，就是遍历 `queue`，顺序执行每个 `watcher` 的 `run` 方法。

```js
function flushSchedulerQueue () {
  currentFlushTimestamp = getNow()
  flushing = true
  let watcher, id

  // 开始flush前要对queue进行排序
  // 排序主要是为了确保：
  // 1. 组件更新顺序是从父组件到子组件（因为父组件早于子组件先创建）
  // 2. 确保 user watchers 在 render watchers 前执行（因为 user watchers 早于 render watchers 创建）
  // 3. 父组件的watcher运行途中如果子组件销毁了，可以及时发现并跳转它
  queue.sort((a, b) => a.id - b.id)

  // 遍历 queue
  // 不缓存 queue 的长度，从 queueWatcher 中可以了解到 queue 在 flushing 途中也会发生变化
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run()
    // ...
  }
  // ...
  resetSchedulerState()
  // ...
}
```

通过 `queueWatcher` 可以看到，当有数据变化时，会先把 `watcher` 推入到待更新的队列中，并触发一次 `nextTick` 调用，在下一个 tick 中遍历队列 run 一遍。

那么下一个 tick 是什么时候？如何去定义它？

## nextTick

目前浏览器平台并没有实现原生的 nextTick 方法，Vue 中源码中分别用 Promise、setTimeout、setImmediate 等方式在 microtask（或是marcotask）中创建一个事件来模拟 nextTick，这样做的目的是在当前调用栈执行完毕以后（不一定立即）才会去执行这个事件。

先来看看 `nextTick` 的实现

```js
function nextTick (cb, ctx) {
  var _resolve;
  /* cb是我们传入的回调函数，cb会先被push到回调队列中 */
  callbacks.push(function () {
    if (cb) {
      try {
        cb.call(ctx);
      } catch (e) {
        handleError(e, ctx, 'nextTick');
      }
    } else if (_resolve) {
      _resolve(ctx);
    }
  });
  if (!pending) {
    // pending用于控制callback队列的调用节流
    pending = true;
    // timerFunc 是执行 callback队列的入口
    timerFunc();
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    /* 在支持Promise环境下，返回一个异步函数，支持nextTick的异步调用 */
    return new Promise(function (resolve) {
      _resolve = resolve;
    })
  }
}
```

nextTick函数主要做了这几件事情：

1. 把回调函数push到回调队列中，等待调用
2. 控制callback队列的调用，timerFunc()
3. 返回promise，支持外部异步调用

那么关键点应该就在 `timerFunc` 上了，我们来看看 `timerFunc` 具体做了什么

### timerFunc

```js
// 回调队列
/* 
这里存放的callback来源包含数据更新时的回调和this.$nextTick调用传入的回调
*/
var callbacks = [];
var pending = false;

/** 遍历回调队列执行回调 */
function flushCallbacks () {
  pending = false;
  // 开始清空队列时，先浅拷贝一份callback队列，目的是为了避免callback万一放生变化，不至于影响到当前的任务执行。
  var copies = callbacks.slice(0);
  callbacks.length = 0;
  for (var i = 0; i < copies.length; i++) {
    copies[i]();
  }
}

// 这段代码在Vuejs代码初始化时就会执行

var timerFunc;

/* istanbul ignore next, $flow-disable-line */
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  var p = Promise.resolve();
  // 第一处赋值
  timerFunc = function () {
    p.then(flushCallbacks);
    if (isIOS) { setTimeout(noop); }
  };
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  // ....
  // 第二处赋值
  timerFunc = function () {
    counter = (counter + 1) % 2;
    textNode.data = String(counter);
  };
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  // 第三处赋值
  timerFunc = function () {
    setImmediate(flushCallbacks);
  };
} else {
  // 第四处赋值
  timerFunc = function () {
    setTimeout(flushCallbacks, 0);
  };
}
```

我们分析一下以上代码的关键部分

| 属性或方法 | 作用 |
| --- | --- |
| callbacks | 回调队列，这里存放的是通过nextTick调用传入的回调函数。比如：数据更新时传入的更新回调，通过this.$nextTick调用传入的回调 |
| pending | 用于控制callback队列的调用节流（见 nextTick 函数内） |
| flushCallbacks | 遍历 callbacks 队列，按照添加顺序执行回调函数，在开始遍历队列前，先浅拷贝一份callback队列，目的是为了避免callback万一放生变化，不至于影响到当前的任务执行。 |
| timerFunc | 代理 flushCallbacks 的执行，由 timerFunc 来控制 flush 的时机 |

从分析上看，`timerFunc` 有四种被赋值的条件，这四种条件就是用来找到 nextTick 中的 tick。

1. 如果是在promise环境，flushCallbacks会在微任务环境中执行，ios的特殊处理（setTimeout）
2. 不支持promise的情况，会选择使用MutationObserver作为microtask
3. 当前环境不支持现有的microtask的情况，如果支持setImmediate，则使用setImmediate在下一个macrotask执行flushCallbacks
4. 上述条件都不支持的情况，选择使用setTimeout在下一个macrotask执行flushCallbacks

总的来说，优先选择Promise，MutationObserver等microtask来执行flushCallbacks。

如果不支持的话，就选择setImmediate和setTimeout等macrotask来执行flushCallbacks。
