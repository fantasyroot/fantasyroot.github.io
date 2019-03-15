# React Router + Webpack 实现异步按需加载

## What

按需加载是什么：单页面多路由的应用场景，按页面拆分成多个 js 模块，访问哪个路由就加载哪块代码，不加载其他不相干的代码。

## Why

为了让用户拥有更好地体验，现在单页面应用（SPA）越来越多了。但随着业务规模越来越大（比如[无线服务市场](https://fuwu.m.taobao.com/wap/ser/index.htm)），页面也越来越多，原来通过 webpack 打包成「一个」js 的方式已经过于「重」了。
其实加载某个路由页面时并不需要其他页面冗余的代码，既 **影响体积白费了流量** ，又 **导致性能越来越差** 。这时，做好按需加载就很有必要了。

## How

### 1. 修改 webpack 配置

修改`webpack.config.js`，在 output 字段加上`chunkFilename`

```js
output: {
    path: 'build',
    publicPath: !DEV ? gitVersion : '/build/',
    filename: '[name].js',
    chunkFilename: '[name].[chunkhash:5].chunk.js'
},
```

-   [name]会被替换成你为 chunk 指定的名字，默认是 webpack 自动生成的 chunk id；取 chunkhash 中的 5 位，防止在文件修改后仍使用旧缓存。
-   publishPath 字段说明：如果当前非本地开发环境，传入 git 当前的版本号，有用，下面第 4 点会讲。
-   如何在 webpack 获取当前 git 版本号？如果你的分支名字都是普遍约定的“daily/x.x.x”形式的，可以用如下方法：

```js
var execSync = require('child_process').execSync;
var gitBranch = execSync(`git symbolic-ref --short HEAD`)
    .toString()
    .trim();
var gitVersion = gitBranch.split('/')[1] || '';
```

### 2. 改造老的 routes

原来的路由（`routes.js`）可能是这样写的：

```js
const Routers = (
    <Route
        path="/"
        component={Layout}
        onEnter={onRouteEnter}
        onChange={onRouteChange}
    >
        <IndexRedirect to="home" />
        <Route path="home" component={Index} spm={'a1z8w.8005165'} />
        <Route path="my" component={My} spm={'a1z8w.8005215'} />
        <Route path="search" component={Search} spm={'a1z8w.8005193'} />
        <Route
            path="serviceDetail"
            component={ServiceDetail}
            spm={'a1z8w.8005173'}
        />
    </Route>
);
export default Routes;
```

做如下改造：

```js
const rootRoute = {
    path: '/',
    onEnter: onRouteEnter,
    onChange: onRouteChange,
    getComponents(location, callback) {
        require.ensure(
            [],
            function(require) {
                callback(null, require('./layout/layout').default);
            },
            'app'
        );
    },
    indexRoute: {
        onEnter: (nextState, replace) => replace('/home'),
    },
    getChildRoutes(location, callback) {
        require.ensure(
            [],
            function(require) {
                callback(null, [
                    require('./r/index/routes').default,
                    require('./r/my/routes').default,
                    require('./r/search/routes').default,
                    require('./r/serviceDetail/routes').default,
                ]);
            },
            'app'
        );
    },
};
export default rootRoute;
```

-   由于要按需加载，需要让路由动态加载模块，将原来的`component`用`getComponents`方法来替代。
-   这里用到了很多`require.ensure`方法，`require.ensure`方法是 webpack 提供的方法，也是按需加载的 **核心方法** ，详见[Webpack 的代码分割功能](http://webpack.github.io/docs/code-splitting.html)。可以对传入的模块单独打包，生成对应的 chunk file。

```js
require.ensure(dependencies, callback, chunkName);
```

-   indexRoute 是首页，这里通过配置 react router api 提供的`onEnter`属性，redirect 到 home 页。
-   这里`onEnter`、`onChange`是用来设置单页面 spm 的，视业务情况使用。详情可以参见[@alife/set-spm](http://web.npm.alibaba-inc.com/package/@alife/set-spm)
-   各个页面的子路由写到`getChildRoutes`方法里，例如上面的`/r/index/routes.js`。

### 3. 为各个页面目录下创建子路由

以 index 路由为例，新建子路由`/r/index/routes.js`，文件内容：

```js
const routes = {
    path: 'home',
    spm: 'a1z8w.8005165',
    getComponents(location, callback) {
        require.ensure(
            [],
            function(require) {
                callback(null, require('./index').default);
            },
            'index'
        );
    },
};
export default routes;
```

-   也用到了`require.ensure`方法，这里第三个参数是'index'，所以 chunkName 就叫'index'，配合上面 Webpack 的配置，就会单独打包出一个`index.chunk.js`文件。当访问 index 路由时，就会动态加载这个模块。
-   `/r/index/index.js`仍是原来的页面文件不用修改。

### 4. 修改 webpack 入口文件

`app.js`里增加以下代码：

```js
// 配置运行时按需加载的静态资源url，本地/日常/线上
let url = window.location.origin;
let publicPath = __webpack_require__.p;
__webpack_public_path__ =
    url.includes('localhost') || url.includes('127.0.0.1')
        ? publicPath
        : url.includes('daily.taobao.net') || url.includes('waptest.taobao.com')
        ? `//g-assets.daily.taobao.net/sj/h5.qnservice/${publicPath}/`
        : `//g.alicdn.com/sj/h5.qnservice/${publicPath}/`;
```

-   这一步是和第 1 步修改`webpack.config.js`相结合的。
-   为什么要做这一步：
    -   一般来说前端的 js 文件是放到 cdn 上的，跟线上业务网站的 url 不同，所以 publicPath 不能写相对路径，一定要写 cdn 的绝对路径。
    -   一个完整的 js 路径是应该包含（daily/线上环境 cdn 域名）+（git 版本号）两部分的，但只有在 runtime 运行时环境下才能知道当前是 daily 还是线上，只有在编译打包环境下才知道当前的 git 版本号。
-   runtime 运行时环境下通过 webpack 暴露的`__webpack_require__.p`变量可以拿到 webpack 配置里`output`中`publicPath`的值；可以通过修改`__webpack_public_path__`变量，改变浏览器运行时环境下的 js 地址的 cdn 前缀。
-   通过这种方式可以把 git 版本号从编译环境传给运行时环境，然后加上 daily/线上的 cdn 域名，拼成最终完整的 url 如：`//g-assets.daily.taobao.net/sj/h5.qnservice/0.6.0/guideList.98f27.chunk.js`。

## Result - 优化结果

### webpack 打包结果对比

-   原来的：
    ![](https://img.alicdn.com/tps/TB1z3L5OFXXXXXcaFXXXXXXXXXX-526-89.png)

-   优化之后：
    ![](https://img.alicdn.com/tps/TB1wyL8OFXXXXXzaFXXXXXXXXXX-678-385.png)
-   将一个大包 app.js（**800+KB**）拆解成多个模块，拆解后的公共脚本 app.js 仅有 **400+KB**（cdn 压缩后仅 **163KB**）。

### 浏览器加载情况对比

-   原来的：
    ![](https://img.alicdn.com/tps/TB1bakiOFXXXXa_XVXXXXXXXXXX-2832-1452.png)
-   优化之后-首页：
    ![](https://img.alicdn.com/tps/TB145j7OFXXXXcQapXXXXXXXXXX-2848-1474.png)
-   优化之后-类目列表页：
    ![](https://img.alicdn.com/tps/TB1JO.fOFXXXXaIaXXXXXXXXXXX-2718-1410.png)
-   优化之后会多出一个获取当前页面 js 模块的 http 请求，但可以明显看出优化后的 js 已经比原来的 js 体积小了很多，在逻辑比较少的页面(如上图)尤其明显，可以由原来的 **238KB** 减少到 162 + 2.7 = **164.7KB**
-   ps：优化之后的其实还加了新需求，包含新增了的两个页面 ;-)

## 参考文章

-   [React router 官网提供的按需加载 demo](https://github.com/ReactTraining/react-router/tree/master/examples/huge-apps)
-   [react-router 按需加载(segmentfault)](https://segmentfault.com/a/1190000007141049)
