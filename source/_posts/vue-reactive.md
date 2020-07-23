---
title: vue2.0响应式原理
date: 2020-07-16 09:49:44
tags:
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
``` js {19-21}
// observe/index.js
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
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
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


``` js {18-45}
// index.js
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
  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend() // get的时候开始收集依赖，将依赖存放在dep中,dep中存放的是什么？查看代码发现存放的是[Watcher，Watcher...]
        if (childOb) {
          childOb.dep.depend() // 收集依赖
          if (Array.isArray(value)) {
            dependArray(value) // 收集依赖
          }
        }
      }
      return value
    },
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

当访问数组中的方法时，由于添加了拦截器，当访问数组的方法时，会访问伪造的方法。
``` js
// 拦截器方法
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

import { arrayMethods } from './array'
const arrayKeys = Object.getOwnPropertyNames(arrayMethods)

export class Observer {
  ...
  constructor (value: any) {
    ...
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
