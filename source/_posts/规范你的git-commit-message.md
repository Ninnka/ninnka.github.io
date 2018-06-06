---
title: 规范你的git-commit-message
date: 2016-12-16 12:24:42
categories:
	- Git
tags:
	- git
	- commit
	- 规范
---

> git commit是日常开发中经常会用到的一项操作。在我所知道的大部分人里经常有人为了偷懒使用`git commit -m ''`这种操作，但是，我秉承积极向上的学习态度查阅了不少资料后明白了commit的优雅之处。

**现今，有许多优秀的开源的commit工具： `commitizen/cz-cli`; `commitizen/cz-conventional-changelog` `conventional-changelog/standard-version`，这些功具为你提供了提交信息和版本发布一条龙服务。再深入一点，还可以集成`marionebl/commitlint`来检查commit信息是否符合规范。**

我深入了解后，将按照以下顺序对实践结果进行记录：

1. git commit message的格式
2. git commit 的模板
3. 使用Commitizen：规范你的commit
4. 使用cz-customizable：自定义adapter
5. 使用Commitlint：检查你的message
6. 使用standard-version：全自动CHANGELOG

## 实践记录

### git commit message的格式

git commit message的格式我们将沿用[angular团队的commit规范](https://github.com/angular/angular.js/blob/master/DEVELOPERS.md#-git-commit-guidelines)
后面要说的模板和工具很多都是基于此规范而衍生的。

> **每个提交消息由标题，正文和页脚组成。标题有特别的格式，它包含类型，范围和主题：标题是必须有的，但是标题的范围是可选的。**
```
<type>(<scope>): <subject>
<BLANK LINE>
<body>
<BLANK LINE>
<footer>
```

#### type(类型)：描述这次的commit的类型，有以下类型：
- feat: 增加新的特性，功能
- fix: 修改BUG
- refactor: 重构代码
- docs: 修改文档
- style: 修改代码格式
- test: 修改测试用例
- chore: 其他的一些非代码修改, 比如构建流程, 依赖管理等.

#### scope: 列出 commit 影响的范围
- view
- router
- component
- utils
- dev.env
- prod.env
- <...>

#### subject: 描述 commit, 符合50行/72列的格式

#### body: 具体的 commit，描述具体的修改内容, 建议分为多行, 符合50行/72列的格式

#### footer: 通常为一些备注, BREAKING CHANGE 或者修复的 bug 的链接

------

### git commit 的模板

> 你可以创建一个自定义的提交模板来提醒自己书写一个正确的格式的commit message，[官方文档](https://git-scm.com/book/en/v2/Customizing-Git-Git-Configuration)

比如：你可以在用户目录下创建这样一个`.gitmessage`文件
```
Subject line (try to keep under 50 characters)

Multi-line description of commit,
feel free to be detailed.

[Ticket: X]
```

创建完后你就拥有了一份自己的`commit` 模板，但是你还没办法使用它
我们需要打开bash或cmd或其他命令行工具，然后执行以下命令

- mac os / Linux
```bash
git config --global commit.template ~/.gitmessage
```

- window
```bash
git config --global commit.template C:\Users\<用户名>\.gitmessage
```

成功后，每次commit都会出现.gitmessage中的模板

### 使用Commitizen：规范你的commit

> 每次commit时，不再需要通过CONTRIBUTING.md来查找首选的commit格式，使用committizen可以立即获取到你提交消息的格式，并且提示你哪些必填是必填字段。

按照上一个步骤，我们可以创建一个属于自己的模板，但是仅仅如此是不够的，我们需要更加智能的工具，这就是commitizen

- 我们需要安装commitizen的命令行工具，建议全局安装
```bash
npm install -g commitizen
```

- 你还可以安装cz-conventional-changelog，一个首选的`commitizen`适配器
```bash
npm install -g cz-conventional-changelog
```

- 在用户目录下创建`.czrc`
mac os / Linux
```bash
echo '{ "path": "cz-conventional-changelog" }' > ~/.czrc
```

window（使用bash）
```bash
echo { "path": "cz-conventional-changelog" } > C:\Users\<用户名>\.czrc
```

- 使用新的提交命令
```bash
# 代替git commit
git cz
```

效果如下：
![效果图](https://github.com/commitizen/cz-cli/raw/master/meta/screenshots/add-commit.png)

### 使用cz-customizable：自定义adapter

> 显然地，不同团队有不同的习惯，或许你不需要不熟悉Angular的团队规范，你可以使用[cz-customizable](https://github.com/leonardoanalista/cz-customizable)来自定义一份adapter

- 全局安装
```bash
npm install -g commitizen
```

- 项目安装
```bash
npm install cz-customizable --save-dev
```

- 在`.czrc`或`package.json`中添加一下内容：
- .czrc
```
{ "path": "cz-customizable" }
```
- package.json
```json
...
"config": {
  "commitizen": {
    "path": "node_modules/cz-customizable"
  }
}
```

同时在用户目录下或项目目录下创建 `.cz-config.js` 文件
```javascript
'use strict';

module.exports = {

  types: [
    {
      value: 'feat',
      name : 'a new feature.'
    },
    // ...
  ]
  scopes: [
    // ...
  ],

  allowCustomScopes: true,
  
  allowBreakingChanges: [
    'feat',
    // ...
  ],
}
```

### 使用Commitlint：检查你的message

如果团队对提交的信息格式非常严格的话，那么使用commitlint是一个很好的选择。

- 安装
```bash
npm install --save-dev @commitlint/cli
```

- 同时推荐安装Angular团队的检查规范
```bash
npm install --save-dev @commitlint/config-conventional
```

- 在项目中使用时，记得在项目的根目录下创建`.commitlintrc.js`
```javascript
module.exports = {
  extends: [
    '@commitlint/config-conventional'
  ],
  rules: {
  },
};
```

- 如果使用了自定义的adapter，则需要安装`commitlint-config-cz`，并修改`.commitlinerc.js`的内容
```bash
npm i -D commitlint-config-cz
```

```javascript
module.exports = {
  extends: [
    'cz'
  ],
  rules: {
  }
};
```

- 配合git hook来检查 commit message，

1. 安装husky
```bash
npm i husky@next
```

2. 在package.json中添加
```json
'husky': {
  'hooks': {
    ...,
    'commit-msg': 'commitlint -e $GIT_PARAMS'
  }
},
```

### 使用standard-version：全自动CHANGELOG

使用以上工具后，commit message会偏向Angular的那一套规范，同时也方便我们使用[standard-version](https://link.zhihu.com/?target=https%3A//github.com/conventional-changelog/standard-version)这样的工具自动生成版本和更新日志

- 安装
```bash
npm i --save standard-version
```

- 修改项目的`package.json`
```json
"scirpt": {
    ...,
    "release": "standard-version"
}
```

**standard-version还有很多功能，比如：**
- Release as a pre-release
- Prevent Git Hooks
- ...

有兴趣和需求的可以前往repo阅读readme

## 结论

- 优秀的良好的规范有助于团队的代码review
- 自我约束，可以形成一个良好的习惯































