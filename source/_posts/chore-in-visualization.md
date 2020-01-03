---
title: 可视化开发
date: 2019-12-31 18:10:54
tags: 
- d3
- svg
- cavans
- threeJs
---
这篇文章本来不知道该什么时候起笔，怎么起笔，要涵盖什么，最后记成了流水账。干了三个月的可视化开发，就当总结一下，记录一下遇到的，防止遗忘吧。毕竟老一辈可视化开发留下的宗旨是，*上动图*。。。。
<!--more-->

啥子是可视化啊？数据可视化研究的是，如何将数据转化成为交互的图形或图像等，以视觉可以感受的方式表达，增强人的认知能力，达到发现、解释、分析、探索、决策和学习的目的。
三个月后回过头来，发现这都是扯淡。对于前端来说，就是把UED的设计稿，好看的、炫酷的，翻译成页面。这其中蕴含的蛮多的数学知识，可以让人觉得，初高中大学并不是白学数学的～～奈何要用时都已经忘光了。
## 可视化开发

* 贝塞尔曲线。https://juejin.im/post/5b854e1451882542fe28a53d
* 各种坐标系
## svg相关
### svg各类标签
svg有多种功能的标签，比如动画标签、容器标签、渐变标签等等。
(svg标签)(https://developer.mozilla.org/en-US/docs/Web/SVG/Element#Container_elements)
* <g>标签可以用来分组，也可以用来被<use>引用是最常使用的标签之一。对它做的变换都会被应用到它到子元素中。但是，用<g cy="xxxx">来对<circle>设置属性是无效对。
* <use>标签对引用方式: <use href="#myCircle" x="10" fill="blue"/>。渐变的引用方式fill: (url(#id))。注意区分。

### <path>
* ie下  stroke-dashoffset: 0%的动画不生效 edge生效
https://stackoverflow.com/questions/33812303/svg-animation-is-not-working-on-ie11

### svg<image>标签的兼容性
* svg中也有<image>标签，可以让我们引用图片。最直接的使用方式是:
``` html
<image href="...">
```
但是在IE下，这样引入图片无效，应改为
``` html
<image
  width="60"
  height="90"
  href="..."
  preserveAspectRatio="none meet"
/>
```
* `Edge`中，<image>，如果动态改变href的值，不会重新渲。在React中，使用了一个key来重载这个标签。
``` jsx
  <image
    key={trend}
    width="16"
    height="16"
    href={TREND_ICON[getIconTrend(trend)]}
    preserveAspectRatio="none meet"
    x={imagePos}
    y={9}
  />
```

* IE11不支持<image> css transform。必须使用`transform`属性。<image transform="translate(xx, yy)">
* 当<image>引入的是一个svg时，用animateMotion做动画时，会引发composite合成时消耗大量时间。改为png。


### svg<text>
对于文本来说主要是两个需求，水平居中和垂直居中。
* 水平居中 设置属性text-textAnchor="middle"。主流浏览器都支持。
* 垂直居中 设置属性dominant-baseline="middle"。在ie11、edge浏览器下都不支持。一般需要在chrome中设置该属性，得到大概的dy偏移量，转而设置dy。一般14px的文本，设置dy=".25em"可以达到垂直居中的效果。[Centering SVG text on IE11](https://stackoverflow.com/questions/48831606/centering-svg-text-on-ie11-with-text-anchor-middle-and-alignment-baseline-midd)
* <tspan>就像html中的<span>标签一样，用来分割文本。
* IE11不支持tspan透明度`opacity`属性和css。涉及字体颜色透明度，应使用fill-opacity。（测试时发现一个现象，在IE11下设置内联style opacity，字体颜色无变化，但取消后再使用，字体颜色却正常了，等待进一步测试。）

### svg元素如何实现div中的title属性？
使用`<title>`标签来实现。注意，`<title>`必须作为第一个子元素才生效。
The <title> element provides an accessible, short-text description of any SVG container element or graphics element.

Text in a <title> element is not rendered as part of the graphic, but browsers usually display it as a tooltip. If an element can be described by visible text, it is recommended to reference that text with an aria-labelledby attribute rather than using the <title> element.

``` html
<text>
  <title>啦啦啦后面还有呢</title>
  啦啦啦。。。
</text>
```
[svg title element](https://developer.mozilla.org/en-US/docs/Web/SVG/Element/title)
[svg alt title properties](https://stackoverflow.com/questions/33317207/can-i-use-alt-and-title-properties-within-svg-elements)
[accessible-svgs](https://css-tricks.com/accessible-svgs/)

### svg滚动条
不能。参考d3.drag模拟实现。

### 改变线性渐变的方向
渐变是有方向的，有时候需要对称的渐变效果，可以使用`gradientTransform`属性来实现。
``` html
<defs>
  <linearGradient id="myCustomGradient">
    <stop>...</stop>
    <stop>...</stop>
  </linearGradient>
</defs>
<linearGradient id="myCustomRotatedGradient" xlink:href="#myCustomGradient" gradientTransform="rotate(180, 150, 25)"/>
```

### 有关clip-path
用来截取clip-path视图内的像素。一共有两种clip-path。
* css中的(clip-path)[https://css-tricks.com/clipping-masking-css/]属性。需要IE10以上、Edge12以上，且只支持url类型的。
* svg的(clip-path)[https://developer.mozilla.org/en-US/docs/Web/SVG/Element/clipPath]属性，IE10以上都支持。

``` jsx
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

## cavans相关


## 总结
