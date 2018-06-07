---
title: vue-webApp + cordova构建nativeApp（hybrid模式）
date: 2018-03-05 14:24:57
categories:
	- 前端
	- JavaScript
tags:
	- Vue
	- cordova
	- 打包
---

> Vue.js 作为一套构建用户界面的渐进式框架，经常被用作开发webApp，但是运行在不同系统的浏览器中总会遇到不少兼容性问题，尤其是Android下的各种魔改内核，而且没办法与native组件进行交互也是很大的遗憾。综上所述，我们决定将vue-webApp通过cordova打包为nativeApp，也是一种hybrid app。

## 准备开发环境及工具

- 安装Java SDK
Java SDK的安装可以通过官网下载的安装包进行安装，推荐1.8版本，安装完后记得设置 `classPath` 和 `path`。具体步骤这里不做复述，可以自行google解决。

- Android SDK
Android SDK的安装最好通过下载最新的稳定版Android Studio安装。
安装成功后记得在系统变量添加ANDROID_HOME的路径（右键点击我的电脑–>属性–>高级–>环境变量–>系统变量–>新建）
```
ANDROID_HOME=android sdk所在目录
```

- 安装cordova
```bash
npm i cordova -g
```

- 其他工具
  1. Nodejs（不做复述）
  2. Npm（不做复述）
  3. Xcode（MacOS）

<!-- more -->

## 建立Cordova项目

- 使用命令行建立cordova项目
```bash
cordova create project-name
cd project-name
```

- 添加需要支持的平台（Android和IOS，二选一或者两者都选）
```bash
cordova platform add android --save // 添加Android平台
cordova platform add ios --save // 添加IOS平台
```

- 添加 Android 平台时你可能会遇到这样的问题
进度卡在`> Configuring > 0/2 projects`
解决办法：
1. 最简单的解决方法是使用科学上网工具进行全局代理。
2. 修改下载地址：
- 找到你的 gradle 目录
- 修改，解压gradle，找到`src/core/org/gradle/api/artifacts/ArtifactRepositoryContainer.java` 文件，把 `MAVEN_CENTRAL_URL` 变量修改为 `repo1.maven.org/maven2`
- 修改 `src/core/org/gradle/api/internal/artifacts/dsl/DefaultRepositoryHandler.java` 文件，把 `BINTRAY_JCENTER_URL` 变量修改为 `jcenter.bintray.com/`
- 运行
```bash
cordova platform remove android && cordova platform add android
```
重新添加Android平台和就成功了。

## 打包vue-webApp到cordova项目下

> cordova打包web时是将cordova项目目录下的www内的文件，所以我们打包vue的webApp时需要把打包的dist位置直接指定到www目录下。

这里我们直接使用vue-cli2.x.x作为脚手架

- 将项目移动到cordova项目根目录下

- 修改build/prod.env.js
```javascript
'use strict'

const BUILD_TYPE = process.env.BUILD_TYPE ? process.env.BUILD_TYPE : '';

module.exports = {
  NODE_ENV: '"production"',
  BUILD_TYPE: `'${BUILD_TYPE}'`
}
```
我们指定了一个环境变量`BUILD_TYPE`，打包vue-webApp时传入`BUILD_TYPE=app`就可以将打包模式指定为app。

- 修改build/index.js
添加以下代码
```javascript
const path = require('path')

const BUILD_APP = process.env.NODE_ENV === 'production' && process.env.BUILD_TYPE === 'app';

BUILD_APP && console.log('BUILD_TYPE is app');

const BUILD_INDEX = BUILD_APP ?
  path.resolve(__dirname, '../../www/index.html') :
  path.resolve(__dirname, '../dist/index.html');

const BUILD_ASSETS_ROOT = BUILD_APP ?
  path.resolve(__dirname, '../../www') :
  path.resolve(__dirname, '../dist');

const BUILD_ASSETS_PUBLIC_PATH = BUILD_APP ?
  './' :
  '/';
```
修改build对象下的部分代码
```javascript
build: {
  // ...
  // index: path.resolve(__dirname, '../dist/index.html')
  index: BUILD_INDEX,
  // assetsRoot: path.resolve(__dirname, '../dist')
  assetsRoot: BUILD_ASSETS_ROOT,
  // assetsPublicPath: '/',
  assetsPublicPath: BUILD_ASSETS_PUBLIC_PATH,
  // ...
}
```

修改完后就可以通过环境变量的方式选择打包的方式。

- 处理`css`的`background-image`属性在`BUILD_TYPE`为app时，路径错误的问题
处理的方法有两种：
1. 将所有背景图片移动到static文件夹，这个目录在vue-cli中默认不会被webpack处理，打包时直接移动到dist/statis目录，路径不会变。
2. 修改webpack配置
2.1 修改build/utils.js，找到`generateLoaders`方法
有这样一段注释（vue-cli@2.x.x）
>  // Extract CSS when that option is specified
>  // (which is the case during production build)

修改注释下的那段代码
```javascript
if (options.extract) {
  const params = {
    use: loaders,
    fallback: 'vue-style-loader',
  }
  if (process.env.NODE_ENV === 'production' && config.build.assetsPublicPath !== '/') {
    params.publicPath = '../../';
  }
    return ExtractTextPlugin.extract(params)
  } else {
    return ['vue-style-loader'].concat(loaders)
  }
}
```

修改后可以解决背景图片引用的路径问题。

## 添加Cordova.js

> 可以在index.html直接以script的方式加入cordova.js，但是这种方式在以普通网页的形式打开时会报错，所以我们在`main.js`中加入以下代码

```javascript
// * 判断是否以原生APP的形式运行
if (window.location.protocol === 'file:' || window.location.port === '3000') {
  let cordovaScript = document.createElement('script');
  cordovaScript.setAttribute('type', 'text/javascript');
  cordovaScript.setAttribute('src', 'cordova.js');
  document.body.appendChild(cordovaScript);
}
```

需要调用 Cordova 的插件但是不清楚 cordova 和设备是否加载完毕时
```javascript
document.addEventListener("deviceready", yourCallbackFunction, false);
function yourCallbackFunction () {
  // ...
}
```

## 添加Cross-walk webview插件

- 进入cordova项目根目录
```bash
cordova plugin add cordova-plugin-crosswalk-webview@latest --save
```
这里必须加上`@latest` 后缀或者`^2.2.0`，安装 2.2.0+ 版本，否则会遇到 [#XWALK-7422](https://crosswalk-project.org/jira/browse/XWALK-7422) 的问题。

**加入Cross-walk webview后，打包时默认有两个apk，分别对应arm平台和x86平台，arm为手机用，体积会比原来增加20MB+，多的体积就是cross-walk的runtime**

- 如果想打包后合并到到一个文件，可以修改cordova项目根目录的`config.xml`
```xml
<plugin name="cordova-plugin-crosswalk-webview" spec="~2.2.0">
    <variable name="XWALK_VERSION" value="22+" />
    <variable name="XWALK_LITEVERSION" value="xwalk_core_library_canary:17+" />
    <variable name="XWALK_COMMANDLINE" value="--disable-pull-to-refresh-effect" />
    <variable name="XWALK_MODE" value="embedded" />
    <variable name="XWALK_MULTIPLEAPK" value="true" />
</plugin>
```

- ios系统的`UIWebView` 替换为 `WKWebView`
1. 安装
```bash
cordova plugin add cordova-plugin-wkwebview-engine@latest --save
```
WKWebView目的是给出一个新的高性能的 Web View 解决方案，摆脱过去 UIWebView的旧笨重慢等问题，针对ios8前的系统。

2. 修改`config.xml`
在`<platform name="ios">`添加
```xml
<feature name="CDVWKWebViewEngine">
    <param name="ios-package" value="CDVWKWebViewEngine" />
</feature>
```

- 调试Cross-walk webview

`Cross-walk webview` 基于 `Chromium`，可以使用 `Chrome` 远程调试的功能。 PC连接 Android 手机并开启 USB 调试模式，开启 Cordova 项目，在 Chrome 地址栏中输入 `chrome://inspect` ，即可打开远程控制台。Vue Devtool 同理。

## Cordova-plugin-whitelist 与 CSP 安全策略

> Cordova 4.0 以上环境中，需要安装cordova-plugin-whitelist插件并 对 config.xml 中的 <access origin="your-policy" /> 标签和 index.html 中的 META 标签做一定设置，防止出现共享 Webview 模式下的跨站攻击等安全问题。新的cordova项目创建时默认安装Cordova-plugin-whitelist，可以通过查看`config.xml`知道安装了哪些插件，插件的字段名为`plugin`，`name`是插件名，`spec`是版本号。

- 打包时在`index.html`的`header`添加
```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self' data: gap: cdvfile: https://ssl.gstatic.com 'unsafe-eval'; style-src 'self' 'unsafe-inline'; media-src *; font-src * data:; img-src *; script-src *;">
```

## 添加 [CodePush](https://microsoft.github.io/code-push/) 热更新

> `CodePush`是微软开发的用于 `Cordova` 和 `React Native` 开发者热更新他们的APP到用户的设备上的一款开发工具。

- 安装
```bash
cordova plugin add cordova-plugin-code-push@latest --save
```

- 安装 code-push，需要注册一个账号，用于开发测试，正式部署时需要专门的账号
```bash
npm install -g code-push-cli
code-push -v // 查看 code-push 版本，有输出说明成功
```
```bash
code-push register // 通过 Github 或者 微软帐号登录
code-push login // 登录后会在本地生成 `session` 文件用来验证你的发布权限
```

- 添加app
```bash
code-push app add <appName>
```
添加后会生成key，复制好 staging 中的 key。

- 在 app 的 config.xml 中粘贴上一步复制的 key
```xml
<platform name="android">
    <preference name="CodePushDeploymentKey" value="YOUR-ANDROID-DEPLOYMENT-KEY" />
</platform>
<platform name="ios">
    <preference name="CodePushDeploymentKey" value="YOUR-IOS-DEPLOYMENT-KEY" />
</platform>
```

- 在 `index.html` 中的安全策略 `meta` 中添加域名
```html
<meta http-equiv="Content-Security-Policy" content="default-src https://codepush.azurewebsites.net ... />
```

- 添加热更新处理
```javascript
var app = {
    // Application Constructor
    initialize: function() {
        document.addEventListener('deviceready', this.onDeviceReady.bind(this), false);
    },

    onDeviceReady: function() {
        const downloadProgress = (progress) => { console.log(`Downloaded ${progress.receivedBytes} of ${progress.totalBytes}`); }
        codePush.sync( null, {updateDialog: true, installMode:InstallMode.IMMEDIATE, mandatoryInstallMode: InstallMode.IMMEDIATE}, downloadProgress );
    }
};
app.initialize();
```

- 重新打包app，并发布app

> CodePush 发布更新的方式是通过 `code-push release-cordova` 命令

通过code-push release-cordova发布更新
```bash
code-push release-cordova <appName> <platform>
```
比如说：
```bash
code-push release-cordova MyApp-Android android  --t 1.0.0 --dev false --d Production --des "1.优化操作流程" --m true
```

> 其中参数--t为二进制(.ipa与apk)安装包的的版本；--dev为是否启用开发者模式(默认为false)；--d是要发布更新的环境分Production与Staging(默认为Staging)；--des为更新说明；--m 是强制更新。
关于code-push release更多可选的参数，可以在终端输入code-push release进行查看。

## Android 协议认证问题和生成签名

- 同意协议
- 
1. MacOS
```bash
cd ~/Library/Android/sdk/tools/bin
./sdkmanager --licenses
```

2. window
```bash
cd <SDK目录>/tools/bin
./sdkmanager --licenses
```

一路同意到最后就可以了。

- 生成签名

- 打开 Android Studio 的 `"File"->"Open"`
- 定位到 cordova 项目下的`/platforms/android`
- 选择"Build"->"Generate Signed APK"
- 选择一个已有的签名，或者新建一个
- 选择release生产版本，点击finish完成

## IOS 开发者证书

- 在苹果开发者网 `developer.apple.com/account/` 站生成了需要的appid,证书和profile文件
- 在Display Name 修改应用名称
- 用Xcode打开工程，注意是.xcworkspace(白颜色)的工程，不是.xcodeproj(蓝颜色)的工程
- 在Resources/Images.xcassets里修改启动图片和应用图标，AppIcon是对应平台的图标，LaunchImage是启动图片
- 选择“Product”->"Edit Scheme..."，在“Build Configuration”中选择“Release”,单击"Close"
- 选择菜单栏中的"Product"->"Archive"，选择刚刚的工程，选择“Export...”，选择第一个(Save for iOS App Store Deployment)，点击Export。然后会生成一个文件夹，里面是ipa包
- 打开 Application Loader，选择“交付您的应用程序”，选取之前生成的ipa包，然后下一步
- 去 `itunesconnect.apple.com/WebObjects/…` 完成app相关信息的填写

## 添加极光推送

> 地址：[极光推送](https://github.com/jpush/jpush-phonegap-plugin)

- 安装
```bash
cordova plugin add jpush-phonegap-plugin --variable API_KEY=your_jpush_appkey
```

- 极光推送的相关事件
```javascript
document.addEventListener("deviceready", onDeviceReady, false); 
document.addEventListener("jpush.openNotification", onOpenNotification, false);
document.addEventListener("jpush.receiveNotification", onReceiveNotification, false);
document.addEventListener("jpush.backgroundNotification", onBackgroundNotification, false);
document.addEventListener("jpush.receiveMessage", onReceiveMessage, false);
```

- 更多使用方法和信息可以去仓库地址查看

















