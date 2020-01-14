---
title: gulp系列之undertaker源码笔记
date: 2020-01-13 20:32:19
tags:
  - gulp
---

undertaker部分的代码，是gulp任务管理部分的核心源码。在讲解gulp本身仓库源码之前，我着重讲讲undertaker。这个库自然也涉及到一些依赖，没关系，这次我们关注undertaker流程本身。

请泡好🍵或者☕️慢慢看，这篇文章篇幅较长。算是回应某些朋友对我的博客期待吧，他们觉得我一开始写的文章太水了。

## undertaker简介
undertaker是gulp团队开发出来为解决gulp的任务执行问题的库。
> Task registry that allows composition through series/parallel methods.

undertaker提供series和parallel两种方法解决任务流程管理。series是串行执行任务，parallel方法是并行执行任务。串行执行应该比较简单理解，就是按顺序执行，只有前面的任务完成，后面的任务才能执行。那么并行呢？很可能是通过for循环执行所有异步任务，这样就能实现并行，这个类似Promise.all方法。

我非常赞叹undertaker的设计，可能是我对代码流程管理本身没有很好的思路，看了之后豁然开朗。想想javascript，异步代码从callback，写到promise，写到generator，写到async/await。异步代码的书写随着时间的推移变得越来越友好，但是比如Promise的实现思想，更加值得我们学习和思考。回到undertaker，这个任务管理的框架，其实就是非常友好地解决了处理串行或者并行任务的代码组织问题。通过undertaker书写的代码可读性和可维护性都很强。

之前写到复杂业务流程的时候，串行或者并行的时候，直接用Promise，总感觉有所欠缺，现在看了undertaker，有一种亲切的感觉，那就是这种实现就是我写流程管理需要的非常好的实践模板。

<!-- more -->

## undertaker用法
### 用法
这里就不废话了，直接复制undertaker的readme:

```js
var fs = require('fs');
var Undertaker = require('undertaker');

var taker = new Undertaker();

taker.task('task1', function(cb){
  // do things

  cb(); // when everything is done
});

taker.task('task2', function(){
  return fs.createReadStream('./myFile.js')
    .pipe(fs.createWriteStream('./myFile.copy.js'));
});

taker.task('task3', function(){
  return new Promise(function(resolve, reject){
    // do things

    resolve(); // when everything is done
  });
});

taker.task('combined', taker.series('task1', 'task2'));

taker.task('all', taker.parallel('combined', 'task3'));
```

在分析源码之前，我简单从用法这里说说undertaker。

简单总结方法如下：
- Undertaker.task
- Undertaker.series
- Undertaker.parallel

Undertaker.task看上去是注册task，每个task name对应一个需要执行的方法。同时，Undertaker.task应该也会返回一个函数，以供其嵌套使用。Undertaker.series和Undertaker.parallel就从名字上可以看出是串行和并行执行任务的方法，同时也推断出会返回一个函数。

### 扩展
既然对于用法我做了简单的分析，也稍微扩展一下源码阅读的问题。专注undertaker源码解析的同学可以**跳过**这部分。

有同事不久前问我，需要怎么阅读源代码。其实这个方法因人而异，每个人在探索的过程中肯定都会得出适合自己的方法。我的方法就是先根据api或者说usage去揣测这个库的可能实现，尤其关注参数和输出。

通过自己的思考反推实现，是一个非常好的思维锻炼过程。你可能会发现自己能猜测出大概的实现思路，又或者毫无头绪，这个前置的思考和预习功课有着类似的功效，就是能帮你很好的衔接实际阅读源码过程中的思索。

也有同事曾经问我，如果有很多依赖，要怎么阅读源码？这个其实是阅读源码过程中肯定会遇到的问题，这个很简单，直接去依赖包中看readme，了解简介知道它做什么就足够了，然后注意力集中在主流程。如果主流程的关键代码就在依赖中，那也很简单啊，看依赖的源码就完事了。如果依赖多到记不清每个依赖是做什么的，那你又想研究，就做一个表格把核心依赖的描述都登记起来用来查询就好。

## undertaker主流程分析
### 目录
老办法，先看看目录结构。注：为了方便阅读，直接取node_modules里面的依赖代码，忽略test文件。
```
./node_modules/undertaker
├── LICENSE
├── README.md
├── index.js
├── lib
│   ├── get-task.js
│   ├── helpers
│   │   ├── buildTree.js
│   │   ├── createExtensions.js
│   │   ├── metadata.js
│   │   ├── normalizeArgs.js
│   │   └── validateRegistry.js
│   ├── last-run.js
│   ├── parallel.js
│   ├── registry.js
│   ├── series.js
│   ├── set-task.js
│   ├── task.js
│   └── tree.js
└── package.json

2 directories, 17 files
```

从api出发，我就很快找到了我感兴趣的文件。
- set-task.js
- get-task.js
- task.js
- parallel.js
- series.js

文件很多，但是我大胆猜测上面五个文件是我需要重点关注的，其他文件都是辅助参考。反正猜错了也不亏，😄。

### 入口文件(index.js)
猜测归猜测，学习还是得老老实实从入口文件看起。
```js
'use strict';

var inherits = require('util').inherits;
var EventEmitter = require('events').EventEmitter;

var DefaultRegistry = require('undertaker-registry');

var tree = require('./lib/tree');
var task = require('./lib/task');
var series = require('./lib/series');
var lastRun = require('./lib/last-run');
var parallel = require('./lib/parallel');
var registry = require('./lib/registry');
var _getTask = require('./lib/get-task');
var _setTask = require('./lib/set-task');

function Undertaker(customRegistry) {
  EventEmitter.call(this); // 继承EventEmitter

  this._registry = new DefaultRegistry(); // 猜测是存储tasks的对象
  if (customRegistry) { // 如果有customRegistry，应用customRegistry
    this.registry(customRegistry);
  }

  this._settle = (process.env.UNDERTAKER_SETTLE === 'true'); // 姑且看作某个叫做UNDERTAKER_SETTLE的开关，具体作用要看后面的源码
}

inherits(Undertaker, EventEmitter); // 继承EventEmitter原型链方法


Undertaker.prototype.tree = tree;

Undertaker.prototype.task = task;

Undertaker.prototype.series = series;

Undertaker.prototype.lastRun = lastRun;

Undertaker.prototype.parallel = parallel;

Undertaker.prototype.registry = registry;

Undertaker.prototype._getTask = _getTask;

Undertaker.prototype._setTask = _setTask;

module.exports = Undertaker;
```

入口文件来看
1. Undertaker继承自EventEmitter
2. 同时内部有一个_registry对象（可以自定义，猜测是用来存储tasks的对象)
3. prototype上面绑定了很多方法或者属性，还是按照api倒推的方法，我挑选task, _getTask, _setTask, series, parallel重点关注

### task(lib/task.js)
首先看看这个task，到底是如何实现的呢？
```js
'use strict';

function task(name, fn) {
  if (typeof name === 'function') { // 如果参数只是一个函数，自动获取Function.displayName或者Function.name作为name
    fn = name;
    name = fn.displayName || fn.name;
  }

  if (!fn) {
    return this._getTask(name); // 如果参数没有传入fn，用_getTask获取name
  }

  this._setTask(name, fn); // 如果name和fn都有，那么用_setTask设置，估计是存储到registry对象
}

module.exports = task;

```
总结一下，task方法其实就是封装了Undertaker.prototype._getTask和Undertaker.prototype._setTask方法。

### get-task(lib/get-task.js)
继续看看这个get-task，👌，看名字，大家应该都猜得到大概了。
```js
'use strict';

function get(name) {
  return this._registry.get(name);
}

module.exports = get;
```
这个就不用解释了，和猜想一致，_registry是用来存储tasks的对象。_getTask就是获取对应的task。

### set-task(lib/set-task.js)
顺势看到set-task，耐心点继续看看：
```js
'use strict';

var assert = require('assert');

var metadata = require('./helpers/metadata');

function set(name, fn) {
  assert(name, 'Task name must be specified'); // 参数检验
  assert(typeof name === 'string', 'Task name must be a string'); // 参数检验
  assert(typeof fn === 'function', 'Task function must be specified'); // 参数检验

  function taskWrapper() {
    return fn.apply(this, arguments); // 执行task，绑定this, 执行taskWrapper，等于执行绑定了当前上下文的fn
  }

  function unwrap() {
    return fn; // 直接返回fn
  }

  taskWrapper.unwrap = unwrap; // 把unwrap绑定到taskWrapper函数的属性
  taskWrapper.displayName = name; // 把name绑定到taskWrapper的displayName属性。不过现在displayName已经是非标准的属性了，https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/displayName

  var meta = metadata.get(fn) || {}; // 咋一看不知道干啥的，不怕，我直接跳过
  var nodes = []; // 咋一看不知道干啥的，不怕，我直接跳过
  if (meta.branch) {
    nodes.push(meta.tree); // // 咋一看不知道干啥的，不怕，我直接跳过
  }

  var task = this._registry.set(name, taskWrapper) || taskWrapper; // 通过_registry设置name属性为taskWrapper，如果返回为空，task使用taskWrapper替代

  metadata.set(task, {
    name: name,
    orig: fn,
    tree: {
      label: name,
      type: 'task',
      nodes: nodes,
    },
  }); // 咋一看不知道干啥的，不怕，我直接跳过。但是我可以猜测metadata是保存一些元数据信息，姑且猜测是用来获取task之间的依赖关系。
}

module.exports = set;
```

_setTask方法稍微复杂一点，多了metadata相关的东西。不知道具体做什么的，没什么关系，不影响我们专注看自己感兴趣的主流程。敲黑板，看源码看到大概率是分支流程的东西，千万别慌，大概率是可以不关注也不影响主流程的内容。

### series(lib/series.js)
好的，总算来到核心代码部分。话不多说，看代码：
```js
'use strict';

var bach = require('bach');

var metadata = require('./helpers/metadata');
var buildTree = require('./helpers/buildTree');
var normalizeArgs = require('./helpers/normalizeArgs');
var createExtensions = require('./helpers/createExtensions');

function series() {
  var create = this._settle ? bach.settleSeries : bach.series; // 根据UNDERTAKER_SETTLE开关选择create函数，这里涉及到bach库，看上去series的核心方法和它有关，怎么办？待会直接翻出来看。

  var args = normalizeArgs(this._registry, arguments); // 将arguments扁平化之后获取对应的task function。helpers里面的源码比较简单，不细讲。
  var extensions = createExtensions(this); // 创建扩展，包含create, before, after, error生命周期方法
  var fn = create(args, extensions); // 待会讲解bach的时候分析create方法
  var name = '<series>';

  metadata.set(fn, { // 咋一看不知道干啥的，不怕，我直接跳过。但是我可以猜测metadata是保存一些元数据信息。
    name: name,
    branch: true,
    tree: {
      label: name,
      type: 'function',
      branch: true,
      nodes: buildTree(args),
    },
  });
  return fn; // 返回create之后的函数
}

module.exports = series;
```

### bach库
OK，我们看源码到series方法的时候，发现被bach库的源码阻塞了，而且显然是核心逻辑。不用犹豫，直接深入进入了解下原理。从节奏上来说，这部分分析理应拆多一篇文章。但是为了分享自己阅读源码时候的思路和细节，不妨直接贯穿下来。

同样的，我们看下bach的简介和用法：

> Compose your async functions with elegance.

```js
var bach = require('bach');

function fn1(cb) {
  cb(null, 1);
}

function fn2(cb) {
  cb(null, 2);
}

function fn3(cb) {
  cb(null, 3);
}

var seriesFn = bach.series(fn1, fn2, fn3);
// fn1, fn2, and fn3 will be run in series
seriesFn(function(err, res) {
  if (err) { // in this example, err is undefined
    // handle error
  }
  // handle results
  // in this example, res is [1, 2, 3]
});

var parallelFn = bach.parallel(fn1, fn2, fn3);
// fn1, fn2, and fn3 will be run in parallel
parallelFn(function(err, res) {
  if (err) { // in this example, err is undefined
    // handle error
  }
  // handle results
  // in this example, res is [1, 2, 3]
});
```

其实看了简介和用法就知道了，bach就是负责undertaker的任务串行或者并行执行的核心库。

接下来看看目录结构：
```
./node_modules/bach
├── LICENSE
├── README.md
├── index.js
├── lib
│   ├── helpers.js
│   ├── parallel.js
│   ├── series.js
│   ├── settleParallel.js
│   └── settleSeries.js
└── package.json

1 directory, 9 files
```
入口文件就不说了，只是暴露lib里面的函数。
我们先讲series.js和settleSeries.js，这个和前面的series(lib/series.js)关联最大。

series.js如下：
```js
'use strict';

var initial = require('array-initial');
var last = require('array-last');
var asyncDone = require('async-done');
var nowAndLater = require('now-and-later');

var helpers = require('./helpers');

function iterator(fn, key, cb) {
  return asyncDone(fn, cb); // 通过callback方式统一处理异步函数
}

function buildSeries() {
  var args = helpers.verifyArguments(arguments); // 验证arguments

  var extensions = helpers.getExtensions(last(args)); // 获取extensions参数

  if (extensions) {
    args = initial(args); // 如果有extensions，arguments为排除extensions的那些
  }

  function series(done) {
    nowAndLater.mapSeries(args, iterator, extensions, done); // 通过nowAndLater真正实现series串行执行
  }

  return series; // 返回series供使用
}

module.exports = buildSeries;

```

通过阅读series.js方法，发现离series串行执行的谜底还是隔着一层nowAndLater的面纱。不知道你有没有觉得要失去信心了，我反正是没有，因为还没打破沙锅问到底，不能轻易放弃。我阅读源码的过程如今在这里分享出来，带你克服内心恐惧。咬咬牙，继续看看nowAndLater是何方神圣。

### now-and-later库

事到如今，已经不想看nowAndLater的简介了，我只知道，他的mapSeries和series的谜底很靠近了额，让我们直接看看mapSeries.js的代码结束战斗！
```js
'use strict';

var once = require('once');

var helpers = require('./helpers');

function mapSeries(values, iterator, extensions, done) {
  // Allow for extensions to not be specified
  if (typeof extensions === 'function') {
    done = extensions;
    extensions = {};
  }

  // Handle no callback case
  if (typeof done !== 'function') {
    done = helpers.noop;
  }

  done = once(done);

  // Will throw if non-object
  var keys = Object.keys(values); // 获取values的key值数组
  var length = keys.length;
  var idx = 0;
  // Return the same type as passed in
  var results = helpers.initializeResults(values); // 初始化result

  var exts = helpers.defaultExtensions(extensions); // 获取生命周期相关的集合

  if (length === 0) {
    return done(null, results);
  }

  var key = keys[idx]; // 递归开始的key
  next(key); // 开始执行next

  function next(key) {
    var value = values[key]; // 获取当前value

    var storage = exts.create(value, key) || {}; // 执行exts的create生命周期

    exts.before(storage); // 执行exts的before的生命周期
    iterator(value, key, once(handler)); //执行iterator，同时传递只执行一次的handler作为回调

    function handler(err, result) {
      if (err) {
        exts.error(err, storage); // 如果报错，执行error的生命周期
        return done(err, results); // 执行done函数，同时结束递归
      }

      exts.after(result, storage); // 如果没有err，则执行生命周期的after钩子
      results[key] = result; // results中保存result

      if (++idx >= length) {
        done(err, results); // 如果结束了，调用done
      } else {
        next(keys[idx]); // 如果没有结束，调用next，继续递归操作
      }
    }
  }
}

module.exports = mapSeries;
```

呼，松了一口气，bach和nowAndLater连起来看，总算清晰串行执行的原理了。同时有一个async-done库我只是一笔带过，感兴趣可以自己看看，它把不同风格的异步代码统一成callback的方式。
```js
function iterator(fn, key, cb) {
  return asyncDone(fn, cb); // 通过callback方式统一处理异步函数
}
```
有了这一步，iterator在执行过程中，才能保证异步函数成功之后执行下一个函数。这就是一整个串行执行的原理了。

### 接下来
接下来是不是轮到要讲解parallel的实现了呢？😄，都已经看我写到这里了，是不是有思路去自己阅读parallel的实现了呢？加油，相信你可以做到的。

简单来说，parallel把所有需要执行的函数都改成异步执行的方式，通过for循环去执行，就做到了并行执行代码。

同时如果你对最开始我没有详细讲解的metadata有兴趣，也可以自己去了解。不是偷懒，开头已经说了，我们关注undertaker流程本身，对于分支流程，有余力的同学自己探索吧。

## 总结
undertaker这个任务管理库非常优雅，可以很好的组合异步任务，通过并行或者串行的方式执行一个任务流程。我也曾经遇到相对复杂的流程控制代码，当时写完之后是一个头两个大。undertaker通过很多黑魔法，抹平多个异步任务组合的问题，让我们更加专注管理流程本身，而不是关注于怎么解决各种异步代码组合。

这篇文，一半是为了总结undertaker的源码，一半是分享下我的阅读源码思路。我也一样会面对下面的问题。

- 依赖非常多
- 分支代码看不懂
- 阅读过程忘记前面的代码

相信这些问题的答案，认真看过本文的你都能找到一些参考和解答。希望新的一年我们永远年轻，永远乐于探索，善于学习总结！
