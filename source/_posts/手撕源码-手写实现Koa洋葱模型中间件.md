---
title: 手撕源码-手写实现Koa洋葱模型中间件
date: 2021-05-15 15:11:45
categories:
	- Web
	- 前端
	- NodeJs
	- Koa
	- 洋葱模型
	- 中间件
tags:
  - Web
	- 前端
	- NodeJs
	- Koa
	- 洋葱模型
	- 中间件
---

# Koa洋葱模型分析

在实现Koa的洋葱模型之前，先回顾一下洋葱模型的运作方式

![洋葱模型](https://camo.githubusercontent.com/d80cf3b511ef4898bcde9a464de491fa15a50d06/68747470733a2f2f7261772e6769746875622e636f6d2f66656e676d6b322f6b6f612d67756964652f6d61737465722f6f6e696f6e2e706e67)

## 中间件的执行机制

先来看个简单例子

```javascript
// 声明一个异步函数作为中间件
app.use(async (ctx, next) => {
  console.log(1);
  await next();
  console.log(2);
});
app.use(async (ctx, next) => {
  console.log(3);
  await next();
  console.log(4);
});

// 输出顺序是：
// 1
// 3
// 4
// 2
```

所有的请求经过一个中间件的时候都会执行两次，执行next前和执行next后的代码分为两部分执行。

## 中间件是怎么保存的

通过上面的案例，可以把中间件的注册关键方法锁定在 `app.use` 上

我们先看 `use` 方法

koa/lib/application.js
![koa-application-use](https://tva1.sinaimg.cn/large/008i3skNgy1gqj59zns7nj31b20fmn0m.jpg)

简化一下可以得到
```javascript
use(fn) {
  // 保存在middleware数组中
  this.middleware.push(fn);
  return this;
}
```
在 `app.use` 中，中间件会被推入到一个 `middleware` 数组中保存起来。

中间件被保存起来后，在什么时候，用什么方式调用的呢？

<!-- more -->

## 中间件执行器

server在启动的时候会传入一个一个 `callback`，这个callback实际就是被包装后的中间件执行器

我们先看看`app.listen`做了什么

```javascript
// lib/application.ts app.listen
listen(...args) {
    const server = http.createServer(this.callback());
    return server.listen(...args);
}
```

listen实际是创建了一个http server，然后通过server的监听端口功能来启动服务，
注意在创建server时传入了一个callback，这个callback会在请求到server时被调用，
这个callback似乎就是我们想要的线索。

```javascript
callback() {
    // middleware数组在这里经过 compose 方法包装得到一个新的函数
    const fn = compose(this.middleware);

    if (!this.listenerCount('error')) this.on('error', this.onerror);

    // 这里包装了一下 handleRequest，用与处理ctx
    const handleRequest = (req, res) => {
      const ctx = this.createContext(req, res);
      return this.handleRequest(ctx, fn);
    };

    // 返回包装后的 handleRequest
    return handleRequest;
}
```

当请求到koa server时，会执行被 `koa-compose` 包装后的中间件，也就是，洋葱模型的关键逻辑在 `koa-compose` 中。
`koa-compose` 的代码很简短

```javascript
// koa-compose
function compose (middleware) {
  // ...这里会有几个判断，校验中间件数组的合法性


  // 返回一个函数，这个函数使用来作为中间件的遍历执行器
  return function (context, next) {
    // index 是用来记录最后执行的那个中间件的下标
    let index = -1
    return dispatch(0)

    // 返回一个函数 dispatch，这个函数是用来代理中间件的执行
    // 内部封装了context和next的传参逻辑
    // 用 i 来标识执行的中间件
    function dispatch (i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      // 取出当前需要执行的中间件
      let fn = middleware[i]

      if (i === middleware.length) fn = next
      if (!fn) return Promise.resolve()

      try {
        // 中间件的两个参数：context，next
        // next实际就是下一个中间件执行器，这里把下一个中间件的 i 通过bind绑定到 dispatch 上
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}
```

`koa-compose` 会返回一个中间件启动函数，这个函数主要负责了两件事
1、记录最后执行的那个中间件的下标，也就是在中间件数组中的位置
2、启动第一个中间件执行器（dispatch(0)）

`dispatch` 中间件执行器对中间件进行了一层包装，主要负责三件事
1、通过下标取出当前需要执行中间件
2、执行中间件，参数中的 `next` 预先绑定为下一个中间件执行器
3、返回Promise

## 思考🤔

到这里其实洋葱模型中间件其实已经分析完了，关键点就在于 `dispatch` 函数返回了 `Promise`，在 `rsolve` 时执行中间件，并把下一个中间件作为 `next` 参数瞧瞧传给了下一个中间件，一层套下一层，直接看上面的源码获取会有些抽象，我们把它用具体代码表示一下

```javascript
const middleware1 = async (ctx, next) => {
  console.log('middleware1 before next');
  await next();
  console.log('middleware1 after next');
};
const middleware2 = async (ctx, next) => {
  console.log('middleware2 before next');
  await next();
  console.log('middleware2 after next');
};
const middleware3 = async (ctx, next) => {
  console.log('middleware3 before next');
  await next();
  console.log('middleware3 after next');
};

const ctx = {};

return Promise.resolve(middleware1(ctx, async () => {
  return Promise.resolve(middleware2(ctx, async () => {
    return Promise.resolve(middleware3(ctx, Promise.resolve()));
  }));
}));

```

## 动手实现一个超简洁版洋葱模型中间件🔍

分析完koa的洋葱模型中间件后，我们动手实现一个自己的中间件模型执行器

```javascript
const middleware1 = async (ctx, next) => {
  console.log('middleware1 before next');
  await next();
  console.log('middleware1 after next');
};
const middleware2 = async (ctx, next) => {
  console.log('middleware2 before next');
  await next();
  console.log('middleware2 after next');
};
const middlewares = [
  middleware1,
  middleware2,
];

// 模拟获取context
function getCurrContext() {
  const ctx = {};
  // ... 一堆操作
  return ctx;
}

// 执行器入口
function composeMiddles() {
  let i = 0;
  let middle = middlewares[i];
  const ctx = getCurrContext();
  return Promise.resolve(
    middle(ctx, middlewares[i + 1] || (async => {}))
  );
}

async function invoke() {
    composeMiddles();
}
invoke();

```
