---
title: 虚拟DOM简介
date: 2020-12-20 13:49:44
tags: [vue]
categories: VUE
---

## 为什么要引入虚拟DOM?
### 频繁的操作DOM应发样式的改变，比较耗费浏览器的性能，那么为什么会耗费浏览器性能？
从解析HTML到生成DOM树再到页面渲染，浏览器需要渲染引擎处理一系列的操作，才能将页面绘制出来。
{% img /images/vue/webkit.png "点击查看大图:vi/vim-cheat-sheet" %}
webkit主流程
如图，在DOM树修改了之后，还需要经过样式计算、布局、分成、图层绘制、栅格化操作、合成和显示这些流程，如果频繁操作DOM,那么势必会影响浏览器的性能。
#### 样式计算
* 把CSS转换成浏览器能理解的结构
    * link外部css文件
    * style标记内的css
    * 元素内嵌css
执行转换操作，将css文本转换成styleSheets
* 标准化样式表属性值
如：red标准化为rgb(255,0,0) bold标准化为700
* 计算DOM节点具体样式
    * css继承
    所有子节点都会继承父节点的样式
    * 层叠规则

#### 布局
* 创建布局树
遍历DOM树所有节点，把这些节点加到布局树中，不可见节点忽略。
* 布局计算

#### 分成
{% img /images/vue/layer.png "点击查看大图:vi/vim-cheat-sheet" %}
布局树和图层树的关系示意图
并不是布局树的每个节点都包含一个图层，如果一个节点没有对应的图层，这个节点就从属于父节点的图层。
满足什么条件，渲染引擎才会为特定的节点创建新的层呢？
* 拥有层叠上下文属性的元素会被提升为单独的一层
* 需要裁减的地方会被创建为图层

#### 图层绘制
把图层的绘制拆分成很懂小的绘制指令，把这些指令按照顺序组成一个待绘制列表。
{% img /images/vue/render.png "点击查看大图:vi/vim-cheat-sheet" %}
渲染流水线示意图
如图，渲染进程的主线程执行完图层绘制之后，将绘制列表提交给合成线程。

#### 栅格化操作
* 图块（tile）
用户能看到的部分叫视口（viewport）,有的图层很大，页面滚动很久才能滚动到底部，要绘制图层所有内容的话，产生太大的开销，基于这个原因，合成线程将图层划分为图块。
栅格化是将图块转换为位图，图块是栅格化执行的最小单位。
渲染进程维护了一个栅格化的线程池。
{% img /images/vue/pools.png "点击查看大图:vi/vim-cheat-sheet" %}
合成线程提交图块给栅格化线程池

渲染进程把生成图块的指令发送给GPU,然后再GPU中执行生成图块位图，并保存在GPU内存中。

#### 合成和显示
浏览器接收合成线程绘制图块的命令----”DrawQuard“,将页面内容绘制到内存中，最终显示在屏幕上。

所以，生成DOM树之后，浏览器还需要经过样式计算、布局、分成、图层绘制、栅格化操作、合成显示一系列的流程，才能渲染显示在屏幕上，如果用户频繁的操作DOM,造成页面重绘或回流，浏览器仍需要按流程处理生成页面，会影响浏览器的性能。

### 什么是虚拟DOM
虚拟DOM是普通的JavaScript对象，用来描述一个真实的DOM元素。
VNode类的代码如下，用VNode来描述DOM节点对象。
```
export default class VNode {
  constructor (tag, data, children, text, elm, context, componentOptions, asyncFactory) {
    this.tag = tag
    this.data = data
    this.children = children
    this.text = text
    this.elm = elm
    this.ns = undefined
    this.context = context
    this.functionalContext = undefined
    this.functionalOptions = undefined
    this.functionalScopeId = undefined
    this.key = data && data.key
    this.componentOptions = componentOptions
    this.componentInstance = undefined
    this.parent = undefined
    this.raw = false
    this.isStatic = false
    this.isRootInsert = true
    this.isComment = false
    this.isCloned = false
    this.isOnce = false
    this.asyncFactory = asyncFactory
    this.asyncMeta = undefined
    this.isAsyncPlaceholder = false
  }

  get child () {
    return this.componentInstance
  }
}
```

### 为什么要使用虚拟DOM?
当状态发生变化时，组件的监听器watcher监听到状态的变化，通知组件对比新创建的虚拟DOM和缓存的虚拟DOM节点进行patch对比，且根据对比结果只更新需要更新的真实DOM节点，从而避免不必要的DOM操作。

### Vue虚拟节点
* 注释节点
* 文本节点
* 元素节点
* 组件节点
* 函数式组件
* 克隆节点
