---
title: JavaScript-错误处理与调试
date: 2021-05-19 14:07:08
categories:
	- Web
	- 前端
	- JavaScript
	- 错误处理
	- 异常处理
tags:
  - Web
	- 前端
  - JavaScript
	- 错误处理
	- 异常处理
---

## 静态资源加载的错误处理

### 图片资源加载的错误处理

当一项资源（如img或script）加载失败，加载资源的元素会触发一个Event接口的error事件，并执行该元素上的onerror()处理函数。这些error事件不会向上冒泡到window，不过能被单一的window.addEventListener捕获。

图片也支持 error 事件，可以使用 addEventListener 或者 onerror 来添加。

```javascript
var image = new Image();
image.addEventListener("load", (event) => { console.log("Image loaded!");
});
// 使用 onerror 函数监听错误事件
image.onerror = (event) => {console.log("Image not loaded!")};
image.src = "123"; // 不存在，资源会加载失败
```
![图片资源加载错误1](https://tva1.sinaimg.cn/large/008i3skNgy1gqnqh9teh6j31160jmgpr.jpg)

```javascript
var image = new Image();
image.addEventListener("load", (event) => { console.log("Image loaded!");
});
// 使用 addEventListener error 函数监听错误事件
image.addEventListener("error", (event) => {
console.log("Image not loaded!"); });
image.src = "doesnotexist.gif"; // 不存在，资源会加载失败
```
![图片资源加载错误2](https://tva1.sinaimg.cn/large/008i3skNgy1gqnqhwbuvnj310o0kwwjl.jpg)

window.onerror 和 window.addListener('error', () => {}) 都可以捕获到js的错误
区别是：
1.捕获到的错误参数不同
2.window.addEventListener('error')可以捕获资源加载错误，但是window.onerror不能捕获到资源加载错误，window.addEventListener('error')捕获到的错误，可以通过target?.src || target?.href区分是资源加载错误还是js运行时错误

### script资源加载的错误处理

script资源加载与图片等其他资源加载没有太大区别

但是有需要注意的地方：
当加载自不同域的脚本中发生语法错误时，为避免信息泄露（参见bug 363897），语法错误的细节将不会报告，而代之简单的"Script error."。在某些浏览器中，通过在script使用crossorigin属性并要求服务器发送适当的 CORS HTTP 响应头，该行为可被覆盖。一个变通方案是单独处理"Script error."，告知错误详情仅能通过浏览器控制台查看，无法通过JavaScript访问。

<!-- more -->

通过script加载跨域脚本时，脚本如果报错的情况，报错信息会缺少 `stack` 信息，error显示为null，并且错误事件中的 message、lineno、filename 显示是不正确的
![跨域脚本错误1](https://tva1.sinaimg.cn/large/008i3skNgy1gqnqm053gpj30u00wkk0w.jpg)

这里说的“看不到stack和正确的错误信息”是指使用 onerror 或者 addEventListener 捕获的错误对象
如果想知道详细信息还是可以通过打开控制台，查看error窗口来获取真正的信息
![跨域脚本错误2](https://tva1.sinaimg.cn/large/008i3skNgy1gqnpm3xokej30k103s74m.jpg)

如果想要获取正确的信息，则加载跨域脚本时，需要添加 `crossorigin` 属性，并且资源的请求头需要带上 `Access-Control-Allow-Origin: http://www.yourdomain.com`
错误事件中的 message、lineno、filename 会显示正确
![跨域脚本错误3](https://tva1.sinaimg.cn/large/008i3skNgy1gqnqngr8vkj30u00zx14h.jpg)

通用的处理方式，参考MDN
```javascript
window.onerror = function (msg, url, lineNo, columnNo, error) {
    var string = msg.toLowerCase();
    var substring = "script error";
    if (string.indexOf(substring) > -1){
        alert('Script Error: See Browser Console for Detail');
    } else {
        var message = [
            'Message: ' + msg,
            'URL: ' + url,
            'Line: ' + lineNo,
            'Column: ' + columnNo,
            'Error object: ' + JSON.stringify(error)
        ].join(' - ');

        alert(message);
    }

    return false;
};
```

## Promise异常全局处理

```javascript
Promise.resolve().then(() => console.log(c));
```
![Promise异常1](https://tva1.sinaimg.cn/large/008i3skNgy1gqnpmraue8j30k207sgmr.jpg)

当promise被reject并且错误信息没有被处理的时候，会抛出一个unhandledrejection，并且这个错误不会被window.onerror以及window.addEventListener('error')捕获。

```javascript
window.addEventListener("unhandledrejection", event => {
  event.preventDefault();
  console.log(">>>", event);
  console.log(">>>", event.reason);
});
Promise.resolve().then(() => console.log(c));
```
![Promise异常2](https://tva1.sinaimg.cn/large/008i3skNgy1gqnpn8sv2rj30js04zq3s.jpg)

如果需要在全局捕获unhandledrejection，需要用专门的window.addEventListener('unhandledrejection')捕获处理。
使用 `event.preventDefault()` 可以阻止在开控制台输出默认的错误信息。

使用 `window.onunhandledrejection` 也可以实现同样的效果
```javascript
window.onunhandledrejection = event => {
  event.preventDefault();
  console.log(">>>", event);
  console.log(">>>", event.reason);
});
Promise.resolve().then(() => console.log(c));
```
![Promise异常3](https://tva1.sinaimg.cn/large/008i3skNgy1gqnpo7md9nj30jf0fwtbt.jpg)

## Ajax请求异常

### XMLHttpRequest

举一个简单的例子：
```javascript
var xhr = new XMLHttpRequest();
xhr.addEventListener('error', error => console.log(error));
xhr.addEventListener('load', () => console.log('load'));
xhr.open("GET", "http://127.0.0.1:8080/index.error")
xhr.send();
```

需要注意的时：XMLHttpRequest的error事件与http status无关，只有在请求过程中，发生 Network error（网络错误）才会触发此事件，比如：
1、请求过程中，网络中断
2、断网环境
![Ajax异常1](https://tva1.sinaimg.cn/large/008i3skNgy1gqnpq51akej30k007t75v.jpg)

对于应用层级别的异常，如响应返回的xhr.statusCode是 4xx 或 5xx 时，并不属于 Network error ，所以不会触发 onerror 事件，而是会触发 onload 事件。
![Ajax异常2](https://tva1.sinaimg.cn/large/008i3skNgy1gqnpqd5hrij30k606y75g.jpg)

### Fetch

举个简单的例子：
```javascript
fetch('http://127.0.0.1:8080/index.error')
    .then(async res => console.log(await res.text()))
    .catch(error => console.log(error));
```

fetch 与 XMLHttpRequest 捕获异常上不同的点在于：
fetch 可以 catch 应用层的异常，比如：4xx,5xx
![fetch异常1](https://tva1.sinaimg.cn/large/008i3skNgy1gqnpr567baj30e601wjre.jpg)
![fetch异常2](https://tva1.sinaimg.cn/large/008i3skNgy1gqnpragpzrj30jp048wf2.jpg)

## 常见错误与异常

### EvalError

创建一个error实例，表示错误的原因：与 eval() 有关。
EvalError 不在当前ECMAScript规范中使用，因此不会被运行时抛出. 但是对象本身仍然与规范的早期版本向后兼容.

常见于：
1、给 eval 赋值
```javascript
eval = '';
```
但是现代引擎有可能会允许这种情况，比如：chrome下会出报错

2、eval在非函数调用的情况下被使用
```javascript
new eval();
```

但是现代引擎有可能不会报 EvalError，如果满足其他错误的场景会优先展示别的错误类型
![evalerror1](https://tva1.sinaimg.cn/large/008i3skNgy1gqnprpsvitj30lk025glq.jpg)


```javascript
try {
  throw new EvalError('Hello', 'someFile.js', 10);
} catch (e) {
  console.log(e instanceof EvalError); // true
  console.log(e.message);              // "Hello"
  console.log(e.name);                 // "EvalError"
  console.log(e.fileName);             // "someFile.js"
  console.log(e.lineNumber);           // 10
  console.log(e.columnNumber);         // 0
  console.log(e.stack);                // "@Scratchpad/2:2:9\n"
}
```

### InternalError（非Web标准的错误类型，目前只有Firefox实现了）

创建一个代表Javascript引擎内部错误的异常抛出的实例。 如: "递归太多"。

### RangeError

表示错误的原因：数值变量或参数超出其有效范围。

常见于：
1、创建一个负长度数组
![internalError1](https://tva1.sinaimg.cn/large/008i3skNgy1gqnps7drvrj30jm05d3yx.jpg)

2、Number 对象的方法参数超出范围
![internalError2](https://tva1.sinaimg.cn/large/008i3skNgy1gqnpsctk7rj30jb071wfc.jpg)


### ReferenceError

表示错误的原因：无效引用。
常见的错误提示：xxxx is not defined

常见于：
1、引用了一个不存在变量
```javascript
try {
  var a = undefinedVariable;
} catch (e) {
  console.log(e instanceof ReferenceError); // true
  console.log(e.message);                   // "undefinedVariable is not defined"
  console.log(e.name);                      // "ReferenceError"
  console.log(e.fileName);                  // "Scratchpad/1"
  console.log(e.lineNumber);                // 2
  console.log(e.columnNumber);              // 6
  console.log(e.stack);                     // "@Scratchpad/2:2:7\n"
}
```

2、主动抛出一个引用错误
```javascript
try {
  throw new ReferenceError('Hello', 'someFile.js', 10);
} catch (e) {
  console.log(e instanceof ReferenceError); // true
  console.log(e.message);                   // "Hello"
  console.log(e.name);                      // "ReferenceError"
  console.log(e.fileName);                  // "someFile.js"
  console.log(e.lineNumber);                // 10
  console.log(e.columnNumber);              // 0
  console.log(e.stack);                     // "@Scratchpad/2:2:9\n"
}
```

### SyntaxError

JS 引擎解析 js 代码块时，先进行词法语法分析，若发现不符合语法规范的 token 或 token 顺序时抛出 SyntaxError(语法错误)，会导致整个 js 文件无法执行。
比如：Chrome中会报"Uncaught SyntaxError: Unexpected token"

常见于几种场景：
1、JS资源文件下载失败
2、JS中使用了当前引擎不支持的语法特性

```javascript
try {
  eval('hoo bar');
} catch (e) {
  console.log(e instanceof SyntaxError); // true
  console.log(e.message);                // "missing ; before statement"
  console.log(e.name);                   // "SyntaxError"
  console.log(e.fileName);               // "Scratchpad/1"
  console.log(e.lineNumber);             // 1
  console.log(e.columnNumber);           // 4
  console.log(e.stack);                  // "@Scratchpad/1:2:3\n"
}
```

### TypeError

表示错误的原因：变量或参数不属于有效类型；对象用来表示值的类型发生了非预期类型时的操作。

常见于：
1、基础类型的值用于方法调用
```javascript
var foo = 1;
foo();
```
![typeerror1](https://tva1.sinaimg.cn/large/008i3skNgy1gqnpswqnlfj30jo05kgm0.jpg)

```javascript
try {
  null.f();
} catch (e) {
  console.log(e instanceof TypeError); // true
  console.log(e.message);              // "null has no properties"
  console.log(e.name);                 // "TypeError"
  console.log(e.fileName);             // "Scratchpad/1"
  console.log(e.lineNumber);           // 2
  console.log(e.columnNumber);         // 2
  console.log(e.stack);                // "@Scratchpad/2:2:3\n"
}
```

### URIError

用来表示以一种错误的方式使用全局URI处理函数而产生的错误。

常见于：
1、使用decodeURIComponent时，传入了错误格式的url，其他的URL操作方法也会出现这种错误，比如：decodeURI，encodeURL等
```javascript
decodeURIComponent('%');
```
![urierror1](https://tva1.sinaimg.cn/large/008i3skNgy1gqnptaghrkj30jz041gm6.jpg)

```javascript
try {
  decodeURIComponent('%');
} catch (e) {
  console.log(e instanceof URIError); // true
  console.log(e.message);             // "malformed URI sequence"
  console.log(e.name);                // "URIError"
  console.log(e.fileName);            // "Scratchpad/1"
  console.log(e.lineNumber);          // 2
  console.log(e.columnNumber);        // 2
  console.log(e.stack);               // "@Scratchpad/2:2:3\n"
}
```

## 自定义错误

```javascript
class MyError extends Error {
    constructor(message) {
        super();
        this.name = 'MyError';
        this.message = message || 'default error message';
        this.stack = (new Error()).stack;
    }
}

try {
    throw new MyError();
} catch (e) {
    console.log(e.name);     // 'MyError'
    console.log(e.message);  // 'Default Message'
}

try {
    throw new MyError('custom message');
} catch (e) {
    console.log(e.name);     // 'MyError'
    console.log(e.message);  // 'custom message'
}
```

## 错误处理

### 局部处理

常用的处理方式使用 `try catch` 语句包含执行的代码块

```javascript
try {
    throw new Error('');
} catch (error) {
    console.error(error);
}
```

### 全局处理
任何没有被 try/catch 语句处理的错误都会在 window 对象上触发 error 事件。当JavaScript运行时错误（包括语法错误）发生时，window会触发一个ErrorEvent接口的error事件，并执行window.onerror()。
触发 error 事件会传入 3 个参数:错误消息、发生错误的 URL 和行号

```javascript
window.onerror = (message, url, lineNumber) => {
    console.log(message, url, lineNumber);
    // 返回true可以阻止浏览器的默认报告错误行为，控制台不会输出error
    return true;
};

window.addEventListener('error', (errorEvent) => {
    console.log(errorEvent);
});

function a() {
    console.log(q);
}
a();
```
![错误处理1](https://tva1.sinaimg.cn/large/008i3skNgy1gqnptv73tej30kc0a0jth.jpg)

浏览器在使用这个事件处理错误时存在明显差异。在IE中发生error事件时，正 常代码会继续执行，所有变量和数据会保持，且可以在 onerror 事件处理程序中访问。 然而在 Firefox 中，正常代码会执行会终止，错误发生之前的所有变量和数据会被销毁， 导致很难真正分析处理错误。

## 结论

* 使用window.onerror 或 window.addEventListener('error') 捕获JS运行时错误
* 使用window.onunhandledrejection 或 window.addEventListener('unhandledrejection')捕获未处理的promise reject错误
* 在跨域脚本上配置 `crossorigin="anonymous"`，并且配置资源的 `Access-Control-Allow-Origin: http://www.yourdomain.com` 来正确捕获跨域脚本错误
* 使用window.addEventListener('error')捕获资源加载错误。因为它也能捕获js运行时错误，为避免重复上报js运行时错误，此时只有event.srcElement inatanceof HTMLScriptElement或HTMLLinkElement或HTMLImageElement时才上报
* XMLHttpRequest 使用 addEventListener('error') 来处理异常
* fetch 使用通常的 Promise catch 的方式即可，try catch 也可以

## 参考
> [常见的JS错误分类-文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Error)
> [常见的JS错误列表](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Errors)
> [what-is-script-error](https://blog.sentry.io/2016/05/17/what-is-script-error)