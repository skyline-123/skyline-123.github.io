---
title: node-delegates学习
date: 2019-12-10 21:52:54
tags:
  - Koa
  - 委托模式
---
最近在学习Koa的源码，除了middleware的思想，我发现其中用到的委托模式也是非常值得学习。说实话，源码学习分享的文章更适合自己看，因为有时候真的觉得好代码胜过千言万语分析，自己写写阅读体会供参考的价值倒还有一点，只是更推荐大家主动阅读源码，学习自己好奇的黑魔法。

## 使用
```javascript
const delegate = require('delegates');

const innerObject = {
  a: 'a',
  methodB: function (val) {
    console.log(val)
  },
  c: 'c',
  d: 'd',
  e: 'e'
}

const proto = {
  innerObject
}

delegate(proto, 'innerObject')
  .access('a')          // 把a属性的getter和setter都委托到proto对象上面
  .method('methodB')    // 把methodB方法委托倒proto对象上面
  .getter('c')          // 把c属性的getterproto对象上面
  .setter('d')          // 把d属性的setter都委托倒proto对象上面
  .fluent('e')          // 把设置e或者读取e的方法委托到proto对象上面

const q = proto.e()

proto.e(1)
  .e(2)

console.log(proto)

// 打印信息如下
// {
//   innerObject: { a: 'a', methodB: [Function: methodB], c: 'c', d: 'd', e: 2 },
//   a: [Getter/Setter],
//   methodB: [Function],
//   c: [Getter],
//   d: [Setter],
//   e: [Function]
// }
```

其实看了使用方法就知道delegates库是做什么的了，说白了就是将对象内部其他对象的属性或者方法委托到对象上，在未来使用这个对象的过程可以更加方便而不用关注对象内部的对象。
<!-- more -->
## 优点
在Koa框架层面的应用场景，其实就是将request对象和response对象上的属性方法，委托到被委托的context对象上，这样才有了类似context.URL这种方式便捷的方式去获取request或者response对象上面的属性或方法。让我总结这种委托模式的优点，主要有两点
- 代码更具表现力，可以在阅读代码的过程中知道动作
- 被委托对象更易使用

更具表现力这一点，看看如下代码：
```javascript
delegate(proto, 'innerObject')
  .access('a')          // 把a属性的getter和setter都委托到proto对象上面
  .method('methodB')    // 把methodB方法委托倒proto对象上面
  .getter('c')          // 把c属性的getterproto对象上面
  .setter('d')          // 把d属性的setter都委托倒proto对象上面
  .fluent('e')          // 把设置e或者读取e的方法委托到proto对象上面
```
其实如果了解这个delegate库的方法之后，你看到这样的链式调用代码会觉得很舒适，也很清晰知道代码要表达的含义。个人觉得，所谓代码即注释的层次已经是达到了。

## 原理
原理层面相对简单，建议自己打开源码食用[node-delegates](https://github.com/tj/node-delegates/blob/master/index.js)。链式调用的原理就是```return this```，委托的主要原理是getter和setter，源码实现方式用的api比较旧，现在可以用Object.defineProperty方法实现。

## 吹捧
TJ Holowaychuk 就是这个node-delegates库的作者，据网上信息了解，他最初是一位平面设计师，学习编程就是去阅读别人的代码，并搞清楚那些代码是如何工作的。额，TJ是我学习路上的榜样，未来要更多去学习优秀的库，提高自己的编程思维和能力！