---
title: Vue2源码细读-响应式原理
date: 2021-06-02 10:01:17
categories:
  - Vue2
  - 响应式原理
tags:
  - Web
	- Vue2
	- 源码
	- 响应式原理
	- Observer
	- Dep
	- Watcher
---

## 如何理解响应式

「响应式数据」的特性是数据改变时自动更新视图，当数据变化后，自动通知到对应的观察者，更新操作函数

Vue的响应式则用到了 `Object.defineProperty`，可能很多小伙伴之前都了解过，在分析 Vue 的响应式原理之前，我们先简单介绍一下
`Object.defineProperty` 的用法

## Object.defineProperty

该方法允许精确地添加或修改对象的属性。

通过设置 get/set 属性，可以代理属性的获取和写入操作，这两个是实现 Vue 响应式的关键

```js
const object1 = {};

Object.defineProperty(object1, 'property1', {
  // 赋值
  set(val) {
    console.log('set', val);
    this.value = val;
  },
  get() {
    console.log('get');
    return this.value || 1;
  },
//   value: 1,
});

// object1.property1 = 77;

console.log(object1.property1);
object1.property1 = 2;
console.log(object1.property1);
```

此外，`defineProperty` 可以实现控制属性的可覆写、可枚举、可配置等操作

```js
const object1 = {};

Object.defineProperty(object1, 'property1', {
  // 赋值
  value: 42,
  // 不可覆写
  writable: false
});

object1.property1 = 77;
// 严格模式下抛出错误，不可覆写

console.log(object1.property1); // 42
```

简单了解 `defineProperty` 后我们开始进入正题

## Vue响应式

![reactive1](https://tva1.sinaimg.cn/large/008i3skNgy1gqz9uduek2j30xc0kuq4o.jpg)

这是官方的响应式流程图，我们今天需要重点关注的地方有几个

1. Data 模块是如何转换成响应式数据结构？
2. getter 是如何收集依赖？依赖具体是什么？
3. setter 如何通知 Watcher？
4. Watcher 是什么？Watcher 的作用是什么？
5. Data 和 Watcher 的关系是什么？
6. Watcher 是如何触发组件 render？

<!-- more -->

### 简单回顾 `_init` 初始化过程

我们先回想下在 `Vue初始化` 篇章中的 `_init` 函数

`_init` 函数主要是对初始化组件的生命周期，事件，渲染VNode，数据响应式，数据注入和提供等函数做了初始化，其中数据响应式 `initState` 是这次的关键点

```js
// src/core/instance/init.js

Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    // ...
    // 触发组件的 beforeCreate 事件
    callHook(vm, 'beforeCreate')
    // ...
    // 1、data/props/computed的响应式
    // 2、绑定methods到实例this
    // 3、初始化watch
    initState(vm)
    // ...
    // 触发组件的 created 事件
    callHook(vm, 'created')
}
```

### initState 初始化数据状态

```js
export function initState (vm: Component) {
  // 创建私有属性 _watchers，用于保存当前组件中创建的render watcher,internal watcher,user watcher
  vm._watchers = []
  const opts = vm.$options
  
  // 初始化 props
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    // 初始化 data
    initData(vm)
  } else {
    // 如果没有data属性，则给 _data 属性赋值空对象
    observe(vm._data = {}, true /* asRootData */)
  }
  // 初始化 computed
  if (opts.computed) initComputed(vm, opts.computed)
  // 初始化 watch
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

`initState` 方法主要是对我们设置的 props、methods、data、computed 和 wathc 等属性做初始化，下面我们重点分析 `props` 和 `data`

对于 `methods` 会在后续的篇幅中单独解读

`computed` 和 `watch` 都是利用了 `watcher` 来做处理，这两个同样会在后续的篇幅中单独解读

#### initProps props响应式化

```js
function initProps (vm: Component, propsOptions: Object) {
  const propsData = vm.$options.propsData || {}
  const props = vm._props = {}
  // 第一次遍历 props 对象时把key保存到数组，避免后续遍历对象
  const keys = vm.$options._propKeys = []
  const isRoot = !vm.$parent
  // 根节点不需要响应式
  if (!isRoot) {
    // 如果是根节点，会把全局变量 shouldObserve 置位 false
    toggleObserving(false)
  }
  // 遍历 props
  for (const key in propsOptions) {
    keys.push(key)
    const value = validateProp(key, propsOptions, propsData, vm)
    // ...
    defineReactive(props, key, value)
    // ...
    // static props are already proxied on the component's prototype
    // during Vue.extend(). We only need to proxy props defined at
    // instantiation here.
    if (!(key in vm)) {
      proxy(vm, `_props`, key)
    }
  }
  toggleObserving(true)
}
```

props 的初始化主要过程，就是遍历定义的 props 配置，把有效的 props 都转换为响应式数据。

遍历时主要做两件事情：

1. 先调用 `defineReactive` 方法把每个 prop 对应的值变成响应式，可以通过 `vm._props.xxx` 访问到定义 props 中对应的属性
2. 另一个是通过 `proxy` 函数把 `vm._props.xxx` 的访问代理到 `vm.xxx` 上

这里用到了两个很重要的函数：`defineReactive` 和 `proxy`

这两个函数在后面会一块介绍，我们继续看 `initData` 的实现

#### initData data响应式化

```js
function initData (vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    // 初始化 data 的数据时，会对 data 与 props，methods 中的 key 做个检查
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      // 代理 _data 的访问到 vm 实例上， 访问 vm.xxx 可以访问到 vm._data.xxx，与前面的 _props 一样的处理
      proxy(vm, `_data`, key)
    }
  }
  // 这个函数是用来把 data 变成响应式的关键
  observe(data, true /* asRootData */)
}
```

data 初始化的过程主要也是做了两件事

1. 一个是对定义 data 函数返回对象的遍历，通过 proxy 把每一个值 `vm._data.xxx` 都代理到 `vm.xxx` 上
2. 另一个用 observe 方法把整个 data 转换成响应式的结构

通过观察 initProps 和 initData 的实现可以看到，无论是 props 或是 data，都是把它们变成响应式对象

在这个过程我们接触到几个重要方法

* proxy
* defineReactive // 响应式数据结构的关键函数
* observe

## proxy

大家都知道有个新的API叫做 Proxy，这里的 proxy 与 js的api API并不是同一个

vue2 中的 proxy 函数是用来把绑定实例的 data,props,methods等属性

比如：我们通常在组件中会这样访问 data，props，`this.xxx`，但是这个 `xxx` 实际不存在与 vm 实例上，只是通过 `defineProperty` 代理到对应的属性上了

```js
export function proxy (target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

代码很简单，清晰明了，就只做一件事

* 通过 `defineProperty` 代理了 vm 属性的访问，比如前面提到的 `this.num` 访问到 `this._data.num`

## defineReactive

```js
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  // Dep 对象，这个很关键，后续会单独开一个篇章解读
  const dep = new Dep()
  // 过滤不可配置的对象
  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // 保存原来的 set get 方法
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  // 如果是深层遍历，那么会先把属性中的对象先转换成响应式结构
  let childOb = !shallow && observe(val)
  
  // 响应式的关键
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    
    // 设置 get 属性，目的是做依赖收集
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    
    // 设置 set 属性，目的是做更新事务收集
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      
      // 如果新的值是对象，在深层遍历的模式下会把新值也转换成响应式结构
      childOb = !shallow && observe(newVal)
      
      // 通知watchers有更新，这一步很关键
      dep.notify()
    }
  })
}
```

`defineReactive` 函数做的事情也很简单

1. 创建 dep 实例，dep实例可以用来收集当前的watcher
2. 如果是深层遍历，那么会先把属性中的对象先转换成响应式结构
3. 通过 `defineProperty` 给对象设置 get 属性，目的是做依赖收集，收集的是watcher（render watcher,user watcher）
4. 通过 `defineProperty` 给对象设置 set 属性，目的是做更新事务收集，dep通知收集的watcher队列，watcher去做对应的处理

## observe 方法

observe 函数是给对象（不包括 VNode和组件实例vm）创建 `Observer` 实例对象

```js
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    // 满足一下条件的可以创建 Observer 对象
    // 1、可以被 observe
    // 2、非服务端渲染
    // 3、数组或普通对象
    // 4、可扩展
    // 5、不是Vue组件实例
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    // 根节点数据会统计当前对象中的 Observer 对象个数
    ob.vmCount++
  }
  return ob
}
```

`observe` 函数并没有出现 `defineReactive` 相关的操作，响应式转换的操作其实放到了 `Observer` 对象中处理

## Observer 对象

在 `observe` 函数中提到过 `Observer` 对象，它的核心作用是将普通对象转换为响应式数据结构

![](https://tva1.sinaimg.cn/large/008i3skNgy1gr2zpu5eb7j31dv0u0q6d.jpg)

```js
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      // 数组转换成响应式数据结构
      this.observeArray(value)
    } else {
      // 对象转换成响应式数据结构
      this.walk(value)
    }
  }

  walk (obj: Object) {
    const keys = Object.keys(obj)
    // 遍历对象执行 defineReactive
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  observeArray (items: Array<any>) {
    // 遍历对象执行 observe
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

`Observer` 对象比较简单，主要就两件事

1. 创建一个 dep 对象
2. 深度遍历 value，将对象属性都转成响应式数据结构

## Dep 对象

`Dep` 对象核心作用是：管理依赖，用于辅助 响应式 Data 和 Watcher

想象这样一个场景：某个函数执行时用到了某个响应式数据，但是它是无法直接知道是哪个函数使用了这个响应式数据，当数据更新时也就无法在数据找到这个函数重新执行。

为了解决这个问题，需要用到 依赖管理 `Dep` 对象

通过前面的 `defineProperty` 函数可以了解到，每次触发属性的取值逻辑时都会 get 函数，然后执行 `dep.depend()`

这是管理依赖的第一点：`收集 watcher`，用一个 subs 数组管理，依赖收集的目的是为了知道是谁在使用数据

通过前面的 `defineProperty` 函数还可以了解到，每次修改属性值时都会 set 函数，然后执行 `dep.notify()`

这是管理依赖的第二点：`通知 watcher`，遍历subs数组逐个执行 `watcher.update()`，通知的目的是为了告诉正在使用数据的watcher需要更新了

![](https://tva1.sinaimg.cn/large/008i3skNgy1gr311meb9oj31qb0sqacf.jpg)

```js
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  // 把当前watcher添加到 dep.subs
  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  notify () {
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      subs.sort((a, b) => a.id - b.id)
    }
    // 遍历 watchers 逐个通知
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}
```

## Watcher 对象

### Watcher 对象是什么

`Watcher` 是用在 依赖收集 和 组件更新代理 的对象

`Watcher` 中需要先关注几个重要的方法和属性

1. this.getter // `vm` 组件实例的更新函数
2. get() {} // 代理 `this.getter` 执行，做了些运行时的判断
3. update() {} // 接收 `dep.notify()`，把当前 watcher 推入到 `dep.subs` 中
4. run() {} // 代理 `this.get` 执行，做了些运行时的判断

```js
// 简化了原有的代码，保留了这次我们需要关注的部分

export default class Watcher {
  vm: Component;
  expression: string;
  cb: Function;
  id: number;
  getter: Function;
  value: any;
  // ... 还有很多成员属性

  // 构造函数
  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    this.vm = vm
    if (isRenderWatcher) {
      vm._watcher = this
    }
    vm._watchers.push(this)
    // 很多成员属性的初始化
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      // getter 实质就是更新的函数
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = noop
        process.env.NODE_ENV !== 'production' && warn(
          `Failed watching path: "${expOrFn}" ` +
          'Watcher only accepts simple dot-delimited paths. ' +
          'For full control, use a function instead.',
          vm
        )
      }
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  /**
   * 用于执行 getter，并开始收集依赖
   */
  get () {
    // watcher 推入到 Dep.target 中
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      // getter 开始执行
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }

  /**
   * 订阅接收更新
   */
  update () {
    // ....
    // 推入待更新的 watcher 队列
    queueWatcher(this)
  }

  /**
   * Scheduler 执行 getter 的接口
   */
  run () {
    if (this.active) {
      // 执行 getter
      const value = this.get()
      // ...
    }
  }
}
```

通过阅读代码我们可以简单了解到，watcher有区分几种情况

1. isRenderWatcher 这个对应的是 `vm` 组件实例级别的 watcher，一般称为 `render watcher`
2. lazy 这个对应的是 `computed` 创建的 watcher，一般称为 `internal watcher`，后续会独立一个篇幅解读
3. cb 参数和 this.user 判断，这个对应的是通过 `watch` 属性创建的 watcher，一般称为 `user watcher`，后续会独立一个篇幅解读

### Watcher 如何接收通知

前面的 Dep 提到过 setter 触发时会通知 Watcher 需要更新了，watcher 是怎么接收通知的呢？

![](https://tva1.sinaimg.cn/large/008i3skNgy1gr31d577k6j31rs0u0gq5.jpg)

`watcher` 通过暴露 `update` 函数来接收更新的通知，收到通知后把当前的 watcher 推入待更新的队列中，然后等待下一个 `tick` 去遍历更新

下一个 `tick` 就是我们熟悉的 `nextTick`，具体会在后面的篇章中单独解读

当更新队列开始 `flush` 时，会把排好序的 `Watchers` 队列遍历一遍，取出 `Watcher` 执行 `watcher.run()`

通过 watcher 的 run 实现可以看到，run 实际是 get 方法的代理执行

get 方法会调用 watcher 实例上的 getter 函数，也就是组件创建 Watcher 时传入的组件更新方法 `vm._update`

`vm._update` 组件 `patch` 的入口，具体实现会在后面的篇章中单独解读

## 总结

经过前面对流程和函数的拆解，Vue2中的响应式原理应该有了一个大致的轮廓

![](https://tva1.sinaimg.cn/large/008i3skNgy1gr32nsmyerj32dn0u047q.jpg)

1. props options 会直接遍历然后通过 `defineReactive` 生成响应式数据结构
2. data options 通过创建 `Observer` 对象，由 `Observer` 对象将 data 属性转换成响应式数据结构
3. 组件 render 时会如果使用了 data 或 props 中的值，会触发对应的 getter
4. `getter` 触发后，会调用 `dep.depend()` 收集依赖，也就是当前的 `watcher`
5. 当组件修改 data 时，会触发对应的 `setter`
6. `setter` 出发后，会调用 `dep.notify()` 去通知收集到的 `watcher`
7. `watcher` 暴露的 `update` api 在接收到 `dep.notify()` 的通知时会被调用
8. 被执行 `watcher.update()` 的 `watcher` 会推入到待更新的队列中 `queue`，这个会在其他篇幅中解读
9. 最后通过执行 `watcher.run()` 来调用组件实例的更新方法

以上便实现了Vue2中的响应式原理的闭环，其中涉及到 `observe` `Observer` `Dep` `watcher` 函数和模块

## 参考

> [Object.defineProperty](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)