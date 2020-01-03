---
title: gulp系列之pump源码笔记
date: 2019-12-30 10:36:18
tags:
  - gulp
---

## 前言
近期在翻阅gulp的源码，它引用到了很多外部包，不用慌，待我抽丝剥茧逐个击破。接下来会写gulp系列的源码解析，会将gulp使用的核心库逐个介绍，最后再串起来梳理gulp的流程。

选择用分总的结构讲解gulp源码，是因为总分总的意义不太大。gulp本身引用了很多第三方库，如果聚焦于其主库去说流程，实际上没有太多值得说的东西。

抛开引用的诸多库，gulp最核心的还是使用了Stream。对于Stream的概念，我就不赘述了，不熟悉的同学去回顾一下node的相关文档。这里有一篇文章值得一读 [*stream-handbook*](https://github.com/substack/stream-handbook)。

另外，源码分析的文章，我的初衷更多是总结回顾，如果同时能对读者有所帮助，也是一件好事。

## pump简介
[pump](https://github.com/mafintosh/pump)是一个小node模块，它通过管道将流连接在一起，并在其中一个关闭时销毁所有流。

首先分析pump，两点原因
1. gulp源码阅读过程中一路追溯到pump，pump算是基础。
2. pump的实现中有闭包的应用场景，值得加深对闭包的学习。

## pump源码解析
首先来看看pump库的目录结构：
```
❯ tree ./node_modules/pump
./node_modules/pump
├── LICENSE
├── README.md
├── index.js
├── package.json
├── test-browser.js
└── test-node.js

0 directories, 6 files
```
可以看出如简介所说，确实是一个小node module。源码有且只有index.js这个入口文件。

接下来，我们看看代码的框架结构。

```javascript
var once = requrie('once')
var eos = require('end-of-stream')
var fs = require('fs')

// 空函数
var noop = function () {}
// 判断process.version是否为旧版本。此处存疑，v.0或者.0开头即可，没见过这种版本号。
var ancient = /^v?\.0/.test(process.version)

// 判断入参fn是否为function
var isFn = function (fn) {
  ...
}

// 判断入参stream是否为fs相关
var isFs = function (stream) {
  ...
}

// 判断入参stream是否为request相关
var isRequest = function (stream) {
  ...
}

// 将stream destroy的函数
var destroyer = function (stream, reading, writing, callback) {
  ...
}

// 执行入参fn函数
var call = function (fn) {
  ...
}

// 返回from.pipe(to)
var pipe = function (from , to) {
  ...
}

// pump主函数
var pump = function () {
  ...
}

module.exports = pump
```

如上，将index.js文件结构梳理了一下。其中ancient这个变量我没有看懂，如果有了解的小伙伴欢迎评论探讨。接下来着重讲讲pump函数和destroyer函数。

```javascript
var pump = function () {
  var streams = Array.prototype.slice.call(arguments) // 将入参转化为streams数组
  var callback = isFn(streams[streams.length - 1] || noop) && streams.pop() || noop // callback的获取
  if (Array.isArray(streams[0])) streams = streams[0] // 兼容支持第一个参数是数组，传入所有的stream
  if (streams.length < 2) throw new Error('pump requires two streams per minimum') // 敲黑板，至少需要两个stream，否则你用pump干啥？

  var error
  var destroys = streams.map(function (stream, i) { // 通过streams.map获取destroy函数的数组，这个数组在未来的destroyer的callback中可能用到
    var reading = i < streams.length - 1 // streams没到最后一个的时候，reading状态都是true
    var writing = i > 0 // streams从第二个开始，writing状态都是true
    return destroyer(stream, reading, writing, function (err) {
      if (!error) error = err // streams共享的error值，如果不存在，先赋值
      if (err) destroys.forEach(call) // 如果存在err，将destroys数组中的每一个函数执行一遍，这个函数详情件destroyer分析
      if (reading) return // 如果没有err，而且正在reading，return
      destroys.forEach(call) // 如果reading为false，那么destroyer的回调里面，将destroys数组中的每一个函数执行一遍，这个函数详情件destroyer分析
      callback(error) // 最后，执行用户传进来的callback参数
    })
  })

  streams.reduce(pipe) // 通过数组的reduce方法，逐步执行Stream.pipe
}
```

接下来我们重点看看destroyer函数
```javascript
var destroyer = function (stream, reading, writing, callback) {
  callback = once(callback) // 保证callback只执行一次，once来自依赖，感兴趣可以自行查看

  var closed = false  // 标志stream是否已经close
  stream.on('close', function () {
    closed = true // 触发close事件，将closed设置为true
  })

  // eos依赖简介：A node module that calls a callback when a readable/writable/duplex stream has completed or failed.
  eos(stream, {readable: reading, writable: writing}, function (err) {
    if (err) return callback(err)
    closed = true
    callback()
  })

  var destroyed = false // 标志stream是否已经destroy
  return function (err) {
    if (closed) return // 如果closed，return
    if (destroyed) return // 如果destroyed，return
    destroyed = true // 将destroyed设置为true，防止重复执行下方逻辑

    // 根据stream的类型，destroy
    if (isFS(stream)) return stream.close(noop) // use close for fs streams to avoid fd leaks
    if (isRequest(stream)) return stream.abort() // request.destroy just do .end - .abort is what we want

    if (isFn(stream.destroy)) return stream.destroy()
    // 如果stream不是以上类型，调用callback返回error
    callback(err || new Error('stream was destroyed'))
  }
}
```

源码已经分析完了，我们梳理一下核心主流程，分支情况请忽略。
1. 入参传多个stream或者streams数组，最后一个参数传入callback到pump函数。
2. pump解析参数，同时通过streams.reduce(pipe)将多个stream合并成一个stream。
3. 在第二步的过程中，根据stream生成destroys数组，此数组包含了调用destroyer函数返回的一个匿名函数，该匿名函数会销毁当前的stream。
4. eos执行过程中，如果遇到错误, 执行callback(err)，即destroys.forEach(call)。这样pump函数入参的所有stream进行销毁。

好的，分析完毕。其中涉及到比较多闭包的应用，比如在destroyer函数中closed和destroyed变量，就是在返回的匿名函数中可以访问。再比如pump主函数中，destroys变量，return的destroyer函数执行过程中的callback中可以访问。对于未来需要访问的时候就能自由访问，这就是闭包的一个非常具体的学习例子。

这个源码分析还是比较简单，其中稍微有点难的点，是下方的代码：
```javascript
var destroys = streams.map(function (stream, i) { // 通过streams.map获取destroy函数的数组，这个数组在未来的destroyer的callback中可能用到
  var reading = i < streams.length - 1 // streams没到最后一个的时候，reading状态都是true
  var writing = i > 0 // streams从第二个开始，writing状态都是true
  return destroyer(stream, reading, writing, function (err) {
    if (!error) error = err // streams共享的error值，如果不存在，先赋值
    if (err) destroys.forEach(call) // 如果存在err，将destroys数组中的每一个函数执行一遍，这个函数详情件destroyer分析
    if (reading) return // 如果没有err，而且正在reading，return
    destroys.forEach(call) // 如果reading为false，那么destroyer的回调里面，将destroys数组中的每一个函数执行一遍，这个函数详情件destroyer分析
    callback(error) // 最后，执行用户传进来的callback参数
  })
})
```
一开始看到destroys赋值之后并没有在后面用到，觉得代码有问题，仔细一看，其实它在destroyer函数的callback中有用到。这段逻辑个人感觉会有点绕，如果不认真看。

Ok，今天，我们就开心的分析完pump的源码啦，写代码的想象力又可以更丰富一点！
