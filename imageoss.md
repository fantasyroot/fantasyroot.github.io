---
title: 前端图片压缩与上传 OSS 组件实现
layout: default
date: 2017-01-03
tag: oss
---

# 前端图片压缩与上传 OSS 组件实现

在日常的前端需求中，对文件的处理可能不常用到会比较陌生。例如在浏览器端计算文件的 MD5、压缩图片……
还有这种操作？？没错，当我们要写一个功能完备的 OSS 上传 组件时，可能都会涉及到。
一睹为快：[@alife/react-oss-upload](http://web.npm.alibaba-inc.com/package/@alife/react-oss-upload)、[react-oss-upload](https://www.npmjs.com/package/react-oss-upload)、[@alife/qn-upload](http://web.npm.alibaba-inc.com/package/@alife/qn-upload)、[示例 demo](http://groups.demo.taobao.net/alife/react-oss-upload/demo/index.html)

## 前端需求

1. 实现一个 React 上传组件，文件上传到阿里云 OSS 私人空间（上传后的链接有时效性，防止客户的隐私图片泄漏）；
2. 可限制上传文件大小、个数，可配置，实时显示上传进度；
3. 为节省如果上传图片超过了限制的大小，要在上传之前 **压缩图片**。如果压缩后还是超过限制大小，报错提示；
4. 支持在 input 里 Ctrl + V 直接上传剪贴板截图；
5. 如果文件已经在 OSS 中存在，实现 **秒传**（参考网盘的秒传功能）；

## 整体流程与技术方案

大多数上传方案一般是应用服务端给前端开一个上传接口，文件流可能是：
![](https://docs-aliyun.cn-hangzhou.oss.aliyun-inc.com/internal/oss/0.0.4/assets/image/practice-post-callback-1.png)

但这种方案有三个缺点：

1. 上传慢。先上传到应用服务器，再上传到 OSS，网络传送多了一倍。如果数据直传到 OSS，不走应用服务器，速度将大大提升，而且 OSS 是采用 BGP 带宽，能保证各地各运营商的速度；
2. 扩展性不好。如果后续用户多了，应用服务器会成为瓶颈；
3. 费用高。由于 OSS 上传流量是免费的。如果数据直传到 OSS，不走应用服务器，可以省下几台应用服务器；

所以，既然文件是存到 OSS，其实文件流没必要经过应用服务器， **前端直接传到 OSS** 即可。这既解决了以上问题，节省了不必要的网络流量，又能将集中式变成分布式，为应用服务器分担压力。

但这也带来了几个问题：

1. 实现秒传，基本原理实际就是计算本地文件的 MD5，如果远程 OSS 已存在相同的文件（MD5 比对），就不用再上传一遍了。前端如何计算文件 MD5 呢？
2. 前端如何实现图片的压缩？
3. 安全如何保证？

首先解决安全问题。如果采用客户端直接签名有一个很严重的安全隐患，OSS AccessId/AccessKey 暴露在前端页面，其他人可以随意拿到 AccessId/AccessKey 然后上传文件，这是非常不安全的做法。
解决方案：这里采用基于 session 的[服务端签名后直传](https://help.aliyun.com/document_detail/31926.html?spm=5176.doc31923.2.1.QNKmZa)的方案：

![](https://docs-aliyun.cn-hangzhou.oss.aliyun-inc.com/internal/oss/0.0.4/assets/image/practice-post-callback-5.png)

剩下的两个问题：前端计算 MD5、实现图片压缩，下面我们逐个搞定。
最终的整体流程时序图如下（省略了服务端与 OSS 交互细节）：
![上传流程](https://img.alicdn.com/tfs/TB1ls3fQXXXXXbgXXXXXXXXXXXX-1510-1146.png)

## 前端计算 MD5

### spark-md5

这个可以帮到我们：[spark-md5](https://www.npmjs.org/package/spark-md5)。要实现在浏览器端计算 MD5，需要浏览器支持 [FileAPI](https://developer.mozilla.org/zh-CN/docs/Web/API/FileReader) 才可以(具体兼容性可以看[这里](http://caniuse.com/#feat=fileapi))。
实现原理：

-   利用 FileReader 的 [readAsArrayBuffer( )](https://developer.mozilla.org/zh-CN/docs/Web/API/FileReader/readAsArrayBuffer) 方法，将 `file.slice()` 分片后的数据读到一个数组中（每个数组项代表一个字节），切分成更小的二进制数据块(chunk)，逐块处理。MD5 算法支持流式计算，读一块算一块，最后再一次性生成完整 hash。因此，即使读取大文件，内存也占用很少，性能优异。
-   [MD5 算法](https://zh.wikipedia.org/wiki/MD5)的基本原理是将任意长度字符串，通过使用函数转换为新的字符串，并且这种运算不可逆转。它以 16 个 32 位子分组即 512 位分组来提供数据杂凑，经过一系列处理，生成 4 个 32 位分组，最后联合起来成为一个 128 位散列。一般 128 位二进制的 MD5 散列被表示为 **32 位十六进制数字**，这就是我们通常看到的计算结果。

具体可以看[spark-md5 源码实现](https://github.com/satazor/js-spark-md5/blob/master/spark-md5.js)，以及[我的使用方式](http://gitlab.alibaba-inc.com/alife/react-oss-upload/blob/master/src/getFileMD5.js)。官方还提供了一个[在线 demo](http://9px.ir/demo/incremental-md5.html)。

### 性能

你一定想知道，在浏览器里读文件流计算 MD5，性能好吗？如果读大文件，浏览器会不会卡死崩溃？这也是我之前担心的，如果浏览器为此而 crash，那体验太差了。

md5 算法有很多种实现。spark-md5 是基于 Joseph's Myers 的 [JKM md5](http://www.myersdaily.org/joseph/javascript/md5-text.html) 实现，这也是目前最快的实现。[性能对比](https://jsperf.com/md5-shootout/63)。
经过在不同环境中简单测试，可以放心了。结果如下：

-   虚拟机千牛环境（低配）：Windows 10；cpu：i7，2.2GHz，4 核；内存：2G；浏览器：Chrome/44.0.2403.155 Aef/3.16
-   宿主机环境（高配）：MacOS 10.12.5；cpu：i7，2.2GHz；内存：16G；浏览器：Chrome/59.0.3071.115

| 文件大小 | 虚拟机计算 MD5 耗时（ms） | MacOS 计算 MD5 耗时（ms） |
| -------- | ------------------------- | ------------------------- |
| 291K     | 14                        | 8                         |
| 4.1M     | 168                       | 88                        |
| 27.5M    | 932                       | 542                       |
| 52.6M    | 1792                      | 976                       |
| 82.6M    | 2621                      | 1473                      |
| 880.7M   | 26740                     | 15501                     |

-   在浏览器异步计算 MD5 的过程中，仍可以进行其他操作，浏览器不会卡死和 block；
-   即使电脑是较低的配置，只要文件不大于 100M，MD5 计算时间一般也不超过 3 秒。

所以看业务场景，如果经常有大于 100M 文件的场景，为了良好体验，建议给用户一个 loading 或提示。

## 前端压缩图片

> 传统的方案一般是前端原样上传文件，由应用服务器来做图片压缩处理。但现在图片大小动辄就有几 MB，有些场景下为了节省流量和存储空间，更好的做法是在上传之前就进行一次压缩。有了 canvas 之后，前端对图片的处理也能游刃有余。

### 原理

前端压缩图片的原理：

1. 利用 [Canvas 2D Context](https://developer.mozilla.org/zh-CN/docs/Web/API/CanvasRenderingContext2D) 的 [drawImage( )](https://developer.mozilla.org/zh-CN/docs/Web/API/CanvasRenderingContext2D/drawImage) 方法传入原始图片，绘制新的图片，设置绘制的图片的 width、height;
2. 然后通过 canvas 的 [toDataURL( )](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLCanvasElement/toDataURL) 方法，设置压缩比率，生成压缩后的 [dataURI](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/data_URIs) ( MIME + base64 字符串)；
3. 最后将 dataURI 转换成 [Blob](https://developer.mozilla.org/zh-CN/docs/Web/API/Blob) (二进制 jpeg 文件)。

部分核心代码：

```js
function html5ImgCompress(file) {
    let cvs = document.createElement('canvas');
    let ctx = cvs.getContext('2d');
    let img = new Image();
    let fileURL = window.URL.createObjectURL(file);
    img.src = fileURL;
    img.onload = () => {
        cvs.width = img.width;
        cvs.height = img.height;
        ctx.drawImage(img, 0, 0, cvs.width, cvs.height); // 绘制新图片
        dataURI = cvs.toDataURL('image/jpeg', 0.6); // 设置压缩比率
        window.URL.revokeObjectURL(img.src); // 为了性能最好主动释放
    };
    let newImg = convertToBinary(dataURI, { type: 'image/jpeg' });
    resolve(newImg);
}
```

代码说明：

-   [window.URL.createObjectURL( )](https://developer.mozilla.org/zh-CN/docs/Web/API/URL/createObjectURL) 方法，接收一个 `Blob` 或 `File` 对象，创建一个 [DOMString](https://developer.mozilla.org/zh-CN/docs/Web/API/DOMString)。这个方法经常用来[在页面上预览本地图片](https://developer.mozilla.org/en-US/docs/Using_files_from_web_applications#Example.3A_Using_object_URLs_to_display_images)。
-   canvas 的 `toDataUrl()` 方法可以将内容导出为 base64 编码格式的图片，此方法可以指定压缩比率（默认 0.92），但是用 base64 编码后，将比源文件大 33.3%。为什么 base64 编码后数据量会变大：

    -   Base64 编码的思想是是采用 64 个基本的 ASCII 码字符对数据进行重新编码。它将需要编码的数据拆分成字节数组。以 3 个字节为一组。按顺序排列 24 位数据，再把这 24 位数据分成 4 组，即每组 6 位。再在每组的的最高位前补两个 0 凑足一个字节。这样就把一个 3 字节为一组的数据重新编码成了 4 个字节。当所要编码的数据的字节数不是 3 的整倍数，也就是说在分组时最后一组不够 3 个字节。这时在最后一组填充 1 到 2 个 0 字节。并在最后编码完成后在结尾添加 1 到 2 个"="。（ 注 BASE64 字符表：ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/）
    -   从以上编码规则可以得知，通过 Base64 编码，原来的 3 个字节编码后将成为 4 个字节，即字节增加了 33.3%，数据量相应变大。所以 20M 的数据通过 Base64 编码后大小大概为 20M \* 133.3% = 26.67M。

-   Blob 对象简介：Blob 对象可以看做是存放二进制数据的容器，允许我们可以通过 js 直接操作二进制数据。一个 Blob 对象就是一个包含有只读原始数据的类文件对象。File 接口基于 Blob，继承了 Blob 的功能，并且扩展支持了用户计算机上的本地文件。
-   最后一步 `convertToBinary()` 方法的实现可以看[这里](http://gitlab.alibaba-inc.com/alife/react-oss-upload/blob/master/src/imgCompressor.js#L14)。

### 压缩效果

明白基于 canvas 压缩图片原理之后，有没有已经实现了的压缩工具呢？还真有：[html5ImgCompress](https://github.com/mhbseal/html5ImgCompress)（[demo](http://mhbseal.com/demo/html5/html5ImgCompress/demo/index.html))。
有一点遗憾的是作者没有发 npm 包，但是提供了几个 js bundle，可以作为项目 vendor 资源使用。

如果想看到不同压缩比率下的效果，我也做了一个简单的 demo，可以输入不同的质量参数，在线看到压缩后的效果：[canvas-compressor demo](http://andong.alidemo.cn/canvas-compressor/)
![](https://img.alicdn.com/tfs/TB151sqSpXXXXcZXpXXXXXXXXXX-2480-1108.jpg)

对一个大小 4.5MB，尺寸 2387 × 3264 的原图进行压缩，如果限制宽度最大 1000px，不同压缩比率下，生成文件大小对比如下：

| 压缩比率   | 文件大小(KB) |
| ---------- | ------------ |
| 0.1        | 22           |
| 0.5        | 63           |
| 0.8        | 117          |
| 0.92(默认) | 203          |
| 0.95       | 273          |
| 1          | 1029         |

## 上传功能

-   上传功能基于 [plupload](http://www.plupload.com/) 封装。参考[阿里云 OSS web 直传 demo](http://oss-demo.aliyuncs.com/oss-h5-upload-js-php/index.html)。plupload 可以帮我们做分片上传、文件过滤、队列管理、防止重复上传、异常报错等事情。
-   在 input 里 Ctrl + V 直接上传剪贴板截图：给 input 绑一个 [paste](http://gitlab.alibaba-inc.com/alife/react-oss-upload/blob/master/src/service.js#L92) 事件即可。
-   注意一点，阿里云 OSS 的 bucket 必须设置 Cors(Post 打勾），不然不能上传。

## UI 和功能分离

初期刚完成这个上传组件的时候，UI 和功能代码耦合度比较高，复用性很差。后来对 UI 和功能进行了分离，使业务和功能尽量解耦。

所以现在提供两个组件：

1. [@alife/react-oss-upload](http://web.npm.alibaba-inc.com/package/@alife/react-oss-upload)：抽出底层的计算 MD5、图片压缩、上传等功能，可以看做是一个 SDK，没有 UI 样式。
2. [@alife/qn-upload](http://web.npm.alibaba-inc.com/package/@alife/qn-upload)：基于上面组件进行的业务封装。增加了千牛业务的 UI，指定了业务接口地址。

其他业务如果想用这个组件，可以参考这个实现，在 `@alife/react-oss-upload` 这个 SDK 层之上，加入自己想要的 UI，让服务端遵循里面的接口字段约定，很快就能封装出来。

## 后续完善

-   其实上传 SDK 层还可以再做拆解。现在这种方案仍对接口的返回格式有要求，后续考虑能更灵活一点，加一个接口适配层，允许开发者自定义配置。
-   对比业界类似的上传组件，看还有哪些欠缺。例如百度的 [Web Uploader](http://fex.baidu.com/webuploader/)。

## 参考资料

-   [File API](https://developer.mozilla.org/zh-CN/docs/Web/API/FileReader)
-   [MD5 算法](https://zh.wikipedia.org/wiki/MD5)
-   [JKM md5](http://www.myersdaily.org/joseph/javascript/md5-text.html)
-   [Base64](https://zh.wikipedia.org/wiki/Base64)
-   [Base64 编码为什么会使数据量变大？](http://blog.csdn.net/sallay/article/details/3550740)
-   [阿里云 - Web 端直传实践简介](https://help.aliyun.com/document_detail/31923.html)
-   [阿里云 - 服务端签名后直传](https://help.aliyun.com/document_detail/31926.html?spm=5176.doc31923.2.1.QNKmZa)
