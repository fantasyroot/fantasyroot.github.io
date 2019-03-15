# 基于 Egg 的同构应用解决方案

## 前言

云音乐目前有大量 webview 页面采用同构直出的方案做服务端渲染，其中营收团队 To C 页面绝大多数都是同构输出的。包括会员、数字专辑、演出票务、订单，还有一些内部平台等。
<img src="https://p1.music.126.net/KbjAA7D6xL6Yovlkze9KDA==/109951163753198355.png" width="100%" />

本文将介绍一下同构原理，我们的技术方案、以及在我们大量业务实践中的经验分享。

<br/>

## 同构应用简介

### 什么是同构

“同构”（Isomorphic）概念是 2013 年 Airbnb 的工程师 Spike Brehm 的一篇文章 [《Isomorphic JavaScript: The Future of Web Apps》](https://medium.com/airbnb-engineering/isomorphic-javascript-the-future-of-web-apps-10882b7a2ebc) 火起来的。文章里阐述的同构：“…Isomorphic JavaScript apps, which are JavaScript applications that can run both on the client-side and the server-side.”，即一套 js 代码同时运行在客户端和服务端。
和传统方案不同的是，我们不再需要 Java freemarker/velocity 模板来渲染页面主体结构，而是由 Node.js 来完成这一步。

<img src="https://p1.music.126.net/o3lvGbZ0gLIBRHucGI4V5g==/109951163753232235.png" width="100%" />

### 同构的好处

#### 1. 减少开发维护成本

前后端模板统一，仅需一套 js 代码。

#### 2. 首屏性能提升

用户能够更快的看到实际页面。主要源于以下三个方面：

-   同构带来了可插拔的 SSR 能力，Server 端直出最终页面
-   Server 端代理了接口请求
-   Client 端所需资源提前已知，可以 [preload](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Preloading_content) 预加载

<img src="https://p1.music.126.net/wTIDGzGwn1tJ8FE-xczrIg==/109951163753229938.png" width="100%" />

#### 3. SEO 友好

-   服务端直接渲染出最终完整的 HTML，方便搜索引擎抓取页面数据。

<br/>

## 同构的原理

一套 js 代码要同时跑在 Node 端和浏览器端，而 Node 环境和浏览器环境有一些差别。同构的原理，更多的是要解决两端兼容的问题。

### 1. View 渲染要一致

View 需要选择对 DOM 做过抽象化的模板(virtual dom)，比如 React。在 Node 端渲染出 DOM 结构。

### 2. Ajax 与 Cookie 处理

Node 端没有自带的请求工具，可以使用 isomorphic-fetch 或 axios。由于需要代理 API 请求，所以需要把用户浏览器带过来的 UA 和 cookie 一并带上。

### 3. 全局变量兼容

Node 全局变量是 global，没有 window、document、File 等全局变量。如果依赖了第三方组件里调用了 document 方法，在 Node 端会报错。

一个可行的解决方案是在 Node 端引入 jsdom，把 jsdom 中的全局变量挂到 global 上。

Note: 实际业务中发现，很多三方组件判断当前是否是 Node 端，是简单用 `typeof window !== 'undefined'` 来判断的。如果 Node 端也有了 window，要注意是否违背了组件的初衷，可能并不想在 Node 端执行某些方法。

### 4. ES6 语法兼容

Client 端有 Webpack、Babel 帮我们处理。Node 端可以引入 [@babel/register](https://babeljs.io/docs/en/babel-register) 解决。

### 5. alias 兼容

例如 `require("$client/foo")` 需要解析成 `require("/cwd/client/foo")`, require 路径里包含我们自定义变量，在 Node 端如何做自定义解析？

一个可行的方案是重写 [Module.\_resolveFilename](https://github.com/nodejs/node/blob/master/lib/internal/modules/cjs/loader.js#L569) 替换 request string 来解决。

### 6. require 资源文件处理

这个是重点，也是比较棘手的地方。在 Node 端直接 require 图片、字体或 CSS 等非 js 文件是会报错的。
查阅部分 [Node 源码](https://github.com/nodejs/node/blob/master/lib/internal/modules/cjs/loader.js#L224) 可以发现，如果没给静态资源扩展名 ext 定义 `Module._extensions[ext]`，Node 默认全部当 js 文件处理。那么图片类的文件自然会报错了。

<img src="https://p1.music.126.net/eey7D3zMIhvlQ5CrH0B9_A==/109951163753393190.png" width="100%" />

一个可行的方案是为 [`Module._extensions`](https://nodejs.org/api/modules.html#modules_require_extensions) 添加 ext 处理方法。上图最右边是 `.json` 文件在 Node 中的处理方法，可供参考。

业界大部分同构工具，例如 [webpack-isomorphic-tools](https://github.com/catamphetamine/webpack-isomorphic-tools)、[asset-require-hook](https://github.com/aribouius/asset-require-hook) 也都是这个原理。
PS: 事实上，了解过 `@bable/register` 原理的人可能知道，[`@bable/register`](https://github.com/babel/babel/blob/master/packages/babel-register/src/node.js#L7) 内部也是这么做的，它主要功能就是 require hook，依赖的 [`pirates`](https://github.com/ariporad/pirates/blob/master/src/index.js#L64) 里，核心代码也是 `Module._extensions`。

<img src="https://p1.music.126.net/olDSF_uTEfJUfPSdqJ6hew==/109951163753421556.png" width="100%" />

关于 require 的原理，知乎上也有篇文章总结不错：[《浅析当下的 Node.js CommonJS 模块系统》](https://zhuanlan.zhihu.com/p/38382637)。

<br/>

## 为什么基于 Egg 做同构框架

Egg 的设计原则和 Koa 类似，“微内核+插件”，灵活可扩展，约定优于配置。而且 Egg 中 [“渐进式开发”](https://eggjs.org/zh-cn/tutorials/progressive.html) 哲学其实和 [Unix 哲学](https://en.wikipedia.org/wiki/Unix_philosophy) 很像，贴合现实场景下的团队协作要求。和 Koa 的关系大致可以理解为：
`Egg = Koa + 规范 + 扩展和插件机制 + 进程模型`

### Egg 插件机制

Egg 的 [插件机制](https://eggjs.org/zh-cn/basics/plugin.html) 是我们必须选择它的原因之一，你可以将一个可复用的功能封装成一个插件，不必侵入核心代码，方便团队协作。
一个插件里可以包含多个 Middleware、环境配置、扩展等。
插件兼容 Koa 中间件生态，所以原 Koa 应用可以很轻松地迁移到 Egg 应用。

### Egg 进程模型

传统 Node 应用的进程模型一般是简单的 Master-Woker 模型，其中 Woker 数量一般取决于 CPU 核数。 但是实际业务中，难免会出现公共资源访问（比如读写文件或 DB）、日志切割、后端长连接等需求，这类需求放在每个 woker 里面去做一遍是不合适的。

<img src="https://p1.music.126.net/oZqYVzjmoYxTr9gitmvBiw==/109951163753277781.png" width="400px" />

在 Egg 进程模型中，增加了 [Agent 进程](https://eggjs.org/zh-cn/core/cluster-and-ipc.html#agent-%E6%9C%BA%E5%88%B6)。和 Woker 不同的是，每个应用只会有一个 Agent 进程。公共事务处理类的需求，我们可以交给 Agent 来做。Agent 和 Worker 之间可以很方便地进程间通讯（IPC）。

### Egg 上层框架支持

Egg 提供了[框架扩展](https://eggjs.org/zh-cn/basics/extend.html)支持，在 Egg 之上，加上需要的插件，可以封装成一个适合团队业务场景的框架。框架是可以无限级继承的，可以基于 Egg 封装企业框架，上层再封装部门框架...
框架基于 NPM 发布，有版本控制，易于升级和维护。

除此之外，我们还可以利用 Egg 提供的单元测试方案、多环境配置、定时任务方案、还有大量社区工等，帮助我们高效地开发高健壮性的 Node 应用。

<br/>

## 基于 Egg 的同构技术方案

了解了上述同构原理、要解决的问题、基于 Egg 做框架的好处之后，我们的技术方案也基本确定下来。
业界也已有基于 Egg 做的框架，例如 [alibaba/beidou](https://github.com/alibaba/beidou)。基于北斗有一些问题，比如不支持多目录自动路由和打包、默认配置和我们业务不符，考虑到稳定性等方面，我们 fork from 北斗，对功能、配置、模板进行改进，产出了我们的框架：
**Bass**

### Bass 整体架构

<img src="https://p1.music.126.net/uRU5UoqgoyneJaIzJsRjLQ==/109951163753450892.png" width="600px" />

底层基于 Egg，将各个功能封装成 Egg 插件：

-   Bass-view: 页面渲染
-   Bass-router: 根据目录结构和名称自动路由
-   Bass-music: 封装云音乐公共的 middleware (例如健康检查)、组件(例如 env)、启动容器工具
-   Bass-isomorphic: Node 端同构兼容处理
-   Bass-webpack: Webpack 构建打包
-   Bass-sentry: 监控异常报错，自动上报 sentry

同时，Bass 提供一套脚手架和示例模板，可以快速生成业务工程。还提供工程化 CLI 工具，服务 `dev`（启动开发环境）、`build`（一键构建打包）、`start`（部署上线）整个开发周期。 业务工程里如果有可复用的功能模块，可以插件的形式，下沉到框架。

### 静态资源处理

静态资源的处理，其实围绕着一个核心思路：
如何实现 **`require(path) = (path) => assets[path]`**

<img src="https://p1.music.126.net/pthNxq_YsPXVsnz4eeyUlg==/109951163747526631.png" width="600px" />

基本工作流程：

1. 利用 Agent 只有一个进程的机制，本地在 Agent 里读取 webpack 配置（包括配置 合并）。同时用 `chokidar` 监听目录结构变动让 webpack 重新编译
2. 得到 Webpack 最终配置，运行 webpack-dev-server
3. webpack-dev-server 进行 compile
4. 构建完成，IsomorphicPlugin (webpack 插件) 生成 `assets.json` 文件，并把 webpack-dev-server 运行端口号以消息的方式告知 woker
5. 开发者访问页面，Node 端执行 Render 流程。如果有 require 图片等资源，将读取 `assets.json` 里对应的 value，渲染到 HTML
6. 浏览器请求图片资源，到 Node 端 woker 处理
7. woker 中的 middleware 拦截请求，替换为 webpack 的端口号，尝试内部访问资源。如果返回状态 200，则命中资源，返回结果，否则交给下一个 middleware 继续处理

<br/>

## React 同构应用实践

### React Fiber

React 16 采用了 Fiber 架构，将之前同步更新的过程，变成了异步、可中断的调度策略。体验上不一定更快，但更流畅。
给开发带来的影响，主要是要注意第一阶段涉及的生命周期，比如常用的 `componentWillMount`、`componentWillReceiveProps`(已废弃) ，可能会被打断，**多次执行**。

<img src="https://p1.music.126.net/Zia1w5NoENJhbbZ2s2a4NA==/109951163753554420.png" width="600px" />

### React SSR

React 服务端渲染主要由 `ReactDOM.renderToString ()` 产出 HTML 字符串来做的。
React 16 SSR 性能相对 React 15 有很大的提升（[《What’s New With Server-Side Rendering in React 16》](https://hackernoon.com/whats-new-with-server-side-rendering-in-react-16-9b0d78585d67)）：

<img src="https://p1.music.126.net/z2E0rnuR0v6txaJ_LtRPcA==/109951163753568229.png" width="600px" />

主要得益于几个方面：

1. 重写了服务端渲染引擎
2. 更宽松、高效的差异算法
3. 支持 Render to a Node stream，缩短首字节（TTFB）时间

其中更宽松高效的差异算法，带来了新的 API：`ReactDOM.hydrate`， 也带来了一个严格的要求：
**首次渲染内容两端必须一致！**
和 React 15 不同的是，如果 Node 端和 client 端渲染不一致，React 16 仅会尝试修改 DOM （一般都会导致样式错乱）。而之前的 React 15 会以 client 端渲染结果为准，将 Node 端渲染的 DOM 全部替换。

### 可能导致渲染不一致的情况

#### 1. 随机数

服务端和客户端各自执行 `Math.random()` 拿到的结果肯定是不一致的。
解决方案：可以由服务端计算随机数渲染到 DOM 里，和客户端共用一套。

#### 2. 依赖客户端设备的变量

服务端拿不到设备信息，比如 DPR、客户端版本等。
解决方案：可以让客户端把这类信息写入 UA 或 cookie 中。

如果还有其他情况，实在做不到两端渲染一致，官方在 issue 里也给出了终极方案：把这部分逻辑放到 `componentDidMount` 里执行 `setState`，仅由客户端渲染。

### 同构实践注意

#### 1. 不是所有东西都要 SSR

一般情况下只需要首屏 SSR + 底部 client 端懒加载。这样既让用户更快看到首屏，又减轻了服务端渲染压力。

#### 2. 控制服务端 fetch 超时时间

服务端请求对 render 来说是同步的，等服务端拿到接口数据之后才会走渲染流程。所以如果 fetch 返回时间很长，用户侧将一直白屏。
建议超时时间小于 1 秒，超时则降级为客户端渲染，至少先给用户一个 loading。

#### 3. 变量声明

要注意这个变量的生命周期，是请求级的（ctx），还是应用级的（app）？
业务代码中大部分都是请求级别的变量。一般来说，用户共享方法，而不是变量。
例如 redux 中的 store，千万注意不要做成了应用级的，导致用户看到的是别人的数据。

#### 4. 内存泄露

有些全局变量在 Node 端是应用级的，所以要避免往 window 等全局变量挂载定时器，判断仅在客户端使用 window。

### 应用稳定性保障

#### 1. 单元测试 / CI

使用 Egg 提供的工具，应用可以很方便地写单元测试。一个实例：

<img src="https://p1.music.126.net/N7BKg0JbWCY4bLOqbQtjjw==/109951163753627782.png" width="100%" />

#### 2. 监控报警

-   Bass 集成了 bass-sentry 插件，只需要配置一行 DSN，即可接入 Sentry，应用报错将被自动上报。
-   alinode 在启动容器集成，帮助我们检测 Node 进程健康状态。

#### 3. 进程守护

由 [graceful](https://github.com/node-modules/graceful) 和 [egg-cluster](https://github.com/eggjs/egg-cluster) 来保障。

#### 4. 压测

应用上线前，根据日常峰值流量，评估压测目标，进行压测。可选的压测工具例如 [loadtest](https://www.npmjs.com/package/loadtest)。

以我们实际页面压测为例：
<img src="https://p1.music.126.net/QGRbKiXKvqkK94KCFUg8gA==/109951163753653525.png" width="100%" />

由于 SSR 很耗费 CPU 性能，一个正常页面，Node 能抗住的 QPS 大概在 100+。而降级到 CSR (客户端渲染)时可以抗住很高的 QPS，实测压到 1000 也依然轻松。

#### 5. SSR 自动降级机制

根据机器负载，自动切换降级开关。
例如 load 过高时，将部分请求降级为客户端渲染，不做 API 请求代理，只渲染一个 HTML 壳。当 load 恢复正常时，再全量开启 SSR。
这个功能正在开发中，将作为插件在 Bass 框架里集成。

<br/>

## 关于 Bass

### 与 Next.js 对比

<img src="https://p1.music.126.net/Q0SbgOHRyyYGqdA5DkfXhA==/109951163753670482.png" width="700px" />

-   [Next.js](https://github.com/zeit/next.js) 也是一个 React 同构框架，非常容易上手，也是目前业界最流行的 React 同构方案。但是框架高度封装，深入了解比较困难。最大的痛点是没有插件机制，扩展性有限，不适合团队协作贡献功能模块。
-   Bass 基于 Egg 提供的扩展能力、插件机制、多环境配置、单元测试配套工具等，功能模块独立，Owner 明确，帮助我们把开发和维护做得更高效。
-   同时，Bass 也吸收了 Next.js 的部分优点，根据目录结构自动路由、HMR 支持、服务端预请求处理等。

### 使用文档

Bass 目前已经在网易云音乐的会员、订单、演出票务等营收团队业务中广泛落地使用。
文档完善中，请持续关注。
