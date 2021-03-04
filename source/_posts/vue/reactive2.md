---
title: vue2.0响应式原理
date: 2020-07-27 09:49:44
tags: [Vue]
categories: VUE
---


在介绍响应式原理之前我们先来了解一下如何侦测对象的变化，目前侦测对象变化的方式有2种：Object.defineProperty和ES6的Proxy。在Vue2.0阶段，浏览器对Proxy的支持还不够理想，所以2.0还是基于Object.defineProperty来实现的。本文也是基于Object.defineProperty来介绍如何实现响应式，在下篇文章中也会基于Proxy来介绍Vue3.0如何实现响应式。

<!-- more -->

### 基础知识
在解析源码的过程中，会针对Object.defineProperty、观察者模式为切入点解析vue是如何实现双向绑定，数据的变化来驱动视图的更新。

#### Object.defineProperty
Object.defineProperty是ES5新添加的对象方法，该方法会直接在一个对象上定义一个新属性，或者修改一个对象的现有属性，并返回此对象。

ECMAScript有两种属性:数据属性和访问器属性
* 数据属性包含[[Configurable]]、[[Enumerable]]、[[Writable]]、[[Value]]；
* 访问器属性包含一对set和get函数，在读取访问器属性时，会调用 getter 函数，这个函数负责返回有效的值，在写入访问器属性时，会调用setter 函数并传入新值，这个函数负责决定如何处理数据访问器属性包含[[Configurable]]、[[Enumerable]]、[[Get]]、[[Set]]。

``` javascript
var obj = {};
var a;
Object.defineProperty(obj, 'a', {
  get: function() {
    console.log('get val');　
    return a;
  },
  set: function(newVal) {
    console.log('set val:' + newVal);
    a = newVal;
  }
});
obj.a;     // get val 
obj.a = '111'; // set val: 111
```

示例代码中 Object.defineProperty 把 obj 的 a 属性转化为 getter 和 setter，可以实现 obj.a 的数据监控，Vue正式基于这个特性实现了响应式。
Vue 会遍历对象所有的 property，并使用 Object.defineProperty 把这些 property 全部转为 getter/setter。

#### 观察者模式
vue是基于观察者模式来实现数据更新之后触发一系列的相关依赖来自动更新视图。那么先来了解一下什么是观察者模式，观察者模式是指一个对象维持一系列的依赖于他的对象，将有关状态变更自动的通知给他们。
观察者模式的基本要素
* Subject (目标)
* Observer （观察者）

{% img /images/vue/observer.png "点击查看大图:vi/vim-cheat-sheet" %}

定义一个收集所有依赖的容器
```javascript
// 目标者类
class Subject {
  constructor() {
    this.observers = []
  }
  // 添加
  add(observer) {
    this.observers.push(observer)
  }
  // 删除
  remove(observer) {
    let idx = this.observers.find(observer)
    idx > -1 && this.observers.splice(idx,1)
  }
  // 通知
  notify() {
    for(let oberver of this.observers) {
      observer.update()
    }
  }
}

// 观察者类
class Observer{
  constructor(name) {
    this.name = name
  }
  update() {
    console.log(`目标通知我更新了,我是${this.name}`)
  }
}
```

### 源码解析

#### 整体概览
下面就进入vue源码开始解析vue是如何实现响应式的。

vue在初始化的时候会做一系列的init操作，我们把关注的重点放在如何将data转换成响应式的数据。一步步的解析源码，在`init.js`文件中，找到在初始化的时候会执行`initState(vm)`,在`state.js`文件中找到`initData(vm)`,最终会执行 `observe(data, true /* asRootData */)`,最终找到核心的observe相关代码。


由于javascript的限制，Object.defineProperty()不能监测到数组的改变，vue对数组和对象使用了2种不同的方式实现，对于Object类型来说，通过劫持getter和setter来实现监测改变；对于Array来说，通过拦截器，拦截数组相关api（push、pop、shift、unshift...）来实现监测改变。
``` js
// instance/observer
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */) 
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}

function initData (vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  ... 省略部分代码
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  ... 省略部分代码
  // observe data
  observe(data, true /* asRootData */)
}
```

``` js
// observe/index.js
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) { // 如果是基本类型 或虚拟node 则直接返回
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}

export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    def(value, '__ob__', this)
    // 分数组和对象来分别处理
    if (Array.isArray(value)) {
      if (hasProto) {
        protoAugment(value, arrayMethods) // 添加拦截器
      } else {
        copyAugment(value, arrayMethods, arrayKeys) // 添加拦截器
      }
      this.observeArray(value) // 将数组转换成响应式
    } else {
      this.walk(value) // 将对象转换成响应式
    }
  }
}
```

#### data是Object类型
{% img /images/vue/observer1.png "点击查看大图:vi/vim-cheat-sheet" %}


``` js
// observe/index.js
// 循环遍历每一个key,劫持添加getter setter
walk (obj: Object) {
  const keys = Object.keys(obj)
  for (let i = 0; i < keys.length; i++) {
    defineReactive(obj, keys[i])
  }
}

export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep() // Dep 对应于观察者模式中的Subject，用户收集用户的依赖，以及发送通知
  ... // 省略部分代码
  let childOb = !shallow && observe(val) // 递归每一个可以，将数据转换成observe
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    // 给数据添加访问器属性get
    // 什么时候触发get? 当页面组件“mount”阶段，会调用render渲染页面，在渲染的时候会获取数据，自动触发reactiveGetter，
    // Dep.target 是什么？ 通过查看lifecycle.js 在mountComponent阶段会new Watcher并将全局的Dep.target指向这个Watcher
    // dep.denpend() 做了什么？ 会将该Watcher添加到dep的subs队列中
    get: function reactiveGetter () { 
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend() 
        if (childOb) {
          childOb.dep.depend() // 收集依赖
          if (Array.isArray(value)) {
            dependArray(value) // 收集依赖
          }
        }
      }
      return value
    },
    // 给数据添加访问器属性set
    // 什么时候触发set? key对应的值改变会自动触发reactiveSetter调用，执行notify通知
    // notify做了什么？ 遍历subs（Watcher）,执行watcher中的update,并将watcher添加至待更新队列
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      ... // 省略部分代码
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      dep.notify() // 数据更新后调用dep 通知存放的所有的依赖
    }
  })
}
```

Dep (目标：Subject)
defineReactive用到了一个很重要的对象 Dep,那么Dep是干嘛的？Dep是一个目标对象，负责管理Watcher（添加watcher、删除watcher、将自己添加至Watcher的deps队列、通知自己管理的每一个watcher进行更新）
``` js
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    ... // 省略部分代码
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}
```

Watcher是一个中介的角色，数据发生变化时通知它，然后它再通知其他地方。
他就是负责具体的脏活累活
* 1、收集依赖
* 2、负责执行cb来更新所有的依赖
 
``` js
// Watcher.js
export default class Watcher {
  ... // 省略部分代码
  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function
  ) {
    this.vm = vm
    ... // 省略部分代码
    this.cb = cb
    ... // 省略部分代码
    this.expression = process.env.NODE_ENV !== 'production'
      ? expOrFn.toString()
      : ''
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
      ... // 省略部分代码
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
  get () {
    pushTarget(this)
    let value
    const vm = this.vm
    value = this.getter.call(vm, vm)
    ... // 省略部分代码
    return value
  }

  /**
   * Add a dependency to this directive.
   */
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }

  update () {
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }

  run () {
    if (this.active) {
      const value = this.get()
      if (
        value !== this.value ||
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        if (this.user) {
          try {
            this.cb.call(this.vm, value, oldValue) // 具体的执行更新
          } catch (e) {
            handleError(e, this.vm, `callback for watcher "${this.expression}"`)
          }
        } else {
          this.cb.call(this.vm, value, oldValue) // 具体的执行更新
        }
      }
    }
  }

  depend () {
    let i = this.deps.length
    while (i--) {
      this.deps[i].depend()
    }
  }
}
```

#### data是Array类型
{% img /images/vue/array.png 440 320"点击查看大图:vi/vim-cheat-sheet" %}
下面将一步步梳理 data中的数据结构是array类型,vue源码是如何实现拦截并转换成响应式

``` js
// 以数据结构为列
data: {
  array: [1, 2, 3]
}
```
``` js
// instance/state.js
// 入口
...
observe(data, true /* asRootData */)
...
```
``` js
// observer/index.js
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```
将data转换成observer,执行walk(value)
``` js
import { arrayMethods } from './array'
const arrayKeys = Object.getOwnPropertyNames(arrayMethods)

export class Observer {
  ...
  constructor (value: any) {
    this.value = value
    this.dep = new Dep() // important ! 这边的dep实际用于类型是数组的数据 收集依赖
    this.vmCount = 0
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      if (hasProto) { // 判断浏览器是否支持 __proto__
        protoAugment(value, arrayMethods) // 使用__proto__将拦截器中的方法直接覆盖原型
      } else {
        copyAugment(value, arrayMethods, arrayKeys) // 通过复制将拦截器中的方法挂载到value上
      }
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }
}

// 遍历每一个key
walk (obj: Object) {
  const keys = Object.keys(obj)
  for (let i = 0; i < keys.length; i++) {
    defineReactive(obj, keys[i])
  }
}


export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep() // Dep 对应于观察者模式中的Subject，用户收集用户的依赖，以及发送通知
  ... // 省略部分代码
  let childOb = !shallow && observe(val) // 这一步很重要，递归的将array转换成observer
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    // 页面在mount阶段获取数据时 会触发reactiveGetter，给数组添加依赖
    get: function reactiveGetter () { 
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend() 
        if (childOb) {
          childOb.dep.depend() // 数组添加收集依赖
          if (Array.isArray(value)) {
            dependArray(value) // 收集依赖
          }
        }
      }
      return value
    },
    // 数组变化并不会触发这边的set回调，实际上会执行拦截器中的__obj__.dep.notify() （见array.js中的方法）
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      ... // 省略部分代码
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      dep.notify() // 数据更新后调用dep 通知存放的所有的依赖
    }
  })
}

```

当访问数组中的方法时，由于添加了拦截器，当访问数组的方法时，会访问伪造的方法。
``` js
// 拦截器方法
// observer/array.js
const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]
methodsToPatch.forEach(function (method) {
  // cache original method
  const original = arrayProto[method]
  def(arrayMethods, method, function mutator (...args) {
    const result = original.apply(this, args)
    const ob = this.__ob__
    let inserted
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    if (inserted) ob.observeArray(inserted)
    // notify change
    ob.dep.notify() // 数组变化时，会调用dep 目标去通知所有的依赖进行更新
    return result
  })
})
```

``` js 
// observer/index.js


// 使用 __proto__ 覆盖原型
function protoAugment (target, src: Object) {
  target.__proto__ = src
}

// 通过复制将拦截器中的方法挂载到value上
function copyAugment (target: Object, src: Object, keys: Array<string>) {
  for (let i = 0, l = keys.length; i < l; i++) {
    const key = keys[i]
    def(target, key, src[key])
  }
}
```

### 总结
vue如何实现响应式?具体实现上对象和数组稍有不同：
* 1、对象：在create阶段，会递归的将data中的数据递归的添加get、set访问器属性,页面在mount阶段会创建全局的Watcher,并且mount阶段需要执行render渲染，会调用页面数据对应的get函数，每个数据的key都有对应的dep依赖，执行dep.depend()时会将 将dep 添加至当前watcher的subs队列中去。当页面数据更新后，调用set函数，执行通知。
* 2、数组：在create阶段，如果是数组类型时，给会执行数组改变方法添加拦截器，同时也会给数据添加get和set访问器属性，只是数组改变时并不会触发set函数，页面在mount阶段执行render,调用数据对应的get函数，并调用childObj.dep.depend()收集watcher,(childObj.dep是什么？在初始化的data的时候会递归的将array转成observer,所以childObj.dep指的是数组array的依赖)。在array数据更新之后，会执行拦截器中的__obj__.dep.notify()执行通知，set并不会触发。

通知之后页面怎么更新渲染？
当发送通知之后，会将watcher添加至队列中由vue统一调度执行更新，后期vue将会进行patch，对比虚拟dom，以当前页面组件级别做一个整体更新。