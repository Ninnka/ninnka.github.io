---
title: JSON Web Token解惑
date: 2021-06-19 23:24:35
categories:
  - Auth 鉴权
	- JSON Web Token
tags:
	- Auth 鉴权
	- JSON Web Token
---

## JSON Web Token 是什么

`JSON Web Token` 是基于开放标准（RFC 7519）定制的字符串，可以安全地把各方之间的信息作为 JSON 字符串传输

`JSON Web Token` 通过 RSA 或 ECDSA（具有HMAC算法），公共/私钥和消息体生成 JWT 签名，保证 `JSON Web Token` 的安全性和唯一性

`JSON Web Token` 定义为紧凑自包含的形式，可以做到无依赖校验和解析

## JSON Web Token 的结构

JWT由三部分组成，每部分都通过 `.` 链接起来，大概长这样：xxx.yyy.zzz

每部分都经过 base64 编码

### 第一部分 header

`header` 原文是一个 JSON 字符串，其中包含两部分信息

1. typ: 'JWT' token类型
2. alg: 'HS256' 加密算法类型

```js
{
    alg: 'HS256',
    typ: 'JWT',
}
```

然后，用Base64对这个JSON编码就得到JWT的第一部分

```js
const headerBase64 = window.btoa(JSON.stringify({ alg: 'HS256', typ: 'JWT' }));
```

[MDNS btoa](https://developer.mozilla.org/zh-CN/docs/Web/API/WindowOrWorkerGlobalScope/btoa)

<!-- more -->

### 第二部分 payload claims

第二部分是自定义的消息体声明，包含三部分

[JWT Standard_fields FROM WIKI](https://en.wikipedia.org/wiki/JSON_Web_Token#Standard_fields)

1. `Registered claims`，根据 JWT 规范，有部分是固有字段，俗称保留字，一共有7个：iss (issuer), exp (expiration time), sub (subject), aud (audience), nbf（Not Before），iat（Issued at），jti（JWT ID）
2. `Public claims`，约定公用关键字，这个表示业界使用 payload时，规定俗成的一些字段声明，比如：typ（Token type），cty（Content type）等，具体可以参考上面的 wiki 链接
3. `Private claims`，这个表示私有字段，除去以上的保留声明和约定公用关键字，可以随意声明，通常由使用方去自定义声明和使用，举个栗子
```js
// private claims
{
  "name": "zhang ninnka",
  "email": "ninnka@xxx.com",
  "admin": true
}
```

通过 base64 编码后拼接到 header 后面

```js
const headerBase64 = window.btoa(JSON.stringify({ "alg": "HS256", "typ": "JWT" }));
const payloadBase64 = window.btoa(JSON.stringify({
  "name": "zhang ninnka",
  "email": "ninnka@xxx.com",
  "admin": true
}));
`${headers}.${payloadBase64}`;
```

### 第三部分 signature

第三部分是签名部分，签名由 header 中声明的加密算法进行加密，依赖三个参数

1. `header base64编码`
2. `payload base64编码`
3. `secret 秘钥`，这个有加密端定义后端，一般应用场景就是后端进行定义和加密

使用base64编码加密计算后得到 `signature`，最后拼接到 `payloadBase64` 后面

```js
const headerBase64 = window.btoa(JSON.stringify({ alg: 'HS256', typ: 'JWT' }));
const payloadBase64 = window.btoa(JSON.stringify({
  "name": "zhang ninnka",
  "email": "ninnka@xxx.com",
  "admin": true
}));

// 秘钥
const secret = 'ninnka';

const signature = HMACSHA256(
    headerBase64,
    payloadBase64,
    secret,
);

const signatureBase64 = window.btoa(signature);

`${headers}.${payloadBase64}.${signatureBase64}`;
```

可以通过这个链接[jwt debugger](https://jwt.io/)体验 `JSON Web Token` 的生成

![](https://tva1.sinaimg.cn/large/008i3skNgy1grl8xp6g0mj30z30nqwgs.jpg)

## JSON Web Token 的使用场景

### 一次性验证

举个例子：

用户注册后需要发一封邮件让其激活账户，通常邮件中需要有一个链接，这个链接需要具备以下的特性：能够标识用户，该链接具有时效性（通常只允许几小时之内激活），不能被篡改以激活其他可能的账户…这种场景就和 jwt 的特性非常贴近，jwt 的 payload 中固定的参数：iss 签发者和 exp 过期时间正是为其做准备的。

### 无 session 的多设备登录

服务端不需要记录每个设备的登录态，基于 JWT 特性，每个设备的 JWT 独立存在且有效

## JSON Web Token 的好处

`JSON Web Token` 可以交由一个中心化的服务去生成和颁发，这个服务可以在公司内部，也可以是第三方可信任的服务

`JSON Web Token` 的存储交由客户端去做，可以是浏览器，也可以是移动端app

客户端和服务端两边各自负责 `JSON Web Token` 的一部分服务，服务端不再存储客户的 `session`，过去服务端会把 `session` 数据库或者 `redis` 用于校验登录态

通过分离存储，可以使服务端做到 `stateless`，也就是俗称的无状态

服务端的无状态是 `JSON Web Token` 带来的其中一个好处，本质上，`JWT` 是为了能让 `token` 的 `issue（颁发）`和 `validate（校验`）能分开在不同的服务中做，好处在于可扩展性强，微服务之间没有必要再做 `session` 数据共享

## JSON Web Token 是怎么工作的

当用户请求登录成功后，接口会把通过用户信息生成 `payload`，最后经过上述的过程生成 `JWT`，并在页面中写入 Cookie 中。

HTTP 本身是无状态的，后续访问资源或者请求接口时，需要校验用户身份的合法性，身份校验通常由这几种方式

### 基于 HTTP Auth

若服务端开启了 `HTTP Auth` 校验，那么在后续请求中，需要把 JWT 设置在`Authorization`请求头中

```
Authorization: Bearer JWT
```

在访问资源或者请求接口时，服务端会先校验请求头中的 `Authorization` 是否合法

![](https://tva1.sinaimg.cn/large/008i3skNgy1grnr5j5fgzj30jb09lgmh.jpg)

### 基于 Cookie

服务端不一定会使用 `HTTP AUTH` 做鉴权，有可能直接通过 `Cookie` 传输身份校验的 `token`

那么在后续的资源访问或者接口请求，服务端从 cookie 中获取 `JWT`，然后解密 `JWT` 的签名校验签名的合法性

## JSON Web Token 存在什么问题

上述提到了 `JWT` 的一些简单运用场景和好处，但是理想与现实之间肯定有差距

### token 时效问题

由于服务器不保存 `session` 状态，因此无法在使用过程中废止某个 `token`，或者更改 `token` 的权限。也就是说，一旦 JWT 签发了，在到期之前就会始终有效，除非服务器部署额外的逻辑。

如果 `JSON Web Token` 发生泄漏，无法通过后端的手段使其失效，泄漏的 `JSON Web Token` 会一直有效直到过期

### token 不唯一

无法实现同一类型设备的排他登录，比如在android手机A登录后，又在android手机B上登录，只要两个 `JWT` 没有过期，那么他们就会一直有效

### token 续签问题

`JWT` 的 `payload` 通常可以通过设置 `exp` 来标记 `token` 的有效期，但是 `payload` 会参与签名的计算，如果想要延长 `JWT` 的有效期就需要更新 `exp` 字段，也就是导致 `JWT` 发生变化

为了解决 `JSON Web Token` 自身存在的问题，在使用 `JSON Web Token` 时通常会跟 `redis` 配合使用，把用户的 `JSON Web Token` 存起来，作为最新的 `token`，但是这种方法又跟 `session-cookie` 的方案没有太大的区别

![](https://tva1.sinaimg.cn/large/008i3skNgy1grnzirfd1uj30j50ecjrg.jpg)

## 参考

> [jwt introducation](https://jwt.io/introduction)
> [wiki JSON_WEB_TOKEN](https://en.wikipedia.org/wiki/JSON_Web_Token)
