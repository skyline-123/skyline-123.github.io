---
title: 浅谈VurRouter实现原理
date: 2019-12-02 19:03:27
tags:
---
有朋友反馈想看更“干”一点的知识分享，最近库存里面只有我之前在腾讯团队那边分享过的VueRouter原理，现把PPT整理一遍，具体讲解部分不赘述，仅供日后参考。

## 前端路由实现方式
这部分内容算是一个基础回顾，如果已经非常熟悉了，可以跳过直接看源码分析部分。
前端路由实现方式主要有两种，如下：

- Hash模式
- History模式

### Hash模式
Hash指url后面的#号和后面的所有字符。hash也称作锚点，它可以使对应的id节点显示在浏览器可视区域内。
要实现前端路由功能，主要有以下三点原因：
1. hash变更，不会导致浏览器向服务端发送请求，当前页面也不会刷新
2. hash变更，会触发hashchange事件
3. hash变更，会在浏览器历史记录中添加记录，浏览器前进后退按键可以导航

代码参考：
```javascript
// 改变hash值
window.location.hash = 'home'

// 获取hash值
window.location.hash // '#home'

// 监听hash变更
window.addEventListener('hashchange', function onHashChange () {
  console.log('hash changed')
}, false)
```

### History模式
简单介绍下相关的API
1. HTML5引入了history.pushState()和history.replaceState()方法，它们分别可以添加和修改历史记录条目。这些方法通常与window.onpopstate配合使用。
2. pushState() 需要三个参数 一个状态对象, 一个标题 (目前被忽略), 和 (可选的) 一个URL。
3. history.replaceState() 的使用与 history.pushState() 非常相似，区别在于 replaceState() 是修改了当前的历史记录项而不是新建一个。 注意这并不会阻止其在全局浏览器历史记录中创建一个新的历史记录项。

如何实现路由呢？参考下面三点原因：
1. pushState新增历史记录可以实现无刷新更改地址栏链接
2. 当用户点击浏览器的前进，后退按钮，会触发popstate事件
3. 希望不添加一个新记录，而是替换当前的记录，则可以使用replaceState方法

参考代码：
```javascript
// state：需要保存的数据，这个数据在触发popstate事件时，可以在event.state里获取
// title：标题，基本没用，一般传 null
// url：设定新的历史记录的 url。新的 url 与当前 url 的 origin 必须是一樣的，否则会抛出错误。url可以是绝对路径，也可以是相对路径。
window.history.pushState(state, title, url)

// 与 pushState 基本相同，但它是修改当前历史记录，而 pushState 是创建新的历史记录
window.history.replaceState(state, title, url)

 // 监听浏览器前进后退事件，pushState 与 replaceState 方法不会触发
window.addEventListener("popstate", function() {
});
```

## VueRouter实现原理
简单回顾完前端路由实现方式，我们来聚焦VueRouter的具体实现。我们来到了硬核的源码环节。不用担心，我看得懂的代码都比较简单。

### VueRouter支持模式
```javascript
export default class VueRouter {
  constructor (options: RouterOptions = ) {
    let mode = options.mode || 'hash'
    this.fallback = mode === 'history'
      && !supportsPushState
      && options.fallback !== false
    if (this.fallback) {
      mode = 'hash'
    }
    if (!inBrowser) {
      mode = 'abstract'
    }
    this.mode = mode
    switch (mode) {
      case 'history':
        this.history = new HTML5History(this, options.base)
        break
      case 'hash':
        this.history = new HashHistory(
          this, options.base, this.fallback
        )
        break
      case 'abstract':
        this.history = new AbstractHistory(this, options.base)
        break
      default:
        if (process.env.NODE_ENV !== 'production') {
          assert(false, `invalid mode: ${mode}`)
        }
    }
  }
}
```
如上所示，VueRouter支持三种路由模式：
1. history mode
2. hash mode
3. abstract mode

VueRouter还有对history模式有降级的处理，在不支持history的浏览器环境中，如果fallback的option设置为true，VueRouter的mode会回落到hash模式。
abstract模式主要用在测试中，运行在非浏览器环境。我们重点关注history和hash这两种模式。\

### History模式
```javascript
export class HTML5History extends History {
  super(router, base)

  constructor (router: Router, base: ?string) {

    window.addEventListener('popstate', e => {
      const location = getLocation(this.base)
      // transitionTo()方法在父类中定义，用来处理路由变化中的基础逻辑
      this.transitionTo(location, route => {
        if (supportsScroll) {
          handleScroll(router, route, current, true)
        }
      })
    })

  }
}
```
如上所示，HTML5History在初始化的时候，添加popstate事件监听。

如下所示，HTML5History中重要的三个方法，go(), push(), replace()。 pushState和replaceState实际上也是封装了window.history对象上的方法，Vue Router对其做了部分逻辑处理。
```javascript
export class HTML5History extends History {
  go (n: number) {
    window.history.go(n)
  }
  push (
    location: RawLocation,
    onComplete?: Function,
    onAbort?: Function
  ) {
    const { current: fromRoute } = this
    // transitionTo()方法在父类中定义，用来处理路由变化中的基础逻辑
    this.transitionTo(location, route => {
      pushState(cleanPath(this.base + route.fullPath))
      handleScroll(this.router, route, fromRoute, false)
      onComplete && onComplete(route)
    }, onAbort)
  }
  replace (
    location: RawLocation,
    onComplete?: Function,
    onAbort?: Function
  ) {
    const { current: fromRoute } = this
    this.transitionTo(location, route => {
      replaceState(cleanPath(this.base + route.fullPath))
      handleScroll(this.router, route, fromRoute, false)
      onComplete && onComplete(route)
    }, onAbort)
  }
}
```

### Hash模式
在Vue app mounted之后添加监听：
```javascript
export class HashHistory extends History {
  // this is delayed until the app mounts
  // to avoid the hashchange listener being fired too early
  setupListeners () {
    const router = this.router
    const expectScroll = router.options.scrollBehavior
    const supportsScroll = supportsPushState && expectScroll

    if (supportsScroll) {
      setupScroll()
    }

    window.addEventListener(
      supportsPushState ? 'popstate' : 'hashchange',
      () => {
        const current = this.current
        if (!ensureSlash()) {
          return
        }
        this.transitionTo(getHash(), route => {
          if (supportsScroll) {
            handleScroll(this.router, route, current, true)
          }
          if (!supportsPushState) {
            replaceHash(route.fullPath)
          }
        })
      }
    )
  }
}
```

如下所示，HashHistory中重要的三个方法，go(), push(), replace()。 pushState和replaceState实际上也是封装了window.history对象上的方法，Vue Router对其做了部分逻辑处理。
```javascript
export class HashHistory extends History {
  push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    const { current: fromRoute } = this
    this.transitionTo(
      location,
      route => {
        pushHash(route.fullPath)
        handleScroll(this.router, route, fromRoute, false)
        onComplete && onComplete(route)
      },
      onAbort
    )
  }

  replace (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    const { current: fromRoute } = this
    this.transitionTo(
      location,
      route => {
        replaceHash(route.fullPath)
        handleScroll(this.router, route, fromRoute, false)
        onComplete && onComplete(route)
      },
      onAbort
    )
  }

  go (n: number) {
    window.history.go(n)
  }
}
```

其中有个细节，VueRouter做的非常好，就是有优先考虑pushState的方式去存hash。这个的好处就是可以存储同一个hash值的历史记录多次。如果只是更改hash值，同一个hash值访问多次不会存如历史记录栈。
```javascript
function pushHash (path) {
  if (supportsPushState) {
    pushState(getUrl(path)) // 优先考虑pushState方式去pushHash
  } else {
    window.location.hash = path
  }
}

function replaceHash (path) {
  if (supportsPushState) {
    replaceState(getUrl(path))
  } else {
    window.location.replace(getUrl(path))
  }
}
```

### History.transitionTo()
transitionTo方法是History模式实现的重要方法。主要的流程在下方，重点已经在注释中用箭头标注。
```javascript
export class History {
  transitionTo (
    location: RawLocation,
    onComplete?: Function,
    onAbort?: Function
  ) {
    const route = this.router.match(location, this.current)
    this.confirmTransition(
      route,
      () => {
        this.updateRoute(route)  // <--更新路由
        onComplete && onComplete(route)
        this.ensureURL()

        // fire ready cbs once
        if (!this.ready) {
          this.ready = true
          this.readyCbs.forEach(cb => {
            cb(route)
          })
        }
      },
      err => {
        if (onAbort) {
          onAbort(err)
        }
        if (err && !this.ready) {
          this.ready = true
          this.readyErrorCbs.forEach(cb => {
            cb(err)
          })
        }
      }
    )
  }
}
```

```javascript
export class History {

  listen (cb: Function) {
    this.cb = cb
  }

  updateRoute (route: Route) {
    const prev = this.current
    this.current = route
    this.cb && this.cb(route) // <--
    this.router.afterHooks.forEach(hook => {
      hook && hook(route, prev)
    })
  }
}
```

```javascript
export default class VueRouter {
  init (app: any /* Vue component instance */) {
    this.apps.push(app)

    history.listen(route => {
      this.apps.forEach((app) => {
        app._route = route // <--
      })
    })
  }
}
```

```javascript
export function install (Vue) {
  Vue.mixin({
    beforeCreate () {
      if (isDef(this.$options.router)) {
        this._routerRoot = this
        this._router = this.$options.router
        this._router.init(this)
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      } else {
        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
      }
      registerInstance(this, this)
    },
    destroyed () {
      registerInstance(this)
    }
  })
}
```

### VueRouter主流程
1. $router.push()
2. History.push()
3. History.transitionTo()
4. History.updateRoute()
5. app._route = route
6. vm.render()

## VueRouter组件简介

### RouterView组件
主要逻辑是拿到URL匹配路由配置对应的组件进行渲染。

### RouterLink组件
router-link 组件在被点击时，根据prop中的 to 的值去调用 router 的 push 或者 replace 来更新route。同时会检查自身是否和当前路由匹配来决定自身的 activeClass 是否添加。

## VueRouter思考问题
上面只讲到了VueRouter的主流程，没提到的部分供大家阅读源码和思考。
1. RouterView具体如何匹配对应Components？
2. 路由导航守卫的实现原理？
