---
layout: post
title: Async/Await是把双刃剑
abbrlink: 65303
date: 2022-05-19 11:40:29
tags: async await
categories: JavaScript
summary: 不要过度依赖async/await语法糖,因为
---

本周精读内容是 [《async/await 是把双刃剑》](https://medium.freecodecamp.org/avoiding-the-async-await-hell-c77a0fb71c4c)。

## 1 引言

终于，async/await 也被吐槽了。Aditya Agarwal 认为 async/await 语法让我们陷入了新的麻烦之中。

其实，笔者也早就觉得哪儿不对劲了，终于有个人把实话说了出来，async/await 可能会带来麻烦。

## 2 概述

下面是随处可见的现代化前端代码：

```typescript
(async () => {
  const pizzaData = await getPizzaData(); // async call
  const drinkData = await getDrinkData(); // async call
  const chosenPizza = choosePizza(); // sync call
  const chosenDrink = chooseDrink(); // sync call
  await addPizzaToCart(chosenPizza); // async call
  await addDrinkToCart(chosenDrink); // async call
  orderItems(); // async call
})();
```

await 语法本身没有问题，有时候可能是使用者用错了。当 `pizzaData` 与 `drinkData` 之间没有依赖时，顺序的 await 会最多让执行时间增加一倍的 `getPizzaData` 函数时间，因为 `getPizzaData` 与 `getDrinkData` 应该并行执行。

回到我们吐槽的回调地狱，虽然代码比较丑，带起码两行回调代码并不会带来阻塞。

看来语法的简化，带来了性能问题，而且直接影响到用户体验，是不是值得我们反思一下？

正确的做法应该是先同时执行函数，再 await 返回值，这样可以并行执行异步函数：

```typescript
(async () => {
  const pizzaPromise = selectPizza();
  const drinkPromise = selectDrink();
  await pizzaPromise;
  await drinkPromise;
  orderItems(); // async call
})();
```

或者使用 `Promise.all` 可以让代码更可读：

```typescript
(async () => {
  Promise.all([selectPizza(), selectDrink()]).then(orderItems); // async call
})();
```

看来不要随意的 await，它很可能让你代码性能降低。

## 3 精读

仔细思考为什么 async/await 会被滥用，笔者认为是它的功能比较反直觉导致的。

首先 async/await 真的是语法糖，功能也仅是让代码写的舒服一些。先不看它的语法或者特性，仅从语法糖三个字，就能看出它一定是局限了某些能力。

举个例子，我们利用 html 标签封装了一个组件，带来了便利性的同时，其功能一定是 html 的子集。又比如，某个轮子哥觉得某个组件 api 太复杂，于是基于它封装了一个语法糖，我们多半可以认为这个便捷性是牺牲了部分功能换来的。

功能完整度与使用便利度一直是相互博弈的，很多框架思想的不同开源版本，几乎都是把功能完整度与便利度按照不同比例混合的结果。

那么回到 async/await 它的解决的问题是回调地狱带来的灾难：

```typescript
a(() => {
  b(() => {
    c();
  });
});
```

为了减少嵌套结构太多对大脑造成的冲击，async/await 决定这么写：

```typescript
await a();
await b();
await c();
```

虽然层级上一致了，但逻辑上还是嵌套关系，这不是另一个程度上增加了大脑负担吗？而且这个转换还是隐形的，所以许多时候，我们倾向于忽略它，所以造成了语法糖的滥用。

### 理解语法糖

虽然要正确理解 async/await 的真实效果比较反人类，但为了清爽的代码结构，以及防止写出低性能的代码，还是挺有必要认真理解 async/await 带来的改变。

首先 async/await 只能实现一部分回调支持的功能，也就是仅能方便应对层层嵌套的场景。其他场景，就要动一些脑子了。

比如两对回调：

```typescript
a(() => {
  b();
});

c(() => {
  d();
});
```

如果写成下面的方式，虽然一定能保证功能一致，但变成了最低效的执行方式：

```typescript
await a();
await b();
await c();
await d();
```

因为翻译成回调，就变成了：

```typescript
a(() => {
  b(() => {
    c(() => {
      d();
    });
  });
});
```

然而我们发现，原始代码中，函数 `c` 可以与 `a` 同时执行，但 async/await 语法会让我们倾向于在 `b` 执行完后，再执行 `c`。

所以当我们意识到这一点，可以优化一下性能：

```typescript
const resA = a();
const resC = c();

await resA;
b();
await resC;
d();
```

但其实这个逻辑也无法达到回调的效果，虽然 `a` 与 `c` 同时执行了，但 `d` 原本只要等待 `c` 执行完，现在如果 `a` 执行时间比 `c` 长，就变成了:

```typescript
a(() => {
  d();
});
```

看来只有完全隔离成两个函数：

```typescript
(async () => {
  await a();
  b();
})();

(async () => {
  await c();
  d();
})();
```

或者利用 `Promise.all`:

```typescript
async function ab() {
  await a();
  b();
}

async function cd() {
  await c();
  d();
}

Promise.all([ab(), cd()]);
```

这就是我想表达的可怕之处。回调方式这么简单的过程式代码，换成 async/await 居然写完还要反思一下，再反推着去优化性能，这简直比回调地狱还要可怕。

而且大部分场景代码是非常复杂的，同步与 await 混杂在一起，想捋清楚其中的脉络，并正确优化性能往往是很困难的。但是我们为什么要自己挖坑再填坑呢？很多时候还会导致忘了填。

原文作者给出了 `Promise.all` 的方式简化逻辑，但笔者认为，不要一昧追求 async/await 语法，在必要情况下适当使用回调，是可以增加代码可读性的。

## 4 总结

async/await 回调地狱提醒着我们，不要过度依赖新特性，否则可能带来的代码执行效率的下降，进而影响到用户体验。同时，笔者认为，也不要过度利用新特性修复新特性带来的问题，这样反而导致代码可读性下降。

当我翻开 redux 刚火起来那段时期的老代码，看到了许多过度抽象、为了用而用的代码，硬是把两行代码能写完的逻辑，拆到了 3 个文件，分散在 6 行不同位置，我只好用字符串搜索的方式查找线索，最后发现这个抽象代码整个项目仅用了一次。

写出这种代码的可能性只有一个，就是在精神麻木的情况下，一口气喝完了 redux 提供的全部鸡汤。

就像 async/await 地狱一样，看到这种 redux 代码，我觉得远不如所谓没跟上时代的老前端写出的 jquery 代码。

决定代码质量的是思维，而非框架或语法，async/await 虽好，但也要适度哦。

> PS: 经过讨论，笔者把原文 async/await 地狱标题改成了 async/await 是把双刃剑。因为 async/await 并没有回调地狱那么可怕，称它为地狱有误导的可能性。

## 5 更多讨论

> 讨论地址是：[精读《async/await 是把双刃剑》 · Issue #82 · dt-fe/weekly](https://github.com/dt-fe/weekly/issues/82)

**如果你想参与讨论，请[点击这里](https://github.com/dt-fe/weekly)，每周都有新的主题，周末或周一发布。**