---
title: Vue2源码细读-vue初始化
date: 2021-05-30 21:12:43
categories:
  - Vue2
  - 初始化-init
tags:
  - Web
	- Vue2
	- 源码
---

Vue.js 是一个以数据驱动为设计原理的框架。

所谓数据驱动，是指视图是由数据驱动生成的，比起视图如何生成，开发者更关注数据模型和数据流转。

接下来，我们会从源码角度来分析 Vue 是如何实现的，分析过程会以主线代码为主，重要的分支逻辑会放在之后单独分析。

先来看一个熟悉的 demo

## 熟悉的 demo

大家对这简单的案例应该都不会陌生，在我还是萌新的那会常看这个demo

demo的代码很简单，功能是点击数字然后累增

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title></title>
    <script src="/path/to/dist/vue.min.js"></script>
  </head>
  <body>
    <div id="app">
      <div @click="num++">
        {{ num }}
      </div>
    </div>
    <script>
    var vm1 = new Vue({
      el: '#app',
      data: {
        num: 1,
      }
    })
    </script>
  </body>
</html>
```

现在回想起来，这一切似乎就从这个 `new Vue()` 开始了：）

带着些许怀旧的心情，我们从 `new vue()` 开始一探究竟。

## new Vue() 做了什么

new Vue() 本质上是创建了一个 Vue 的实例对象，那么 Vue 是怎么定义的呢

### Vue 的定义

Vue本质是一个构造函数，只能通过 new 操作符去创建一个 Vue 的实例

```js
const app = new Vue({ /* options */ });
```

```js
// src/core/instance/index.js

// vue 构造函数
function Vue (options) {
  // 初始化的入口
  this._init(options)
}

// 下面的都是在 Vue.prototype 上挂载接来下要用到函数
initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)
```

Vue 本身很简单，就只做了一件事，就是调用初始化方法 `this._init`，并发参数传入

Vue 实例的函数并没直接写在 Vue 构造函数内，而是通过各种 mixin 挂载到了 Vue.prototype

其实这些函数都可以直接在当前文件中书写，但是拆开到不同模块分别挂载可以降低当前文件的耦合度，更利于代码的维护

### initMixin 初始化

`initMixin` 用于挂载初始化相关的函数到 Vue.prototype

`_init` 函数则是通过 `initMixin` 注入到 Vue 的原型链上

<!-- more -->

```js
// src/core/instance/init.js

export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    // ...
  }
}
```

### stateMixin 数据状态

`stateMixin` 用于挂载 控制数据状态 相关的函数到 Vue.prototype

```js
export function stateMixin (Vue: Class<Component>) {
  // 初始化 $data 和 $props 的 descriptor
  const dataDef = {}
  dataDef.get = function () { return this._data }
  const propsDef = {}
  propsDef.get = function () { return this._props }
  // ...
  Object.defineProperty(Vue.prototype, '$data', dataDef)
  Object.defineProperty(Vue.prototype, '$props', propsDef)

  // 注入 $set 和 $delete 方法
  Vue.prototype.$set = set
  Vue.prototype.$delete = del

  // 注入手动 $watch api
  Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: any,
    options?: Object
  ): Function {
    // .....
  }
}
```

`stateMixin`比较简单纯粹，主要做了这几件事

1. 初始化 $data 和 $props 的 descriptor
2. 注入 $set 和 $delete 方法
3. 注入手动 $watch api

### eventsMixin 事件发布订阅

`eventsMixin` 用于挂载 事件发布订阅 相关的函数 到 Vue.prototype

```js
export function eventsMixin (Vue: Class<Component>) {
  // 事件订阅
  Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {
    // ...
  }
  
  // 只触发一次的事件订阅
  Vue.prototype.$once = function (event: string, fn: Function): Component {
    // ...
  }

  // 取消订阅
  Vue.prototype.$off = function (event?: string | Array<string>, fn?: Function): Component {
    // ...
  }

  // 触发事件
  Vue.prototype.$emit = function (event: string): Component {
    // ...
  }
}
```

`eventsMixin` 主要做了这几件事

1. 挂载 事件订阅 函数
2. 挂载 只触发一次的事件订阅 函数
3. 挂载 取消订阅 函数
4. 挂载 触发事件 函数

### lifecycleMixin 生命周期

```js
export function lifecycleMixin (Vue: Class<Component>) {
  // vue 内部使用的更新函数
  Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    // ...
  }

  // vue 公开api，强制刷新组件
  Vue.prototype.$forceUpdate = function () {
    // ...
  }

  // vue 公开 api，销毁组件
  Vue.prototype.$destroy = function () {
    // ...
  }
}
```

### renderMixin 渲染VNode

`renderMixin` 用于挂载 控制渲染执行 相关的函数到 Vue.prototype

```js
export function renderMixin (Vue: Class<Component>) {
  // 注入运行时的一些辅助函数，供内部使用，目的是为了简化操作
  installRenderHelpers(Vue.prototype)

  // 公有API，我们熟悉的 $nextTick 方法
  Vue.prototype.$nextTick = function (fn: Function) {
    return nextTick(fn, this)
  }

  // 私有API，返回 render 函数创建的 VNode，非常重要
  Vue.prototype._render = function (): VNode {
    // ...
  }
}
```

`renderMixin` 只做了三件事

1. 注入运行时的一些辅助函数，供内部使用，目的是为了简化操作
2. 挂载我们熟悉的 `$nextTick`
3. 挂载内部API `_render`

从上述各种 mixin 中可以观察到，这里挂载到 prototype 上的方法分为两种

| 函数类型 | 备注 |
| --- | --- |
| 以 `_` 开头的私有方法 | 非公开暴露的API，仅提供给 vue 内部去使用，不建议外部使用这类私有方法，因为在后续是有可能会发生变化的 |
| 以 `$` 开发的公有方法 | 已经被公开暴露的 API，可以安心使用 |

以上就是 Vue 的构造函数定义，其中的函数简化了许多，目的是为了先从宏观的角度上对 Vue 初始化有个大概了解

## 初始化 `_init`

```js
// src/core/instance/init.js

Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    vm._uid = uid++
    
    // ...

    // 标记为 vue，避免被 observe
    vm._isVue = true
    // 合并 options
    if (options && options._isComponent) {
      initInternalComponent(vm, options)
    } else {
      /*
      * 这里合并的options来源有三个
      1、合并 vue 构造器上的通用 options
      2、参数传入的 options，加载优先级大于通用的options
      3、当前vm实例，这个会在加载策略中用到
      */
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    /* istanbul ignore else */
    // _renderProxy 是 vm 的别名，用于需要访问自身实例的情况
    if (process.env.NODE_ENV !== 'production') {
      // 在开发环境时，如果开发环境支持 `Proxy` API，则会用 Proxy 把 vm 包装一下
      initProxy(vm)
    } else {
      // 生产环境时就是 vm 自身
      vm._renderProxy = vm
    }
    // _self 也是 vm 的别名，用于需要访问自身实例的情况，跟 _renderProxy 在开发环境时还是有区别的，但在生产环境中它们是一样的
    vm._self = vm
    
    // 做了两件事
    // 1、初始化生命周期标识中的一些变量
    // 2、把自身vm绑定到parent.$children上
    initLifecycle(vm)
    // 更新父组件的attached事件监听器
    initEvents(vm)
    // 1、绑定生成Component和Element的函数到实例上
    // 2、$attrs和$listeners的响应式，创建$slots
    // 3、初始化_vnode、_staticTrees等虚拟dom和dom tree的关键信息
    initRender(vm)
    // 触发组件的 beforeCreate 事件
    callHook(vm, 'beforeCreate')
    // 在组件data/props初始化前，初始化变量注入
    initInjections(vm)
    // 1、data/props/computed的响应式
    // 2、绑定methods到实例this
    // 3、初始化watch
    initState(vm)
    // 在组件data/props初始化后，初始化变量提供
    initProvide(vm)
    // 触发组件的 created 事件
    callHook(vm, 'created')
    
    // ...

    if (vm.$options.el) {
      // 准备就绪后，把vdom生成的模板挂载到对应父节点上
      vm.$mount(vm.$options.el)
    }
}
```

好家伙，上面一串代码看起来是那么的简单清晰‼️大致浏览后便对 `Vue _init` 阶段有了一个轮廓

感慨完后，我们把其中各部分简单拆解一下，下面对它们做解释

### 合并配置 mergeOptions

```js
if (options && options._isComponent) {
  initInternalComponent(vm, options)
} else {
  vm.$options = mergeOptions(
    resolveConstructorOptions(vm.constructor),
    options || {},
    vm
  )
}
```

可以看到这里有两种场景的配置合并，不同场景合并逻辑也不一样

为了方便梳理，这里会改造一下最初的案例，后面会用到

```js
import Vue from 'vue'

var ChildComp = {
  template: '<div>{{msg}}</div>',
  created() {
    console.log('child created')
  },
  mounted() {
    console.log('child mounted')
  },
  data() {
    return {
      msg: 'Hello World'
    }
  }
}

Vue.mixin({
  created() {
    console.log('mixin created')
  }
})

var vm1 = new Vue({
  el: '#app',
  data: {
    num: 1,
  },
  render: h => h(childComp)
})
```

按照我们的思路，在执行 `this._init` 后，代码进入到这个场景 `手动调用场景`

所谓 `手动调用场景` 其实是指我们手动创建的 Vue 组件实例，用于区分平时我们 `import xxxComponent from '../path/your/vue-component'`，也就是 `组件场景`;

#### 手动调用场景

```js
vm.$options = mergeOptions(
  resolveConstructorOptions(vm.constructor),
  options || {},
  vm
)
```

这里合并的options来源有三个

1、合并 vue 构造器上的通用 options
2、参数传入的 options，加载优先级大于通用的options
3、当前vm实例，这个会在加载策略中用到

这里通过调用 mergeOptions 方法来合并，它实际上就是把 resolveConstructorOptions(vm.constructor) 的返回值和 options 做合并。

| options来源 | 说明 |
| --- | --- |
| resolveConstructorOptions(vm.constructor) | 通用的配置，Vue.options，initGlobalAPI时挂载 |
| options | new Vue时传入的参数 |
| vm | 组件实例 |

##### Vue.options

```js
export function initGlobalAPI (Vue: GlobalAPI) {
  // config
  const configDef = {}
  configDef.get = () => config
  Object.defineProperty(Vue, 'config', configDef)

  // 暴露了一些内置api，这些并不是公有api，不建议使用
  Vue.util = {
    warn,
    extend,
    mergeOptions,
    defineReactive
  }
  Vue.set = set
  Vue.delete = del
  Vue.nextTick = nextTick

  // 下面是 options 的初始化
  Vue.options = Object.create(null)
  /*
      export const ASSET_TYPES = [
          'component',
          'directive',
          'filter'
      ]
  */
  ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })

  // 增加 Vue 构造函数的索引
  Vue.options._base = Vue
  
  // ....
}
```

可以看到 Vue.options 实际是给 `components`，`filters`，`directives`设置默认值

##### resolveConstructorOptions 获取Vue构造器的options

```js
export function resolveConstructorOptions (Ctor: Class<Component>) {
  let options = Ctor.options
  if (Ctor.super) {
    // 父构造器的 options
    const superOptions = resolveConstructorOptions(Ctor.super)
    const cachedSuperOptions = Ctor.superOptions
    if (superOptions !== cachedSuperOptions) {
      Ctor.superOptions = superOptions
      const modifiedOptions = resolveModifiedOptions(Ctor)
      if (modifiedOptions) {
        extend(Ctor.extendOptions, modifiedOptions)
      }
      options = Ctor.options = mergeOptions(superOptions, Ctor.extendOptions)
      if (options.name) {
        options.components[options.name] = Ctor
      }
    }
    // 上述操作是直接操作了 options 对象
  }
  return options
}
```

上面一串代码简单来说

1. 获取当前构造器的 options
2. 如果构造器继承了别的构造器，则把当前构造器和父构造器的options合并

##### mergeOptions

```js
export function mergeOptions (
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  // ...

  normalizeProps(child, vm)
  normalizeInject(child, vm)
  normalizeDirectives(child)
  
  if (!child._base) {
    if (child.extends) {
      // 组件继承
      parent = mergeOptions(parent, child.extends, vm)
    }
    if (child.mixins) {
      // 合并 mixin 配置
      for (let i = 0, l = child.mixins.length; i < l; i++) {
        parent = mergeOptions(parent, child.mixins[i], vm)
      }
    }
  }
  
  const options = {}
  let key
  for (key in parent) {
    mergeField(key)
  }
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }
  function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
  return options
}
```

mergeOptions 主要功能就是把 parent 和 child 这两个对象根据一些合并策略，合并成一个新对象并返回。比较核心的几步，先递归把 extends 和 mixins 合并到 parent 上，然后遍历 parent，调用 mergeField，然后再遍历 child，如果 key 不在 parent 的自身属性上，则调用 mergeField

mergeField 中用到了策略模式，对于特定的 key，会有特殊的处理，mergeOptions中的 `vm` 参数也是用于此


##### merge strate 合并策略

```js
/* 
export const LIFECYCLE_HOOKS = [
  'beforeCreate',
  'created',
  'beforeMount',
  'mounted',
  'beforeUpdate',
  'updated',
  'beforeDestroy',
  'destroyed',
  'activated',
  'deactivated',
  'errorCaptured'
]
*/

function mergeHook (
  parentVal: ?Array<Function>,
  childVal: ?Function | ?Array<Function>
): ?Array<Function> {
  const res = childVal
    ? parentVal
      ? parentVal.concat(childVal)
      : Array.isArray(childVal)
        ? childVal
        : [childVal]
    : parentVal
  return res
    ? dedupeHooks(res)
    : res
}

LIFECYCLE_HOOKS.forEach(hook => {
  strats[hook] = mergeHook
})
```

不难看到，合并策略里的特殊 key 其实都是 Vue 的生命周期 key

这个函数的实现也非常有意思，就是

1、如果没有 childVal ，就返回 parentVal；
2、如果有 dhildval， 再判断是否存在 parentVal
3、如果存在 parentVal 就把 childVal 添加到 parentVal 后返回新数组；否则返回 childVal 的数组。
最后回到 mergeOptions 函数，一旦 parent 和 child 都定义了相同的钩子函数，那么它们会把 2 个钩子函数合并成一个数组。

这样做的目的是为了把同个组件内的相同的声明周期定义从上往下顺序执行

#### 手动调用场景的结果

当我们走完上述流程，案例中的 Vue 实例的 options 最后会变成这样

```js
vm.$options = {
  components: { },
  created: [
    function created() {
      console.log('mixin created')
    },
    function created() {
      console.log('child created')
    },
  ],
  mounted: [
    function mounted() {
      console.log('child mounted')
    },
  ],
  directives: { },
  filters: { },
  _base: function Vue(options) {
    // ...
  },
  el: "#app",
  render: function (h) {
    // ...
  }
}
```

#### 组件场景

所谓 `组件场景` 是指通过全局组件注册 `Vue.component` 或者 局部组件注册 `components: ['xxx']`

所有注册的组件都会经过 `createComponent` -> `Vue.extend` 处理

##### Vue.extend 组件创建和继承

我们来看看 `Vue.extend` 是怎么实现的

```js
  /**
   * Class inheritance
   */
  Vue.extend = function (extendOptions: Object): Function {
    // extendOptions 就是我们写的对象配置
    // 这个大家都熟悉 export default { ... }
    extendOptions = extendOptions || {}
    
    const Super = this
    const SuperId = Super.cid
    const cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {})
    // 缓存 在当前父节点下 时期的 当前构造函数
    if (cachedCtors[SuperId]) {
      return cachedCtors[SuperId]
    }
    // 组件名赋值
    const name = extendOptions.name || Super.options.name
    
    // 定义了一个 VueComponent 构造函数
    const Sub = function VueComponent (options) {
      // 又到了我们熟悉 _init 环节
      this._init(options)
    }
    
    // 下面是熟悉的手写基于prototype继承
    Sub.prototype = Object.create(Super.prototype)
    Sub.prototype.constructor = Sub
    
    // 增加 cid 标识
    Sub.cid = cid++
    
    // 合并基础配置
    Sub.options = mergeOptions(
      Super.options,
      extendOptions
    )
    
    // 标识父构造函数的别名
    Sub['super'] = Super

    if (Sub.options.props) {
      // 这里的 initProps 其实不是对 Props 做响应式，只是把 props 代理到了 prototype._props
      initProps(Sub)
    }
    if (Sub.options.computed) {
      // 同上
      initComputed(Sub)
    }

    // 这三个都是引用，便于继承父组件的功能
    Sub.extend = Super.extend
    Sub.mixin = Super.mixin
    Sub.use = Super.use

    // 前面提到过的基本属性，目的也是继承父组件的属性
    /*
      export const ASSET_TYPES = [
          'component',
          'directive',
          'filter'
      ]
    */
    ASSET_TYPES.forEach(function (type) {
      Sub[type] = Super[type]
    })
    
    // 允许组件使用自己，也就是递归使用自身
    if (name) {
      Sub.options.components[name] = Sub
    }

    // 保存对各个配置的引用。sub.options 才是当前组件真正的配置
    Sub.superOptions = Super.options
    Sub.extendOptions = extendOptions
    Sub.sealedOptions = extend({}, Sub.options)

    // 缓存 在当前父节点下 时期的 当前构造函数
    cachedCtors[SuperId] = Sub
    return Sub
  }
```

`Vue.extend` 主要做了这几件事

1. 缓存 在当前父节点下 时期的 当前构造函数
2. 组件名赋值
3. 定义了一个 VueComponent 构造函数
4. 基于prototype继承 Vue
5. 增加 cid 标识
6. 标识父构造函数的别名
7. 把 props 代理到了 `prototype._props`，computed 代理到 `prototype._computed`
8. 继承父组件的component,directive,filter属性
9. 允许组件使用自己，也就是递归使用自身
10. 保存对各个配置的引用。sub.options 才是当前组件真正的配置

`Vue.extend` 会在 `createComponent` 中被调用，用于获取真正的VueComponent构造函数

##### createComponent 创建组件VNode

我们先不讨论他们是在何时怎么流转到 `createComponent`

```js
export function createComponent (
  Ctor: Class<Component> | Function | Object | void, // 当前组件的 PlainObject
  data: ?VNodeData, // 父节点的 VNode
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  const baseCtor = context.$options._base

  // ctor 是指我们的SFC中 export 的对象，实质是一个普通js对象
  if (isObject(Ctor)) {
    // 这一步是把组件对象Options处理后才会真正的 Vue component 构造函数
    Ctor = baseCtor.extend(Ctor)
  }
  
  // ...
  
  data = data || {}

  // 合并父子 options
  resolveConstructorOptions(Ctor)

  // 注册组件函数钩子，非常重要，在后续的篇章中会详细解读
  installComponentHooks(data) 
  const name = Ctor.options.name || tag
  // 创建一个 VNode对象，因为组件还没有渲染，所以组件实例componentInstance其实还是 undefined
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  )
  return vnode
}
```

`createComponent` 最后返回了一个初始的 VNode，这个VNode的componentInstance其实还是空的，那么真正的组件实例会在什么时候创建呢？

这里关系到 `installComponentHooks`，这里简单提一下，在父组件patch过程中，会对触发子组件的 `init` 钩子，此时子组件才会去真正创建组件实例

这个`init` 钩子做了什么呢？

```js
const componentVNodeHooks = {
  init (vnode: VNodeWithData, hydrating: boolean): ?boolean {
    if (
      vnode.componentInstance &&
      !vnode.componentInstance._isDestroyed &&
      vnode.data.keepAlive
    ) {
      // kept-alive components, treat as a patch
      const mountedNode: any = vnode // work around flow
      componentVNodeHooks.prepatch(mountedNode, mountedNode)
    } else {
      // 在这里第一次创建组件的实例
      const child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance
      )
      child.$mount(hydrating ? vnode.elm : undefined, hydrating)
    }
  },
  // ...
}
```

##### createComponentInstanceForVnode 创建组件实例并挂载到VNode

```js
export function createComponentInstanceForVnode (
  vnode: any,
  parent: any
): Component {
  // 熟悉的配置
  const options: InternalComponentOptions = {
    // 这个标记大家应该有印象，就是前面用于区分手动调用场景和组件场景的关键
    _isComponent: true,
    // 标明父节点的VNode
    _parentVnode: vnode,
    // 父组件实例
    parent
  }
  // ...
  // vnode.componentOptions.Ctor 是创建Vnode时传入的VueComponent构造函数，通过这个构造函数创建出真正的组件实例
  return new vnode.componentOptions.Ctor(options)
}
```

##### initInternalComponent

组件场景的关键函数都已经梳理好了，我们回过头看看 `组件场景` 的关键处理函数

```js
export function initInternalComponent (vm: Component, options: InternalComponentOptions) {
  const opts = vm.$options = Object.create(vm.constructor.options)
  // doing this because it's faster than dynamic enumeration.
  const parentVnode = options._parentVnode
  opts.parent = options.parent
  opts._parentVnode = parentVnode

  const vnodeComponentOptions = parentVnode.componentOptions
  opts.propsData = vnodeComponentOptions.propsData
  opts._parentListeners = vnodeComponentOptions.listeners
  opts._renderChildren = vnodeComponentOptions.children
  opts._componentTag = vnodeComponentOptions.tag

  if (options.render) {
    opts.render = options.render
    opts.staticRenderFns = options.staticRenderFns
  }
}
```

首先执行把组件的保存到当前实例 vm.$options

接着又把实例化子组件传入的子组件父 VNode 实例 parentVnode、子组件的父 Vue 实例 parent 保存到 vm.$options 中，另外还保留了 parentVnode 配置中的如 propsData 等其它的属性。

这样看来，相较于 `手动调用场景`，initInternalComponent 只是做了简单一层对象赋值，并不涉及到递归、合并策略等复杂逻辑。

#### 组件场景的结果

经过上述代码执行，VueComponent 实例的options会变成情况

核心的地方在于我们设置的组件配置都在 `__proto__` 上

```js
vm.$options = {
  parent: Vue /*父Vue组件实例*/,
  propsData: undefined,
  _componentTag: undefined,
  _parentVnode: VNode /*父VNode实例*/,
  _renderChildren: undefined,
  __proto__: {
    components: { },
    directives: { },
    filters: { },
    _base: function Vue(options) {
        //...
    },
    _Ctor: {},
    created: [
      function created() {
        console.log('mixin created')
      }, function created() {
        console.log('child created')
      }
    ],
    mounted: [
      function mounted() {
        console.log('child mounted')
      }
    ],
    data() {
       return {
         msg: 'Hello Vue'
       }
    },
    template: '<div>{{msg}}</div>'
  }
}
```

## 总结

在new Vue()过程中，主要做了这几件事：

1. 合并传入的options和通用的options，把options挂载到vm.$options
2. 初始化生命周期标识中的一些变量，把自身vm绑定到parent.$children上
3. 更新父组件的attached事件监听器
4. 绑定生成Component和Element的函数到实例上，$attrs和$listeners的响应式，创建$slots，初始化_vnode、_staticTrees等虚拟dom和dom tree的关键信息
5. 触发组件的 beforeCreate 事件
6. data/props/computed的响应式，绑定methods到实例this，初始化watch
7. 触发组件的 created 事件
8. 准备就绪后，把vdom生成的模板挂载到对应父节点上

在 `_init` 过程中，做了很多模块的初始化

* 初始化生命周期 initLifecycle
* 初始化事件 initEvents
* 初始化渲染VNode initRender
* 初始化数据注入 initInjections
* 初始化数据状态 initState
* 初始化数据暴露 initProvide
* 挂载 vm.$mount
* 注册组件函数钩子 installComponentHooks
* ...等等

这些将放在后面的篇章中一一细读
