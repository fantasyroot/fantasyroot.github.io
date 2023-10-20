---
layout: post
title: 你可能用的到的 Unix 知识
---

> 分享大纲：
>1.  文件系统相关，inode、权限、硬链接软链接及在 pnpm 中的应用
>2.  Shell 脚本入门和注意事项
>3.  一些实用的命令分享

## 一、背景
工作生活中，少不了和 Unix/Linux 系统接触，比如 MacOS、Linux 服务器、Openwrt 软路由配置和部署等。偶尔也需要写点 Shell 脚本（尽管我们可以用 Shell 脚本当壳，核心内容用 node.js 来写）。希望这点实用为主的分享对大家有用。

列举几个公司内用到的 shell 脚本场景（需内网访问）：
1.  [pr-common 的构建部署](https://gitlab.qunhequnhe.com/fe/febu/pr-common/blob/master/build_pr_common.sh)：主要事项：版本号输出到 njk 、 执行 saas 脚本（因 pr-common 不支持直接引入微应用，构建时同步微应用的产物资源到 njk 中）、打包，上传 cos
2.  [Serverless 部署相关](https://gitlab.qunhequnhe.com/def/docker-images/-/blob/master/serverless/alinode-faas/docker-entrypoint.sh)

## 二、文件系统相关
### 什么是 inode
![inode](https://qhstaticssl.kujiale.com/image/png/1697792205577/4AEDBC46B11B494D51C1E4DD59BF3E36.png)

在 Unix/Linux 系统中，表面上，用户通过文件名打开文件，实际上，系统内部这个过程分成三步：
- 首先，系统找到这个文件对应的 inode 号码；
- 其次，通过 inode 号码，获取 inode 信息；
- 最后，根据 inode 信息，（有访问权限的话）找到文件数据所在的 block，读出数据。
储存文件元信息（创建者、创建日期、大小等）的区域就是 inode。每一个文件都有对应的 inode。

```bash
$ ls -li link.txt
50260683 -rw-r--r--  1 anto  staff  0 Oct 12 12:00 link.txt
```

#### 这么设计有什么优势？
举个例子，软件更新变得简单，可以在不关闭软件的情况下进行更新，不需要重启。（[阮一峰：理解 inode](https://www.ruanyifeng.com/blog/2011/12/inode.html)）

- 因为系统通过 inode 号码（而不是文件名），识别运行中的文件。
- 更新的时候，新版文件以同样的文件名，生成一个新的 inode，不会影响到运行中的文件。
- 等到下一次运行这个软件的时候，文件名就自动指向新版文件，旧版文件的 inode 则被回收。

### 硬链接和软链接

```bash
# 新建硬链接
$ ln file link

# -s 新建软链接（符号链接）, B 是被链接文件，A 是将创建的软链接
$ ln -s B A
```

![硬链接和软链接](https://qhstaticssl.kujiale.com/image/png/1697792251854/8049EF6B5BCAFC368C646F3AF9563790.png)

- 硬链接，和被链接文件指向同一个 inode。所以不能跨文件系统（1 个 inode 只能属于一个文件系统）
    - 无论修改哪一个文件，另一个也会相应变化
    - 删除一个文件名，不影响另一个文件名的访问
    - 只能对文件创建硬链接，不能对目录创建硬链接
- 软链接（符号链接），会生成一个新的 inode，指向被链接文件真实的 block。 类似 windows 中的“快捷方式”
    - 无论打开哪一个文件，最终读取的都是被链接（B）文件
    - 文件 A 依赖于文件 B 而存在，如果删除了文件 B，A 还存在，但无法访问源文件内容
    - 软链接与硬链接最大的不同：文件 A 指向文件 B 的文件名，而不是文件 B 的 inode 号码，文件 B 的 inode "链接数" 不会因此发生变化。

#### 应用 - PNPM 减少 node_modules 的磁盘占用
- node_modules 的一个梗
![](https://res.cloudinary.com/practicaldev/image/fetch/s--BIL81hqp--/c_imagga_scale,f_auto,fl_progressive,h_420,q_auto,w_1000/https://dev-to-uploads.s3.amazonaws.com/i/8bqhs2j0frz5upzfyj90.jpg)

- PNPM 的解决办法
![](https://camo.githubusercontent.com/cdbaa37b72ed202cba93b974b5e4e15ac73ebbb461147553e49ea72aec7e31e8/68747470733a2f2f73312e617831782e636f6d2f323032322f30382f31342f765573454d342e706e67)

- 所有 npm 包都安装在全局目录 `~/.pnpm-store/v3/files` 下，同一版本的包仅存储一份内容，甚至不同版本的包也仅存储 diff 内容。
- 每个项目 `node_modules` 下安装的包结构为树状，符合 node require 的查找规则，以软链接方式将内容指向 `node_modules/.pnpm` 中的包
- 每个包的寻找都要经过三层寻址：`node_modules/package-a` > 软链接 `node_modules/.pnpm/package-a@1.0.0/node_modules/package-a` > 硬链接 `~/.pnpm-store/v3/files/00/xxxxxx`

```sh
$ ls -l node_modules/dohjs
lrwxr-xr-x  1 anto  staff  36 Aug 30 18:48 node_modules/dohjs -> .pnpm/dohjs@0.3.3/node_modules/dohjs
```

- `node_modules` 一级目录很干净，都是 `package.json` 中声明的包，没有幽灵依赖的问题。其他包被拍平到 `.pnpm` 目录下。
- 通过硬链接的方式保证了相同的包不会被重复下载，链接到 pnpm 本机全局存储。比如跨 repo 中的 @dohjs/@0.3.3 相同版本的包文件可以被复用。

```bash
# ls -i 看下 .pnpm 下某个文件的 inode 号码，也看到硬链接次数是 2
$ ls -li /Users/anto/Projects/doh-benchmark/node_modules/.pnpm/dohjs@0.3.3/node_modules/dohjs/package.json
45349431 -rw-r--r--  2 anto  staff  1265 Aug 30 17:33 

# 根据这个 inode 号码，扫一下硬盘中，有几个文件关联了这个 inode，查出来确实是 2 个文件
find ~/ -inum 45349431
# 第一个
/Users/anto/Projects/doh-benchmark/node_modules/.pnpm/dohjs@0.3.3/node_modules/dohjs/package.json
# 第二个
/Users/anto/Library/pnpm/store/v3/files/62/3b004149c528a15325b39235b366c559ef0c1caf380e5ab17e5864d6a0d4b377ecdd721e4231ac2b55843347bb9fcab905a295af59ac290250bb1b05c7a7b1
```

- 但是硬链接的方式也有坑，如果你在 node_modules/ 下修改了某个文件进行调试，忘记改回去，在其他项目也会被误引入。因为本质上你改的是同个文件。

### 文件权限

```bash
# 给所有用户赋予执行权限，等价于 a+x
$ chmod +x script.sh

# 给所有用户读权限和执行权限
$ chmod +rx script.sh
# 或者
$ chmod 755 script.sh
```

脚本的权限通常设为 `755`（拥有者有所有权限，其他人有读和执行权限）。755 到底代表啥呢？
![文件权限](https://qhstaticssl.kujiale.com/image/png/1696748577569/5E07FAB75D43FA6AB1F5270D29587748.png)

比如某个文件的权限是：`-rwxr-xr-x`，chmod 可以用 755 表示。
- 第一位是文件属性，目录: d，普通文件：-，符号链接：l
- 755 是三位数，每位的顺序是用户身份；每位的值表示有哪些权限。比如：
    - 7（=4+2+1，二进制 111）表示权限为可读、可写、可执行；
    - 5（=4+1，二进制 101） 表示可读、可执行；

> 提问：如果想把某个文件权限变成最宽松，所有人都可读可写可执行，用 `chmod` 命令应该怎么操作？

Tips: 目录的执行权限。对于目录来说，如果无执行权限，则对应用户 cd 不进去。（为啥？不是有读权限了吗？）
- 目录也是一种文件，目录文件的读权限（r）是针对目录文件本身。如果只有读权限，无法获取储存在 inode 节点中的其他信息，而读取 inode 节点内的信息需要目录文件的执行权限（x）。

### 输出重定向

```sh
# 执行 command1 然后将标准输出的内容，覆盖式存入file1
$ command1 > file1

# 不覆盖，追加到指定文件
$ command1 >> file1

# `2>`用来将 标准错误 重定向到指定文件。
# 比如我们想把执行中的错误，单独记录在 error.log里
$ ls -l /bin/usr 2> error.log

# 标准输出和标准错误，可以重定向到同一个文件:
$ ls -l /bin/usr > ls-output.txt 2>&1
# 或者
$ ls -l /bin/usr &> ls-output.txt

# 追加到同一个文件
$ ls -l /bin/usr &>> ls-output.txt
```

- 想显示命令输出的同时，保存到文件：`tee`。

```bash
$ ping kujiale.com | tee output.txt
```

### 其他常用命令
- 查看某个目录占用磁盘空间大小：

```bash
$ du -sh node_modules
283M    node_modules
```

- 处理转换文本：awk 或 sed。sed 全名叫 stream editor，流编辑器。常用来通过正则模式匹配，对文件内容做修改。

```bash
$ sed -i -e "s/g_prCmnCdnHost = \"\"/g_prCmnCdnHost = \"\/\/qhstaticssl.kujiale.com\/pub\/$deployVersion\"/g" $cdnInfoFtl
```

- 查看日志文件尾部内容

```bash
# 尾部 50 行
$ tail -n 50 output.log

# `-f`会实时追加显示新增的内容，常用于实时监控日志，按`Ctrl + C`停止。
$ tail -f /var/log/messages
```

## 三、系统和网络相关
- 查看某个端口被哪些进程占用

```bash
$ lsof -i :7000
```

- ps 查看当前系统正在执行的进程信息

```bash
$ ps -ef |grep node
# 或
$ ps aux |grep node
```

- 查看本机网卡和 ip 等网络信息

```bash
$ ifconfig
```

- 磁盘挂载

```bash
# 查看磁盘挂载信息
$ df -h

# 挂载磁盘
$ mount /dev/sdb1 /mnt/sdb1
```

- 防火墙 iptables
- 定时任务 crontab

> 分享个假期前开发的网络小工具（nali），方便查询 ip 归属地： [https://github.com/fantasyroot/nali-ip-cli](https://github.com/fantasyroot/nali-ip-cli)

```bash
$ nali 1.145.1.4

1.145.1.4 [澳大利亚，新南威尔士州，悉尼，澳大利亚电信]
```

## 四、Shell 脚本
当我们需要多个任务编排，重复使用这些命令集合的时候，可以写个 shell 脚本执行它们。

### 特殊变量
- `$?`: 上一个命令的退出码, 正常为 0
- `$$`: 当前 Shell 的进程 ID
- `$_`: 上一个命令的最后一个参数
- `$1`: 脚本的第一个参数（也可以用 getopts 命令解析复杂的脚本命令行参数）

### 调试排查
> 提问：以下脚本中的代码有什么问题？（不要实际尝试）

```bash
#! /bin/bash

dir_name=/path/not/exist

cd ~/Project
cd $dir_name
rm *
```

编写 Shell 脚本的时候，一定要考虑到命令失败的情况，因为默认出错后会继续向下执行。
如果目录 `$dir_name` 不存在，`cd $dir_name` 命令就会执行失败。这时，就不会改变当前目录，脚本会继续执行下去，导致 `rm` 命令删光当前目录的文件。

正确做法是，用 `[ -d file ]` 判断表达式，先判断目录是否存在：

```bash
[[ -d $dir_name ]] && cd $dir_name && rm *
```

同时可以用 `set` 命令设置 Shell 的行为参数，有利于脚本除错。比如：

```bash
#!/usr/bin/env bash
set -e

foo  # 发生错误，就会到此终止，不向下继续执行
echo bar
```

### 命令替换
将一个命令的输出，替换进入另一个命令。

```bash
$ ls -l $(which cp)

# 或者
$ ls -l `which cp`
```

### 多个命令连续执行

```bash
# 第一个命令执行完（不管成功或失败），执行第二个命令
$ command1; command2

# 只有第一个命令成功执行完（退出码0），才会执行第二个命令
$ command1 && command2

# 只有第一个命令执行失败（退出码非0），才会执行第二个命令
$ mkdir foo || mkdir bar
```

### 其他常用命令：
#### exit

```bash
# 退出值为 0（成功）
$ exit 0

# 退出值为 1（失败，走到异常逻辑）
$ exit 1
```

`exit` 与 `return` 差别是，`return` 命令是函数的退出，脚本依然执行。`exit` 是整个脚本的退出。

#### alias 别名

```bash
# 防止误删文件
$ alias rm='trash -F'

# 不加参数时，显示所有有效的别名
$ alias

# 当我们用 alias 自定义命令覆盖了原始命令，但还想用原始命令，怎么做？
# 可以在命令前加上反斜杠（"\"）来绕过别名。例如：
$ alias ping=nali-ping
$ \ping
```

#### type
Shell 可执行命令分为四种类型，可以用 `type` 命令判断命令的来源：
- 可执行程序
- Shell 提供的命令
- Shell 函数
- 前三类命令的别名

```bash
# 平常会用 which 看某个命令的路径
$ which node
/Users/anto/.nvs/default/bin/node

# 也可以用 type
$ type node
node is /Users/anto/.nvs/default/bin/node

$ type type
type is a shell builtin
```

## 五、推荐读物
- [阮一峰的《Bash 脚本教程》](https://wangdoc.com/bash/intro)
- [陈皓 - SED 简明教程](https://coolshell.cn/articles/9104.html)
- 鸟哥的 Linux 私房菜
- [精读《pnpm》](https://github.com/ascoders/weekly/blob/master/%E5%89%8D%E6%B2%BF%E6%8A%80%E6%9C%AF/253.%E7%B2%BE%E8%AF%BB%E3%80%8Apnpm%E3%80%8B.md)
- [菜鸟教程 - Shell 教程](https://www.runoob.com/linux/linux-shell-basic-operators.html)
