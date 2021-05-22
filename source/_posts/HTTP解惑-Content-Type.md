---
title: HTTP解惑-Content-Type
date: 2021-05-22 20:28:13
categories:
  - Http
  - Content-Type
tags:
  - Web
  - Http
  - Content-Type
---

# 什么是 Content-Type

HTTP协议（RFC2616）采用了请求/响应模型。
客户端向服务器发送一个请求，请求头包含请求的方法、URI、协议版本、以及包含请求修饰符、客户信息和内容的类似于MIME的消息结构。
服务器以一个状态行作为响应，相应的内容包括消息协议的版本，成功或者错误编码加上包含服务器信息、实体元信息以 及可能的实体内容。

通常HTTP消息由一个起始行，一个或者多个头域，一个只是头域结束的空行和可选的消息体组成。
HTTP的头域包括通用头，请求头，响应头和实体头四个部分。每个头域由一个域名，冒号（:）和域值三部分组成。域名是大小写无关的，域值前可以添加任何数量的空格符，头域可以被扩展为多行，在每行开始处，使用至少一个空格或制表符。

请求消息和响应消息都可以包含实体信息，实体信息一般由实体头域和实体组成。实体头域包含关于实体的原信息，实体头包括Allow、Content- Base、Content-Encoding、Content-Language、 Content-Length、Content-Location、Content-MD5、Content-Range、Content-Type、 Etag、Expires、Last-Modified、extension-header。

Content-Type是返回消息中非常重要的内容，表示后面的文档属于什么MIME类型。

Content-Type: [type]/[subtype]; parameter。例如最常见的就是text/html，它的意思是说返回的内容是文本类型，这个文本又是HTML格式的。原则上浏览器会根据Content-Type来决定如何显示返回的消息体内容。

在请求中，Content-Type 是实体头部用于指示资源的MIME类型 media type，服务端根据这个类型来做不同的处理。

在响应中，Content-Type标头告诉客户端实际返回的内容的内容类型，浏览器根据Content-Type来对文件做不同的处理。
比如对请求 google.com 返回的content-type: text/html，charset=uft8，浏览器会把文本作为html进行解析，最终渲染到页面中。
![response_content-type1](https://tva1.sinaimg.cn/large/008i3skNgy1gqralnkznwj31b20okn1z.jpg)

响应头中的 Content-Type 类型与请求头中的 Content-Type 并没有太多区别，但是有一些content-type只能在响应头中设置。

| Content-Type | 说明 | 案例 | 备注 |
| --- | --- | --- | --- |
| 请求头中的content-type | 描述请求实体对应的MIME信息 | Content-Type: application/json | 简单理解就是：客户端告诉服务端，我传的数据是什么类型 |
| 响应头中的content-type | 描述响应实体对应的MIME信息 | text/html; charset=UTF8 | 简单理解就是：服务端告诉客户端，我返回的数据是什么类型 |

# MIME

## MIME类型

媒体类型（通常称为 Multipurpose Internet Mail Extensions 或 MIME 类型 ）是一种标准，用来表示文档、文件或字节流的性质和格式。它在IETF RFC 6838中进行了定义和标准化。

互联网号码分配机构（IANA）是负责跟踪所有官方MIME类型的官方机构，您可以在媒体类型页面中找到最新的完整列表。(在文章的最后)

## MIME结构

MIME格式：[type]/[subType]；
type是指主类型；subType为子类型；

例如：text/html text/css image/jpeg image/png application/json 等等

每种MIME类型有独立的配置，我们称之为-可选参数（Optional parameters），比如text/html 可以配置 charset=UTF8
![request_content-type1](https://tva1.sinaimg.cn/large/008i3skNgy1gqraxpdgerj30s60fsmz1.jpg)

MIME类型对大小写不敏感，但是传统写法都是小写。

## MIME有哪些类型

目前已知的MIME中有以下已注册的类型

* application
* audio
* font
* example
* image
* message
* model
* multipart
* text
* video

### 基本的独立类型

| 类型 | 描述 | 典型案例 |
| --- | --- | --- |
| text | 表明文件是普通文本，理论上是人类可读 | text/plain, text/html, text/css, text/javascript |
| image | 表明是某种图像。不包括视频，但是动态图（比如动态gif）也使用image类型 | image/gif, image/png, image/jpeg, image/bmp, image/webp, image/x-icon, image/vnd.microsoft.icon |
| audio | 表明是某种音频文件 | audio/midi, audio/mpeg, audio/webm, audio/ogg, audio/wav |
| video | 表明是某种视频文件 | video/webm, video/ogg |
| application | 表明是某种二进制数据 | application/octet-stream, application/pkcs12, application/vnd.mspowerpoint, application/xhtml+xml, application/xml,  application/pdf |

对于text文件类型若没有特定的subtype，就使用 text/plain。类似的，二进制文件没有特定或已知的 subtype，即使用 application/octet-stream。

application/octet-stream通常与标识二进制文件，常见的场景是：上传文件；
服务端在处理这种类型的MIME时通常就是把数据流写入文件。

## document.contentType

浏览器在请求资源成功后，会把当前页面对应链接的content-type挂载到document.contentType中

比如：

1、请求一个网站后，访问document.contentType 可以得到 text/html
2、请求一个css文件，访问document.contentType 可以得到 text/css
3、请求一个js文件，访问document.contentType 可以得到 application/javascript

## MIME没有设置或错误设置的情况

在缺失 MIME 类型或客户端认为文件设置了错误的 MIME 类型时，浏览器可能会通过查看资源来进行MIME嗅探。每一个浏览器在不同的情况下会执行不同的操作。因为这个操作会有一些安全问题，有的 MIME 类型表示可执行内容而有些是不可执行内容。浏览器可以通过请求头 Content-Type 来设置 X-Content-Type-Options 以阻止MIME嗅探。

很多服务器使用默认的 application/octet-stream 来发送未知的二进制数据。出于一些安全原因，对于这些资源浏览器不允许设置一些自定义默认操作，导致用户必须存储到本地以使用。
比如说：

请求了一个图片文件，希望可以通过浏览器打开查看文件，但是实际因为content-type是application/octet-stream，最后只会弹出一个下载框。

```javascript
const http = require("http");
const fs = require("fs");
const path = require("path");

http
  .createServer(function (request, response) {
    response.writeHead(200, { "Content-Type": "application/octet-stream" });
    var stream = fs.createReadStream(
      path.resolve(process.cwd(), "public/a.png")
    );
    var responseData = [];
    if (stream) {
      stream.on("data", function (chunk) {
        responseData.push(chunk);
      });
      stream.on("end", function () {
        var finalData = Buffer.concat(responseData);
        response.write(finalData);
        response.end();
      });
    }
  })
  .listen(8099, () => {
    console.log("startup in http://127.0.0.1:8099");
  });
```

在浏览器中访问 http://127.0.0.1:8099/public/a.png，只会弹出一个下载框

![content-type6](https://tva1.sinaimg.cn/large/008i3skNgy1gqria1zt67j30v80u0gvt.jpg)

如果返回了错误的MIME类型，
比如说：

音视频播放的资源如果返回了错误的MIME类型，即使使用了正确的video和audio标签也无法播放。

正常的情况：正确返回 Content-Type，可以正常播放音频
```javascript
const mp3 = path.resolve(process.cwd(), "public/One Last Kiss.mp3");
const stat = fs.statSync(mp3);

response.writeHead(200, {
    "Content-Type": "audio/mpeg",
    "Content-Length": stat.size,
});

//创建可读流
const readableStream = fs.createReadStream(mp3);
// 管道pipe流入
readableStream.pipe(response);
```
![](https://tva1.sinaimg.cn/large/008i3skNgy1gqrirlqbudj31fk0ew0vg.jpg)
![](https://tva1.sinaimg.cn/large/008i3skNgy1gqris8feioj30xa0k8wgx.jpg)
![](https://tva1.sinaimg.cn/large/008i3skNgy1gqrivvx00nj31c80u043z.jpg)

错误的情况：无法播放

```javascript
const mp3 = path.resolve(process.cwd(), "public/One Last Kiss.mp3");
const stat = fs.statSync(mp3);

response.writeHead(200, {
    // 这里返回了图片的 MIME
    "Content-Type": "image/png",
    "Content-Length": stat.size,
});

//创建可读流
const readableStream = fs.createReadStream(mp3);
// 管道pipe流入
readableStream.pipe(response);
```

![](https://tva1.sinaimg.cn/large/008i3skNgy1gqrix9t1qfj31ab0u043a.jpg)


# 常见的 Content-Type 讲解

## application/x-www-form-urlencoded

用于POST请求提交数据的格式之一。

常用于FORM表单提交，如果不设置enctype属性,默认为application/x-www-form-urlencoded方式提交数据。

x-www-form-urlencoded有这几个特性：

* Content-Type都指定为application/x-www-form-urlencoded;
* 提交的表单数据会转换为键值对并按照key1=val&key2=val2的方式进行编码,key和val都进行了URL转码。大部分服务端语言都对这种方式有很好的支持。

另外,如利用AJAX提交数据时,也可使用这种方式。例如在jQuery中,Content-Type默认值都是"application/x-www-form-urlencoded;charset=utf-8"。

## application/json

用于POST请求提交数据的格式之一。

application/json在响应头和请求头中都很常见，消息主体是序列化后的JSON字符串。

```javascript
fetch('url', {
    method: 'POST',
    body: JSON.stringify({
        xxx: 'xxx',
    }),
    header: {
        'Content-Type': 'application/json',
    },
});
```

## text/plain

如果没有指定响应头中的content-type，那么content-type默认就是text/plain，纯文本，浏览器中会直接以文本形式展示出来。

## multipart类型

multipart中有的重要类型：
* multipart/form-data
* multipart/byteranges

### multipart/form-data

这个是老生常谈的话题，multipart/form-data通常用在需要文件上传的场景：

* HTML FORM：表单提交很常见，通常在需要上传文件时，会把FORM的enctype设置为multipart/form-data
* AjAX POST：提交异步POST请求时配合 formData 使用

multipart/form-data作为多部分文档格式，主要构成分为这几部分：
1、边界线
2、内容区域，包含content-disposition和content-type，content-type出现在文件上传的时候

```html
<form action="http://localhost:8000/" method="post" enctype="multipart/form-data">
  <input type="text" name="myTextField">
  <input type="checkbox" name="myCheckBox">Check</input>
  <input type="file" name="myFile">
  <button>Send the file</button>
</form>
```

```http
POST / HTTP/1.1
Host: localhost:8000
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.9; rv:50.0) Gecko/20100101 Firefox/50.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Content-Type: multipart/form-data; boundary=---------------------------8721656041911415653955004498
Content-Length: 465

-----------------------------8721656041911415653955004498
Content-Disposition: form-data; name="myTextField"

Test
-----------------------------8721656041911415653955004498
Content-Disposition: form-data; name="myCheckBox"

on
-----------------------------8721656041911415653955004498
Content-Disposition: form-data; name="myFile"; filename="test.txt" // 描述内容的基础配置
Content-Type: text/plain // 描述了上传的文件类型

Simple file.
-----------------------------8721656041911415653955004498--

```

### multipart/byteranges

这个类型通常用来标明当前的数据是分片传输的，分片传输听起来可能有点模糊，我们来举个🌰：
大家应该都看过电影，一步高清的电影动辄几十GB的大小很难在短时间迅速下完，但是如果中途中断下次再下载时会不会重头开始呢？
网络中常用的下载工具肯定都提供了断点续传的功能，multipart/byteranges就是其中的关键。

对于这个content-type，图解HTTP中是这样解释的：

> multipart/byteranges：状态码 206（Partial Content，部分内容）响应报文包含了多个范围的内容时使用
   HTTP/1.1 206 Partial Content   Date: Fri, 13 Jul 2012 02:45:26v GMT   Last-Modified: Fri, 31 Aug 2007 02:02:20 GMT   Content-Type: multipart/byteranges; boundary=THIS_STRING_SEPARATES      --THIS_STRING_SEPARATES   Content-Type: application/pdf   Content-Range: bytes 500-999/8000      ...(范围指定的数据)...      --THIS_STRING_SEPARATES   Content-Type: application/pdf   Content-Range: bytes 7000-7999/8000      ...(范围指定的数据)...   --THIS_STRING_SEPARATES--
   使用 boundary 字符串来划分 multipart 指明的各类实体，在其指定的各个实体的起始行之前插入『—』标记（如 --AaB03x），在对应的字符串的最后插入『--』标记作为结束（如 --AaB03x—）。
   
看起来很疑惑，是的，因为只有这段说明是解释不清楚的。

因为响应头中的content-type: multipart/byteranges需要配合请求头的Range请求使用，那么Range是啥？我们看看图解HTTP的解释：

> 范围请求（Range Request）：指定范围发送的请求。执行范围请求时，用到首部字段 Range 来指定资源的 byte 范围：
5001~10000 字节
   Range: bytes=5001-10000
从 5001 字节之后全部的
   Range: bytes=5001-
从一开始到 3000 字节和 5000~7000 字节的多重范围
   Range: bytes=-3000, 5000-7000
说明：针对范围请求，响应返回 206 Partial Content 的响应报文。对于多重范围的范围请求，响应会在首部字段 Content-Type 标明 multipart/byteranges 后 返回响应报文。如果服务器无法响应范围请求，则会返回 200 OK。

结合Range Request和multipart/bytesranges的说明可以这样理解：请求可以要求服务端只返回数据中的某一段，若服务器能响应`Range`，则把请求头置位`206`并把Content-Type置为multipart/byteranges。如果不支持`Range`，则直接返回整个数据并置状态码为`200`。

我们可以动手试试：

```javascript
// 请求一个图片，要求返回31个字节，从0开始，30结束，[0,30]
fetch('https://developer.mozilla.org/static/media/search.db31d27c.svg', {
    headers: {
        Range: 'bytes=0-30'
    }
}).then(async (res) => {
    const svg = await res.text();
    console.log(svg, svg.length)
})
```
![content-type3](https://tva1.sinaimg.cn/large/008i3skNgy1gqrdgs01r9j31040aa75w.jpg)

嗯，看起来没毛病，但是multipart/byteranges似乎没看出？只是多了content-range，content-length也发生了变化
![content-type4](https://tva1.sinaimg.cn/large/008i3skNgy1gqrdvogdckj30yi0u0q8j.jpg)

重新看看图解http那段说明，Range是支持多段区间的
那我们重新试试

```javascript
// 请求两个区间的数据 [0,30] [90,120]
fetch('https://developer.mozilla.org/static/media/search.db31d27c.svg', {
    headers: {
        Range: 'bytes=0-30, 90-120'
    }
}).then(async (res) => {
    const svg = await res.text();
    console.log(svg, svg.length)
})
```
多个区间是，可以看到相应body通过boundary区分开来了
![content-type5](https://tva1.sinaimg.cn/large/008i3skNgy1gqrdxfyj8gj318m0k6wih.jpg)

请求头中的MIME-TYPE也不再是原来的基础类型
![content-type6](https://tva1.sinaimg.cn/large/008i3skNgy1gqrdzhgldvj314s0nc42r.jpg)


## 参考

> * [Content-Type](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Type)
> * [MIME Type](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Basics_of_HTTP/MIME_types)
> * [mime-types 速查](https://www.iana.org/assignments/media-types/media-types.xhtml)
> * [HTTP请求范围](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Range_requests)
> * [Http请求中的Content-Type](https://segmentfault.com/a/1190000013056786)