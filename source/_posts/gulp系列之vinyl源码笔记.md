---
title: gulp系列之vinyl源码笔记
date: 2020-01-07 11:18:46
tags:
  - gulp
---

## vinyl简介
vinyl是gulp团队维护的一个虚拟文件格式库。Vinyl对象，主要包括path和contents属性，是一个描述文件的元数据对象。

## vinyl用法
```js
var Vinyl = require('vinyl');

var jsFile = new Vinyl({
  cwd: '/', 工作目录
  base: '/test/', base路径
  path: '/test/file.js', //
  contents: new Buffer('var x = 123')
});
```
<!-- more -->
## vinyl源码
总共300多行，老方法，我们先看看代码总体的结构：

```js
构造函数
function File (file) {
  ...
}
File.prototype.isBuffer = function () {
  ...
};
File.prototype.isStream = function () {
  ...
};
File.prototype.isNull = function () {
  ...
};
File.prototype.isDirectory = function () {
  ...
};
File.prototype.isSymbolic = function () {
  ...
};
File.prototype.clone = function () {
  ...
};
File.prototype.inspect = function () {
  ...
};
...
File.isCustomProp = function (key) {
  ...
};
File.isVinyl = function (file) {
  ...
};
Object.defineProperty(File.prototype, 'contents', {
  ...
});
Object.defineProperty(File.prototype, 'cwd', {
  ...
});
Object.defineProperty(File.prototype, 'base', {
  ...
});
Object.defineProperty(File.prototype, 'relative', {
  ...
});
Object.defineProperty(File.prototype, 'dirname', {
  ...
});
Object.defineProperty(File.prototype, 'basename', {
  ...
});
Object.defineProperty(File.prototype, 'stem', {
  ...
});
Object.defineProperty(File.prototype, 'extname', {
  ...
});
Object.defineProperty(File.prototype, 'path', {
  ...
});
Object.defineProperty(File.prototype, 'symlink', {
  ...
});

module.exports = File;
```

我就不一一注释了，具体的作用其实名字已经很清晰了
原型对象上的方法总结下：
- isBuffer
- isSream
- isNull
- isDirectory
- isSymbolic
- clone
- inspect

通过Object.defineProperty定义的属性
- contents 文件内容
- cwd 工作目录
- base 基准路径
- relative 相对路径
- dirname 文件夹名称
- basename 文件名称（包括扩展名）
- stem 文件名
- extname 扩展名
- path 完整路径
- symlink 软链

## vinyl总结
正如简介所说，这个库最重要的就是path和contents属性。整体实现也是相对简单，这里就不赘述了。对于重要但是简单的库，我这边会简单介绍，仅供讲解gulp主流程 的时候回顾参考。
