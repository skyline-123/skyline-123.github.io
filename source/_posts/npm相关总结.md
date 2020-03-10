---
title: npm知识总结
date: 2020-03-10 14:52:04
tags:
  - npm
---

正常来讲，npm是所有前端工程师或者JS工程师都默认熟悉的包管理工具。随着长时间工作的积累，知识逐渐积累，我简单梳理总结一下重点。

## 准备
希望舒适使用npm或者node，下面两个全局依赖应该是默认都要安装：
- nrm 用于切换registry
- nvm node版本管理

## 梳理
### package.json
对于package.json，或许有一些同学会有盲点。

- peerDependencies 提示宿主环境去安装满足插件peerDependencies所指定依赖的包
- scripts 重点关注和version相关的生命周期钩子
  - preversion
  - version
  - postversion

### package-lock.json
这里**重点关注**package-lock.json文件和package.json文件对npm install的处理。
这里需要转载前人的总结，转载自https://www.cnblogs.com/surui/p/9038287.html
自npm 5.0版本发布以来，npm istall的规则发生了三次变化:
1. npm 5.0.x版本，不管package.json怎么变，npm install时都会根据package-lock.json下载；
2. npm 5.1.0版本后，npm install会无视package-lock.json,下载最新的版本；
3. npm 5.4.2版本后，如果package.json与package-lock.json不一致，npm install会根据package.json去下载版本，并更新package-lock.json；如果package.json与package-lock.json一致，npm install会根据package-lock.json去下载。

### version
version往往也是工作中经常出现问题的地方。解决报错一定要先排除version的问题。
#### semver
详情请查看官方文档[语义化版本号](https://semver.org/lang/zh-CN/)
#### 符号注释
示例 | 解释
------- | -------
* | 任意版本
1.1.0 | 指定版本
~1.1.0 | >=1.1.0 && < 1.2.0
^1.1.0 | >=1.1.0 && < 2.0.0

### npx
对于npx，要么你一点都不知道，要么就应该知道它有什么作用。
- 调用项目安装的模块，例如: npx lerna bootstrap，告别node-modules去执行。
- 使用不同版本的node，例如：npx node@10.15.3 -v，方便临时切换node版本。

### 私有npm库
搭建私有npm可能是有一定前端项目规模的公司必须面对的问题。
我列出两种方案供参考。
- 基于阿里的cnpm
- 使用verdaccio

### npm缓存
注意npm v5以后不需要通过npm cache clean --force解决缓存导致的问题。npm的缓存基于pacote实现，https://www.npmjs.com/package/pacote，具体内容我还没看。总之官网的指引就是排查问题不要再把焦点放在npm缓存上。缓存路径可以通过`npm config get cache`获取。

### publish
如果你到2020年还没有发布过npm包，尝试发一个。稍微要留意的是，如果发包到私有库，包名需要加上你公司的命名空间@scope。

### .npmrc
npm自己的配置文件，通过`npm config edit`命令修改。我一般用它修改registry，proxy等。你也可以用`npm config get xxx`和`npm config set xxx`来获取或者设置config。命令设置的config优先级低于.npmrc文件。

## 结束
大概总结就是这些，日后有踩到坑再补充。
