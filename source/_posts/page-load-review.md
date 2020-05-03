---
title: 图解页面加载
date: 2020-04-05 12:11:10
tags:
  - Performance
  - DOM
  - CSSOM
---

最近对页面加载整个流程的细节有点遗忘，抽空利用 Chrome 的 Performance 观察了几遍，记录一下。

<!--more-->

我们带着下面几个问题：

- script 会阻止 DOM 的构建吗？
- css 会阻止 JS 执行吗？css 会阻止 DOM 的构建吗？
- SPA 中为什么 css 放 header，script 放在 body 后？骨架图是什么原理？
- DomContentLoaded、readystate、Load 的关系是什么？
- DOM、CSSOM、Render Tree 之间到关系是什么？

## 总览

我们选用了下面这个简单的 SPA 为例，利用 Chrome 录制了页面加载的全部过程。这是一个标准的 SPA 首页，在 head 中含有一个 CSS 资源，在 body 的最后含有一个 JS 脚本。同时，body 中还有一小段起到“骨架图”作用的片段。

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>首屏测试</title>
    <link rel="icon" href="/favicon.ico" type="image/x-icon" />
    <link href="/assets/styles/index.3c893753.css" rel="stylesheet" />
  </head>
  <body>
    <div id="loading">
      <div id="loadAnimate">
        <span></span>
        <span></span>
        <span></span>
        <span></span>
      </div>
    </div>
    <style>
      #loading {
        background-color: rgba(51, 51, 51, 0.4);
        height: 100%;
        width: 100%;
        position: fixed;
        z-index: 1;
        margin-top: 0;
        top: 0;
        left: 0;
      }

      #loadAnimate {
        position: absolute;
        top: 50%;
        left: 50%;
        width: 42px;
        height: 42px;
        margin: -21px 0 0 -21px;
        animation: loadAnimate 5s infinite linear;
      }

      #loadAnimate span {
        width: 20px;
        height: 20px;
        border-radius: 5px;
        position: absolute;
        background: red;
        display: block;
        animation: loadAnimate_span 1s infinite linear;
      }

      #loadAnimate span:nth-child(1) {
        background: #00b4ed;
      }

      #loadAnimate span:nth-child(2) {
        left: 22px;
        background: #27cbff;
        animation-delay: 0.2s;
      }

      #loadAnimate span:nth-child(3) {
        top: 22px;
        background: #72ddff;
        animation-delay: 0.4s;
      }

      #loadAnimate span:nth-child(4) {
        top: 22px;
        left: 22px;
        background: #b1ecff;
        animation-delay: 0.6s;
      }

      @keyframes loadAnimate {
        from {
          transform: rotate(0deg);
        }

        to {
          transform: rotate(360deg);
        }
      }

      @keyframes loadAnimate_span {
        0% {
          transform: scale(1);
        }

        50% {
          transform: scale(0.5);
        }

        100% {
          transform: scale(1);
        }
      }
    </style>
    <script type="text/javascript">
      function removeLoading() {
        var loading = document.getElementById("loading");
        loading.parentNode.removeChild(loading);
        document.on`readystatechange` = null;
      }
      var timeoutHandle;
      document.on`readystatechange` = function () {
        console.log(document.readyState);
        if (document.readyState === "interactive") {
          timeoutHandle = setTimeout(function () {
            removeLoading();
          }, 5000);
        } else {
          if (document.readyState === "complete") {
            removeLoading();
            if (timeoutHandle) {
              clearTimeout(timeoutHandle);
            }
          }
        }
      };
    </script>
    <div id="root"></div>
    <script
      type="text/javascript"
      src="/assets/scripts/index.a3c6208c.js"
    ></script>
  </body>
</html>
```

我们点击 Performance 面板的`Start profiling and reload page`按钮，等它执行完毕，得到了下面的记录：

![](/post-images/page-load-1.png)

## 白屏到 FP

这个阶段从发出请求获取 HTML 开始，到页面的第一次绘制为止，经历了资源下载、HTML 解析、DOM 构建、获取 CSS、获取 JS、CSSDOM 构建、JS 脚本编译和执行等过程。

### 解析 HTML

下面截图展示的四个片段，是本次页面加载 Chrome 记录的所有`Parse HTML`片段（图中仅显示了 HTML 的解析，DOM 的构建没有明确标出）。我们可以发现，HTML 的解析在接收到部分数据流后就开始了，并不需要等待整个文档加载完成。同时，在解析到需要外部资源时，便会请求这些资源。
下面三个截图发生在 FP 之前，代表了三个时刻：

1. 解析到`link`标签，请求该 CSS 资源。

![](/post-images/page-load-2.png)

2. 解析到 body 中的第二个`script`资源，请求该资源。按照浏览器的执行过程，执行到第一个`script`时，DOM 暂停构建，直到该脚本加载和执行完毕。此处我们发现，HTML 的解析依旧会继续，解析到了一个外部的脚本资源，便发出请求去获取。解析 HTML 和构建 DOM 关联密切，但不是完全等价。

![](/post-images/page-load-3.png)

3. FP 阶段。CSSOM 构建完成，执行内联`script`。在这个阶段后不久，我们的首屏动画就显示出来了。

![](/post-images/page-load-4.png)

4. 在 FP 之后，浏览器等待`index.a3c6208c.js`下载完成，并进行解析和执行。执行完脚本后，浏览器便完成了初始文档的解析和 DOM 树的构建，随后触发了 DOMContendLoaded 事件。额外的，上述脚本又穿插了一系列资源，它们都立刻被请求了。

![](/post-images/page-load-5.png)

### 构建和暂停

第三和第四个片段中，我们发现`Parse HTML`前都有一段`Evaluate Script`的时刻，这即是我们平常所说的遇到 JS 脚本，则暂停 DOM 构建，等待 JS 执行完成。
而第三阶段在脚本执行前又有一段 CSSOM 构建的过程，这便是 JS 脚本会等待 CSSOM 构建完成的一个例子。
我们也会发现，在等待第一个`script`脚本执行的过程中，第二个外联`script`脚本的请求却已经发出了。这是一个浏览器的优化，即使在 DOM 树暂停构建时，后面的资源也会被提前加载。
Chorme 记录的数据似乎没有明显的标志出 DOM 的构建（尚有疑问：解析 HTML 的同时，DOM 解析也在进行吗？）。

对于 script，有两个属性可以使它不阻止 DOM 构建。设置了`defer`的外部脚本，不会在被解析到时立刻执行。浏览器会先下载它，然后在 HTML 文档全部解析后，DCL 事件触发前执行。设置了`async`的外部脚本，会在解析 HTML 的时候同步下载，并且一旦它下载完毕，就会立刻执行；但是它在下载的过程中，解析 HTML 的工作还在执行。
特别的，对于`type=module`的 script，`defer`属性是默认自带的（且不能设为 false），它总是在文档全部解析后才会执行；`async`属性将会使这类脚本在文档解析的同时下载其本身和所有依赖，一旦完成了下载，就立刻执行。更详细的内容，可以参考[这里](https://html.spec.whatwg.org/multipage/scripting.html#attr-script-async)。

对于 css，如果它被设置为`media=print`，那它也就不会阻止 JS 的执行了。其他特殊[媒体查询](https://drafts.csswg.org/mediaqueries/)可以查询这里。

### FP 需要什么

在第三个`Parse HTML`片段，我们发现浏览器第一次执行了`Layout`、`Paint`和`Composite`，不久之后，我们就看到了首屏的 loading 动画。这个阶段里，也是第一出现了`Parse Stylesheet`，之后，CSSOM 便准备就绪了。至此，我们拥有了 CSSOM 和部分 DOM。浏览器为了减少白屏等待的时间，会依照目前已有的资源，生成一颗`render tree`，一旦`render tree`准备就绪了，浏览器就可以绘制页面了。

## DCL 和 Onload

在 FP 之后的一段时间内，DOM 的解析依旧没有恢复，这是因为第二个 JS 脚本还没有下载完成。这段时间内，浏览器只能一直等待，页面上仅展现 FP 阶段的内容。当脚本文件完成下载、解析和执行后，浏览器恢复了 DOM 的构建。在所有的 HTML 解析完成、DOM 树构建完成后，浏览器便触发了[DCL](https://developer.mozilla.org/en-US/docs/Web/API/Document/DOMContentLoaded_event)事件。同时，这个 JS 又给文档添加了额外的 JS/CSS 资源，在这部分资源全部获取并解析后，Window 的 Onload 事件才会被触发。

DCL 和 Onload 事件有什么区别呢？

- 在触发时机上。当`HTML被完全加载以及解析时`，DCL 事件便会触发，而不必等待样式表，图片或者子框架完成加载。Onload 则需要等待整个页面所有资源都加载完毕，才会被触发。
- 除了 window 的 load 事件外，image/JS/CSS/XMLHttpRequest 都有其 load 事件。

readyState有4个值，会在`readystatechange`事件中触发。
- uninitialized - 还未开始载入
- loading - 载入中
- interactive - 已加载，文档与用户可以开始交互
- complete - 载入完成

它们之间的关系是：
1. readystate: interactive
2. DOMContentLoaded
3. readystate: complete
4. load

## FCP、FMP、LCP

这几个概念是 lighthouse 提出的性能指标，用来衡量 web 应用的性能。

- FCP First Contentful Paint
- FMP First Meaningful Paint
- LCP Largest Contentful Paint

## 有关 CSSOM

CSSOM 即 CSS Object Model，是一颗包含 CSS 有关信息的树，和 DOM 结构相似。它也提供了一系列的 API 以供 JS 来操作 CSS。

### CSSStyleSheet

我们可以通过`document.styleSheets`来查询本文档应用的 CSS。一般的，我们可以通过外部样式表、内部样式表、内联样式三种方式引入 CSS 规则，那么它们都能通过`document.styleSheets`查询到吗？我们用下面的片段来进行试验。

```html
<html>
  <body style="color: red;">
    text
  </body>
  <style>
    body {
      color: rgb(139, 195, 74);
    }
  </style>
</html>
```

![](/post-images/page-load-6.png)

可以发现，内联样式是不会写入`document.styleSheets`中的。而外部样式表和内部样式表都会被加入`document.styleSheets`中，并且有一个`ownerNode`属性表明它来自`head`还是`style`标签。根据 HTML 文档，内联 style 会[返回](https://html.spec.whatwg.org/multipage/dom.html#the-style-attribute)一个`CSSStyleDeclaration`对象，挂载在该节点的`style`属性中；而 link、style 标签则会[返回](https://html.spec.whatwg.org/multipage/semantics.html#the-style-element)一个`CSSStyleSheet`对象，挂载在`document.styleSheets`中；同时，还可通过该节点的`sheet`属性访问。

### CSSStyleDeclaration

这是我们最常打交道的对象，它可以从三个地方来访问：

1. HTMLElement.style。用来处理 HTML 元素的内联 CSS。
2. document.styleSheets[0].cssRules[0].style。用来处理外/内部样式表中的 CSS。
3. Window.getComputedStyle(HTMLElement)。该方法返回的只读的`CSSStyleDeclaration`对象。

更多的内容可以阅读 PDF 中的文件。

## TL;DR

- script 会阻止 DOM 的构建。
- 页面加载时如果存在 CSS 资源，则必须等 CSSOM 全部构建完成，才会执行 script，因为 JS 可能查询、修改 CSSOM。如果存在多个 CSS 资源，因为后面的规则可能覆盖前面的规则，所以必须等待全部 CSS 加载完成，才会进入下一个流程。
- DOM 和 CSSOM 结合在一起，才能生成 Render Tree。有了 Render Tree 之后，浏览器就可以进行绘制了。DOM 的构建会被 JS 阻止，而 JS 执行又需要查询 CSSOM。所以 script 和 css 都是 render block 的资源。所以我们应当尽早准备好 CSS 资源，尽量把 JS 资源放在文档的后面来解析。骨架图也是基于这个原理。我们可以参照“[优化关键渲染路径](https://developers.google.com/web/fundamentals/performance/critical-rendering-path/adding-interactivity-with-javascript?hl=zh-cn)”，了解和优化 HTML、CSS 和 JavaScript 之间的依赖关系谱。

## 参考

- [when does parsing html dom tree happen](https://stackoverflow.com/questions/34269416/when-does-parsing-html-dom-tree-happen)
- [when recalculate style event](https://stackoverflow.com/questions/34698433/what-is-involved-in-chromes-recalculate-style-event)
