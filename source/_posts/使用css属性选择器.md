---
layout: post
title: 使用css属性选择器
abbrlink: 59170
date: 2022-05-15 19:41:24
tags: css属性选择器
categories: css
---

# 1 引言

虽然现在 css Module 与 css-in-js 更流行，但使用它们会导致过分依赖 **滥用 class 做唯一定位**，违背了css 选择器的初衷。

本期精读的文章是：[attribute-selectors-splicing-html-dna-css](https://www.smashingmagazine.com/2018/10/attribute-selectors-splicing-html-dna-css/)，带你重新理解强大的 Css 选择器。

# 2 概要

css Module 与 css-in-js 大部分场景使用 className 作为选择器，那么本文以选择器为重点，看看选择器有哪些实用的用法。

## 属性选择器

如果你想选择包含 `title` 属性的 `div`：

```css
div[title]
```

选择包含 `title` 属性的子元素，只需要加个空格：

```css
div [title]
```

选择 `title` 内容是 `dna` 的元素：

```css
div[title="dna"]
```

选择 `title` 属性包含 `dna` 单词的元素：

> 注意 dna 需要是单词，也就是用空格分割，比如 “my beautiful dna” 或 “mutating dna is fun!”

```css
div[title~="dna"]
```

和正则类似，选择 `title` 属性中，以 `dna` 结尾的元素：

```css
div[title$="dna"]
```

以 `dna` 开头：

```css
div[title^="dna"]
```

如果希望选择 `dna` 或 `dna-zh`，但不希望匹配 `dnaer`，可以：

> 这种场景一般用在国际化，比如 en en-us 就可以用 `|="en"`

```css
div[title|="dna"]
```

只要包含 `dna` 这三个字符就选中：

```css
div[title*="dna"]
```

真的很像正则，你可以用 `i` 标识匹配时大小写不敏感：

```css
div[title*="dna" i]

```

如果你想找到一个 `a` 标签，拥有 `title` 属性并且 className 以 `genes` 结尾，可以这样：

```css
a[title][class$="genes"]

```

## 获取标签的值

可以用 `attr` 标识符拿到当前选择器选中元素的属性，比如当 `hover` 状态时，在文字尾部显示其 `title` 属性：

```css
.joke:hover:after {
  content: "Answer:" attr(title);
  display: block;
}

```

## 其它用法

本文还介绍了一些实用技巧，比如

**根据输入框类型设置样式**

```css
input[type="email"] {
  color: papayawhip;
}
input[type="tel"] {
  color: thistle;
}

```

**改变下载标签的 icon**

```css
a[download][href$="pdf"]:after {
  content: url(pdf-icon.svg);
}

```

当然也可以选中一些老代码进行样式重写，比如：

```html
<div bgcolor="#000000" color="#FFFFFF">Old, holey genes</div>

```

```css
div[bgcolor="#000000"] {
  /*override*/
  background-color: #222222 !important;
}

```

不过这种用法要谨慎，写的越多越难以维护。

**结合一些新标签功能**

比如 `details` 标签是 html 原生的手风琴折叠组件：

```html
<details> <summary>List of Genes</summary> Roddenberry Hackman </details>

```

我们可以使用属性选择器，定义其打开时的样式：

```css
details[open] {
  background-color: hotpink;
}

```

**为没有 `async` 标记的 `script` 标签着色**，算是友情提示哪儿有错误：

```css
script[src]:not([async]) {
  display: block;
  width: 100%;
  height: 1em;
  background-color: red;
}
script:after {
  content: attr(src);
}

```

**为 JS 事件着色**，比如触发的鼠标事件可以作为选择器：

```css
[OnMouseOver] {
  color: burlywood;
}
[OnMouseOver]:after {
  content: "JS: " attr(OnMouseOver);
}

```

**选中隐藏元素：**

```css
[hidden],
[type="hidden"] {
  display: block;
}

```

还有更多就不一一列举了，感兴趣的读者可以跳转到原文继续阅读。大部分内容其实都写在了 [w3school 选择器参考手册](http://www.w3school.com.cn/cssref/css_selectors.asp)，只是结合一篇文章来读，可以理解得更深刻，同时文章里确实有一些新鲜的选择器，比如 JS 事件选择器，HTML5 属性标签选择器等等。

# 3 精读

这篇文章确实说明了 Css 选择器的强大性，但回到 css module 或者 css-in-js 的工程代码里，我们往往难以做太多的实践，有如下几个原因：

## 一直在担心的 DOM 结构变动

业务开发中，大量需求涌入，也许过了一周，DOM 结构就已经面目全非了，而且就算是一个普通的圣杯布局，可能老版本用 Table 布局，后面进来一个年轻小伙子直接用 div + flex 重构了，你会担心之前写的 table 选择器在某一天全部失效。

也许今天的 div 选择器，明天因为语义化改造就换成了 article 标签。

最大原因是 **一种视觉界面对应的实现方式太多**，不仅标签可以各异，css 属性还有 table、block、flex、grid 可选，同时 grid 属性还会导致视觉结构与 DOM 结构不完全对应。

如果你今天用 css 选择器做了一套完全贴合现在 DOM 结构的 css 文件，这个 css 文件也许是后面 dom 结构改动的噩梦。

## 你敢做全局样式覆盖吗

我们排除标签，仅对属性做全局覆盖，的确可以部分绕开 DOM 结构的限制，但是这样的全局样式覆盖，不同的人有不同看法。

小明的团队非常懂得 css 运用，他们每天都会花一个小时讨论项目的 css 架构，并对通用需求样式做了抽象，并且每个人都很认可这个方案，在他们的团队，一个非常酷炫的按钮与动画效果，通过 `<button animate />` 就可以完成，页面间交互非常流畅，用户体验统一，前端代码也非常简洁和优雅。

小白的团队水平参差不齐，有人永远只使用 table 布局，有人却总想将一些试验阶段 css 属性用在生产环境，小白自己抽象了一个全局样式 css 文件，可团队没什么时间沟通，甚至有人私下也注入了不少全局 css 样式，总有人抱怨自己的样式被全局覆盖了，最后小白甚至不得不在自己页面入口处写上 `*: unset` 清空各种奇怪的全局样式干扰，他想清空那该死的全局 css 样式文件，但他知道这样做带来的是更大的灾难。

可以看到，并不是每个团队都适合做全局样式覆盖。

## JS 模块化思维的影响

为什么一个项目安装了几百个 npm 三方包，却依然可以正常运行？因为好的三方包都是遵守模块化的，同时也不产生副作用，这样被使用时的效果就可以被预期，试想一下几百个 npm 包里同时定义了不同规范的全局 css 覆盖，你的项目会成为什么样。

当然 js 与 css 是不适合放在一起比较的，css 大多是业务级别的，也就是能写 css 只有做业务的你，第三方包一般是不会提供 css 定义干扰你的项目的。

然而大部分 UI 组件库是自带样式的，他们有自己的设计哲学，但为什么现在你会反感，而当初使用 Bootstrap 不会？

使用 Bootstrap 的时代，Bootstrap 一般是作为项目第一个依赖安装的，我们明确知道它会注入全局样式。我们会泡在他的官方文档目录，一条条理解他做的全局样式规则，他提供的各种 class。

然而现在是一个 Css-in-js 的时代，或者至少是 css-in-npm 的时代，什么都用 npm 装，什么都是模块化的，很多时候我们用一个 UI 组件仅仅是为了在某一处地方使用，而不想接受他带来的全局样式污染，视觉设计哲学，更不想看他的 css 文档。所以好的组件库往往 css 使用的很收敛，尽量不要对用户项目环境造成影响。

如果你项目的样式已经被不得不安装的第三方包全局覆盖得面目全非，每一次对全局样式修改都如履薄冰，可能你会比较反感 css 选择器，你会推崇更安全的 css modules，或甚至是 css-in-js，让每个组件的 className 都唯一，做到标签粒度的隔离。

# 4 总结

笔者认为，在一个确定的环境中，比如一个组件，一个独立负责的模块，是比较适合用 css 选择器的，这样可以让样式代码更易读，DOM 结构更清爽。但请一定注意作用域，如果不是大家一起达成的共识，最好不要放到全局样式中。

就算项目的风格非常明确，`a` 标签一定要用红色，在把这条规则放到全局样式之前，请思考一下，这样会不会破坏了某个用 `a` 标签模拟按钮的组件库的样式？

css 属性选择器的强大功能，需要有良好的项目管理做支撑，或者通过技术手段比如 shadow dom 做支撑。不过 shadow dom 的支持程度 [现在仍然很低](https://caniuse.com/#search=shadow%20dom)，所以使用编译工具做的隔离，在某种程度上模拟了 Css 选择器，承担了 Css 选择器 + shadow dom 的功能。

一切样式都用 className 控制，也许是 shadow dom 出来前的一种妥协方案，这篇文章更多是在描述 Css 选择器设计之美，但需要我们理性去使用。

> 讨论地址是：[精读《使用 CSS 属性选择器》 · Issue #113 · dt-fe/weekly](https://github.com/dt-fe/weekly/issues/113)

**如果你想参与讨论，请[点击这里](https://github.com/dt-fe/weekly)，每周都有新的主题，周末或周一发布。前端精读 - 帮你筛选靠谱的内容。**