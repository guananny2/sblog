---
title: vue-router 源码解析
date: 2021-02-19 20:49:44
tags: [Vue, router]
categories: VUE
---

## 后端路由和前端路由
### 后端路由
当页面切换的时候，浏览器会向服务器发送不同的URL，服务器接收到不同的URL,解析生成不同的页面，发送给浏览器，浏览器将页面进行渲染展示。

### 前端路由
在页面切换的时候，不会向浏览器发送请求，从而保证页面不刷新。

## 前端路由实现的原理
实现前端路由，需解决2个问题：
* 1、实现url变化，页面不刷新
* 2、捕获url变化，执行页面替换逻辑

### 前端路由两种实现
* 1、hash路由
http://www.xxx.com/#/login
满足条件：
> #后面hash值的变化，不会向浏览器发送请求
> location.hash可获取到#号后hash值；location.hash="xxx"可替换#后hash值
> 可通过window.addEventListener捕获hashchange事件，来执行页面替换的逻辑

* 2、history路由
http://www.xxx.com/login
满足条件：
> 基于Html5 history 新增的API,通过该api改变url地址，不会向浏览器发送请求
> pushState()、replaceState()
> 可通过window.addEventListener捕获popstate事件,来执行页面替换的逻辑

## vue-router实现方式
vue通过全局方法Vue.use()使用插件，
``` javascript
// 调用VueRouter.install(vue)
vue.use(VueRouter)

new Vue({
    // ... 组件选项
})
```

* 1、vue.use()的源码实现（注：vue源码）
``` javascript
Vue.use = function (plugin: Function | Object) {
    const installedPlugins = (this._installedPlugins || (this._installedPlugins = []))
    if (installedPlugins.indexOf(plugin) > -1) {
      return this
    }

    // additional parameters
    const args = toArray(arguments, 1)
    args.unshift(this)
    if (typeof plugin.install === 'function') {
      plugin.install.apply(plugin, args)
    } else if (typeof plugin === 'function') {
      plugin.apply(null, args)
    }
    installedPlugins.push(plugin)
    return this
}
```
    * VueRouter插件定义了install方法，并执行install方法
    * 把VueRouter插件存储到installedPlugins


* 2、VueRouter install源码实现
``` javascript
export function install (Vue) {
  if (install.installed && _Vue === Vue) return
  install.installed = true

  _Vue = Vue

  const isDef = v => v !== undefined

  const registerInstance = (vm, callVal) => {
    let i = vm.$options._parentVnode
    if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
      i(vm, callVal)
    }
  }

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

  Object.defineProperty(Vue.prototype, '$router', {
    get () { return this._routerRoot._router }
  })

  Object.defineProperty(Vue.prototype, '$route', {
    get () { return this._routerRoot._route }
  })

  Vue.component('RouterView', View)
  Vue.component('RouterLink', Link)

  const strats = Vue.config.optionMergeStrategies
  // use the same hook merging strategy for route hooks
  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
}
```
主要关注核心的代码`Vue.mixin`部分，通过vue混入，给每个组件注入了`beforeCreate()`和`destory()`这两个生命周期。`beforeCreate()`主要执行了`vueRouter`的`init()`方法、`_route`转换为响应式、调用`registerInstance(this, this)`;`destory()`调用`registerInstance(this, this)`

* 3、VueRouter init()源码
``` javascript
init (app: any /* Vue component instance */) {
    this.apps.push(app)

    // set up app destroyed handler
    // https://github.com/vuejs/vue-router/issues/2639
    app.$once('hook:destroyed', () => {
      // clean out app from this.apps array once destroyed
      const index = this.apps.indexOf(app)
      if (index > -1) this.apps.splice(index, 1)
      // ensure we still have a main app or null if no apps
      // we do not release the router so it can be reused
      if (this.app === app) this.app = this.apps[0] || null

      if (!this.app) this.history.teardown()
    })

    // main app previously initialized
    // return as we don't need to set up new history listener
    if (this.app) {
      return
    }

    this.app = app

    const history = this.history

    if (history instanceof HTML5History || history instanceof HashHistory) {
      const handleInitialScroll = routeOrError => {
        const from = history.current
        const expectScroll = this.options.scrollBehavior
        const supportsScroll = supportsPushState && expectScroll

        if (supportsScroll && 'fullPath' in routeOrError) {
          handleScroll(this, routeOrError, from, false)
        }
      }
      const setupListeners = routeOrError => {
        history.setupListeners()
        handleInitialScroll(routeOrError)
      }
      history.transitionTo(
        history.getCurrentLocation(),
        setupListeners,
        setupListeners
      )
    }

    history.listen(route => {
      this.apps.forEach(app => {
        app._route = route
      })
    })
  }
```
init 主要处理了handleInitialScroll（滚动条状态）、setupListeners（注册监听）、transitionTo（加载至匹配路由组件）

* 4、VueRouter constructor源码
``` javascript
constructor (options: RouterOptions = {}) {
    this.app = null
    this.apps = []
    this.options = options
    this.beforeHooks = []
    this.resolveHooks = []
    this.afterHooks = []
    this.matcher = createMatcher(options.routes || [], this)

    let mode = options.mode || 'hash'
    this.fallback =
      mode === 'history' && !supportsPushState && options.fallback !== false
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
        this.history = new HashHistory(this, options.base, this.fallback)
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
```
根据不同的`mode`创建不同的history实例，且默认为hash路由。
HTML5History、HashHistory、AbstractHistory都继承了History类，其中go、push、replace、ensureURL、getCurrentLocation、setupListeners由各自特性不同，有各自的实现。
主要来对比 HTML5History、HashHistory这push、replace两个接口的实现

    * push
    ``` javascript
    // hash mode
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
    
    function pushHash (path) {
        if (supportsPushState) {
            pushState(getUrl(path))
        } else {
            window.location.hash = path
        }
    }
    ```
    ``` javascript
    // html5 mode
    push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
        const { current: fromRoute } = this
        this.transitionTo(location, route => {
        pushState(cleanPath(this.base + route.fullPath))
        handleScroll(this.router, route, fromRoute, false)
        onComplete && onComplete(route)
        }, onAbort)
    }

    export function pushState (url?: string, replace?: boolean) {
        saveScrollPosition()
        // try...catch the pushState call to get around Safari
        // DOM Exception 18 where it limits to 100 pushState calls
        const history = window.history
        try {
            if (replace) {
            // preserve existing history state as it could be overriden by the user
            const stateCopy = extend({}, history.state)
            stateCopy.key = getStateKey()
            history.replaceState(stateCopy, '', url)
            } else {
            history.pushState({ key: setStateKey(genStateKey()) }, '', url)
            }
        } catch (e) {
            window.location[replace ? 'replace' : 'assign'](url)
        }
    }
    ```

    * replace
    ``` javascript
    // hash mode
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

    function replaceHash (path) {
        if (supportsPushState) {
            replaceState(getUrl(path))
        } else {
            window.location.replace(getUrl(path))
        }
    }
    ```

    ``` javascript
    // html5 mode
    replace (location: RawLocation, onComplete?: Function, onAbort?: Function) {
        const { current: fromRoute } = this
        this.transitionTo(location, route => {
        replaceState(cleanPath(this.base + route.fullPath))
        handleScroll(this.router, route, fromRoute, false)
        onComplete && onComplete(route)
        }, onAbort)
    }

    export function replaceState (url?: string) {
        pushState(url, true)
    }
    ```

继续回到 transitionTo（）方法，通过location匹配到路由组件
``` javascript
match (raw: RawLocation, current?: Route, redirectedFrom?: Location): Route {
    return this.matcher.match(raw, current, redirectedFrom)
}
```
match源码
``` javascript
function match (
    raw: RawLocation,
    currentRoute?: Route,
    redirectedFrom?: Location
  ): Route {
    const location = normalizeLocation(raw, currentRoute, false, router)
    const { name } = location

    if (name) {
      const record = nameMap[name]
      if (process.env.NODE_ENV !== 'production') {
        warn(record, `Route with name '${name}' does not exist`)
      }
      if (!record) return _createRoute(null, location)
      const paramNames = record.regex.keys
        .filter(key => !key.optional)
        .map(key => key.name)

      if (typeof location.params !== 'object') {
        location.params = {}
      }

      if (currentRoute && typeof currentRoute.params === 'object') {
        for (const key in currentRoute.params) {
          if (!(key in location.params) && paramNames.indexOf(key) > -1) {
            location.params[key] = currentRoute.params[key]
          }
        }
      }

      location.path = fillParams(record.path, location.params, `named route "${name}"`)
      return _createRoute(record, location, redirectedFrom)
    } else if (location.path) {
      location.params = {}
      for (let i = 0; i < pathList.length; i++) {
        const path = pathList[i]
        const record = pathMap[path]
        if (matchRoute(record.regex, location.path, location.params)) {
          return _createRoute(record, location, redirectedFrom)
        }
      }
    }
    // no match
    return _createRoute(null, location)
}
```


