---
title: 前端工程化实践系列--ESLint定制规则
date: 2019-11-26 14:33:08
tags:
  - 前端工程化
  - 标准
  - ESLint
---
写过前端项目的同学，对ESLint都应该非常熟悉。业界常用的ESLint规范，一般是以[Airbnb](https://github.com/airbnb/javascript)和[standard](https://github.com/standard/standard)规范为主。我们今天是抛开直接在extends里面引入业界ESLint扩展，来聊聊自己项目实践中定制的ESLint规则。

我们先来看看ESLint官方的[Rules](https://eslint.org/docs/rules/)， 它分为如下多个种类:
  - Possible Errors
  - Best Practices
  - Strict Mode
  - Variables
  - Node.js and CommonJS
  - Stylistic Issues
  - ECMAScript 6
  - Deprecated
  - Removed

我们重点看看Stylistic Issues这个分类，这个分类的Rules是解决什么问题的呢？官方解释称这些规则是关于风格指南，而且是非常主观的。对于这样的描述，我们可能在定制规则的过程中一带而过，而其他最佳实践和一些避免潜在错误的规范，standard等规范又帮我们做了。别着急，我们看看这几个Rules。

  - max-depth(强制可嵌套的块的最大深度)
  - max-len(强制一行的最大长度)
  - max-lines(强制最大行数)
  - max-lines-per-function(强制函数最大代码行数)
  - max-nested-callbacks(强制回调函数最大嵌套深度)
  - max-params(强制函数定义中最多允许的参数数量)
  - max-statements(强制函数块最多允许的的语句数量)
  - max-statements-per-line(强制每一行中所允许的最大语句数量)

这些主观性强的规则，却恰恰帮助我们为团队定制规范提供了很好的帮助。max-depth适当的嵌套数，可以避免写出难以阅读的复杂代码，max-len可以强迫开发过程中主动换行提高代码的可阅读性，max-lines相关的rules可以让开发对代码细粒度的把握有更看得见摸得着的依据，max-nested-callbacks这个可以在ESLint层面解决回调地狱的问题。

为什么能发现这些比较特殊的规则呢？也是我们团队开发过程中，发现有不少同学会把一个方法或者一个函数写成几百行的情况。对于这个问题，明显不能通过code review这种后置笨重的方法去解决，所以我们找时间去阅读ESLint的官方规则，对症下药。

刚刚开始前端开发的时候，我对标准这种东西的认识还是比较肤浅。曾以为业界的规范就是拿来就用即可，但是实际开发过程中，很可能和不同层次的开发合作，面对复杂的开发人员现状，势必要有更高级别的约束和提示。业内的公用标准，对于团队来说，更重要的是借鉴，并根据实际情况做灵活的调整。

希望这种“个性化定制”，能给做团队标准化制定工作的朋友和未来的自己一些启发。
