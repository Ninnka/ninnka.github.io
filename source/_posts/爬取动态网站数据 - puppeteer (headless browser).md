---
title: 爬取动态网站数据 - puppeteer (headless browser)
date: 2018-06-01 18:22:04
categories:
	- JavaScript
tags:
	- Nodejs
	- 命令行工具
	- 爬虫
	- headless browser
	- puppeteer
---

> 我在之前有提到过pixiv用React重写了部分页面，导致以前的方法不能爬取那部分的页面了。
> 熟悉 `React/Vue/Angular` 开发的同学应该很清楚，这类框架开发的Web应用有个很明显的特征，web应用的原始页面的DOM只有简单body和一个id为root的节点（当然，其他人为加入的也会存在）。如果用以前的superagent配合request只能把这部分下载下来，如果pixiv使用了service-side rendering的还好，但是pixiv并没有使用（我觉得以后pixiv可能大概会用上）。

说回正题，因为pixiv的部分页面改为“动态加载”后，直接获取文档不能获取到DOM，那么有没有什么办法能够获取代码然后执行js生成DOM，再获取页面的内容呢？
仔细想想这种行为不就像是一个浏览器了吗？
其实这种情况我之前就遇到过，当时是因为有要求爬取京东和淘宝的部分页面，但是京东和淘宝的首页出了靠近顶部部分以外的都是懒加载。顺着上面所说的思路，我很快想到了以前看到过的 `headless browser`，然后找到了 `puppeteer`，中文名是木偶戏

> Puppeteer 是一个提供了高集成API，通过开发者工具协议来控制无页面Chrome或者Chromium的 node 库. 它同样能用来控制正常的 Chrome 和 Chromium。 
> 我们可以用 Puppeteer 做什么？
> - 生成一个快照或者PDF。
> - 爬取SPA（单页应用）并生成预渲染的内容 (比如： "SSR")。
> - 自动化提交，输入，UI测试等。
> - 创建一个最新的，全自动测试的环境。在最新Chrome上使用最新的javascript和浏览器特性运行测试代码。
> - 按照时间线来跟踪捕获网站的运行，用来帮助诊断性能问题。

Bingo，这正是我在这次爬虫中要用到的，接下来将使用它爬取pixiv的React App。

**[源代码](https://github.com/Rennzh/node-pixiv-crawler)，源代码已经是最新的，如果想看以前的方法的可以 `git checkout` 到以前的仓库节点。**

<!-- more -->

## 初始化puppeteer browser实例

在处理接收的参数后，应该立即创建一个puppeteer的browser实例，并保存起来，接下来的操作都使用这一个实例，如果不小心创建太多browser，会有超过最大emitterListener的警告。

```javascript
const program = require('commander');
const colors = require('colors');
const puppeteer = require('puppeteer');

const parseUrl = require('../reptile/parseUrl');

const pathController = require('../reptile/PathController');
// 用来保存路径相关的对象（单例）
const userController = require('../reptile/UserController');
// 用来保存一些爬取信息相关的变量的对象（单例）

// ... 处理参数

async function finishProcess () {
  try {
    // 关闭browser
    await userController.closeBrowser();
  } catch (err) {
    console.log('closeBrowser catch err', err);
  } finally {
    // 成功或失败都需要结束进程
    process.exit(0);
  }
}

async function initBrowser () {
  return new Promise(async (resolve) => {
    try {
      let browser = null;
      if (!userController.browser) {
        // 如果browser不存在的话就创建一个并保存
        browser = await puppeteer.launch();
        userController.setBrowser(browser);
      }
      resolve();
    } catch (err) {
      resolve();
      console.error('launch browser failed');
      // 启动browser失败也结束node进程
      process.exit(0);
    }
  });
}

async function FetchingData () {
  await initBrowser();
  const paramList = params.split(',');
  let targetPList = [];
  paramList.forEach((item) => {
    if (item) {
      const res = parseUrl.fetchMediumUrl(item.trim());
      targetPList.push(res);
    } else {
      console.log('url或id不能为空'.red);
    }
  });

  Promise.all(targetPList)
    .then(async (res) => {
      // 所有流程结束后结束node进程
      finishProcess();
    })
    .catch(err => {
      // 捕捉到错误后结束node进程
      console.log('paramList forEach Promise all catch err', err);
      finishProcess();
    });
}

FetchingData();
```

通过以上的方法，可以创建一个单例的puppeteer browser实例。

## 使用 puppeteer 打开新页面

`headless browser` 本质上就是一个浏览器，只是没有可视化的界面，但它的操作逻辑是一样，获取一个页面的数据必须打开指定的页面。
除此之外，在打开页面前，我们应该为页面填充好Cookie，因为pixiv需要Cookie来验证登录态，当然不用也可以，那么你需要打开登录页面，并填写好数据然后登录。登录成功后，页面就有cookie了，这个时候你可以把Cookie提取出来并保存好方便后续使用。（pixiv验证你是否登录只需要知道`PHPSESSID`）

```javascript
/**
 * 解析传入的url，打开单个页面
 * @param {String} mediumUrl
 */
async function fetchMediumUrl (mediumUrl, pageAttemptTimes = 0) {
  // 转换地址，详细部分可以看源码
  mediumUrl = transformMediumUrl(String(mediumUrl));
  return new Promise(async (resolve, reject) => {
    let page = null;
    try {
      let browser = null;
      // 还是先判断是否已有单例的browser，没有的话先创建
      if (!userController.browser) {
        browser = await puppeteer.launch();
        userController.setBrowser(browser);
      }
      // 创建一个新的页面
      page = await userController.browser.newPage();
      // 获取页面的cookie
      const cookies = await page.cookies(mediumUrl);
      // 判断是否有cookie存在，没有的话就将准备好的cookie填充进page，详细部分看源码
      if (cookies.length <= 0) {
        page = await Cookie.setCookie(page);
      }
      try {
        userController.spinner.stop();
        userController.spinner.color = 'yellow';
        userController.spinner.text = '进入页面中...';
        userController.spinner.start();
        // 跳转到目标地址，失败的话会被catch到
        await page.goto(mediumUrl);
        userController.spinner.text = '进入页面成功';
        userController.spinner.succeed();
        // 跳转成功后等待页面加载完成并获取页面数据
        const content = await page.content();
        try {
          // 执行解析页面数据的方法。成功的流程便到此为止。
          await parseMediumPage(content);
        } catch (err) {
          // 如果解析出错被catch到则直接运行finall部分，结束promise流程。
        }
      } catch (err) {
        userController.spinner.text = '进入页面失败';
        userController.spinner.fail();
        // 跳转页面或者获取内容失败时或尝试重新连接，默认次数是5次。
        if (pageAttemptTimes < userController.pageAttemptTimes) {
          // 重连之前需要关闭当前这个页面，因为接下来会重新创建一个
          await page.close();
          const newAttempt = pageAttemptTimes + 1;
          console.log('--------');
          console.log(`重连${mediumUrl}`.yellow.bgBlack);
          console.log(`次数${newAttempt}`.yellow.bgBlack);
          console.log('--------');
          resolve(await fetchMediumUrl(mediumUrl, newAttempt));
        } else {
          // 如果重连失败到一定次数后（默认5次），会关闭页面
          console.log(`跳转到目标页面失败:${mediumUrl}`.yellow);
        }
      }
    } catch (err) {
      console.log('[async fetchMediumUrl catch err in promise]', err);
    } finally {
      page && !page.isClosed() && await page.close();
      resolve();
    }
  });
}
```

## 解析内容

获取页面的内容后，还需要解析。解析的方法有多种，比如说常用的 `cheerio`，使用这个的话，解析跟以前没什么不同，还有一种方法是直接使用 `puppeteer` 的 `document.querySelector` ，相信大家都很熟悉，这是JS DOM的原生API。

1. 使用 cheerio：
```javascript
/**
 *
 * @param {String} pageContent
 */
async function parseMediumPage (pageContent) {
  return new Promise(async (resolve, reject) => {
    userController.spinner.stop();
    userController.spinner.color = 'yellow';
    userController.spinner.text = '解析页面内容中...';
    userController.spinner.start();

    const $ = cheerio.load(pageContent);

    // * 图源
    const _illust_modal$ = $('div[role=presentation] > a');

    if (_illust_modal$) {
      try {
        const dataSrc = _illust_modal$.attr('href');
        if (dataSrc && dataSrc.includes('http')) {
          // 图片只有一张的时候
          await pureImg.getPureImg(dataSrc);
        } else if (dataSrc) {
          // 图片有多张的时候
          await fetchMultipleHref(dataSrc);
        } else {
          // 没有找到图片链接的时候
          userController.spinner.stop();
          userController.spinner.color = 'yellow';
          userController.spinner.text = '没有内容';
          userController.spinner.warn();
        }
      } catch (err) {
        resolve();
      }
    } else {
      userController.spinner.stop();
      userController.spinner.color = 'yellow';
      userController.spinner.text = '没有找到节点';
      userController.spinner.fail();
    }
    resolve();
  });
}
```


2. 使用原生API：
```javascript
/**
 *
 * @param {Puppeteer.Page} page
 */
async function parseMediumPage (page) {
  return new Promise(async (resolve, reject) => {
    // ... cheerio 部分
    
    // * 图源
    const _illust_modal$ = await page.evaluate(() => {
      return document.querySelector('div[role=presentation] > a');
    })

    if (_illust_modal$) {
      try {
        const dataSrc = _illust_modal$.getAttribute('href');
        // ... cheerio 部分
      } catch (err) {
        resolve();
      }
    } else {
      // ... cheerio 部分
    }
    resolve();
  });
}
```

以上两种方法都能根据选择出对应DOM节点。

## 获取图片

前面已经拿到了DOM节点并且得到了图片下载地址，那么接下来就是如果获取图片了，这里可能有同学发现了，既然只有图片，那么是不是不需要pupeteer呢？我认为是的，图片是静态资源，与动态加载无关，这里是直接获取资源。那么用以前的方法是可行的。还有一点就是，浏览器的页面在创建，跳转，加载和销毁的消耗比superagent直接获取资源要大得多，所以使用puppeteer爬取是会慢一点，但是无可厚非，毕竟能直接爬取SPA，还能创建PDF和快照之类的也是极好的。

因为puppeteer的API几乎都是异步的，我这里也对原来的获取图片方法小小的改造了以下。其实为了适应异步，所有步骤都已经改成了异步（返回Promise）。

```javascript
/**
 *
 * @param {String} illustUrl
 */
async function fetchPureImg (illustUrl, filename, pageAttemptTimes = 0) {
  return new Promise(async (resolve, reject) => {
    userController.spinner.stop();
    userController.spinner.color = 'yellow';
    userController.spinner.text = `下载图片中:${filename}`.gray;
    userController.spinner.start();
    superagent
      .get(illustUrl)
      // .set('Cookie', cookiesStr)
      .set('Referer', referer)
      .timeout(60 * 1000)
      .end(async (err, res) => {
        if (err) {
          userController.spinner.text = `下载图片失败:${filename}`.red;
          userController.spinner.fail();
          // console.log(err);
          // * 重新连接下载
          if (pageAttemptTimes < userController.pageAttemptTimes) {
            const newAttempt = pageAttemptTimes + 1;
            console.log(`重连次数${newAttempt}`.yellow.bgBlack);
            resolve(await fetchPureImg(illustUrl, filename, newAttempt));
          } else {
            resolve();
          }
        } else {
          userController.spinner.text = `下载图片成功:${filename}`.green;
          userController.spinner.succeed();
          if (res.body) {
            await writeBufferPureImg(res.body, filename);
          }
          resolve();
        }
      });
  })
}
```

在控制异步流程时，有人会有疑问：会什么 catch 到错误时仍然 resolve，而不是 reject 呢？
其实这个疑问是对的，只不过在这里，我只希望在合适的时机 `fulfilled` 掉 Promise，无论是 resolve 还是 reject，当有Error被catch时，该如何应对会在这个函数内处理，例如输出error信息等。（如果 catch 到 error 时，使用 resolve(error) 或者 reject(error) 都可以将错误传递到上级，只是使用 resolve(error) 时需要进一步判断，使用 reject(error) 则可以被catch到）
当然，如果这个错误的信息不希望在正式使用中看到，那么可以加入`debug` 变量控制错误信息的显示。


## 保存图片

保存图片就是将图片写入到硬盘内，核心代码与上一个版本没有区别，只不过流程改为了异步。

```javascript
/**
 *
 * @param {Buffer} buffer
 * @param {String} filename
 */
async function writeBufferPureImg (buffer, filename) {
  return new Promise((resolve, reject) => {
    let dirPath = '';
    if (pathController.output) {
      dirPath = path.resolve(process.cwd(), pathController.output);
    } else {
      const dateFormated = moment().format('YYYY-MM-DD');
      dirPath = path.join(process.cwd(), `${dateFormated} pixiv`);
    }
    if (!fsExistsSync(dirPath)) {
      fs.mkdirSync(`${dirPath}`);
    }
    const filenameList = filename.split('.');
    filename = userController.cFilenamePrefix + filenameList[0] + userController.cFilenameSuffix + `.${filenameList[1]}`;
    const filenameFull = path.join(dirPath, filename);

    userController.spinner.color = 'yellow';
    userController.spinner.text = `保存图片中...`;
    userController.spinner.start();

    fs.writeFile(filenameFull, buffer, (err) => {
      if (err) {
        userController.spinner.text = `保存失败`.red;
        userController.spinner.fail();
        // console.log(err);
        resolve();
      } else {
        userController.spinner.text = `保存成功:${filenameFull}`.cyan;
        userController.spinner.succeed();
        resolve();
      }
    });
  });
}
```
