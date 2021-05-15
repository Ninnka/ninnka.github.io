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

在实现Koa的洋葱模型之前，先从源码的角度分析一遍洋葱模型的工作方式

## 注册中间件

通过官方文档，我们可以简单的了解到中间件的注册方式

```javascript
// 声明一个异步函数作为中间件
app.use(async (ctx, next) => {
  const start = Date.now();
  await next();
  const ms = Date.now() - start;
  console.log(`${ctx.method} ${ctx.url} - ${ms}ms`);
});
```

通过案例可以把中间件的注册关键方法锁定在 `app.use` 上

koa/lib/application.js
![koa-application-use](https://tva1.sinaimg.cn/large/008i3skNgy1gqj59zns7nj31b20fmn0m.jpg)

```javascript
use(fn) {
  this.middleware.push(fn);
  return this.
}
```

在 `app.use` 中，中间件会被推入到一个 `middleware` 数组中保存起来

## 中间件执行器

当请求到koa server时，koa会通过 `koa-compose` 以洋葱模型的方式执行这些中间

![koa-application-listener](https://tva1.sinaimg.cn/large/008i3skNgy1gqj5hlo4a3j30vs07ct9n.jpg)

![koa-application-callback](https://tva1.sinaimg.cn/large/008i3skNgy1gqj5i7wu3wj31140f60uy.jpg)

也就是，洋葱模型的关键逻辑在 `koa-compose` 中，`koa-compose` 的代码很简短

```javascript
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
        // next实际就是下一个中间件，这里把下一个中间的 i 通过bind绑定到 dispatch上
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}
```


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
