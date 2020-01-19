---
title: 多包管理器Lerna实践总结
date: 2019-11-20 00:54:12
tags:
 - 前端工程化
---
## 简介
什么是Lerna？截取github上面的description如下：
> A tool for managing JavaScript projects with multiple packages.

## monorepo
最近工作中维护多个项目，均用到Lerna这个多包管理器。通过使用Lerna, 感受了monorepo维护项目的好处。

对于monorepo的介绍，请[点击此处](https://trunkbaseddevelopment.com/monorepos/)

目前出名的前端开源项目也有很多使用了Lerna去维护。

- [React](https://github.com/facebook/react)
- [Babel](https://github.com/babel/babel)
- [Ember](https://github.com/emberjs/ember.js)
<!-- more -->
其中Babel仓库还专门写了monorepo的优缺点。
摘录如下：

**Pros:**

 * Single lint, build, test and release process.
 * Easy to coordinate changes across modules.
 * Single place to report issues.
 * Easier to setup a development environment.
 * Tests across modules are run together which finds bugs that touch multiple modules more easily.

**Cons:**

 * Codebase looks more intimidating.
 * Repo is bigger in size.
 * [Can't `npm install` modules directly from GitHub](https://github.com/npm/npm/issues/2974)
 * ???

Babel的文档已经把monorepo的好处总结的非常到位了，而monorepo的缺点，也是瑕不掩瑜。

Codebase looks more intimidating.这一个缺点确实一开始令我头疼，但是意识到monorepo的优点（也就是它解决了其他更严重的问题）之后，我就开始慢慢接受并习惯。

## Lerna做了什么
聊完monorepo，我们来看看Lerna帮我们做了哪些事情。

- 开发过程中：维护项目多个npm包之间的关系
  - 减少重复安装依赖
  - 开发中直接映射依赖包最新代码
  - 维护复杂的依赖关系，即版本号关系
- 发布过程中：维护项目多个npm发布包的版本号和自动打tag

无论是开发过程还是发布包的过程，Lerna都帮我们做了很多手动维护难以完成的工作。对于多个npm包还有版本的维护，如果手动维护，将是一个灾难。当然了，又因为这些包之间的依赖深，如果用multirepo的方式维护，会出现更多难以维护的问题。所以可以开出，Lerna对于多个npm包用monorepo的方式维护，算是提供了非常强力的辅助。
