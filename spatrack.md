# 单页面应用（SPA）的 SPM 埋点问题解决方案

> 最近前端开发中越来越多的使用到了单页面+多路由的方式。这种单页面应用有很多优势：让用户体验更好、更快，后端只需要关注数据接口，实现了前后端分离；但同时也带来了数据埋点相关的问题。

## 问题描述

众所周知，单页面应用无论跳转到哪个页面，body 标签始终是不变的，因为用的是同一个 body。而 SPM 埋点的 B 位（代表是哪个页面的标志位）就是埋在 body 标签上，这样就带来了三个问题：

-   SPA 应用进行页面切换时， **SPM 的 B 位永远不会变** ，所有页面统计到的 PV/UV 都是一样的，并且都是整个站点的流量。
-   页面只有第一次进入时会全局刷新，也就是说只有第一次进入时会有 1 个 PV， **后继切换操作都不会产生 PV** 。
-   单页面应用做页面跳转应该是不刷新的，但其实使用了 Aplus 脚本之后，会向 url 里插一段 `?spm=xxx`的参数，每次跳到不同的路由下面，这个参数都会变，导致 **整个页面资源又重新加载了** 。这和 spm 埋点的设计有关，spm 如果在单页面里仍然作为 url 的 query 并不合理。

## 解决方案

-   对于此类场景, aplus 最新的脚本（如何使用下文会有说明）提供了对应的接口和方法, 前端可以通过 js 自主修改 SPM，并发送 PV 日志。
-   **针对上述场景，本人已封装了一个组件: [@alife/set-spm](http://web.npm.alibaba-inc.com/package/@alife/set-spm), 可以省去下面第 2 和第 3 个步骤，强烈推荐使用！**
-   **Weex 场景** 可以直接跳转参考：《[Nuke 与 Rax 应用的 SPM 与黄金令箭埋点指南](https://www.atatech.org/articles/88899)》

### 1. 添加 mata 信息

-   如果是单页面应用，目前是有特殊采集机制的。需要禁止 url 修改的话, 可以在 head 部分加个 mata 声明, aplus.js 就不会去修改 url，导致整个页面重新加载了。同时禁止 aplus 自动发 PV：

```
<meta name="data-spm" data-spm-protocol="i">

<meta name="aplus-wating" content="MAN">
```

### 2. 修改 SPM

-   在页面加载后动态修改页面的 SPM 埋点。 aplus.js 提供了一个`setPageSPM()`方法, 前端需调用此方法实现 spm 编码的重新定义(直接修改对应的 meta 声明是无效的，调用 setPageSPM 会自动更新 meta)
-   调用方法: goldlog.setPageSPM(a, b)
-   参数说明:
    -   a 对应页面 SPM 编码的 A 位,即站点码;
    -   b 对应 spm 编码的 B 位, 及页面码。
    -   A 位和 B 位均应该在 SPM 申请中心内注册。a、b 都可以为 null; b 可以不传
-   下面的调用都是合法的：
    -   goldlog.setPageSPM('123', '456'); // 同时修改页面 SPM 的 a、b 位
    -   goldlog.setPageSPM('123'); // 只修改页面 SPM 的 a 位，b 位保持原值
    -   goldlog.setPageSPM(null, '456'); // 只修改页面 SPM 的 b 位，a 位保持原值

### 3. 用新的 spm 发送日志

-   调用 aplus 提供的 goldlog.sendPV()方法发送日志
-   调用方法: goldlog.sendPV({pageid:"", checksum:"123456"})
-   参数说明:
    -   page_id: 如果不需要做 abtest, 则此处置空即可
    -   checksum: 与当前页面 URL 对应的 checksum 值
    -   chksum 的生成, 可访问此链接 [http://on.alibaba.net/](http://on.alibaba.net/) 输入页面 URL 或者页面的 SPM a.b 自助获得。如果使用 [@alife/set-spm](http://web.npm.alibaba-inc.com/package/@alife/set-spm) 可以自动计算得出。
    -   sendPV()应在页面 body 标签加载后再调用执行.

## 如何使用 aplus 最新脚本

有两种方式：

-   通过 nginx 服务器的 beacon 模块，自动向 HTML 页面注入 aplus.js (大多数应用都是这种)；
-   手动部署 aplus.js 埋点脚本：
    -   无线端脚本： `//g.alicdn.com/alilog/mlog/aplus_wap.js`
    -   PC 端脚本：`//g.alicdn.com/alilog/mlog/aplus_v2.js`

如果你的应用 beacon 模块注入的 aplus 脚本不是以上两者，可能已经过时，需要修改 beacon 配置。具体可参考 aplus 官方的 [阿里日志接入指南](http://log.alibaba-inc.com/log/info.htm?spm=a1z71.7905536/2277.outline.7.4506d6aoby3q0&type=2277&id=60)

## 方案实例（更推荐使用@alife/set-spm）

比如服务市场无线的首页 spm 为 a1z8w.8005165，而订单列表页的 spm 为 a1z8w.8005215，那么订单列表页加载后就需要加一个修改 SPM 和发送打点日志的逻辑：

```js
componentDidMount() {
    // 注：goldlog的加载是异步的，页面加载完成后不一定已经有goldlog对象
    let goldlogTimer = setInterval(() => {
        if(window.goldlog){
            // 手动修改spm-b位,并发送正确的spm统计信息
            window.goldlog.setPageSPM && window.goldlog.setPageSPM('a1z8w', '8005215');
            window.goldlog.sendPV && window.goldlog.sendPV({page_id: '', checksum: "46807199"});
            clearInterval(goldlogTimer);
        }
    }, 1000);
}
```

## 更多

更多关于埋点的相关问题，可以参考：

-   [阿里日志 - SPM](http://spm.alibaba-inc.com/spm/index.htm)
-   [阿里日志 - 单页应用(SPA)](http://log.alibaba-inc.com/log/info.htm?spm=a1z71.7905536/2395.outline.3.66a834a9VIOzpf&type=2395&id=19)
-   [面向前端的 Aplus 打点解惑集](http://groups.alidemo.cn/alilog/manual-for-f2e/index.html)
