---
title: gulp系列之through2源码笔记
date: 2020-01-03 15:41:57
tags:
  - gulp
---

## through2简介
仓库简介：A tiny wrapper around Node.js streams.Transform (Streams2/3) to avoid explicit subclassing noise. 简单来说它就是用readable-stream库创建transform stream。如果使用官方的Stream接口，还要考虑兼容性问题。

<!-- more -->

## through2用法
具体用法直接看through2的[github](https://github.com/rvagg/through2), 另外我在阅读过程中也顺便提交PR修复了readme.md中遗漏的括号。

through2函数：```through2([ options, ] [ transformFunction ] [, flushFunction ])```

options就是和ransform stream相关的配置项。

transformFunction，简单概括，它就是连接readable stream和writable stream的桥梁，它改变可读流之后供可写流消费。这个transformFunction也是duplex stream和transform stream最大的区别。

flushFunction将在所有输入流传输结束后调用，用来结束transform stream。

through2一般用在pipe中，抄官方demo如下作参考：

```js
fs.createReadStream('ex.txt')
  .pipe(through2(function (chunk, enc, callback) {
    for (var i = 0; i < chunk.length; i++)
      if (chunk[i] == 97)
        chunk[i] = 122 // swap 'a' for 'z'

    this.push(chunk)

    callback()
   }))
  .pipe(fs.createWriteStream('out.txt'))
  .on('finish', () => doSomethingSpecial())
```

## 源码简介
如果你深入了解过Stream，看了through2的用法，其实已经可以猜测到它源码是如何工作的。

为什么这么说呢？因为我学习这块的时候直接看through2的源码，发现自己其实是不理解pipe和through2结合起来，怎么就可以做到这种修改可读流数据的黑魔法。说白了是对pipe方法的了解很模糊，Stream的pipe方法，其实只需要传一个dest流作为参数，它内部已经实现了对dest流的data和end事件的监听。

抛开源码去理解，pipe管道只是一个黑盒，传入一个transfrom / duplex / writable stream作为pipe的参数，源头的readable stream可以自动从源头流到出口writable stream，并通过执行每一个pipe方法对stream做相应的处理。

回到through2，它就是帮助你快速创建一个transform stream的工具函数。

看核心源码：
```js
// main export, just make me a transform stream!
module.exports = through2(function (options, transform, flush) {
  var t2 = new DestroyableTransform(options)

  t2._transform = transform

  if (flush)
    t2._flush = flush

  return t2
})
```
可以看到through2确实只是返回一个DestroyableTransform流。
那我们看看DestroyableTransform。
```js
var Transform = require('readable-stream').Transform
  , inherits  = require('util').inherits
  , xtend     = require('xtend')

function DestroyableTransform(opts) {
  Transform.call(this, opts)
  this._destroyed = false
}

inherits(DestroyableTransform, Transform)

DestroyableTransform.prototype.destroy = function(err) {
  if (this._destroyed) return
  this._destroyed = true

  var self = this
  process.nextTick(function() {
    if (err)
      self.emit('error', err)
    self.emit('close')
  })
}
```

继承readable-stream的Transform，然后原型链上增加一个destroy方法。非常简单。
核心源码部分就分析到此。

## 小结
这篇源码笔记看上去没怎么讲解through2，因为它本身确实没啥可以讲的，只是创建transform stream都会用到它处理。

我在研读它之前的困惑，不在于这个库本身，而是对Stream的理解有点模糊。先将理解记录下来，供日后参考。
