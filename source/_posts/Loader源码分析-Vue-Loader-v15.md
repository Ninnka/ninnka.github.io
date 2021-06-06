---
title: Loader源码分析-Vue Loader v15
date: 2021-06-06 17:52:54
categories:
  - Webpack
  - Vue Loader v15
tags:
  - Web
	- Vue Loader
  - Vue Loader v15
	- 源码
---

# 什么是 vue-loader

简单来说，`vue-loader` 是把 `.Vue` 文件编译成 `.js`，即可在浏览器中运行，也可以通过 `vue-server-render` 在 node 环境运行。

# vue-loader 15

`vue-loader 15` 向较于过去的版本，有许多重要的改动，这些改动体现在：

1. loader推导策略变化
2. 独立出 `VueLoaderPlugin`
3. ...等等

更多细节可以查阅官方迁移指南：[Vue Loader 迁移](https://vue-loader.vuejs.org/zh/migrating.html)

# vue-loader 编译过程

vue-loader 的处理流程可以大致分为几个部分

1. 入口函数解析 `.vue` 文件
2. `parse` 解析 `.vue` 文件，生成包含不同模块的 `descriptor`
3. 根据不同模块做 `loader` 推导
4. `VueLoaderPlugin` 处理

## vue-loader 入口函数

vue-loader 入口代码不多，我把入口函数的流程做成了一个简单的 UML 图，通过图也能快速对流程有个初步的印象

![](https://tva1.sinaimg.cn/large/008i3skNgy1gr8mbc91ibj322z0u0as3.jpg)

<!-- more -->

vue-loader 入口函数主要做了这几件事

1. 通过 `parse` 生成 `descriptor`，描述符中包含了vue解析后的各个结果，比如template、style、script
2. 如果 `resourceQuery` 中有 `type` 说明已经被处理过，可以进行分块处理，是loader推导策略的关键步骤点之一
3. 生成 `code`，这里的 `code` 包含 import vue 文件的代码，并且引入携带不同类型的type，会触发新的 `vue-loader` 新一轮的执行，是loader推导策略的关键步骤点之一

通过上面的 UML 图可知，`.vue` 文件初次编译时会走生成 `code`的流程，那么生成的 `code` 究竟是什么呢？

通过调试 `vue-loader`，把 code 打印出来，仔细看下图中红色框中的部分

![](https://tva1.sinaimg.cn/large/008i3skNgy1gr7tp6hmd9j30u00vf7b2.jpg)

可以发现几句 `import` 中，都是从 source.vue 获取对象，并且路径上携带了参数，这些参数就是 `resourceQuery`，type有三种不同类型，分别是 `template` | `script` | `styles`

这些import会继续触发新一轮的 `vue-loader` 执行（简单来说，可以先这么理解，详细过程在下面会分析），于是接下来到了途中 `resourceQuery` 有 `type` 的情况

下面是进行了适当删减后的源码，把上述涉及到的代码做了保留，对代码本身感兴趣的可以浏览

```js
module.exports = function (source) {
  // this 是指 webpack 注入的内容
  // 因为函数是在 webpack 下执行的，this绑定到了 webpack context上
  const loaderContext = this

  // 各种各样的工具属性和方法，先不看
  const {
    target,
    request,
    minimize,
    sourceMap,
    rootContext,
    resourcePath,
    resourceQuery = ''
  } = loaderContext

  // 使用 loader 时可以通过 options 来传参
  // e.g { loader: 'vue-loader', options: {} }
  const options = loaderUtils.getOptions(loaderContext) || {}

  // 通过 parse 解析.vue文件
  // 描述符中包含了vue解析后的各个结果，比如template、style、script
  const descriptor = parse({
    // .vue 文件
    source,
    // 编译器，默认就是 require('vue-template-compiler')
    compiler: options.compiler || loadTemplateCompiler(loaderContext),
    // ...
  })

  // 如果文件还有 type，说明 vue 文件已经被解析过一次了，这里的type有三种
  // type = template | script | style
  // 既然已经给 vue 分块了，那么接下来就是给不同的 block 找到对应的 loader
  // 这里就是推导策略的开始
  if (incomingQuery.type) {
    return selectBlock(
      descriptor,
      loaderContext,
      incomingQuery,
      !!options.appendExtension
    )
  }

  // 处理 styles scoped
  const hasScoped = descriptor.styles.some(s => s.scoped)
  // 处理函数式组件
  const hasFunctional = descriptor.template && descriptor.template.attrs.functional

  // template 处理
  let templateImport = `var render, staticRenderFns`
  let templateRequest
  if (descriptor.template) {
    // ...
    templateImport = `import { render, staticRenderFns } from ${request}`
  }

  // script 处理
  let scriptImport = `var script = {}`
  if (descriptor.script) {
    // ...
    scriptImport = (
      `import script from ${request}\n` +
      `export * from ${request}` // support named exports
    )
  }

  // styles 处理
  let stylesCode = ``
  if (descriptor.styles.length) {
    // 生成样式代码
    stylesCode = genStylesCode(
        // ...
    )
  }

  // 生成编译后的代码
  let code = `
   // ...
  `;

  return code
}
```

## parse .vue组件解析

`parse` 方法内部处理了 vue SFC 文件，前面提到过，编译的方法默认是通过 `vue-template-compiler` 处理

主要是通过 `compiler.parseComponent` 函数对 `.vue` 文件进行编译

那么 `vue-template-compiler` 究竟是什么呢？

### vue-template-compiler

在了解 `vue-tempalte-compiler` 之前，我对 vue 的编译过程有些了解，既然它们都是处理 vue SFC 文件，那么它们会不会是同一份代码呢，抱着疑问的态度，我们先看看 `vue-template-compiler` 的 readme.md

> This package is auto-generated. For pull requests please see [src/platforms/web/entry-compiler.js](https://github.com/vuejs/vue/tree/dev/src/platforms/web/entry-compiler.js).

在 readme.md 可以看到官方对它的说明，实际 `vue-template-compiler` 是一份自动生成的代码，它本质就是 vue 中的 `sfc/parse`

![](https://tva1.sinaimg.cn/large/008i3skNgy1gr8igh1tfdj316k0u078y.jpg)

但今天的主角并不是 `vue-template-compiler`，也不是 `sfc/parse`，我会在后面的篇章中对 `vue build` 的过程做一个详细的解读

## parse 流程

![](https://tva1.sinaimg.cn/large/008i3skNgy1gr8lhmu0s2j31bx0u0agk.jpg)

## vue-loader 推导策略

在 vue-loader 入口函数分析中已经可以了解到，入口函数最终会生成一个 `code`，这个 `code` 包含了几个 import 语句，import 语句都含有 vue 标识并且标明了不同的分块类型

这些 import 语句会被 VueLoaderPlugin 捕捉并做推导策略处理

### VueLoaderPlugin

老规矩，先来看 VueLoaderPlugin 的代码

代码删减后及其简单，就一件事：注入 `pitcher-loader`，用于处理 vue 分块 loader 推导

```js
class VueLoaderPlugin {
  apply (compiler) {
    // ...
    // pitcher loader，用于 vue 分块 loader 推导
    const pitcher = {
      loader: require.resolve('./loaders/pitcher'),
      resourceQuery: query => {
        // 解析 query 上带有 vue 标识的资源
        const parsed = qs.parse(query.slice(1))
        return parsed.vue != null
      },
      options: {
        cacheDirectory: vueLoaderUse.options.cacheDirectory,
        cacheIdentifier: vueLoaderUse.options.cacheIdentifier
      }
    }

    // 重置 webpack 的 rules，把pitcher放在了第一个
    compiler.options.module.rules = [
      pitcher,
      ...clonedRules,
      ...rules
    ]
  }
}
```

### pitcher-loader

`VueLoaderPlugin` 的主要作用就是注入 `pitcher-loader`，由此可先，实际处理推导过程的是 `pitcher-loader`，`VueLoaderPlugin` 不过是loader的注入器

那么 `pitcher-loader` 是怎么做 loader 推导的呢？

前面提到 入口函数生成的 code，code 中包含 import 语句

这些 import 语句会触发 `pitcher-loader`，pitcher 根据resourceQuery来区分不同块，并生成不同的 loader request

```js
module.exports.pitch = function (remainingRequest) {
  // 生成 loader request import
  const genRequest = loaders => {
    loaders.forEach(loader => {
      // ....
    })
    const request = loaderUtils.stringifyRequest(this, '-!' + [
      ...loaderStrings,
      this.resourcePath + this.resourceQuery
    ].join('!'))
    return request
  }

  // .. 处理样式 loader
  if (query.type === `style`) {
    const cssLoaderIndex = loaders.findIndex(isCSSLoader)
    if (cssLoaderIndex > -1) {
      // ...
      return query.module
        ? `export { default } from  ${request}; export * from ${request}`
        : `export * from ${request}`
    }
  }

  // 处理模板 loader
  if (query.type === `template`) {
    // ...
    return `export * from ${request}`
  }
  // ...
  const request = genRequest(loaders)
  return `import mod from ${request}; export default mod; export * from ${request}`
}
```

## loader 推导流程

![](https://tva1.sinaimg.cn/large/008i3skNgy1gr8ma1vrj5j31lw0u07wi.jpg)

# 总结

把上述过程汇聚成一张 UML 图，通过这张图可以对整个流程有个清晰认识

![](https://tva1.sinaimg.cn/large/008i3skNgy1gr8mautex7j324d0u0qv6.jpg)

vue-loader15 的整体过程可以划分为一下几个部分

1. 通过 `parse` 生成 `descriptor`，描述符中包含了vue解析后的各个结果，比如template、style、script
2. 如果 `resourceQuery` 中有 `type` 说明已经被处理过，可以进行分块处理，是loader推导策略的关键步骤点之一
3. 生成 `code`，这里的 `code` 包含 import vue 文件的代码，并且引入携带不同类型的type，会触发新的 `pitcher-loader` 的执行，是loader推导策略的关键步骤点之一
4. `pitcher-loader` 生成的 `loader request` 会触发 `vue-loader` 新一轮的执行，这时 `resourceQuery` 中会包含 `type=xxx`
5. `code` 中的 `import` 都处理完后，当前的 `.vue sfc` 就处理完了
