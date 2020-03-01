---
title: Lerna的version变更
date: 2020-03-02 01:17:57
tags:
 - 前端工程化
 - Lerna
---

之前写了Lerna的使用感受文章，分享了Lerna的整体使用感受。而今天我处理了一个因为不熟悉Lerna导致的小问题，那我再来分享一下这个具体的使用案例。

## 起因
我们都知道（不知道你也假装现在知道了），Lerna会自动维护各个package的package.json文件的version变更。这里看上去非常智能，但是如果你的需求是要在其他地方（非package.json）同步这个version变更要如何处理呢？刚好工作中也遇到了这个问题，我们的模版文件的使用了ejs, 而package.json.ejs文件中的依赖版本号，是需要同步Lerna发布之后的版本号的。

## 解决
我们手动修改package.json.ejs依赖的版本号很长一段时间，但是这个动作明显不合理，所有能够交给代码去处理的事情不能手工完成。这个不仅仅是效率问题，更涉及到可靠性和稳定性。

不过事情的推进往往是出现问题之后，直到一次我们忘记手动更改这个依赖版本号之后，我就着手去处理这个问题。

这个归功于我之前有阅读过npm的部分源码，对preversion和version等npm自己的生命周期钩子有模糊的印象。我顺藤摸瓜，找到lerna version的文档，在最下面果然找到了lifecycle相关文档。lerna根目录的version script可以在lerna更改了packages文件之后执行。那思路就有了，只要我们写一个version.js的脚本，去替换package.json.ejs的版本号就行了。

## 结语
再简单的事情，都要重视，能自动化的都要交给机器。
