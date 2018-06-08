---
title: Vue2加载优化
date: 2018-06-05 17:39:37
categories:
	- 前端
	- JavaScript
tags:
	- Vue
	- 首屏
	- 性能
---

> 以vue-cli2作为开发脚手架为前提，虽然内部做了不少的打包优化来减少文件体积，但是仍然有提升的空间。提升的方向大致就为两种：1. 代码结构；2. webpack的打包配置

## 调整代码结构

### 配合vue-router使用异步加载

如何配合vue-router使用异步加载在官方文档中已有说明，简单的来说就是在需要的时候再加载特定的js，这部分js在打包的时候会被单独打包到一个js中，以此达到减少bundle的体积的目的。

```javascript
const TestView = () => import('@/views/test-view');

const routerOptions = {
  routes: [
    {
      path: '/',
      component: TestView,
    },
    // ...
  ],
}
```

<!-- more -->

### 框架的按需加载

通常我们诸如element-ui，mint-ui，iview等框架时，有可能会直接在main.js中直接导入整个框架，这会造成打包后的体积多出不少，并且使用中有绝大部分的组件用不到。这个时候就需要按需加载。

以element-ui为例：[官方文档案例](https://element.faas.ele.me/#/zh-CN/component/quickstart)
```javascript
import Vue from 'vue';
import { Button, Select } from 'element-ui';
import App from './App.vue';

Vue.component(Button.name, Button);
Vue.component(Select.name, Select);
/* 或写为
 * Vue.use(Button)
 * Vue.use(Select)
 */

new Vue({
  el: '#app',
  render: h => h(App)
});
```

### 取消外链css

vue-cli在打包时默认会抽取css到css文件，然后以外链的方式插入到index.html。这里会产生额外的http请求，所以这里可以考虑 `css in js`，不抽取css到单独的文件。

找到 `build/webpack.prod.conf.js` 文件，将 `module` 对象修改为以下的代码

```javascript
const webpackConfig = merge(baseWebpackConfig, {
  module: {
    rules: utils.styleLoaders({
      sourceMap: config.build.productionSourceMap,
      extract: false,
      usePostCSS: true
    })
  },
  // ...
});
```

**重点是 `extract` 属性改为 `false` 即可**

### 页面加载期间的loading动画

> 即使做了以上种种优化，在网络状况不理想的情况下，用户在网页加载的期间仍然能够看到一小段时间的白屏。

针对这种情况，我们可以手动index.html中加点动画元素，一张图片，或者svg等等。

但是这种做法每次打包都需要手动复制粘贴，显然很不优雅。

我们可以利用 `html-webpack-plugin` 将自定义的 loading 的对象带入到index.html

```javascript
const fs = require('fs');

const loading = {
  html: fs.readFileSync(path.join(__dirname, '../loading.html')),
  css: '<style>' + fs.readFileSync(path.join(__dirname, '../loading.css')) + '</style>'
}

const webpackConfig = merge(baseWebpackConfig, {
  // ...
  new HtmlWebpackPlugin({
    // ...
    loading: loading,
    // ...
  }),
});
```

```html
<!DOCTYPE html>
<html>
  <head>
		<!-- ... -->
    <%= htmlWebpackPlugin.options.loading.css %>
  </head>
  <body>
    <div id="app">
      <%= htmlWebpackPlugin.options.loading.html %>
    </div>
    <!-- built files will be auto injected -->
  </body>
</html>
```

### `prerender-spa-plugin` 预渲染首屏

在某些项目中，loading本身就可能被设计成一个组件，这个时候就需要先渲染出组件的真实DOM。首先用 `headless browser` （比如：[puppeteer](https://github.com/GoogleChrome/puppeteer)）执行js，渲染完成后，读取html插入到我们的`index.html`。

以上的步骤都已经集成在了 `prerender-spa-plugin`

```javascript
const path = require('path')
const PrerenderSPAPlugin = require('prerender-spa-plugin')

module.exports = {
  plugins: [
    // ...
    new PrerenderSPAPlugin({
      // Required - The path to the webpack-outputted app to prerender.
      staticDir: path.join(__dirname, 'dist'),
      // Required - Routes to render.
      routes: [ '/', '/about', '/some/deep/nested/route' ],
    })
  ],
  // ...
}
```

更多的高级用法可以去访问仓库的文档查看。

> 预渲染使用方法参考：[Vue单页面骨架屏实践](https://zhuanlan.zhihu.com/p/31954853)

### 本地静态资源（图片等）的预加载

> 如果有不少图片使用的频率较高，使用的范围比较广，可以考虑在 app mounted 后延时一小段时间，将那些图片做一个预加载。

// app.vue (template)
```html
<div>
	<!-- ... -->
	<div id="preload-imgs" v-if="preloadImg">
		<img src="/path/to/img/..." alt="">
	</div>
</div>
```

// app.vue
```javascript
export default {
	// ...
	data () {
    return {
      preloadImg: false,
    };
	},
  mounted () {
    setTimeout(() => {
      this.preloadImg = true;
    }, 500);
  }
}
```

### 使用动态 `polyfill`

> `polyfill` 为不支持ES2015+的老旧浏览器提供了一个完整的 ES2015+ 环境，这样就可以使用新的内置对象比如 Promise 或者 WeakMap, 静态方法比如 Array.from 或者 Object.assign, 实例方法比如 Array.prototype.includes 和生成器函数（提供给你使用 regenerator 插件）。为了达到这一点， polyfill 添加到了全局范围，就像原生类型比如 String 一样。

但是 `polyfill` 的体积通常都很大，而且几乎所有的现代浏览器都已经能运行绝大部分的ES2015+环境，`polyfill` 只是为那一小部分用户提供的'垫片'。

为了只在需要 `polyfill` 的时候加载特定的一部分，我们可以使用[polyfill.io](https://polyfill.io/v2/docs/)

- 引入polyfill

```html
<script src="https://cdn.polyfill.io/v2/polyfill.min.js"></script>
```

- 引入特定部分的polyfill

```html
<script src="https://cdn.polyfill.io/v2/polyfill.min.js?features=Map"></script>
```

- polyfill.io会根据浏览器的UserAgent判断浏览器需要哪些polyfill，然后只再返回需要的那部分。

你可以使用 Chrome 和 Edge，IE 访问 https://cdn.polyfill.io/v2/polyfill.min.js 试试。

### 在生产环境中部署ES2015+

> 原文链接：[【译】如何在生产环境中部署ES2015+](https://jdc.jd.com/archives/4911)

> 我们在打包时通常会了兼容老旧浏览器，使用babel将es2015+代码转译成es5（语法部分），代价就是js体积会变大不少。
> 如果能让浏览器自动加载他们能够识别的那部分 `JavaScript` 岂不美哉？是的，有这种方案了。
> 虽然说目前并没有一个非常好的解决方案，但是我们可以通过 `<script type="module">` 过滤出能够运行整个es2015+环境的浏览器。

能够识别出 type=module 的浏览器同样支持 async 和 await 函数， Class 类，arrow functions，fetch 、Promises、Map、Set 等更多 ES2015+ 语法。

我们可以单独打包出一份 `es2015+` 的代码，然后另外打包一份降级到 es5 的代码。

**以下说明摘自原文[译]**

- 实现方式

例如，假设你使用了 webpack 并且 JS 的入口文件是 ./path/to/main.js ，你当前的 ES5 版本的配置应该如下所示（注意，由于使用 ES5 语法书写，我给该代码包命名为 main-legacy ）

```javascript
module.exports = {
  entry: {
    'main-legacy': './path/to/main.js',
  },
  output: {
    filename: '[name].js',
    path: path.resolve(__dirname, 'public'),
  },
  module: {
    rules: [{
      test: /\.js$/,
      use: {
        loader: 'babel-loader',
        options: {
          presets: [
            ['env', {
              modules: false,
              useBuiltIns: true,
              targets: {
                browsers: [
                  '> 1%',
                  'last 2 versions',
                  'Firefox ESR',
                ],
              },
            }],
          ],
        },
      },
    }],
  },
};
```

为了支持 ES2015+ 版本，你需要做的是生成第二个配置文件，该配置文件的使用环境是支持 `<script type="module">` 的浏览器， 如下面所示：

```javascript
module.exports = {
  entry: {
    'main': './path/to/main.js',
  },
  output: {
    filename: '[name].js',
    path: path.resolve(__dirname, 'public'),
  },
  module: {
    rules: [{
      test: /\.js$/,
      use: {
        loader: 'babel-loader',
        options: {
          presets: [
            ['env', {
              modules: false,
              useBuiltIns: true,
              targets: {
                browsers: [
                  'Chrome >= 60',
                  'Safari >= 10.1',
                  'iOS >= 10.3',
                  'Firefox >= 54',
                  'Edge >= 15',
                ],
              },
            }],
          ],
        },
      },
    }],
  },
};
```

接下来的步骤就是修改 HTML 代码，有条件的加载浏览器中支持 ES2015+ 的模块。你可以使用下面两个标签 `<script type="module">`  和 `<script nomodule>`  :

```html
<!-- Browsers with ES module support load this file. -->
<script type="module" src="main.js"></script>

<!-- Older browsers load this file (and module-supporting -->
<!-- browsers know *not* to load this file). -->
<script nomodule src="main-legacy.js"></script>
```

注意：这里唯一的问题是 Safari 10 并不支持 `nomodule` 属性，但是为了解决这一问题，你可以在使用 `<script nomodule>` 标签前，在 HTML 中使用内联JavaScript代码片段（注意：这个插件已经安装在 Safari11 版本中了）。

### 添加骨架屏

> 参考链接：[为vue项目添加骨架屏](https://xiaoiver.github.io/coding/2017/07/30/%E4%B8%BAvue%E9%A1%B9%E7%9B%AE%E6%B7%BB%E5%8A%A0%E9%AA%A8%E6%9E%B6%E5%B1%8F.html)

> 相关库：[vue-content-placeholders](https://github.com/michalsnik/vue-content-placeholders)

**具体用法和相关概念可以查看上面列出的链接**


## 优化webpack打包配置

### 禁用 sourceMap

找到`build/index.js`
修改 `build` 对象的 `productionSourceMap`
```javascript
module.exports = {
	// ...
  build: {
    // ...
    productionSourceMap: false,
  }
};
```

**禁用 `sourceMap` 可以有效减小体积，但是线上出问题时没有办法定位问题所在的代码。是否禁用看需求。**

### 升级到webpack4， 使用 SplitChunksPlugin

webpack4默认使用 `SplitChunksPlugin` 分割代码，比起 `CommonChunksPlugin`，`SplitChunksPlugin` 采用不同的方法分割代码，默认规则是：
- 模块被重复引用或者来自node_modules中的模块
- 在压缩前最小为30kb
- 在按需加载时，请求数量小于等于5
- 在初始化加载时，请求数量小于等于3

> 小于30kb的模块不值得再单独发送一次请求，在很小的模块的前提下，相比与多次打包，减少请求次数成本要低。

自定义配置参考：[webpack4——SplitChunksPlugin使用指南](https://blog.csdn.net/qq_26733915/article/details/79458533)
```javascript
new webpack.optimize.SplitChunksPlugin({
    chunks: "all",
    minSize: 20000,
    minChunks: 1,
    maxAsyncRequests: 5,
    maxInitialRequests: 3,
    name: true
)};
```

此外，需要注意的是，如果用到了 `html-webpack-plugin`, 则需要安装 `html-webpack-plugin@next`。

### 其他优化

- 网上有不少资料都会提到DllPlugin 和 DllReferencePlugin，使用cdn等手段进行优化打包，但是这些方法其实都只是加快build的速度，对Vue的首屏加载性能没有起到作用。

## 结论

- 配合vue-router使用异步加载
- 框架的按需加载
- 取消外链css
- 页面加载期间的loading动画
- `prerender-spa-plugin` 预渲染首屏
- 本地静态资源（图片等）的预加载
- 使用动态 `polyfill`
- 在生产环境中部署ES2015+
- 添加骨架屏
- 打包时禁用 `sourceMap`
- 升级到webpack4， 使用 SplitChunksPlugin





























