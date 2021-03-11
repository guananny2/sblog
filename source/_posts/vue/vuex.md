---
title: vuex 源码解析
date: 2021-02-19 20:49:44
tags: [Vuex,Vue]
categories: VUE
---

Vuex 是一个专为 Vue.js 应用程序开发的状态管理模式。

### Vuex模式思想
Vuex的思想借鉴了Flux、Redux,与其他模式不同的是，Vuex是专门为Vue.js设计的状态管理库。

#### Flux架构思想
Flux是一种架构思想，最大的特点是数据的“单向流动”

{% img /images/vue/vuex/array.png "点击查看大图:vi/vim-cheat-sheet" %}
* 1、View: 视图层
* 2、Action: 动作
* 3、Dispatcher: 派发器
* 4、Store: 数据层
数据流动的顺序：
用户访问View,页面需要变化时，发出动作Action；Dispatcher收到动作后，要求Store进行相应的更新；store更新后，发出change事件，更新页面。在这一过程中，数据总是单向流动的，保证了流程的清晰。

#### Vuex架构思想
Vuex也把组件的共享状态抽取出来，以一个全局单例模式管理。并且数据的流动也遵从“单向流动”的设计思想。

{% img /images/vue/vuex/vuex.png "点击查看大图:vi/vim-cheat-sheet" %}
数据流动的顺序：
* 1、同步数据
    用户访问View,页面需要变化时；调用commit来请求store中对应的mutation函数；store改变（Vue检测到数据变化，自动更新视图）
* 2、异步数据
    用户访问View,页面需要变化时，派发动作action;action调用commit来请求store中对应的mutation函数；store改变（Vue检测到数据的变化，自动更新视图）

### Vuex源码解析
#### Vuex的使用
{% img /images/vue/vuex/store.png 378 339 "点击查看大图:vi/vim-cheat-sheet" %}

#### Vuex的注册
``` javascript
export function install (_Vue) {
  if (Vue && _Vue === Vue) {
    if (__DEV__) {
      console.error(
        '[vuex] already installed. Vue.use(Vuex) should be called only once.'
      )
    }
    return
  }
  Vue = _Vue
  applyMixin(Vue)
}
```
已经注册过一次的就不再注册；调用applyMixin(Vue)进行注册
``` javascript
export default function (Vue) {
  const version = Number(Vue.version.split('.')[0])

  if (version >= 2) {
    Vue.mixin({ beforeCreate: vuexInit })
  } else {
    // override init and inject vuex init procedure
    // for 1.x backwards compatibility.
    const _init = Vue.prototype._init
    Vue.prototype._init = function (options = {}) {
      options.init = options.init
        ? [vuexInit].concat(options.init)
        : vuexInit
      _init.call(this, options)
    }
  }

  /**
   * Vuex init hook, injected into each instances init hooks list.
   */

  function vuexInit () {
    const options = this.$options
    // store injection
    if (options.store) {
      this.$store = typeof options.store === 'function'
        ? options.store()
        : options.store
    } else if (options.parent && options.parent.$store) {
      this.$store = options.parent.$store
    }
  }
}
```
根据Vue的版本来分别注册；如果是2.X则使用 beforeCreate来注册；如果是1.X则使用_init来注册。
当从new Vue的时候，从options.store获取；当是非根组件时，子组件从父组件获取到$store，这样保证了每个组件都能取到$store

#### store的初始化
``` javascript
export default new Vuex.Store({
  state: {
  },
  mutations: {
  },
  actions: {
  },
  modules: {
  }
})

new Vue({
  router,
  store,
  render: h => h(App)
}).$mount('#app')
```
``` javascript
constructor (options = {}) {
    // 如果当前环境是浏览器环境，且没有安装 vuex ，那么就会自动安装
    if (!Vue && typeof window !== 'undefined' && window.Vue) {
      install(window.Vue)
    }

    // 断言指定的环境是否满足
    if (__DEV__) {
      assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
      assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)
      assert(this instanceof Store, `store must be called with the new operator.`)
    }

    const {
      plugins = [],
      strict = false
    } = options

    // store internal state
    this._committing = false // 是否在进行提交状态标识
    this._actions = Object.create(null) // acitons 操作对象
    this._actionSubscribers = [] // action 订阅列表
    this._mutations = Object.create(null)  // mutations操作对象
    this._wrappedGetters = Object.create(null)  // 封装后的 getters 集合对象
    this._modules = new ModuleCollection(options)  // vuex 支持 store 分模块传入，存储分析后的 modules
    this._modulesNamespaceMap = Object.create(null)  // 模块命名空间 map
    this._subscribers = [] // 订阅函数集合
    this._watcherVM = new Vue()  // Vue 组件用于 watch 监视变化
    this._makeLocalGettersCache = Object.create(null)

    // 替换 this 中的 dispatch, commit 方法，将 this 指向 store
    const store = this
    const { dispatch, commit } = this
    this.dispatch = function boundDispatch (type, payload) {
      return dispatch.call(store, type, payload)
    }
    this.commit = function boundCommit (type, payload, options) {
      return commit.call(store, type, payload, options)
    }

    // 是否使用严格模式
    this.strict = strict
    // 数据树
    const state = this._modules.root.state

    // 加载安装模块
    installModule(this, state, [], this._modules.root)

    // 重置虚拟 store
    resetStoreVM(this, state)

    // 如果使用了 plugins 那么挨个载入它们
    plugins.forEach(plugin => plugin(this))

    const useDevtools = options.devtools !== undefined ? options.devtools : Vue.config.devtools
    if (useDevtools) {
      devtoolPlugin(this)
    }
  }
```

##### 模块收集
``` javascript
this._modules = new ModuleCollection(options)
```
``` javascript
export default class ModuleCollection {
  constructor (rawRootModule) {
    // register root module (Vuex.Store options)
    this.register([], rawRootModule, false)
  }

  get (path) {
    return path.reduce((module, key) => {
      return module.getChild(key)
    }, this.root)
  }

  getNamespace (path) {
    let module = this.root
    return path.reduce((namespace, key) => {
      module = module.getChild(key)
      return namespace + (module.namespaced ? key + '/' : '')
    }, '')
  }

  update (rawRootModule) {
    update([], this.root, rawRootModule)
  }

  register (path, rawModule, runtime = true) {
    if (__DEV__) {
      assertRawModule(path, rawModule)
    }

    const newModule = new Module(rawModule, runtime)
    if (path.length === 0) {
      this.root = newModule
    } else {
      const parent = this.get(path.slice(0, -1))
      parent.addChild(path[path.length - 1], newModule)
    }

    // register nested modules
    if (rawModule.modules) {
      forEachValue(rawModule.modules, (rawChildModule, key) => {
        this.register(path.concat(key), rawChildModule, runtime)
      })
    }
  }

  unregister (path) {
    const parent = this.get(path.slice(0, -1))
    const key = path[path.length - 1]
    const child = parent.getChild(key)

    if (!child) {
      if (__DEV__) {
        console.warn(
          `[vuex] trying to unregister module '${key}', which is ` +
          `not registered`
        )
      }
      return
    }

    if (!child.runtime) {
      return
    }

    parent.removeChild(key)
  }

  isRegistered (path) {
    const parent = this.get(path.slice(0, -1))
    const key = path[path.length - 1]

    if (parent) {
      return parent.hasChild(key)
    }

    return false
  }
}
```
接着再看Module
``` javascript
export default class Module {
  constructor (rawModule, runtime) {
    this.runtime = runtime
    // Store some children item
    this._children = Object.create(null)
    // Store the origin module object which passed by programmer
    this._rawModule = rawModule
    const rawState = rawModule.state

    // Store the origin module's state
    this.state = (typeof rawState === 'function' ? rawState() : rawState) || {}
  }

  get namespaced () {
    return !!this._rawModule.namespaced
  }

  addChild (key, module) {
    this._children[key] = module
  }

  removeChild (key) {
    delete this._children[key]
  }

  getChild (key) {
    return this._children[key]
  }

  hasChild (key) {
    return key in this._children
  }

  update (rawModule) {
    this._rawModule.namespaced = rawModule.namespaced
    if (rawModule.actions) {
      this._rawModule.actions = rawModule.actions
    }
    if (rawModule.mutations) {
      this._rawModule.mutations = rawModule.mutations
    }
    if (rawModule.getters) {
      this._rawModule.getters = rawModule.getters
    }
  }

  forEachChild (fn) {
    forEachValue(this._children, fn)
  }

  forEachGetter (fn) {
    if (this._rawModule.getters) {
      forEachValue(this._rawModule.getters, fn)
    }
  }

  forEachAction (fn) {
    if (this._rawModule.actions) {
      forEachValue(this._rawModule.actions, fn)
    }
  }

  forEachMutation (fn) {
    if (this._rawModule.mutations) {
      forEachValue(this._rawModule.mutations, fn)
    }
  }
}
```
这里调用 ModuleCollection 构造函数，通过 path 的长度判断是否为根 module，首先进行根 module 的注册，然后递归遍历所有的 module，子 module 添加其父 module 的 _children 属性上，最终形成一棵树


##### 模块安装
``` javascript
// 这里是module处理的核心，包括处理根module、命名空间、action、mutation、getters和递归注册子module
installModule(this, state, [], this._modules.root)
```
``` javascript
function installModule (store, rootState, path, module, hot) {
  const isRoot = !path.length
  // 获取模块的命名空间
  const namespace = store._modules.getNamespace(path)

  // 存在命名空间时，加入到 moduleNamespaceMap 中
  if (module.namespaced) {
    if (store._modulesNamespaceMap[namespace] && __DEV__) {
      console.error(`[vuex] duplicate namespace ${namespace} for the namespaced module ${path.join('/')}`)
    }
    store._modulesNamespaceMap[namespace] = module
  }

  // 如果当前不是子模块也不是热更新状态，那么就是新增子模块，这个时候要取到父模块
  // 然后插入到父模块的子模块列表中
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      if (__DEV__) {
        if (moduleName in parentState) {
          console.warn(
            `[vuex] state field "${moduleName}" was overridden by a module with the same name at "${path.join('.')}"`
          )
        }
      }
      Vue.set(parentState, moduleName, module.state)
    })
  }
  // 拿到当前的上下文环境
  const local = module.context = makeLocalContext(store, namespace, path)

  // 使用模块的方法挨个为 mutation, action, getters, child 注册
  module.forEachMutation((mutation, key) => {
    const namespacedType = namespace + key
    registerMutation(store, namespacedType, mutation, local)
  })

  module.forEachAction((action, key) => {
    const type = action.root ? key : namespace + key
    const handler = action.handler || action
    registerAction(store, type, handler, local)
  })

  module.forEachGetter((getter, key) => {
    const namespacedType = namespace + key
    registerGetter(store, namespacedType, getter, local)
  })

  module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot)
  })
}
```
首先保存 namespaced 模块到 store._modulesNamespaceMap，再判断是否为根组件且不是 hot，得到父级 module 的 state 和当前 module 的 name，调用 Vue.set(parentState, moduleName, module.state)将当前 module 的 state 挂载到父 state 上。
