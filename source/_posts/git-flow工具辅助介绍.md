---
title: git-flow工具辅助介绍
date: 2019-11-30 22:47:39
tags:
  - git
  - git-flow
  - 工具
---

最近在实践过程中，使用了git-flow的命令行工具进行管理，对git-flow的流程更加清晰，现在总结出来供参考。

我使用的是macOS，安装git flow命令行方法如下：
```bash
brew install git-flow-avh
```

## 第一步
```bash
git flow init
```
简单来说，git flow初始化主要做了两件事:
1. 初始化git仓库
2. 基于master分支创建develop分支
```bash
git init
git checkout -b develop
```

## 第二步
假设我们需要进行user功能的开发
```bash
git flow feature start user
```
这时git flow做了一件事情，就是基于develop分支，新开一个feature/user分支。类似于如下git命令：
```bash
git checkout -b feature/user
```

## 第三步
我们在feature/user分支把user模块的功能开发完成，此时就可以执行如下命令：
```bash
git flow feature finisth user
```
以上命令，其实就是切换回develop分支，merge feature/user的代码，然后将feature/user的分支删除。
类似于如下git命令：
```bash
git checkout develop
git merge feature/user
git branch --delete feature/user
```

**注意**：实际工作中，我们的feature分支合并到develop分支，是全部在线上完成。由完成feature的同学上传对应的feature分支到远程git仓库，然后创建合并请求申请。项目负责人在code review确认后，同意合并该分支到develop分支，并删除该feature分支。

第三步在开发过程中是一个持续的过程，会不断有新的feature分支合并到develop分支。

## 第四步
feature开发完之后，开始准备release分支提测。类似于start一个feature分支，release分支命令如下：
```bash
git flow release start v1.0
git flow release publish v1.0
```
上面两行命令，就是基于develop分支新建一个release/v1.0分支，然后将改分支推送到远程仓库。测试回归过程中，遇到有问题，所有bugfix都在改分支进行调整。
类似于如下git命令：
```bash
git checkout develop
git checkout -b release/v1.0
git push -u origin release/v1.0
```

## 第五步
release分支完成之后，完成如下命令，将所有代码回补到develop分支，最后合并到master分支。
```bash
git flow release finish v1.0
```
**注意** 实际生产环境，涉及到develop分支和master分支的代码merge，一般都是在线上发起merge request来完成。git flow工具只是做一个辅助参考，不需要所有命令都使用。
类似于如下git命令：
```bash
git checkout develop
git merge release/v1.0
git tab v1.0
git checkout master
git merge develop
```
👌，经过上面的操作，一个相对完整的git-flow流程就愉快地走了一遍。由于git flow的命令非常清晰易懂，而且背后执行的git操作其实也相对简单，就不把每一步执行的结果截图演示了。

git flow的命令行工具，在一定程度上辅助我们简化git flow流程的操作，但是最重要的是我们如果真的在用git flow的方法论开发，一定要对git flow本身的流程有非常清晰的认识，才能更好借助git flow命令提高工作中的效率。

另外，在试用git flow命令行工具的过程中，我和公司的同事也讨论了下其限制。比如我们如果有超过一条主线，比如，多个定制版本的时候，标准的git flow就不能满足开发需求。但是要结合git flow本身的规范性去衍生一些解决方案出来也是有的，如下：
- 新开仓库走git flow开发定制版
- 增加特殊的分支，用来定制使用，平行于master分支。例如master_a, master_b等定制化分支。
基于git flow衍生的解决方案，也是可以相对规范地解决一些复杂的生产开发问题。
