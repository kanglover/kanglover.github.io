---
layout:     post
title:      Nodejs-Process
date:       2022-11-01
author:     shaokang
catalog: true
tags:
    - Node
---

## process 对象
官网解释：process 对象是一个全局变量，它提供当前 Node.js 进程的有关信息，以及控制当前 Node.js 进程。 因为是全局变量，所以无需使用 require()。

而 process 通常用于：

获取进程信息（资源使用、运行环境、运行状态）
执行进程操作（监听事件、调度任务、发出告警）

## 常用属性
### process.env
process.env 返回一个包含用户环境信息的对象，可以拿到用户环境的一些环境变量。

process.env 允许用户对当前环境变量进行添加、删除操作，所以我们可以在process.env上挂载一些变量标识当前的环境，比如最常见的用 process.env.NODE_ENV 区分 development 和 production，但其实process.env 中并不存在 NODE_ENV 这个东西，NODE_ENV 是用户自定义的一个环境变量。

不同的环境有不同的修改方式：
windows
```bash
#node中常用的到的环境变量是NODE_ENV，首先查看是否存在 
set NODE_ENV 

#如果不存在则添加环境变量 
set NODE_ENV=production 

#环境变量追加值 set 变量名=%变量名%;变量内容 
set path=%path%;C:\web;C:\Tools 

#某些时候需要删除环境变量 
set NODE_ENV=

```
linux或者mac
```bash
#node中常用的到的环境变量是NODE_ENV，首先查看是否存在
echo $NODE_ENV

#如果不存在则添加环境变量
export NODE_ENV=production

#环境变量追加值
export path=$path:/home/download:/usr/local/

#某些时候需要删除环境变量
unset NODE_ENV

#某些时候需要显示所有的环境变量
env
```

直接修改 package.json 文件
```json
"scripts": {
  "dev": "NODE_ENV=development webpack-dev-server",
  "build": "NODE_ENV=production webpack",
   // 跨平台可以使用 cross-env
   "dev": "cross-env NODE_ENV=development webpack-dev-server"
},
```

如果需要自定义的参数很多，我们可以创建.env文件来添加自定义的环境变量。
通过 dotenv 依赖包可以很方便的将其加到 process.env 下
```js
require('dotenv').config();
const xxx = process.env.xxx;
```

### process.argv
process.argv 属性返回一个数组，这个数组包含了启动Node.js进程时的命令行参数。

主要用于在终端通过 Node 执行命令的时候，通过 process.argv 可以获取传入的命令行参数，返回值是一个数组，数组组成如下：

第一个元素：process.execPath，即启动 Node.js 进程的可执行文件所在的绝对路径。
第二个元素：当前执行的 JavaScript 文件路径。
其余元素：其他命令行参数。


当我们想要拿到传入的命令行的参数时，使用slice方法直接截取就行。
```js
const args = process.argv.slice(2)
```

我们可以借助一些工具来解析，常见的解析工具有optimist、yargs、minimist、mri等（optimist 和 yargs 内部使用的解析引擎是minimist）。

### 生命周期事件: process.env.npm_lifecycle_event
判断当该文件执行时所处哪个生命周期事件
```js
const ENVIRONMENT = process.env.npm_lifecycle_event;
// npm run build
if (ENVIRONMENT === "build") {
    console.log("Running your build tasks!");
}
// npm run dev
if ( ENVIRONMENT === "dev") {
    console.log("Running the dev server!");同
}
```

### process.env.npm_package_xxx
通过 process.env.npm_package_[name] 拿到 package.json 中的 [name] 字段

```json
{
  "name": "foo",
  "version": "1.2.5",
  "scripts": {
    "view": "node view.js"
  }
}
```
```js
console.log(process.env.npm_package_name); // foo
console.log(process.env.npm_package_version); // 1.2.5
```

## 常见方法
### process.cwd()
process.cwd() 返回的是当前 Node.js 进程执行时的工作目录，process.cwd() 与 __dirname 的区别在于 __dirname 返回源代码所在的目录。