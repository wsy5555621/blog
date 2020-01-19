---
title: 可视化开发
date: 2019-12-31 18:10:54
tags:
  - d3
  - svg
  - cavans
  - threeJs
---

这篇文章本来不知道该什么时候起笔，怎么起笔，要涵盖什么，最后记成了流水账。干了三个月的可视化开发，就当总结一下，记录一下遇到的，防止遗忘吧。毕竟老一辈可视化开发留下的宗旨是，_上动图_。。。。

<!--more-->

## 概念

可视化中蕴含的都是数学知识。理解一些概念后，才能帮助我们更好的还原设计师的图稿。

- 贝塞尔曲线。实现各类曲线的生成，一般 n 阶贝塞尔曲线会有 n+1 个控制点。可以参考[A Primer on Bézier Curves](https://pomax.github.io/bezierinfo/)和[深入理解贝塞尔曲线](https://juejin.im/post/5b854e1451882542fe28a53d);
- 坐标系。svg 和 cavans 的坐标系都以页面的左上角为(0,0)坐标点，坐标以像素为单位，x 轴正方向是向右，y 轴正方向是向下。在svg中，通过`scale(1, -1)`来把坐标系与我们属性的坐标系联系起来。[一个例子](https://stackoverflow.com/questions/31200679/how-to-animate-svg-rect-to-grow)

## svg 相关

### svg 各类标签

svg 有各种功能的[标签](https://developer.mozilla.org/en-US/docs/Web/SVG/Element#Container_elements)，比如动画标签、容器标签、渐变标签等等。

- `<g>`标签可以用来分组，也可以用来被`<use>`引用，是最常使用的标签之一。对它做的变换都会被应用到它到子元素中。但是，用`<g cy="xxxx">`来对`<circle>`设置属性是无效对。
- `<use>`标签对引用方式和渐变的引用方式，注意区分。

```javascript
  <use href="#myCircle" x="10" fill="blue" />
  <circle fill="url(#id)" />
```

### stroke-dasharray和stroke-dashoffset
利用`stroke-dasharray`和`stroke-dashoffset`，可以实现一些路径动画。如结合`<circle>`，可以实现进度条的功能。
- `stroke-dasharray`。线段数组，单值时表示实现和虚线都是该值，双值分别表示实线和虚线的值。***特别的***，如果值采用`percentage`来表示，则其长度是以当前`viewport`为[base](https://www.w3.org/TR/SVG11/painting.html#StrokeProperties)的。
- `stroke-dashoffset`代表线段的偏移量，最大不超过路径的长度，负值不生效。如果数值为正，则顺时针偏移该值再展示，这显示到我们的试图中，就表现为路径往左边偏移了该值。即，数值为正，路线左偏；为负，路径右偏。
- 注意：IE下`stroke-dashoffset`是[不生效](https://stackoverflow.com/questions/33812303/svg-animation-is-not-working-on-ie11)的。Edge中是有效的。

### \<path\>
- [各种path的指令](https://developer.mozilla.org/zh-CN/docs/Web/SVG/Tutorial/Paths)。如画半圆，`d="M40.5 40.5 L80,40.5 A45,45 0 0 1 40.5,80"`；`d3.arc()`画圆环。
- 如何得出`<path>`的长度呢？svg元素有一个`getTotalLength`方法。
- `<marker>`可以在`<path>`上添加一个标记，例如箭头。
- 碰到的现象：`<g><defs><marker>...<marker/></defs></g>`在`<g>`标签定义`<marker>`再引用不生效。

那么，结合`<marker>`和`stroke`是不是可以实现箭头移动的动画呢？这当然是不行的。
> You can't animate a marker using a stroke-dash-array. The position of the "draw/no-draw" array is being slid along the path, but the path itself doesn't move, which means the marker doesn't move.

但是，我们可以画一个箭头，然后采用`<animateMotion>`，让这个箭头按`path`的轨迹移动。同一时刻，`<path>`执行相应的`stroke`动画。
``` html
<!-- https://stackoverflow.com/questions/54993441/add-arrow-to-svg-stroke-animation -->
<style>
.outline {
  stroke-dasharray: 3253;
  animation: dash 7s linear forwards;
  stroke-linejoin: round;
}

@keyframes dash {
  from {
    stroke-dashoffset: 3253;
  }
  to {
    stroke-dashoffset: 0;
  }
}
</style>

<html>
<svg
  id="svg2"
  version="1.1"
  xmlns="http://www.w3.org/2000/svg"
  xmlns:xlink="http://www.w3.org/1999/xlink"
  viewBox="0 0 1350 370"
  preserveAspectRatio="xMidYMid meet"
>
  <path
    class="outline"
    id="border"
    d="m1317.2 348c6.1-3.3 10.3-8.3 11.6-14 1.7-7.2 1.7-299.5 0.1-307.2-1.4-6.1-4.6-10.8-9.6-13.8l-3.5-2H677 38.1l-3.5 2c-4.6 2.8-8 7.5-9.4 13.2-1.7 6.8-1.7 300.5 0 307.8 1.3 5.8 5.6 10.8 11.8 14 3.9 2 6.7 2 640.2 2 629.4-0.1 636.3-0.1 640-2z"
    style="fill:none;stroke-width:20;stroke:#2e3464"
    stroke-linecap="round"
  ></path>

  <polyline
    transform="translate(7 -25) rotate(90)"
    points="0,0 25,43.3 50,0"
    fill="#4B55A3"
  >
    <animateMotion
      id="an"
      dur="7s"
      repeatCount="1"
      rotate="auto-reverse"
      begin="0s"
      fill="freeze"
      restart="whenNotActive"
    >
      <mpath xlink:href="#border"></mpath>
    </animateMotion>
    <set attributeName="fill-opacity" to="0" begin="an.end"></set>
  </polyline>
</svg>
</html>
```

### svg\<image\>标签的兼容性

- svg 中\<image\>标签，可以让我们引用图片。最使用方式和注意点:

```html
<image href="...">
  <!-- 但是在IE、firefox下，这样引入图片无效，应加上 -->
  <image width="60" height="90" href="..." preserveAspectRatio="none meet"
/></image>
```

- `Edge`中，\<image\>，如果动态改变 href 的值，不会重新渲。在 React 中，使用了一个 key 来重载这个标签。

```html
<image
  key="{trend}"
  width="16"
  height="16"
  href="{TREND_ICON[getIconTrend(trend)]}"
  preserveAspectRatio="none meet"
  x="{imagePos}"
  y="{9}"
/>
```

- IE11 不支持\<image\> css transform。必须使用`transform`属性。

```html
<image transform="translate(xx, yy)"></image>
```

- 当\<image\>引入的是一个 svg 时，用 animateMotion 做动画时，会引发 composite 合成时消耗大量时间。改为 png。

### svg\<text\>

对于文本来说主要是两个需求，水平居中和垂直居中。

- 水平居中 设置属性 text-textAnchor="middle"。主流浏览器都支持。
- 垂直居中 设置属性 dominant-baseline="middle"。在 ie11、edge 浏览器下都不支持。一般需要在 chrome 中设置该属性，得到大概的 dy 偏移量，转而设置 dy。一般 14px 的文本，设置 dy=".25em"可以达到垂直居中的效果。[Centering SVG text on IE11](https://stackoverflow.com/questions/48831606/centering-svg-text-on-ie11-with-text-anchor-middle-and-alignment-baseline-midd)
- \<tspan\>就像 html 中的\<span\>标签一样，用来分割文本。
- IE11 不支持 tspan 透明度`opacity`属性和 css。涉及字体颜色透明度，应使用 fill-opacity。（测试时发现一个现象，在 IE11 下设置内联 style opacity，字体颜色无变化，但取消后再使用，字体颜色却正常了，等待进一步测试。）

### svg 元素如何实现 div 中的 title 属性？

使用`<title>`标签来实现。注意，`<title>`必须作为第一个子元素才生效。

> The `<title>` element provides an accessible, short-text description of any SVG container element or graphics element.
> Text in a `<title>` element is not rendered as part of the graphic, but browsers usually display it as a tooltip. If an element can be described by visible text, it is recommended to reference that text with an aria-labelledby attribute rather than using the `<title>` element.

```html
<text>
  <title>啦啦啦后面还有呢</title>
  啦啦啦。。。
</text>
```

[svg title element](https://developer.mozilla.org/en-US/docs/Web/SVG/Element/title)
[svg alt title properties](https://stackoverflow.com/questions/33317207/can-i-use-alt-and-title-properties-within-svg-elements)
[accessible-svgs](https://css-tricks.com/accessible-svgs/)

### svg 滚动条

不能。参考 d3.drag 模拟实现。

### 改变线性渐变的方向

渐变是有方向的，有时候需要对称的渐变效果，可以使用`gradientTransform`属性来实现。

```html
<defs>
  <linearGradient id="myCustomGradient">
    <stop>...</stop>
    <stop>...</stop>
  </linearGradient>
</defs>
<linearGradient
  id="myCustomRotatedGradient"
  xlink:href="#myCustomGradient"
  gradientTransform="rotate(180, 150, 25)"
/>
```

### 有关 clip-path

用来截取 clip-path 视图内的像素。一共有两种 clip-path 使用情况。

- [clip-path for html](https://css-tricks.com/clipping-masking-css/)属性。需要 IE10 以上、Edge12 以上，且只支持 url 类型的。
- [clip-path for svg element](https://developer.mozilla.org/en-US/docs/Web/SVG/Element/clipPath)属性，IE10 以上都支持。

```jsx
<defs>
  <clipPath id="bsp-clip">
    <circle strokeWidth="4.37" cx={0} cy={0} r="65" stroke="url(#clp-1)" />
  </clipPath>
</defs>

<rect
  x="0"
  y="0"
  width="130"
  height="46"
  clipPath="url(#clp-1)"
  fill="#46D2F4"
/>
// 可以把长方形的两个角角截去
```

### z-index
svg里面的标签元素可以使用[`z-index`](https://stackoverflow.com/questions/17786618/how-to-use-z-index-in-svg-elements)吗？目前不能，只能通过svg中节点的先后顺序来实现。

``` javascript
// 改变元素位置
const { parentNode } = $0;
const grandfaterNode = parentNode.parentNode;
return grandfaterNode.appendChild(parentNode);
```

### animateMotion
是svg动画的一种，可以用来做沿着`<path>`的动画。
- IE11不支持。
- animateMotion的path不能为`ellipse`等元素，`mpath`也不能指向`ellipse`,即`url(#一个ellipse)`是无效的，[需要吧ellipse转为path](http://complexdan.com/svg-circleellipse-to-path-converter/)。


## cavans 相关

日后在写。。。