---
title: 作用域
date: 2021-03-10 20:49:44
tags: [JavaScript]
categories: scope
---

### 什么是作用域？
作用域是一套规则，用于确定在何处以及如何查询变量。
* 如果查找的目的是对变量进行赋值，那么就会使用LHS查询
* 如果查找的目的是获取变量的值，那么就会使用RHS查询

### 什么是作用域链？
当一个块或者函数嵌套在另一个块或函数中时，会发生作用域的嵌套。当需要查找某个变量时，会在当前作用域中查找，如果无法找到某个变量时，会在外层嵌套的作用域中继续查找，直到找到该变量或找到全局作用域为止。
``` javascript
var color = "blue"
function changeColor() {
    var anotherColor = "red"

    function swapColor() {
        var tempColor = anotherColor
        anotherColor = color
        color = tempColor
    }

    swapColor()
}

changeColor()
```
{% img /images/js/scope/scope.png "点击查看大图:vi/vim-cheat-sheet" %}
如图: 内部环境可以通过作用域链访问外部环境，但是外部环境不能访问内部环境中的任何变量和函数。
`swapColor()`环境中可以访问外部环境`changeColor()`中的`anotherColor`和全局环境中的`color`。

### 异常
``` javascript
function foo(a) {
    console.log(a + b)
}
foo(2)
```
上述代码，对`b`进行`RHS`嵌套查询作用域，是查不到该变量的，这是引擎会抛出`ReferenceError`异常。

``` javascript
function foo() {
    b = 2
    console.log(b)
}
foo()

console.log(window.b) // 2
```
上述代码，在非严格模式下 对`b`进行了`LHS`嵌套查询作用域，如果在全局作用域也无法查到该变量时，
全局作用域中就会创建一个具有该名称的变量，并将其返回给引擎。

### 词法作用域
JavaScript中的作用域就是词法作用域。词法作用域是由你在写代码时将变量和块作用域写在哪里决定的，
词法分析器在处理代码时会保持作用域不变。

``` javaScript
function foo() {
    console.log(a)
}

function bar() {
    var a = 3
    foo()
}

var a = 2
bar() // 2
```
上述代码，打印的结果是`2`,因为在词法分析阶段，a的值通过作用域链取到的是全局作用域下的`a = 2`,
在调用`bar()`时，执行`foo()`函数时，是取不到`bar()`下的`a = 3`

### 作用域闭包
闭包是指有权访问另一个函数作用域中的变量的函数。
``` javascript
function foo() {
    var a = 2
    function bar() {
        console.log(a)
    }
    return bar
}

var baz = foo()

baz()
```



