---
layout: post
title: Node 应用从申请到部署上线全流程总结
---

> 内网上之前有一些文档，但部分已经过时可能会有误导，比如之前约定监听 7001 端口，现在是 6001 端口。本应用从申请到上线都是在 2015 年 12 月，目前应该仍然通用，本文引用的参照文档也都以最新的为准。

> 本流程以黄金眼项目为例。[黄金眼](http://goldeneyes.taobao.com/)是一个 A/B 测试平台，新版黄金眼使用 Node 开发服务端，本文通过我自己的亲身经历阐述了阿里集团内一个 Node 应用从申请到上线的流程，以及这个过程中自己遇到的一些问题和解决办法。

> 由于想深入理解 Koa 的机制和原理，本项目当时没有使用 Midway 和 Egg（当时也还没出来。。。）

## 在 aone 申请应用

> 申请应用可参照：[申请新应用及发布](http://docs.alibaba-inc.com:8090/pages/viewpage.action?pageId=239013204)

1. 先在 aone2 上申请新应用，找自己部门对应的 PE（共享@粟裕）审批。<br/>

2. git 代码仓库关联应用：找 SCM（共享@红巾）审批应用的 gitlab 代码模块，提醒 SCM 部署类型选 node.js。这样，在 aone2 应用里，代码模块里才会有应用的 git 仓库地址，后面才能成功新建日常(审核通过后有半个小时左右延迟)。

    ![Alt text](https://img.alicdn.com/tps/TB1wtBjKVXXXXXPXFXXXXXXXXXX-236-295.png)

<br/>

## 新建日常，申请代码变更

> 同可参照：[申请新应用及发布](http://docs.alibaba-inc.com:8090/pages/viewpage.action?pageId=239013204)

1. 在 Aone 上新建日常。请先尽量保证自己的 node 应用符合集团的打包规范（仅作推荐，不强制，但 bin 目录下的 server.js 一定要有），应用要监听**6001**端口，如果接入[alinode](http://alinode.alibaba-inc.com/)，在 package.json 中加入：

```
"engines": {
    "install-alinode": "1.2.1"
}
```

Node 应用的打包规范详见：[集团 Node 应用打包规范](http://docs.alibaba-inc.com:8090/pages/viewpage.action?spm=0.0.0.0.f8ADox&pageId=241992025)<br/>

2. 申请变更（新建日常时可同时申请）
   ![Alt text](https://img.alicdn.com/tps/TB10IlxKVXXXXbXXXXXXXXXXXXX-680-544.png)

<br/>

## 提交集成，进入日常，日常部署

1. 申请变更之后，在“我的变更”中点“提交集成”。

> -   第一次日常部署时，aone2 会 **自动分配日常服务器** ，给出日常环境服务器 ip 地址，如果第一次显示日常环境创建失败，请 10 分钟后重试日常部署。<br/>
> -   如果 aone 上部署失败，日志看不到，不要灰心，日志页面刷新个几十遍就会出来了。。。<br/>
> -   如果日常部署失败了，找 SCM 也很忙没有回复的话，可以加旺旺群【群号 1424680928 密码 ali20140919】，在群里问问题，一般还是可以得到回复的。

-   日常部署会做以下事情： 1. 日常构建打包，会在临时的被随机分配的机器上执行。构建会通过命令`sh bin/build.sh daily`，执行自己应用下的 build.sh（如何写 build.sh 可以参照这一份[build.sh](http://gitlab.alibaba-inc.com/cm/goldeneye-node/blob/master/bin/build.sh)）。 2. 打开 build.sh 可以看到，构建过程实际上是首先用 nvm 安装了 node，然后安装了 tnpm，执行了 tnpm install。日常环境下构建，执行 build.sh 时会带上 daily 参数（预发下是 prepub，生产环境下是 publish），我们可以根据\$1 参数，获取到当前的环境，并把当前的环境输出到一个文件（.env）里。这样部署后，我们的 node 应用可以通过读取这个文件的内容，获取当前的 NODE_ENV。 3. 构建完成后，会将所有文件打包成 tar.gz 压缩包，然后上传到我们的日常服务器上去，解压缩。日常服务器会执行`nodejsctl start`命令启动应用，然后执行`preload.sh`检测 node 应用是否开启成功（检测原理下面会提到），如果返回 success 则会自动启动 nginx，至此日常部署完成。

2. 日常域名申请：参照[测试环境域名管理](http://docs.alibaba-inc.com/pages/viewpage.action?pageId=248128184)
   黄金眼申请到的日常地址是：[http://goldeye.daily.taobao.net/](http://goldeye.daily.taobao.net/)。
   注：如果要申请的是\*.daily.taobao.net 的域名，走[ATEP](https://atep.alibaba-inc.com/myatep)系统， **默认就支持 https** ，域名解析会自动指向我们的日常机器，不需要自己再配置。如果是非业务的想应用申请域名，可以在[KFC](http://kfc.alibaba-inc.com/#/resource/new/dns)系统里自行申请。<br/>
3. 部署不顺利怎么办？登陆日常服务器查错。
   如果需要登陆日常环境服务器（你一定会需要的）排查错误或安装东西： 1. 首先去[KFC](http://kfc.alibaba-inc.com/#/resource/server/account/push)申请对应日常服务器 ip【 **应用管理员** 】权限（黄金眼日常是`10.189.197.77`）。 2. 通过 ssh 登陆后，先执行`sudo su admin`，切换到 admin 用户身份。应用的目录在`/home/admin/`下。例如黄金眼的应用源码目录：`/home/admin/goldeneye-node/target/goldeneye-node/`。 3. 第一次部署如果服务器没启动 nginx，则需要手动启动 nginx，执行`/home/admin/cai/bin/nginxctl start`。
   如果因为健康检查的原因，nginx 没启动起来，可以修改一下 nginx 配置文件：`vim /home/admin/cai/conf/nginx-proxy.conf`
   添加如下几行：

```
location = /status.taobao {
      stub_status on;
}

location = /check.node {
      stub_status on;
}
```

<br/>

### 日常部署其他常见报错及解决办法

> [测试环境常见问题集](http://docs.alibaba-inc.com/pages/viewpage.action?pageId=162308951)这里有一些官方回答的问题。

-   如果部署过程出错，可以在服务器上手动执行`/home/admin/goldeneye-node/bin/nodejsctl start`，进行排错，node 应用输入日志在`/home/admin/goldeneye-node/logs/nodejs_stdout.log`。

-   **健康检查** ：预检会执行(除非调试否则不需要手动执行)：`/home/admin/goldeneye-node/bin/preload.sh`，
    系统通过检查执行`curl http://127.0.0.1:6001/check.node`，返回`success`才说明应用正常启动了，才会接着启动 nginx。
    所以如果全站接入 BUC，要保证 BUC 不会覆盖到/check.node，不然系统检查 node 状态时会做跳转，得不到 success，部署会报错。如果用的是 Koa，可以在 use buc 之前，先 use [taobaostatus](http://npm.taobao.net/package/koa-taobaostatus)，做健康检查。

-   服务器上应用 **启动和检查过程** 可以参考：[自动发布启动过程](http://npm.taobao.net/package/koa-taobaostatus)

-   服务器上的 node 路径：`/home/admin/goldeneye-node/target/goldeneye-node/node_modules/node/bin/node`

-   **其他要注意** ：不管出什么问题，千万 **不要删除** 服务器上`/home/admin/goldeneye-node`这个目录。不要试图删掉这个目录，认为再次部署会重建目录，不会的！！要改最多改到`target/goldeneye-node/`里面的内容，千万别把`target/goldeneye-node/`目录外面的东西删了，不然要联系 SCM 重新分配日常机器，重建日常环境！。。。

<br/>

## 预发部署

1. 日常环境自测没问题后，Aone 2 进入预发环境，点击预发部署，然后勾选变更，由日常环境进入预发：![进入预发](https://gw.alicdn.com/tps/TB1xm0SLXXXXXXMXpXXXXXXXXXX-792-262.png)<br/>
2. 第一次预发部署由于没有机器，预发部署会失败。不用担心，就预发部署一次，第一次失败后 Aone 会给出申请预发机器的入口：![预发申请机器入口](https://img.alicdn.com/tps/TB1GffMKVXXXXaZXFXXXXXXXXXX-939-345.png) 点击申请机器和线上域名。详见[新应用上线新流程使用帮助说明](http://docs.alibaba-inc.com:8090/pages/viewpage.action?spm=0.0.0.0.4GKXbh&pageId=245970375)。同时数据库在 iDB 上也要申请上线。

3. 再进行一次预发部署。

4. 登陆预发机器排错。

    - 如果预发部署失败，需要去预发机器上查看日志的话，需要去 [PSP](http://psp.alibaba-inc.com/paas/account/index.htm#/apply) 申请服务器 ssh 登陆权限（还记得日常是去 KFC 申请吗，PSP 对应的是线上机器的操作）。
    - 生产环境下的服务器必须要 **通过跳板机登陆** ，不允许直接用办公电脑 ssh 登陆。可以参见[集团跳板机使用规范和方法](http://docs.alibaba-inc.com:8090/pages/viewpage.action?spm=0.0.0.0.kAmCAv&pageId=203557709)。所以也要申请一下跳板机权限，在[PSP](http://psp.alibaba-inc.com/paas/account/index.htm#/fortress)申请，比如可以申请`login1.cm3.alibaba.org`这台跳板机。
    - 申请成功后，首先 ssh 登陆跳板机，`ssh login1.cm3.alibaba.org` (采用域密码+令牌六位数字登录)，然后在跳板机里再 ssh 登陆我们的生产环境机器，接下来的排错过程同上面日常环境。

5. 预发机器上安装 VIPServer（可选，根据应用实际情况）
   预发机器上可能默认没有安装 vipserver，无法解析数据库域名。所以需要手动安装一下 vipserver：

-   服务器上以 admin 身份执行`sudo yum install t-midware-vipserver-dnsclient -b current`，安装。
-   然后尝试 `ping tddl.sh05.tbsite.net`，如果能 ping 通，证明 vipserver 安装成功。详细可参考[vip 安装说明](http://gitlab.alibaba-inc.com/middleware/vipserver/wikis/vipsrv-dns-client)。

6. 部署成功后，访问预发页面：
   电脑绑定预发 host：`110.75.98.154 pre.ge.taobao.com`，然后访问[pre.ge.taobao.com](pre.ge.taobao.com)。需要绑定的 host 在 Aone 的预发部署页面“预发测试绑定平台”里可以看到：![预发绑定](https://gw.alicdn.com/tps/TB1qBFxLXXXXXaLXVXXXXXXXXXX-1932-732.png)

<br/>

## 应用上线

> 恭喜终于走到这一步了，最蛋疼的坑你肯定都踩完了 : -）激动人心的时刻到来了！颤抖吧！gogogo！此步可以参照 [新应用上线新流程使用帮助说明](http://docs.alibaba-inc.com:8090/pages/viewpage.action?spm=0.0.0.0.NrbdFF&pageId=245970375#)

1. Aone 点击“正式环境”，进入正式。
2. 进行正式构建，构建成功后，提交发布单。PE 审批以后，等待发布完成就好啦。
3. 到 [PSP](http://psp.alibaba-inc.com/paas/account/index.htm#/apply) 系统申请线上机器登陆权限，几台机器就申请几个。
4. 线上机器安装 VIPServer（同预发，每台机器都要安装，根据应用实际情况）
5. 如果申请了线上域名，比如`*.taobao.com`，则要走 VIP 申请审批流程。这里的 VIP 是外面可以访问的一个 ip，对应用分组的机器做了负载均衡。首先 PE 创建工单，然后安全@墨禅会审批，给安全的同学描述一下为什么申请 VIP（为什么要暴露给外网）。
6. VIP 审批成功以后，系统会走域名审批流程。`*.taobao.com`的域名归属人是@圆心，PE 审批完以后，需要@圆心最终审批。注：`*.taobao.com`二级域名名字长度必须 **大于 4 个字母** ，否则@圆心不会审批通过。。

<br/>

## 接入集团 HTTPS（可选）

> 可参见[接入说明](http://tbdocs.alibaba-inc.com/pages/viewpage.action?pageId=247369731)

1. 在 vipserver 配置里面，[添加域名](http://ops.jm.taobao.org:9999/vipserver-ops/domain_detail.htm)。如何填写可以参照[这里](http://tbdocs.alibaba-inc.com/pages/viewpage.action?pageId=254942818)。
2. 上一步完成，注册了域名以后，到 psp 提交[统一接入申请单](http://psp.alibaba-inc.com/paas/uniconnect/appApply.htm)。注意“VipServer Key 名称”要填的是 **域名名称** ，不是 token。
3. 工单审批完成后，自己先验证 VIP 绑定，如果验证没有问题，通知 PE 切域名。wagbridge.taobao.com ，hz.wagbridge.taobao.com ，sh.wagbridge.taobao.com，dig 一下域名的 VIP，绑定验证，根据你的机房部署，第一个是杭州+上海，第 2 个是杭州，第 3 个是上海。HTTPS 方面如果有问题可以找@千山。

<br/>

## 其它

### 接入 AliNode

> 可参见 [接入 Alinode 教程](http://alinode.alibaba-inc.com/doc/deploy)

1. [将 Node.js 打包到应用代码中](http://node.alibaba-inc.com/deploy/node-with-app.html?spm=0.0.0.0.FaR5h8)
2. 服务器上安装命令集，执行: `$ sudo yum install alinode-commands -b current`
3. 配置监控项: [alimonitor](https://m.alibaba-inc.com/monitor/monitoritem/monitorItemAdd.htm)->监控管理->单机|聚合->添加监控项

### 接入 BUC 和 ACL

> 可参考 [BUC](http://docs.alibaba-inc.com/display/RC/BUC)、[集团统一登录中心用户帮助手册](http://gitlab.alibaba-inc.com/buc/sso/tree/master)、[ACL 接入流程](http://docs.alibaba-inc.com/pages/viewpage.action?pageId=239841933)。

1. 首先接入 BUC，然后接入 ACL。[日常 buc](https://login-test.alibaba-inc.com)+[日常 acl](http://acl-test.alibaba-inc.com)，[线上 buc](https://login.alibaba-inc.com)+[线上 acl](https://acl.alibaba-inc.com)。
2. 如果需要调用 ACL 的写接口，比如需要调用 api 创建一个权限时，需要先申请一个公共账号。申请完公共账号以后，去 ACL 平台申请 HSF 接口权限（用申请到的公共账号）。申请完 HSF 接口的写权限以后，才可以调用 ACL 的写权限接口。
    > 为何使用公共账号？
    > 每个应用对应一个公共账号，因为应用申请到的公共账号权限稳定，不像员工会因离职转岗权限发生变化。公共账号可以固定写在接口调用的代码中。

<br/>

### cluster 模式和异常重启

> 工具选型：pm2、forever 与 cfork

-   放弃使用 pm2 的原因： 1. 服务器上不支持全局安装 pm2 的 npm 包，启动命令是集团约定写死的诸如`node server.js`，无法使用`pm2 start` 启动应用。 2. pm2 所有操作均为命令操作，而由于第一点的原因，服务器上不方便也无法进行命令操作。 3. pm2 虽然暴漏出几个 api 供 web 调用（比如 pm2 list），但返回的都是 json 数据，没有实现好的可视化页面。 4. 由于以上几点原因，如果继续使用 pm2，也只能用到其最简单的功能（cluster 和错误自动重启），这和 cfork 没有什么区别了。所以不如使用更轻量的 cfork。 5. D2 大会上，不四也提到了 pm2 太重，集团内的场景推荐使用更轻量的 cfork，天猫那边也是使用了 cfork。
-   所以，最终我们还是选择了 cfork。

<br/>

## Q&A

### 名词解释：

-   BUC：是 Backend User Center 的英文缩写，中文名为“后台用户管理中心”，包含的功能有：统一认证、统一验权、单点登录、后台用户信息统一管理、后台用户权限统一管理。
-   ACL：是 BUC 旗下的产品，ACL 是 Access Control Layer 的英文缩写，中文名为“集团权限平台”，致力于解决后台应用权限管理问题

### 各个环境介绍：

-   项目环境：淘系应用在开发过程中使用的测试环境，每个应用都是独立的环境（这里一般用不到）
-   日常、预发、正式，灰度这四个环境都属于 **集成环境**
-   日常：由 SCM 管理，各个变更集成到一个分支上进行测试的环境
-   预发：由 PE 管理，跟正式环境配置一样的环境，用于正式上线前的最后校验
-   正式：对外服务的应用环境

### 各个环境的机器需要自己单独申请吗

-   不需要。
-   日常环境机器，进入日常后，第一次日常部署自动提供。日常的域名申请参见上文。
-   预发机器和线上机器，第一次预发部署失败后，Aone 会有申请入口，填一下申请单就 ok 了。

<br/>

## 参考资料

-   Node.js 阿里手册（集团官方部署文档）：[http://node.alibaba-inc.com/deploy/nodejs.html](http://node.alibaba-inc.com/deploy/nodejs.html)
-   midway 上的部署流程参考：[https://midway.taobao.net/docs/deploy/index.html](https://midway.taobao.net/docs/deploy/index.html)
-   ATA 上关于 node 应用部署的文章：
    -   [nodejs 应用接入、部署流程整理](http://www.atatech.org/articles/39606)
    -   [如何发布一个 node 应用](http://www.atatech.org/articles/16767)
    -   [在阿里体系内玩转 node，坑你踩过了么？](http://www.atatech.org/articles/22012)
    -   [1688 Node.js 应用部署、发布的流程、规范整理及运维指引](http://www.atatech.org/articles/30602)
    -   [Aone2 Nodejs 构建部署](http://www.atatech.org/articles/32462)
    -   [阿里 node 手册](http://www.atatech.org/articles/13675)
