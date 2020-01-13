---
title: 什么是passive listeners
date: 2020-1-13 15:30:30
tags:
  - passive
  - addEventListener
---

最近年末等着过年，干的是修修补补的活。这不，在某项目里发现了个2018年的问题还没改呢。

<!--more-->

### 什么问题

最近 Chrome 更新到了 79，打开某项目赫然发现一条**error**level 的日志:

> [Intervention] Unable to preventDefault inside passive event listener due to target being treated as passive. See https://www.chromestatus.com/features/66626470

顺藤摸瓜，发现我们在一个`ul`的节点上绑定了一个`onWheel`方法，并且在这个方法里调用了`preventDefault`方法。但是，此处`onWheel`是一个`passive`事件，该方法无效了。
为什么以前没有发现呢？chrome从56+就支持了事件的`passive`属性，并且从 73+开始，强制爆出此错误。我们还可以方便的在`Event Listener`中切换这个属性：
![](/post-images/passive-flag.png)
因为我们在此处没有过多的性能需求，所以解决方法很简单：`onWheel`事件改成非`passive`事件。

### 原理
- addEventListener 第三个参数除了可以为布尔值外，还可以是一个对象。若`passive`为`true`，表明listener不会调用`preventDefault`，即使调用了，也不会生效。
``` javascript
target.addEventListener(type, listener[, options]);
target.addEventListener(type, listener[, useCapture]);

/*
  [options]
  { 
    capture: 捕获或冒泡，默认为false，冒泡
    once: 是否只能调用一次，默认为false
    passive: 一般事件默认为false，特殊的scroll、wheel等为true
  }
 */
```
- 一般来讲，滚动事件过于复杂可能会阻塞浏览器的主线程，导致主线程不能处理滚动的重绘，造成极大的卡顿。`passive listeners`主要是为了解决滑动的性能问题，现在，一些浏览器默认把`scroll`、 `wheel`、`touchstart`、`touchmove`、`touchend`事件默认设置`passive`为`true`。据说，在移动端，这样的处理带来的性能提升是显而易见的。

> 简而言之就是当我们在滚动页面的时候（通常是我们监听touch事件的时候），页面其实会有一个短暂的停顿（大概200ms），浏览器不知道我们是否要preventDefault，所以它需要一个延迟来检测。这就导致了我们的滑动显得比较卡顿。

- 一段巧妙的特性检测代码
```javascript
let passiveIfSupported = false;
try {
  window.addEventListener(
    "test",
    null,
    Object.defineProperty({}, "passive", {
      get: function() {
        passiveIfSupported = { passive: true };
      }
    })
  );
} catch (err) {}

window.addEventListener(
  "scroll",
  function(event) {
    /* do something */
    // can't use event.preventDefault();
  },
  passiveIfSupported
);
```
### 参考
[Support Passive Event Listeners](https://github.com/facebook/react/issues/6436)
[Improving scrolling performance with passive listeners](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener#Improving_scrolling_performance_with_passive_listeners)