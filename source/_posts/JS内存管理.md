---
layout: post
title: JS内存管理
abbrlink: 33540
date: 2022-05-24 22:45:53
tags: 内存管理
categories: JavaScript
summary:   JS作为一门高级语言,前端程序员也有必要掌握基础内存分配原理
---

本期精读的文章是：

[How JavaScript works: memory management + how to handle 4 common memory leaks](https://blog.sessionstack.com/how-javascript-works-memory-management-how-to-handle-4-common-memory-leaks-3f28b94cfbec)

# 1 引言

我为什么要选这篇文章呢？

sessionstack 最近接连发了好几篇文章, 深入探讨 JS, 以及  JS 中一些内部原理. 文中也讲到了, 伴随深入了解 JS 中的一些工作原理, 才有可能写出更好的代码和程序.

而 JS 中的内存管理, 我的感觉就像 JS 中的一门副科, 我们平时不会太重视, 但是一旦出问题又很棘手. 所以可以通过平时多了解一些 JS 中内存管理问题, 在写代码中通过一些习惯, 避免内存泄露的问题.


# 2 内容概要

### 内存生命周期

![](https://raw.githubusercontent.com/dt-fe/weekly/master/assets/29/1.jpg)

不管什么程序语言，内存生命周期基本是一致的：

1. 分配你所需要的内存
2. 使用分配到的内存（读, 写）
3. 不需要时将其释放/归还

在 C 语言中, 有专门的内存管理接口, 像`malloc()` 和 `free()`. 而在 JS 中, 没有专门的内存管理接口, 所有的内存管理都是"自动"的. JS 在创建变量时, 自动分配内存, 并在不使用的时候, 自动释放. 这种"自动"的内存回收, 造成了很多 JS 开发并不关心内存回收, 实际上, 这是错误的.

## JS 中的内存回收
### 引用
垃圾回收算法主要依赖于引用的概念. 在内存管理的环境中, 一个对象如果有访问另一个对象的权限（隐式或者显式）, 叫做一个对象引用另一个对象. 例如: 一个 Javascript 对象具有对它原型的引用（隐式引用）和对它属性的引用（显式引用）.

### 引用计数垃圾收集
这是最简单的垃圾收集算法.此算法把“对象是否不再需要”简化定义为“对象有没有其他对象引用到它”. 如果没有引用指向该对象（零引用, 对象将被垃圾回收机制回收.
示例:
```javascript
let arr = [1, 2, 3, 4];
arr = null; // [1,2,3,4]这时没有被引用, 会被自动回收
```

### 限制: 循环引用
在下面的例子中, 两个对象对象被创建并互相引用, 就造成了循环引用. 它们被调用之后不会离开函数作用域, 所以它们已经没有用了, 可以被回收了. 然而, 引用计数算法考虑到它们互相都有至少一次引用, 所以它们不会被回收.
```javascript
function f() {
  var o1 = {};
  var o2 = {};
  o1.p = o2; // o1 引用 o2
  o2.p = o1; // o2 引用 o1. 这里会形成一个循环引用
}
f();
```
![](https://raw.githubusercontent.com/dt-fe/weekly/master/assets/29/2.jpg)

实际例子:
```javascript
var div;
window.onload = function(){
  div = document.getElementById("myDivElement");
  div.circularReference = div;
  div.lotsOfData = new Array(10000).join("*");
};
```
在上面的例子里, myDivElement 这个 DOM 元素里的 circularReference 属性引用了 myDivElement, 造成了循环引用. IE 6, 7 使用引用计数方式对 DOM 对象进行垃圾回收. 该方式常常造成对象被循环引用时内存发生泄漏.  现代浏览器通过使用标记-清除内存回收算法, 来解决这一问题.

### 标记-清除算法
这个算法把“对象是否不再需要”简化定义为“对象是否可以获得”.

这个算法假定设置一个叫做根`root`的对象(在 Javascript 里，根是全局对象). 定期的, 垃圾回收器将从根开始, 找所有从根开始引用的对象, 然后找这些对象引用的对象, 从根开始,垃圾回收器将找到所有可以获得的对象和所有不能获得的对象.

从 2012 年起, 所有现代浏览器都使用了标记-清除内存回收算法。所有对 JavaScript 垃圾回收算法的改进都是基于标记-清除算法的改进.
![](https://raw.githubusercontent.com/dt-fe/weekly/master/assets/29/3.gif)

### 自动 GC 的问题
尽管自动 GC 很方便, 但是我们不知道 GC 什么时候会进行. 这意味着如果我们在使用过程中使用了大量的内存, 而 GC 没有运行的情况下, 或者 GC 无法回收这些内存的情况下, 程序就有可能假死, 这个就需要我们在程序中手动做一些操作来触发内存回收.

### 什么是内存泄露?
本质上讲, 内存泄露就是不再被需要的内存, 由于某种原因, 无法被释放.

![](https://raw.githubusercontent.com/dt-fe/weekly/master/assets/29/4.jpg)

## 常见的内存泄露案例
#### 1. 全局变量
```javascript
function foo(arg) {
    bar = "some text";
}
```
在 JS 中处理未被声明的变量, 上述范例中的 `bar`时, 会把`bar`, 定义到全局对象中, 在浏览器中就是 `window` 上. 在页面中的全局变量, 只有当页面被关闭后才会被销毁. 所以这种写法就会造成内存泄露, 当然在这个例子中泄露的只是一个简单的字符串, 但是在实际的代码中, 往往情况会更加糟糕.

另外一种意外创建全局变量的情况.
```javascript
function foo() {
    this.var1 = "potential accidental global";
}
// Foo 被调用时, this 指向全局变量(window)
foo();
```
在这种情况下调用`foo`,  this 被指向了全局变量`window`, 意外的创建了全局变量.

我们谈到了一些意外情况下定义的全局变量, 代码中也有一些我们明确定义的全局变量. 如果使用这些全局变量用来暂存大量的数据, 记得在使用后, 对其重新赋值为 null.
#### 2. 未销毁的定时器和回调函数
在很多库中, 如果使用了观察者模式, 都会提供回调方法, 来调用一些回调函数. 要记得回收这些回调函数. 举一个 setInterval 的例子.
```javascript
var serverData = loadData();
setInterval(function() {
    var renderer = document.getElementById('renderer');
    if(renderer) {
        renderer.innerHTML = JSON.stringify(serverData);
    }
}, 5000); // 每 5 秒调用一次
```
如果后续 `renderer` 元素被移除, 整个定时器实际上没有任何作用. 但如果你没有回收定时器, 整个定时器依然有效, 不但定时器无法被内存回收, 定时器函数中的依赖也无法回收. 在这个案例中的 `serverData` 也无法被回收.

#### 3. 闭包
在 JS 开发中, 我们会经常用到闭包, 一个内部函数, 有权访问包含其的外部函数中的变量. 下面这种情况下, 闭包也会造成内存泄露.
```Javascript
var theThing = null;
var replaceThing = function () {
  var originalThing = theThing;
  var unused = function () {
    if (originalThing) // 对于 'originalThing'的引用
      console.log("hi");
  };
  theThing = {
    longStr: new Array(1000000).join('*'),
    someMethod: function () {
      console.log("message");
    }
  };
};
setInterval(replaceThing, 1000);
```
这段代码, 每次调用`replaceThing`时, `theThing` 获得了包含一个巨大的数组和一个对于新闭包`someMethod`的对象.  同时 `unused` 是一个引用了`originalThing`的闭包.

这个范例的关键在于, 闭包之间是共享作用域的, 尽管`unused`可能一直没有被调用, 但是`someMethod` 可能会被调用, 就会导致内存无法对其进行回收. 当这段代码被反复执行时, 内存会持续增长.

该问题的更多描述可见[Meteor 团队的这篇文章](https://blog.meteor.com/an-interesting-kind-of-javascript-memory-leak-8b47d2e7f156).
#### 4. DOM 引用
很多时候, 我们对 Dom 的操作, 会把 Dom 的引用保存在一个数组或者 Map 中.
```Javascript
var elements = {
    image: document.getElementById('image')
};
function doStuff() {
    elements.image.src = 'http://example.com/image_name.png';
}
function removeImage() {
    document.body.removeChild(document.getElementById('image'));
    // 这个时候我们对于 #image 仍然有一个引用, Image 元素, 仍然无法被内存回收.
}
```
上述案例中, 即使我们对于 image 元素进行了移除, 但是仍然有对 image 元素的引用, 依然无法对齐进行内存回收.

另外需要注意的一个点是, 对于一个 Dom 树的叶子节点的引用. 举个例子: 如果我们引用了一个表格中的`td`元素, 一旦在 Dom 中删除了整个表格, 我们直观的觉得内存回收应该回收除了被引用的 `td`外的其他元素. 但是事实上, 这个`td` 元素是整个表格的一个子元素, 并保留对于其父元素的引用. 这就会导致对于整个表格, 都无法进行内存回收. 所以我们要小心处理对于 Dom 元素的引用.
# 3 精读

ES6 中引入`WeakSet` 和 `WeakMap`两个新的概念, 来解决引用造成的内存回收问题. `WeakSet` 和 `WeakMap`对于值的引用可以忽略不计, 他们对于值的引用是弱引用,内存回收机制, 不会考虑这种引用. 当其他引用被消除后, 引用就会从内存中被释放.

JS 这类高级语言，隐藏了内存管理功能。但无论开发人员是否注意，内存管理都在那，所有编程语言最终要与操作系统打交道，在内存大小固定的硬件上工作。不幸的是，即使不考虑垃圾回收对性能的影响，2017 年最新的垃圾回收算法，也无法智能回收所有极端的情况。

唯有程序员自己才知道何时进行垃圾回收，而 JS 由于没有暴露显示内存管理接口，导致触发垃圾回收的代码看起来像“垃圾”，或者优化垃圾回收的代码段看起来不优雅、甚至不可读。

所以在 JS 这类高级语言中，有必要掌握基础内存分配原理，在对内存敏感的场景，比如 nodejs 代码做严格检查与优化。谨慎使用 dom 操作、主动删除没有业务意义的变量、避免提前优化、过度优化，在保证代码可读性的前提下，利用性能监控工具，通过调用栈定位问题代码。

同时对于如何利用 chrome 调试工具, 分析内存泄露的方法和技巧. 可以参考上期精读[精读《2017 前端性能优化备忘录》](https://zhuanlan.zhihu.com/p/30349982)

# 4 总结

即便在 JS 中, 我们很少去直接去做内存管理. 但是我们在写代码的时候, 也要有内存管理的意识, 谨慎的处理可能会造成内存泄露的场景.

> 讨论地址是：[精读《JS 中的内存管理》 · Issue #40 · dt-fe/weekly](https://github.com/dt-fe/weekly/issues/40)

> 如果你想参与讨论，请[点击这里](https://github.com/dt-fe/weekly)，每周都有新的主题，每周五发布。




参考文章: [MDN 的内存管理介绍](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Memory_Management)