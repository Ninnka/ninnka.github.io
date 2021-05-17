---
title: 手撕源码-redux从原理到实现
date: 2021-05-15 16:36:58
categories:
	- Web
	- 前端
	- redux
tags:
  - Web
	- 前端
	- redux
---

# 前言

本文主要分为两部分，第一部分是分析redux的原理，第二部分是根据我们分析的结果来实现一个五脏六腑俱全的mini-redux

分析redux原理的过程中，会对必要但不是全部的概念都解释一遍，所以需要对 `redux` 有基础的认识

> [redux 官方教程传送门](https://redux.js.org/tutorials/essentials/part-1-overview-concepts)

在认识 `redux` 之前，我只知道 `react-redux`，并且认为 redux 是 react 的一部分，相信大多入门的新人也会有这样的误解

正文开始之前，先同步一下我们的认识点
- redux 与 react 没有任何关系，redux 可以用在任何框架中，比如 React，Vue 等等；
- redux 是一个数据状态管理器，用来实现数据的中心化管理
- 正文不会对 react-router 做原理分析，react-router 会另开一篇做分析

# Redux

redux 基础功能是数据管理和通知订阅，redux是如何实现的呢？

## Store

先来看一段官方demo

```javascript
import { createStore } from 'redux'

function counterReducer(state = { value: 0 }, action) {
  switch (action.type) {
    case 'counter/incremented':
      return { value: state.value + 1 }
    case 'counter/decremented':
      return { value: state.value - 1 }
    default:
      return state
  }
}
let store = createStore(counterReducer, { value: 0 })

store.subscribe(() => console.log(store.getState()))

store.dispatch({ type: 'counter/incremented' })
// {value: 1}
store.dispatch({ type: 'counter/incremented' })
// {value: 2}
store.dispatch({ type: 'counter/decremented' })
// {value: 1}
```

redux 通过 `createStore` 创建一个数据中心管理器 `store`
createStore 可以说是数据流的入口

```javascript
// src/createStore.ts
function createStore(
  reducer: Reducer<S, A>,
  preloadedState?: PreloadedState<S> | StoreEnhancer<Ext, StateExt>,
  enhancer?: StoreEnhancer<Ext, StateExt>
) {
  // 对 reducer，preloader，enhancer做了很多空值校验
  // ...

  // 把 reducer，state，listener 都保存到闭包中，供下面的函数处理时使用
  let currentReducer = reducer
  let currentState = preloadedState as S
  let currentListeners: (() => void)[] | null = []
  let nextListeners = currentListeners
  let isDispatching = false
  
  // 创建了若干函数用于操作数据流

  // 获取当前的数据状态
  function getState() {}

  // 订阅
  function subscribe() {}

  // 派发事件
  function dispatch() {}

  // 替换 reducer
  function replaceReducer() {}

  // 监听 state 变化的观测器
  // https://github.com/tc39/proposal-observable
  function observable() {}

  // 发起初始化的 action，这个是内部的action，需要在业务代码中有对应的 reducer 才会触发响应
  dispatch({ type: ActionTypes.INIT } as A)

  const store = {
    dispatch: dispatch as Dispatch<A>,
    subscribe,
    getState,
    replaceReducer,
    [$$observable]: observable
  } as unknown as Store<ExtendState<S, StateExt>, A, StateExt, Ext> & Ext

  // 最后返回了一个大对象，把上述创建的函数都丢了出去
  return store;
}
```

<!-- more -->

### getState

```javascript
// src/createStore.ts

// 获取当前的数据状态
function getState() {
  return currentState as S
}
```

getState很好理解，就是返回闭包中保存的 currentState

### subscribe

```javascript
// src/createStore.ts

// 订阅state的变化，当state change时，会触发监听器
function subscribe(listener: () => void) {
  // 订阅状态锁
  let isSubscribed = true
  // 把监听器推入 nextListeners，供后续响应式使用
  nextListeners.push(listener)

  // 返回一个取消订阅的函数
  return function unsubscribe() {
    if (!isSubscribed) {
      return
    }

    isSubscribed = false

    const index = nextListeners.indexOf(listener)
    nextListeners.splice(index, 1)
    currentListeners = null
  }
}
```

subscribe也很好理解，主要是把我们的订阅的监听器都保存到数组中，并提供一个取消订阅的函数

### dispatch

```javascript
// src/createStore.ts

function dispatch(action: A) {
  if (isDispatching) {
    // ...输出校验log
    return;
  }
  try {
    isDispatching = true
    // 通过我们定义的 reducer去处理后得到一个新的state
    currentState = currentReducer(currentState, action)
  } finally {
    isDispatching = false
  }
  // 通过 subscribe 添加的订阅器会在经过 reducer 处理完数据后响应
  const listeners = (currentListeners = nextListeners)
  for (let i = 0; i < listeners.length; i++) {
    const listener = listeners[i]
    listener()
  }
  return action
}
```

`dispatch` 做的事情很简单，派发一个事件，然后通过我们定义的 reducer去处理后得到一个新的state，并遍历监听器数组，逐个触发


### replaceReducer

```javascript
// src/createStore.ts

function replaceReducer<NewState, NewActions extends A>(
  nextReducer: Reducer<NewState, NewActions>
): Store<ExtendState<NewState, StateExt>, NewActions, StateExt, Ext> & Ext {
  ;(currentReducer as unknown as Reducer<NewState, NewActions>) = nextReducer

  // This action has a similar effect to ActionTypes.INIT.
  // Any reducers that existed in both the new and old rootReducer
  // will receive the previous state. This effectively populates
  // the new state tree with any relevant data from the old one.
  dispatch({ type: ActionTypes.REPLACE } as A)
  // change the type of the store by casting it to the new store
  return store as unknown as Store<
    ExtendState<NewState, StateExt>,
    NewActions,
    StateExt,
    Ext
  > &
    Ext
}
```

`replaceReducer` 就更简单了，主要是把旧的 `reducer` 替换成新的，并触发一个内部的 `action`


### observable

```javascript
function observable() {
  const outerSubscribe = subscribe
  return {
    // 定义一个新的 observable.subscribe 来包装 store.subscribe
    subscribe(observer: unknown) {
      // 定义一个满足 `observable` 条件的执行器函数，用于操作我们传入的 observable
      function observeState() {
        const observerAsObserver = observer as Observer<S>
        if (observerAsObserver.next) {
          observerAsObserver.next(getState())
        }
      }

      observeState()
      // 最后使用 store 中的 subscribe订阅一下，然后把observable执行器传入
      const unsubscribe = outerSubscribe(observeState)
      return { unsubscribe }
    },

    [$$observable]() {
      return this
    }
  }
}
```

大家可能对 `observable` 比较陌生，这是一个新的 `proposal`
[proposal-observable](https://github.com/tc39/proposal-observable)

具体可以戳上面的链接查看

redux 通过 store 把这个 api 暴露出来了，但是实际官方并不希望它在外部被调用，
非要使用的话，可以通过这种方式


```javascript
let subscription = store[$$observable].subscribe({
    next(val) { console.log(val) },
    error(err) { console.log(err) },
    complete() { console.log("complete") },
});
```

这个api是把 `subscribe` 包了一层，然后可以以 `observable` 的形式调用，需要注意的是$$observable，这是 redux 的一个内部变量
```javascript
const $$observable = /* #__PURE__ */ (() =>
  (typeof Symbol === 'function' && Symbol.observable) || '@@observable')()
```

官方并没有把 `$$observable` 变量主动暴露出来，侧面说明官方不希望外部使用这个api

# Redux-toolkit
