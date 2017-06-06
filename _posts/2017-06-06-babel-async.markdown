---
layout: post
title: 以 async/await 为例，说明 babel 插件怎么搭
date: 2017-06-06 12:43:23
author: sabakugaara
comments: true
---

## 你一定碰到过这些库 

### babel-polyfill

项目地址：https://github.com/babel/babel/blob/master/packages/babel-polyfill

通过两个依赖实现功能
- `core-js/shim` 提供 ES5/6/7 标准方法的实现
- `regenerate-runtime`  提供 async 语法编译后的的运行时环境（下文会专门说明）

### babel-plugin-transform-runtime

项目地址：https://github.com/babel/babel/blob/master/packages/babel-plugin-transform-runtime

开发 ES6/7 新特性的库推荐使用该插件，需要注意的是，安装时，必须同时安装 `babel-runtime` 作为依赖：
```
npm install --save-dev babel-plugin-transform-runtime
npm install --save babel-runtime // `babel-plugin-transform-runtime` 插件本身其实是依赖于 `babel-runtime` 的，但为了适应 `npm install --production` 强烈建议添加该依赖。
```

插件会将 es6/7 的语法转化为 es5 兼容的格式，并提供运行时依赖。什么是运行时依赖？比如你要用 Array.from 方法，该方法的具体实现必须在代码的执行环境中提供，这就是运行时依赖。

该插件在转化语法时，不会污染全局环境。而 `babel-polyfill` 则会污染全局环境。

### babel-plugin-external-helpers

项目地址：https://github.com/babel/babel/blob/master/packages/babel-plugin-external-helpers/

代码很少，只依赖于 `babel-runtime`。相比较 `babel-plugin-transform-runtime` 会在每个模块注入运行时代码，该插件会将运行时代码打包，类似封装到一个对象下，这样避免注入重复的代码。

## 让 async/await 跑起来 

通过最简单的一个函数：

```
async function foo() {
  return await 1
}

foo().then(function(val) {
  console.log(val)  // should output 1
})
```

说明这些 babel 插件怎么搭配，三种方案：

#### 方案一：`regenerator`
**.babelrc** 如下配置：
```
{
  "plugins": ["transform-runtime", "babel-plugin-transform-regenerator", "babel-plugin-transform-es2015-modules-commonjs"]
}
```
- babel-plugin-transform-regenerator 将 `async/await` 语法转化成 [regenerator](https://github.com/facebook/regenerator) 库支持的语法
- transform-runtime 将运行时注入，类似：`import regenerator from 'babel-runtime/regenerator'`
- babel-plugin-transform-es2015-modules-commonjs 只是为了将 import 转化为 require，便于在 node.js 模块下执行（如果你的执行环境支持 es6 的模块机制，则不需要该插件）。

#### 方案二：`generator`
这种方式，最适合 node.js 环境，node.js 最早从 0.11 开始，便支持 generator。**.babelrc** 如下配置：
```
{
  "plugins": ["babel-plugin-transform-async-to-generator"]
}
```
生成的代码，在 node.js 环境下可以直接执行，此时便不再需要 babel 提供任何有关 generator 相关的运行时环境了，直接 node.js 自带~

#### 方案三：`babel-polyfill`
**.babelrc** 如下配置：
```
{
  "plugins": ["babel-plugin-transform-regenerator"]
}
```
其实和前面 `regenerate` 一样，去掉了 `runtime` 配置。编译结束后，需要手动在结果文件的第一行加入：
```
require('babel-polyfill')
```
通过 `babel-polyfill` 向全局注入运行时依赖。那什么时候该用 `babel-polyfill` 什么时候用 `babel-runtime`？官网给出了解释：
>This will emulate a full ES2015+ environment and is intended to be used in an application rather than a library/tool.

- 如果是应用级别的开发，可以考虑使用 `babel-polyfill`：大而全，支持所有的 es2015+ 特性。可以在项目入口处统一添加，也可以通过打包工具配置入口。
- 如果是开发一个库，使用 `babel-runtime`，不会污染全局，而且是根据情况注入需要的运行时环境。

关于 babel-runtime 更多细节，强烈建议阅读官方文档：https://github.com/babel/babel/blob/master/packages/babel-plugin-transform-runtime/README.md

#### 别忘了 externel-helpers

刚刚只是一个简单的 foo 函数，一个文件。多个文件时，存在每个文件都注入类似 `asyncToGenerator` 等辅助方法，导致重复。举例说明：

foo.js

```
'use strict'

const bar = require('./bar')

async function foo () {
  const val = await bar()
  console.log(val)
}

foo()
```
bar.js
```
'use strict'

module.exports = async function bar () {
  return await 'bar'
}
```
采用前文提到的 `generator` 方式，去编译，会发现结果文件中，都有 `_asyncToGenerator` 定义。修改 **.babelrc** 如下：
```
{
  "plugins": ["babel-plugin-transform-async-to-generator", "babel-plugin-external-helpers"]
}
```
再编译，`_asyncToGenerator` 都变成了 `babelHelpers.asyncToGenerator`。这样，多个模块之间没有重复的代码注入，更加干净清爽。不过此时 `babelHelpers` 是未定义，仍然需要引入运行时环境： `transform-runtime`，最终可以运行的配置如下：
```
{
  "plugins": [
    "babel-plugin-transform-async-to-generator",
    "babel-plugin-external-helpers",
    "transform-runtime",
    "babel-plugin-transform-es2015-modules-commonjs"
  ]
}
```

#### 示例代码见：https://github.com/sabakugaara/babel-example

本文整理自：https://github.com/sabakugaara/sabakugaara.github.io/issues/8
