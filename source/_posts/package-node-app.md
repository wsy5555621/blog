---
title: 打包node应用
date: 2020-03-03 13:14:15
tags:
  - node pack
  - egg
  - pkg
---

平时交付 node 应用，前后端混淆一下代码，打包成一个`docker`镜像就交付了。虽然懂的人进入 docker 下依旧能拿到所有代码，但好歹防了一手小白。突然碰到一个客户，要求应用在`windows`下使用，还`不允许`装 docker。如果进行裸部，显得十分不专业。

<!--more-->

项目中用了 egg + egg-mysql。

## 心路历程

- 如果裸部，则需要给客户安装 node 环境，安装 node 依赖（还包括内部私有的依赖），混淆代码。而客户现场又不通网，简直是地狱难度。
- `UglifyJs`不支持 ES6。项目使用 node 12 来写，直接就用了 ES6 语法。如果要使用这个混淆工具，还要配置一遍 babel，过于繁琐了。
- egg 的配置是动态载入的，混淆时一个不小心，这部分还可能出问题，很忧伤。
- 去 node 社区一番寻觅，发现有很多打包的解决方案。`pkg`是一款不错的打包工具。它支持 win、linux、macos 等平台，构建产物是对应平台下的一个可执行文件（包含了 node 环境），一键执行，十分便捷。

## 过程

### 配置

安装了`pkg`后，在 package.json 中添加相关配置。

```json
{
  "name": "app",
  "bin": "run.js",
  "pkg": {
    "assets": [
      "./config/*.js",
      "./app.js",
      "./app/**/*.js",
      "./node_modules/nanoid/**/*.js",
      "./node_modules/egg-mysql/**/*.js",
      "./node_modules/egg/**/*.js",
      "./config/manifest.json",
      "./package.json"
    ],
    "targets": ["node12-macos-x64", "node12-win-x64", "node12-linux-x64"]
  }
}
```

`bin`很重要，它是你应用对外暴露的入口。我们使用了 egg，这里需要把启动命令提出来：

```js
// run.js
const path = require("path");

require(path.join(__dirname, "/node_modules/egg-scripts/bin/egg-scripts.js"));
```

`pkg-assets/scripts`是需要的资源文件。
`pkg-targets`是配置产物运行的环境。

### 下载 node 环境的依赖

配置完了，进入项目目录输入一下命令开始构建应用。

> pkg . --out-path ./dist --debug

因为网络问题，第一步下载依赖就卡住了。
![](/post-images/pkg-1.png)

通过看`pkg`源码，它使用了一个`pkg-fetch`的包来管理依赖文件。它会去`os.homedir()/.pkg-cache/v2.6/`目录下看有没有缓存，没有就下载，有就直接使用。那么我们去[这里](https://github.com/zeit/pkg-fetch/releases)把依赖下下来，放到这个目录下就好了。记得改名适应它的匹配规则:
![](/post-images/pkg-2.png)

ps：`node12`会默认下载 latest 的版本，node 发版本了会造成缓存失效，又进行依赖下载。可以指定`node12.13.1`来使用以前下的缓存。

ps：过了几天在`su`下执行了构建，发现又要下依赖。心想这不科学啊，一番跟踪发现`su`下的`os.homedir()`和普通用户的 home 目录不同～～

### 构建出错

我们在构建时加了`--debug`，输出日志方便定位问题。

> Error! This experimental syntax requires enabling one of the following parser plugin(s): 'decorators-legacy, decorators' (14:0)
> /Users/bm/Documents/personal-project/projectprojectname/node_modules/loaders.css/loaders.css

这个错误是因为`pkg`会分析配置文件里的`dependencies`的各种包。有些前端的包含有 less 文件，有@语法，引起了这个报错。我们在上面进行构建时，没有特别指定配置文件，`pkg`则使用了默认的 package.json，由于前后端都放在一个工程开发了，这才引起了这个问题。
我们可以指定自己的配置文件，剔除前端的依赖。

### 运行出错

解决了构建时的问题，美滋滋的得到了第一个产物包。一运行，发现数据库实例没有初始化，egg 也没有报错。又经过了漫长的追踪，发现因为 plugin 是 egg 动态引入的，pkg 在构建时无法分析出这一点，所以 egg-mysql 不会自动被打入构建产物中（虽然它已经在 dependencies 中了）。我们要在`pkg/assets`中手动加入。

### 使用脚本构建

```js
// package.json中
{
  "name": "app",
  "scripts": {
    "pkg": "npm run build && node ./pkg.js",
  }
}

// pkg.js
const { exec } = require('pkg');
const { copySync, remove, pathExistsSync } = require('fs-extra');
const path = require('path');

const packageNow = async () => {
  await remove(path.join(__dirname, './dist'));
  const frontBuildPath = path.join(__dirname, './app/public');
  if (pathExistsSync(frontBuildPath)) {
    await copySync(path.join(__dirname, './app/public'), path.join(__dirname, './dist/public'));
  }
  await exec('run.js --config ./pkg.json --out-path ./dist --debug'.split(' '));
};

packageNow();

// pkg.json

{
  "name": "app",
  "bin": "run.js",
  "pkg": {
    "assets": [
      "./config/*.js",
      "./app.js",
      "./app/**/*.js",
      "./node_modules/nanoid/**/*.js",
      "./node_modules/egg-mysql/**/*.js",
      "./node_modules/egg/**/*.js",
      "./config/manifest.json",
      "./package.json"
    ],
    "targets": ["node12.13.1-macos-x64", "node12.13.1-win-x64", "node12.13.1-linux-x64"]
  },
  "dependencies": {
    "cross-env": "^5.2.0",
    "...": "^2.12.0",
  }
}
```

### 运行

我们成功的打出了包，该怎么运行呢？我们的构建产物就是一个可执行文件，和原来的启动脚本执行相同的操作，只是文件路径变了，会有一个前缀`/snapshot`。进入文件目录下，执行：

```bash
./app-macos start /snapshot/app --port=7001 --env=prod --workers=2
# 环境变量 unix export
export MYSQL_HOST=localhost ... && ./app-macos start /snapshot/app --port=7001 --env=prod --workers=2
#  win set
SET MYSQL_HOST=localhost ... && .\app-win.exe start C:\snapshot\app --title=app --workers=2 --env=prod
```

snapshot是`pkg`虚拟出来的目录结构。

## 原理

抄了一段`pkg`的打包原理：

> pkg 的打包原理简单来说，就是将 js 代码以及相关的资源文件打包到可执行文件中，然后劫持 fs 里面的一些函数，使它能够读到可执行文件中的代码和资源文件。例如，原来的 require('./a.js')会被劫持到一个虚拟目录 require('/snapshot/a.js')。

和[node-packer](https://github.com/pmq20/node-packer)的对比：
> Pkg hacked fs.* API's dynamically in order to access in-package files, whereas Node.js Compiler leaves them alone and instead works on a deeper level via libsquash. Pkg uses JSON to store in-package files while Node.js Compiler uses the more sophisticated and widely used SquashFS as its data structure.

## 参考
[Egg.js线上打包部署](https://blog.csdn.net/qq_35241223/article/details/97306900)
[例子](https://github.com/MrSmallLiu/pkg-egg-example)
[node打包讨论帖子](https://cnodejs.org/topic/5bc712ae37a6965f59052301)
[http://enclose.io/](http://enclose.io/)

## 结尾

看看这文章，吧唧吧唧就这么点，也不是很困难嘛。可是我回想起捣鼓的这一天，一步一步调试的绝望和挣扎，就在质疑当时的自己，搞什么 egg，搞事情。以防遗忘，记录一下。

顺便说一句，总有大牛还能反编译出咱的代码～～