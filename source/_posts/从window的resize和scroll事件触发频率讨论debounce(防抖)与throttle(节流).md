---
title: 从window的resize和scroll事件触发频率讨论debounce(防抖)与throttle(节流)
date: 2016-2-11 21:50:33
categories:
	- 前端
	- JavaScript
tags:
	- debounce
	- throttle
	- 性能优化
---

## 使用debounce(防抖)来优化window的resize和scroll事件

用户在调整浏览器窗口大小时，会触发resize事件，如果在resize事件中有导致回流和重绘的操作的话会大量消耗性能，在低端机和低版本的IE都会悲剧，大概率进入假死状态。

用以下的方法可以调整resize事件中的代码的调用频率。scroll事件同理

```javascript
var timer = null;
window.addEventListener("resize", function() {
  if(timer){
    clearTimeout(timer);
  }
  timer = setTimeout(function (){
    console.log("resize");
  }, 500);
});
```

当然，以上的只是一个非常简单的例子和使用场景。这种处理方式有一个比较正式的名称，我们称之为防抖。
**（以上的案例的处理方法不是最终版）**

<!--more-->

## debounce(防抖)

防抖是指在一段时间内连续触发执行同一个函数时，只有最后一次触发能执行函数。

假如现在有这么一个场景，我们使用websocket与后台进行通信，后台大概一秒会有100条左右的数据传来，这一秒内的数据只有最后一秒是最终的数据，之前的数十条我们可以选择性的显示出来（可以理解为k线的每一秒的多个数据的最后一个）

```javascript
// 以下代码默认经过babel转译和webpack打包处理
import { debounce } from 'lodash';
import io from 'socket.io-client';
const address = '服务器地址';
const socketIns = io(address);
// ...
const dataResolve = () => {
    // ...
    /* 假如在这里处理数据，并使用canvas绘制出来，每一秒绘制上百条对于性能不太好的机器		会导致页面卡顿，严重的时候socket会与服务器断开连接 */
};
socketIns.on('data', debounce(dataResolve, 20));
// 这里使用lodash提供的防抖函数进行处理，防抖时间设置为20ms，页面要保持绘制的流畅度，每秒要有60帧，每一帧最多需要16ms，设置为20ms的间隔是为了最大限度防止浏览器在处理数据上花太多时间个阻塞了ui的绘制处理。如果在绘制ui的压力不会太大的前提下，时间也可以设置为16ms。
```

如果上个案例太复杂的话，可以看下面这个，这是取自vue文档中的一个案例，我们只讨论js部分

```javascript
new Vue({
  el: '#editor',
  data: {
    input: '# hello'
  },
  computed: {
    compiledMarkdown: function () {
      return marked(this.input, { sanitize: true })
    }
  },
  methods: {
    update: _.debounce(function (e) {
      // 这个函数使用了lodash的debounce处理，确保300毫秒内的最后一次输入再执行函数处理
      this.input = e.target.value
    }, 300)
  }
})
```
> 案例地址：https://vuefe.cn/v2/examples/index.html


## throttle(节流)

throttle指在一段时间内只允许一个函数执行一次。

这种方法通常用于点击事件的处理等需要限制触发次数的场景。假如有这样一个场景：我们点击某个按钮之后获取一个列表的数据，如果用户手抖不小心快速点了两下，如果没有节流处理的话，我们的app变回在极短时间内向后端请求两次数据。

这个时候就需要节流了，节流的使用方法很简单。

```javascript
import { throttle } from 'lodash';
const div = document.querySelector('#btn');
const resolve = () => {
    // resolve
};
div.addEventListener('click', throttle(resolve, 300));
// 只允许300毫秒内触发一次点击事件，这样便能够防止意外的多次触发函数
```

## 结论

通俗的讲，debounce是取一段时间内的最后一次触发来执行函数，而throttle是取一段时间内的第一次触发来执行函数。

### 使用场景上
- debouce
    - 持续的大量数据处理，或者说事件本身会被频繁触发，为了保证浏览器的进程不会把大量时间消耗在处理js上，可以使用debounce保证一段时间内函数只在最后一次触发后执行。
- throttle
    - 为了避免没有必要的重复执行，只需要第一次成功执行即可，例如点击按钮后请求数据等。

















