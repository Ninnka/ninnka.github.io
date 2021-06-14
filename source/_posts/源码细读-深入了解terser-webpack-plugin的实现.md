---
title: 源码细读-深入了解terser-webpack-plugin的实现
date: 2021-06-14 21:29:55
categories:
	- Webpack
	- plugin
	- terser-webpack-plugin
tags:
  - Web
	- 前端
	- Webpack
	- plugin
	- terser-webpack-plugin
---

## terser-webpack-plugin 是什么

`terser-webpack-plugin` 内部封装了 [terser](https://github.com/terser/terser) 库，用于处理 js 的压缩和混淆，通过 `webpack plugin` 的方式对代码进行处理

`terser-webpack-plugin` 的使用方式也很简单

官方文档提供了一份通用的配置：`terser` 配置的使用和具体含义可以参考 [minify-options](https://github.com/terser/terser#minify-options)

```js
module.exports = {
  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          ecma: undefined,
          parse: {},
          compress: {},
          mangle: true,
          module: false,
          output: null,
          format: null,
          toplevel: false,
          nameCache: null,
          ie8: false,
          keep_classnames: undefined,
          keep_fnames: false,
          safari10: false,
        },
      }),
    ],
  },
};
```

## terser-webpack-plugin 执行机制

`terser-webpack-plugin` 本质是个 webpack-plugin，通过注册运行时的某个钩子，可以在合适的时间点对代码做压缩和混淆的优化

那么 `terser-webpack-plugin` 是在哪个钩子中做这件事的呢，我们先看看插件的 `apply` 函数

`apply` 是 `webpack-plugin` 插件的初始化入口函数，`terser-webpack-plugin` 本身并不复杂且易读，简化后的代码如下：

```js
apply(compiler) {
    const { devtool, output, plugins } = compiler.options;
    const pluginName = this.constructor.name;
    // ... 中间做了一些别的处理
    // 比如 sourceMap 处理，弱缓存的初始化 weakCache，terserOptions 的初始化
    
    compiler.hooks.compilation.tap(pluginName, (compilation) => {
      // ...
      // 注册 optimizeChunkAssets 钩子，这个钩子对应了 webpack4 中 optimise 配置
      compilation.hooks.optimizeChunkAssets.tapPromise(pluginName, (assets) =>
          // 优化处理的真正入口
          this.optimize(compiler, compilation, assets, CacheEngine, weakCache)
        );
    });
}
```

通过代码很容易了解到，`terser-webpack-plugin` 先通过 compilation 钩子获取到 `compiler` 的 `compilation` 实例（这里不展开讨论 `webpack-plugin` 插件系统架构和 `tapable` 的设计，会在后面的篇幅中展开解读 `webpack-plugin` 插件系统架构 和 `tapable v2`）

注册 `webpack compilation` 提供的 `hooks: optimizeChunkAssets` 运行时钩子，注册时使用了 `tapPromise` 异步注册的方式，注册的回调函数返回一个 `Promise`

在 `webpack` 执行 `optimise` 阶段，每个 `chunk` 都会触发这个异步钩子然后执行回调函数，回调函数本质是通过 `optimise` 执行代码的优化任务处理

可以理解为 `terser-webpack-plugin` 中的实际任务处理入口是 `optimise` 函数

![apply流程](https://user-images.githubusercontent.com/9619419/121899669-3762c200-cd57-11eb-9140-173b26bdb0de.png)

<!-- more -->

### optimise

`optimise` 的源码很长，下面做代码解读会简化很多代码，比如 `cache` `comments` 相关的处理会被省略，把中心放在执行流程上

![optimise流程](https://user-images.githubusercontent.com/9619419/121899779-4f3a4600-cd57-11eb-9c1c-2163c71e3405.png)

```js
async optimize(compiler, compilation, assets, CacheEngine, weakCache) {
  let assetNames;
  assetNames = [/* 获取资源文件名 */]

  // 获取 cpu core 数，用于并行模式处理
  const availableNumberOfCores = TerserPlugin.getAvailableNumberOfCores(
    this.options.parallel
  );

  let concurrency = Infinity;
  let worker;

  if (availableNumberOfCores > 0) {
    // 如果开启 parallel 并行模式，会创建新的 `Worker` 线程池来做多进程处理
    worker = new Worker(require.resolve('./minify'), { numWorkers });
  }

  // promise 的并发数量限制
  const limit = pLimit(concurrency);
  // 处理任务队列
  const scheduledTasks = [];

  for (const name of assetNames) {
    scheduledTasks.push(
      limit(async () => {
        // 任务处理函数的开始
        // asset 资源的基本信息和代码
        const { info, source: inputSource } = TerserPlugin.getAsset(
          compilation,
          name
        );
        // 避免二次压缩
        if (info.minimized) {
          return;
        }

        let input;
        let inputSourceMap;

        input = inputSource.source();
        inputSourceMap = null;

        const minimizerOptions = {/* 压缩配置 */};

        try {
          // 启动 worker 做处理，没有worker的情况则在主线程做压缩处理
          output = await (worker
            ? worker.transform(serialize(minimizerOptions))
            : minifyFn(minimizerOptions));
        } catch (error) {
          compilation.errors.push(
            TerserPlugin.buildError(
              error,
              name,
              inputSourceMap && TerserPlugin.isSourceMap(inputSourceMap)
                ? new SourceMapConsumer(inputSourceMap)
                : null,
              new RequestShortener(compiler.context)
            )
          );

          return;
        }
        // 根据优化后的代码生成新的代码
        output.source = new RawSource(output.code);
        // 缓存起来做增量编译
        await cache.store({ ...output, ...cacheData });
        const newInfo = { ...info, minimized: true };
        const { source, /** ... */ } = output;
        // 更新资源的内容
        TerserPlugin.updateAsset(compilation, name, source, newInfo);
      })
    );
  }
  // 启动任务队列
  await Promise.all(scheduledTasks);
  // 等待任务结束
  if (worker) {
    await worker.end();
  }
}
```

总结一下，`optimise` 函数主要做了这几件事：

1. 组装资源文件名，便于后续从 `compilation` 中获取对应的资源文件
2. 获取 cpu core 数，用于 parallel 并行模式处理，并行模式的实现使用 worker_thread 实现
3. 使用 pLimit 对 promise 任务队列做并发处理限制
4. 遍历 assetNames 资源队列并创建对应的任务体 scheduleTask
5. 启动任务队列，await Promise.all(scheduledTasks) 等待 Promise 任务执行完毕
6. await worker.end() 等待 worker 线程关闭

`optimise` 做的事情并不复杂

我们继续分析 `任务体 scheduleTask` 做了哪些事：

1. 获取 `asset` 资源的基本信息和代码
2. 根据 `minimized` 标识判断是否已处理，避免二次压缩
3. 启动 `worker` 线程做 `minify` 处理，没有 worker 的情况则在主线程 调用 `minifyFn` 函数做压缩处理
4. 根据优化后的代码生成新的代码 `RawSource`
5. 缓存起来做增量编译
6. 更新 `compilation` 中对应资源的内容

### parallel

`terser-webpack-plugin` 提供了 `parallel` 用于做并行模式处理

内部执行时会先获取当前 cpu 核心数，如果开启 `parallel 模式`，会启动 cpu 核心数 - 1 个 `worker`

```js
static getAvailableNumberOfCores(parallel) {
    const cpus = os.cpus() || { length: 1 };
    return parallel === true
      ? cpus.length - 1
      : Math.min(Number(parallel) || 0, cpus.length - 1);
}
```

```js
apply() {
    if (availableNumberOfCores > 0) {
        // 如果开启 parallel 并行模式，会创建新的 `Worker` 线程池来做多进程处理 numWorkers 就是 cpu 核心数 - 1
        worker = new Worker(require.resolve('./minify'), { numWorkers });
      }
}
```

### Worker

`terser-webpack-plugin` 使用的 `Worker` 线程池管理是基于 `jest` 提供的 `jest-worker`

`jest` 大家都熟悉，是 `Facebook` 开源的自动化测试工具

`jest-worker` 本身是一个复杂的线程池管理模块，这里先对其做一个较为简单的解读

`jest-worker` 的线程池 `WorkerPool` 使用了两种线程模式

1. `worker_threads`
2. `child_process`

```js
class WorkerPool extends _BaseWorkerPool.default {
  // ...
  createWorker(workerOptions) {
    let Worker;

    if (this._options.enableWorkerThreads && canUseWorkerThreads()) {
      // 使用 worker_threads 模式，需要高版本的node支持
      Worker = require('./workers/NodeThreadsWorker').default;
    } else {
      // 不支持 worker_threads 时的降级处理
      Worker = require('./workers/ChildProcessWorker').default;
    }
    return new Worker(workerOptions);
  }
}
```

对于 高版本的 node，会优先使用 `worker_threads` 模式，如果不支持 `worker_threads` 则降级处理使用 `child_process`

```js
const canUseWorkerThreads = () => {
  try {
    require('worker_threads');
    return true;
  } catch {
    return false;
  }
};
```

`jest-worker` 本身在管理线程池的同时，也会管理任务队列，在 `new Worker()` 时，`jest-worker` 会把任务队列保存在 `Worker` 实例中的 `_taskQueue` 上，同时实例上暴露出一些 `public api` 给外部使用，言外之意，这个 `_taskQueue` 是私有只读属性，外部尽量避免直接依赖它

![parallel机制](https://user-images.githubusercontent.com/9619419/121899893-6a0cba80-cd57-11eb-9127-daced5b3cdd6.png)

#### `_taskQueue`

`_taskQueue` 使用 `先进先出队列` 进行维护

```js
export default class FifoQueue implements TaskQueue {
  private _workerQueues: Array<InternalQueue<WorkerQueueValue> | undefined> =
    [];
  private _sharedQueue = new InternalQueue<QueueChildMessage>();

  enqueue(task: QueueChildMessage, workerId?: number): void {
    // ....
    workerQueue.enqueue(item);
  }

  dequeue(workerId: number): QueueChildMessage | null {
    // ....
    if (workerTop != null && sharedTaskIsProcessed) {
      return this._workerQueues[workerId]?.dequeue()?.task ?? null;
    }
    return this._sharedQueue.dequeue();
  }
}
```

`Fifo` 队列维护了两份 `queue`，分别对应 `独立 workerId` 和 `共用 worker` 的场景，这里不展开说明

### minify

`minify` 函数是用于处理代码的入口，内部实现很简单，就是使用了 `terser` 这个库对代码做了压缩和混淆处理（`terser` 本身是一个功能强大且较为复杂的包，在后面的篇幅会单独作为一个系列去解读，这里就不展开了）

```js
async function minify(options) {
  const {
    name,
    input,
    inputSourceMap,
    minify: minifyFn,
    minimizerOptions,
  } = options;
  if (minifyFn) {
    return minifyFn({ [name]: input }, inputSourceMap, minimizerOptions);
  }
  const terserOptions = buildTerserOptions(minimizerOptions);
  
  // 省略了 comments 的处理
  
  // 调用 terser export 的 api 对代码做处理
  const result = await terserMinify({ [name]: input }, terserOptions);

  return { ...result, extractedComments };
}
```

`minify` 函数做的事情很简单：

1. 合并 `terserOptions` terser配置
2. 处理 `comments`
3. 调用 `terser export` 的 api 对代码做处理

## 总结

`terser-webpack-plugin` 优化代码的流程主要体现在这几个方面：

1. 异步注册 `compilation.hooks.optimizeChunkAssets`
2. 在回调中调用 `plugin` 实例的 `optimise` 方法
3. 并行模式：创建 `Worker` 进行多线程编译
4. `minify` 过程调用 `terser` 库对代码进行处理

![terser-webpack-plugin v4 流程](https://user-images.githubusercontent.com/9619419/121900401-f7e8a580-cd57-11eb-99ad-6717eab39c49.png)