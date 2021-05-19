---
title: 从零创建基于nodejs的命令行工具
date: 2017-04-12 14:28:49
categories:
	- Nodejs
tags:
	- Nodejs
	- 命令行工具
	- 爬虫
	- npm-package
---

> 有时突然奇想，想要快速获取某些网站的页面数据，但是没有合适的工具。
> 其实，开发一个爬虫工具是非常简单的。
> 这次我准备以爬取著名的[插画网站:pixiv](https://www.pixiv.net/)为案例，介绍如何爬取网站的页面数据
> 代码已经准备好，[github地址](https://github.com/Rennzh/node-pixiv-crawler)
> **pixiv已用React重写，没有使用Service-side Rendering，所以普通的爬取方法是行不通的，日后再使用 `puppeteer` 进行重写爬取方法。**

## 创建项目

- 现在任意文件夹中创建一个名为 test-crawler
- 进入 test-crawler 文件夹，开启 bash 或者 cmd 并cd到这个文件内，使用 `npm init --yes` 快速初始化 `package.json` 文件

创建好后，文件夹中会有如下的文件：
```
-(test-crawler)
|- package.json
```

<!-- more -->

### 创建可执行脚本

- 在文件夹中创建 bin 文件夹，

- 并在 bin 文件中创建 `crawling.js` 文件

- 创建好后，文件夹中会有如下的文件：
```
- (test-crawler)
|_ bin
	 |_ crawling.js
|_ package.json
```

- 打开 `crawling.js`
- 加入以下代码
```javascript
#!/usr/bin/env node
console.log('test crawling');
```
**第一行代码是用在 Linux 以及 Unix 的注释，用来判断node的文件位置。
如果你使用的操作系统是Linux 或者 Unix，MacOs，可以先使用 `which node`命令得到 node 的位置。**

### 创建全局的脚本命令

- 创建完脚本后，打开 `package.json`。
- 加入以下代码：
```json
{
  // ...
  "bin": {
    "crawling": "bin/crawling.js",
  },
  // ...
  script: {
    "crawling": "bin/crawling.js",
  }
}
```

**使用 `bin` 可以告诉 node，你需要创建的全局命令名以及命令需要执行的文件。**

- 在项目根目录下使用 `npm link` 为命令创建一个全局的path。这样就可以在任意位置执行命令。
```bash
npm link
```
**这个不建议在开发进行时使用，因为每次修改后需要重新 `npm link`，使用 `npm link` 时需要重新安装 `node_modules`。所以，开发时仍然是手动执行 `crawling.js` 文件。**

**使用这个命令前，要先删掉当前文件夹中的 `node_modules`，否则会遇到 `syscall unlink` 的问题。**

### 测试命令

- 任意文件夹中执行命令
```bash
$ crawling
crawling // 输出结果
```

- 测试结束后记得执行 `npm unlink` 删除链接。

## 命令行参数

- 当我们执行以下命令时可以将命令标识后面的文字输入到nodejs的进程中。
```bash
crawling somevalue
```

- 接收这些参数的变量名为 `process.argv`

### process.argv

- `process.argv` 是一个数组，将传入的值以空格分开。
- 打开 `crawling.js`，修改代码：
```javascript
#!/usr/bin/env node
console.log('test crawling');
for (let item of process.argv) {
  console.log(item);
}
```

- 执行
```bash
$ node ./bin/crawling somevalue othervalue
test crawling // 输出test crawling
node // 第一个参数
./bin/crawling // 第二个参数
somevalue // 第三个参数
othervalue // 第四个参数
```

**node进程会把上面的命令所有值以空格分开保存到 `process.argv`。**

### commander

> commander 是一个简化命令行参数操作的库。

- 修改 `crawling.js`
```javascript
#!/usr/bin/env node
const program = require('commander');
program
  .version('test-crawler v0.0.1', '-v, --version')
  .option('-u, --urls [address]', 'Set the url [address] for img', '')
  .option('-i, --ids [illust_id]', 'Set the [illust_id] which belong to img', '')
  .option('-o, --output [output_path]', 'Set the img [output_path]', '')
  .option('-n, --file-name [file_name]', 'Custom [file_name]', '')
  .parse(process.argv);
```

- `commander` 内置了 `version`方法，可以生成一个版本的命令。
- 此外，我们试着先加入一些命令
'-u, --urls' 都是表示同一个命令，前者是缩写，后者是全写，跟在后面的方括号以及里面的字符串是用来表示这个命令可以接收一个参数
比如：命令名  -u somgarg 或者 命令名  --urls somgarg

- 执行命令，输出版本号
```bash
$ node ./bin/crawling -v
test-crawler v0.0.1
```

> `Commander` 的更多介绍和使用方法可以查看[仓库](https://github.com/tj/commander.js)


## 爬虫代码

> 关羽爬虫的核心，主要集中在：
> 1. 获取网页的真实内容
> 2. 分析DOM节点

对此我们需要使用到
1. cheerio
2. superagent
3. superagent-charset

- 安装
```bash
npm install -S cheerio superagent superagent-charset
```

### 分析和获取需要爬取的网站内容，然后解析内容

- 以[插画网站:pixiv](https://www.pixiv.net/)的插画页面为例，首先获取页面内容

```javascript
/**
 * 解析传入的url
 * @param {String} mediumUrl 网站的地址或者插画的id
 */
function fetchMediumUrl (mediumUrl, pageAttemptTimes = 0) {
  mediumUrl = transformMediumUrl(String(mediumUrl));
  return new Promise((resolve, reject) => {
    superagent
      .get(mediumUrl)
      .set('Cookie', Cookie.cookiesStr)
      .timeout(60 * 1000)
      .end((err, res) => {
        if (err) {
          console.log(`下载网页失败:${mediumUrl}`.yellow);
          console.log(err);
          // * 重新连接下载
          if (pageAttemptTimes < userController.pageAttemptTimes) {
            const newAttempt = pageAttemptTimes + 1;
            console.log(`重连次数${newAttempt}`.yellow.bgWhite);
            fetchMediumUrl(mediumUrl, newAttempt);
          }
        } else {
          console.log(`下载网页成功:${mediumUrl}`.green);
          res.res && res.res.text && parseMediumPage(res.res.text);
          resolve({
            code: 0,
            type: 'fetchMediumUrl'
          })
        }
      });
  });
}
```

- 分析网页中图片的DOM结构，获取对应的DOM节点，再获取图片地址

经分析得出：
图片在 `._illust_modal .wrapper .original-image`

```javascript
/**
 * @param {String} pageContent
 */
function parseMediumPage (pageContent) {
  const $ = cheerio.load(pageContent);
  const _multiple$ = $('.works_display a.multiple');
  if (_multiple$[0]) {
    // * 图源是multiple的情况
    const multipleDataSrc = _multiple$.attr('href');
    multipleDataSrc && fetchMultipleHref(multipleDataSrc);
  } else {
    // * 图源不是multiple的情况
    const _illust_modal$ = $('._illust_modal .wrapper .original-image');
    const dataSrc = _illust_modal$.data('src');
    dataSrc && pureImg.getPureImg(dataSrc)
  }
}
```

- 下载片，并写入本地（代码较多，可以去[仓库](https://github.com/Rennzh/node-pixiv-crawler)查看）

```javascript
/**
 * @param {String} imgPath
 */
function getPureImg (imgPath) {
  const { illustId, name } = spliceIllustInfoFormPath(imgPath);
  const illustUrl = transformIllustUrl(imgPath);
  fetchPureImg(illustUrl, name);
}

/**
 * @param {String} illustUrl
 */
function fetchPureImg (illustUrl, filename, pageAttemptTimes = 0) {
  return new Promise((resolve, reject) => {
    console.log(`下载图片中:${filename}`.gray);
    superagent
      .get(illustUrl)
      // .set('Cookie', cookiesStr)
      .set('Referer', referer)
      .timeout(60 * 1000)
      .end((err, res) => {
        if (err) {
          console.log(`下载图片失败:${filename}`.yellow);
          console.log(err);
          // * 重新连接下载
          if (pageAttemptTimes < userController.pageAttemptTimes) {
            const newAttempt = pageAttemptTimes + 1;
            console.log(`重连次数${newAttempt}`.yellow.bgWhite);
            fetchPureImg(illustUrl, filename, newAttempt);
          }
        } else {
          console.log(`下载图片成功:${filename}`.green);
          res.body && writeBufferPureImg(res.body, filename);
        }
      });
  })
}

/**
 * @param {Buffer} buffer
 * @param {String} filename
 */
function writeBufferPureImg (buffer, filename) {
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
  fs.writeFile(filenameFull, buffer, (err) => {
    if (err) {
      console.log(`写入失败:${filenameFull}`.red);
      console.log(err);
    } else {
      console.log(`写入成功:${filenameFull}`.cyan);
    }
  });
}

/**
 * @param {String} path
 */
function fsExistsSync (path) {
  try {
    fs.accessSync(path, fs.F_OK);
  } catch (e) {
    return false;
  }
  return true;
}
```

## 发布npm-package

1. 注册一个npm账号

2. 在本地执行命令
```bash
npm login
```
3. 登录成功后
执行
```bash
npm publish
```

## 结论

创建一个基于 `Nodejs` 的命令行工具，最基本的几点：
1. 在 `package.json` 中加入命令名和命令的执行脚本路径
2. 添加 `bin` 文件夹，在 `bin` 文件夹中中加入执行脚本
3. 执行脚本的顶部需要声明
```
#!/usr/bin/env node
```
4. 使用 `Commander` 或其他工具库处理命令行参数，也可以不使用。
5. 根据情况使用 `cheerio` 或者 `puppeteer` 爬取网页内容。


























































