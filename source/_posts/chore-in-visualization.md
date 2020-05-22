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

## 概念

可视化中蕴含的都是数学知识。理解一些概念后，才能帮助我们更好的还原设计师的图稿。

- 贝塞尔曲线。实现各类曲线的生成，一般 n 阶贝塞尔曲线会有 n+1 个控制点。可以参考[A Primer on Bézier Curves](https://pomax.github.io/bezierinfo/)和[深入理解贝塞尔曲线](https://juejin.im/post/5b854e1451882542fe28a53d);
- 坐标系。svg 和 cavans 的坐标系都以页面的左上角为(0,0)坐标点，坐标以像素为单位，x 轴正方向是向右，y 轴正方向是向下。在 svg 中，通过`scale(1, -1)`来把坐标系与我们属性的坐标系联系起来。[一个例子](https://stackoverflow.com/questions/31200679/how-to-animate-svg-rect-to-grow)

## svg 相关

### svg 各类标签

svg 有各种功能的[标签](https://developer.mozilla.org/en-US/docs/Web/SVG/Element#Container_elements)，比如动画标签、容器标签、渐变标签等等。

- `<g>`标签可以用来分组，也可以用来被`<use>`引用，是最常使用的标签之一。对它做的变换都会被应用到它到子元素中。但是，用`<g cy="xxxx">`来对`<circle>`设置属性是无效对。
- `<use>`标签对引用方式和渐变的引用方式，注意区分。
- `<foreignObject>`，svg 元素允许包含不同的 XML 命名空间。在浏览器的上下文中，很可能是 XHTML / HTML。可以用这个包裹 div。

```javascript
  <use href="#myCircle" x="10" fill="blue" />
  <circle fill="url(#id)" />
```

### stroke-dasharray 和 stroke-dashoffset

利用`stroke-dasharray`和`stroke-dashoffset`，可以实现一些路径动画。如结合`<circle>`，可以实现进度条的功能。

- `stroke-dasharray`。线段数组，单值时表示实现和虚线都是该值，双值分别表示实线和虚线的值。**_特别的_**，如果值采用`percentage`来表示，则其长度是以当前`viewport`为[base](https://www.w3.org/TR/SVG11/painting.html#StrokeProperties)的。
- `stroke-dashoffset`代表线段的偏移量，最大不超过路径的长度，负值不生效。如果数值为正，则顺时针偏移该值再展示，这显示到我们的试图中，就表现为路径往左边偏移了该值。即，数值为正，路线左偏；为负，路径右偏。
- 注意：IE 下`stroke-dashoffset`动画是[不生效](https://stackoverflow.com/questions/33812303/svg-animation-is-not-working-on-ie11)的。Edge 中是有效的。

### \<path\>

- [各种 path 的指令](https://developer.mozilla.org/zh-CN/docs/Web/SVG/Tutorial/Paths)。如画半圆，`d="M40.5 40.5 L80,40.5 A45,45 0 0 1 40.5,80"`。
- 如何得出`<path>`的长度呢？svg 元素有一个`getTotalLength`方法。
- `<marker>`可以在`<path>`上添加一个标记，例如箭头。
- 碰到的现象：`<g><defs><marker>...<marker/></defs></g>`在`<g>`标签定义`<marker>`再引用不生效。

那么，结合`<marker>`和`stroke`是不是可以实现箭头移动的动画呢？这当然是不行的。

> You can't animate a marker using a stroke-dash-array. The position of the "draw/no-draw" array is being slid along the path, but the path itself doesn't move, which means the marker doesn't move.

但是，我们可以画一个箭头，然后采用`<animateMotion>`，让这个箭头按`path`的轨迹移动。同一时刻，`<path>`执行相应的`stroke`动画。

```html
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

### 画一个圆环

- 可以用`<circle>`画一个圆，用`stroke-width`设置宽度，还可以用`stroke-dasharray`设置间隔。
- 用 d3.arc()来绘制`<path>`。

```html
<circle
  fill="transparent"
  cx="0"
  cy="0"
  r="425"
  stroke="#46d2f4"
  stroke-width="5"
  stroke-dasharray="200px, 4px"
></circle>

<path
  d="M-168.79570687645705,-379.1213649216794A415,415,0,0,1,-7.2427486714726514,-414.93679348990236L-7.0682246070998165,-404.9383165383385A405,405,0,0,0,-164.72834044569905,-369.9859103452534Z"
  fill="#fff171"
></path>
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
  x="imagePos"
  y="9"
/>
```

- IE11 不支持\<image\> css transform。必须使用`transform`属性。

```html
<image transform="translate(xx, yy)"></image>
```

- 当\<image\>引入的是一个 svg 时，用 animateMotion 做动画时，会引发 composite 合成时消耗大量时间。改为 png。

### svg preserveAspectRatio

`symbol`、`image`、`feImage`、`marker`、`pattern`、`view`标签都有`preserveAspectRatio`属性，它用来控制缩放比例。

- `svg`等标签中如果要应用该属性，应当设置好`viewport`
- 对于 `image` 元素, `preserveAspectRatio` 指示引用的图像应该如何与参考矩形进行匹配，以及是否应该相对于当前用户坐标系保留参考图像的长宽比

它有两个属性，`align`和`meetOrSlice`，以空格分隔，默认值为`xMidYMid meet`；

- `align`的值不为`none`时将强制统一缩放。简单来说就是把 viewbox 属性与视图属性宽高按照设定对齐。
- `align`为`none`时，不会进行强制统一缩放，会缩放指定元素的图形内容，使元素的边界完全匹配视图矩形。同时忽略`meetOrSlice`。
- `meetOrSlice`的可选值有`meet`和`slice`，效果分别相当于 CSS`background-size`的`contain`和`cover`，这在我们[自适应 svg](https://css-tricks.com/scale-svg/)时很有用。

更多细节可以参考[此处](https://www.zhangxinxu.com/wordpress/2014/08/svg-viewport-viewbox-preserveaspectratio/)。

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

svg 里面的标签元素可以使用[`z-index`](https://stackoverflow.com/questions/17786618/how-to-use-z-index-in-svg-elements)吗？目前不能，只能通过 svg 中节点的先后顺序来实现。

```javascript
// 改变元素位置
const { parentNode } = $0;
const grandfaterNode = parentNode.parentNode;
return grandfaterNode.appendChild(parentNode);
```

### animateMotion

是 svg 动画的一种，可以用来做沿着`<path>`的动画。

- IE11 不支持。
- animateMotion 的 path 不能为`ellipse`等元素，`mpath`也不能指向`ellipse`,即`url(#一个ellipse)`是无效的，[需要吧 ellipse 转为 path](http://complexdan.com/svg-circleellipse-to-path-converter/)。

### gradientUnits 问题

有时我们给一条直线添加线性渐变的`stroke`时，如果它是水平或垂直时，会发现这条直线无法显示。这是由于线性渐变`<linearGradient>`默认采用的是`gradientUnits=objectBoundingBox`。下面是一段测试代码，同时，列出了三种办法

```xml
<svg>
  <defs>
    <linearGradient id="grad" x1="0%" x2="100%" y1="0%" y2="0%">
      <stop class="" offset="0%" style="stop-color: red;"></stop>
      <stop class="" offset="33%" style="stop-color: yellow;"></stop>
      <stop class="" offset="66%" style="stop-color: pink;"></stop>
      <stop class="" offset="100%" style="stop-color: blue"></stop>
    </linearGradient>
    <linearGradient id="grad_2" x1="0%" x2="100%" y1="0%" y2="0%" gradientUnits="userSpaceOnUse">
      <stop class="" offset="0%" style="stop-color: red;"></stop>
      <stop class="" offset="33%" style="stop-color: yellow;"></stop>
      <stop class="" offset="66%" style="stop-color: pink;"></stop>
      <stop class="" offset="100%" style="stop-color: blue"></stop>
    </linearGradient>
  </defs>
  <-- Gradient not applied -->
  <path stroke="url(#grad)" d="M20,20L400,20" style="stroke-width: 10px;"></path>

  <-- Gradient applied since height of 1px -->
  <path stroke="url(#grad)" d="M20,40L400,41" style="stroke-width: 10px;"></path>

  <-- Gradient applied because of fake initial "move to" -->
  <path stroke="url(#grad)" d="M-1,-1,M20,60L400,60" style="stroke-width: 10px;"></path>

  <-- Gradient userSpaceOnUse applied -->
  <path stroke="url(#grad_2)" d="M20,80L400,80" style="stroke-width: 10px;"></path>
</svg>
```

![](/post-images/gradientUnits.jpg)
这是为什么呢？我们从[spec](https://www.w3.org/TR/SVG11/coords.html#ObjectBoundingBox)中可以找到原因：

> Keyword objectBoundingBox should not be used when the geometry of the applicable element has no width or no height, such as the case of a horizontal or vertical line, even when the line has actual thickness when viewed due to having a non-zero stroke width since stroke width is ignored for bounding box calculations. When the geometry of the applicable element has no width or height and objectBoundingBox is specified, then the given effect (e.g., a gradient or a filter) will be ignored.

### GeoJSON 相关

在可视化领域，地图方向的开发十分常见。在实践中，我们采用了`d3-geo`和`GeoJSON`的方式来进行。`GeoJSON`是一种对各种地理数据结构进行编码的格式，我们使用`d3-geo`可以很方便的来使用它。

- 如何居中？有时候直接使用 GeoJson 生成的地图十分小，这时候需要我们做一些放大操作。现在，`d3-geo`提供了一个方便的方法，即下面代码事例中。
- 从哪里下载格式？网络上有很多方法，有很多 git 库我 star 了，可以供参考。推荐使用阿里云的[`dataV`](http://datav.aliyun.com/tools/atlas/#&lat=29.87994609419456&lng=119.53399658203125&zoom=9)工具集。

```javascript
const getPath = () => {
  const projection = d3.geoMercator().fitExtent(
    [
      [originX, originY],
      [originX + width, originY + height],
    ],
    GeoJSON
  );

  return [projection, d3.geoPath().projection(projection)];
};

// 手动计算缩放的方法
// https://stackoverflow.com/questions/14492284/center-a-map-in-d3-given-a-geojson-object
var width = 300;
var height = 400;

var vis = d3
  .select("#vis")
  .append("svg")
  .attr("width", width)
  .attr("height", height);

d3.json("nld.json", function (json) {
  // create a first guess for the projection
  var center = d3.geo.centroid(json);
  var scale = 150;
  var offset = [width / 2, height / 2];
  var projection = d3.geo
    .mercator()
    .scale(scale)
    .center(center)
    .translate(offset);

  // create the path
  var path = d3.geo.path().projection(projection);

  // using the path determine the bounds of the current map and use
  // these to determine better values for the scale and translation
  var bounds = path.bounds(json);
  var hscale = (scale * width) / (bounds[1][0] - bounds[0][0]);
  var vscale = (scale * height) / (bounds[1][1] - bounds[0][1]);
  var scale = hscale < vscale ? hscale : vscale;
  var offset = [
    width - (bounds[0][0] + bounds[1][0]) / 2,
    height - (bounds[0][1] + bounds[1][1]) / 2,
  ];

  // new projection
  projection = d3.geo.mercator().center(center).scale(scale).translate(offset);
  path = path.projection(projection);

  // add a rectangle to see the bound of the svg
  vis
    .append("rect")
    .attr("width", width)
    .attr("height", height)
    .style("stroke", "black")
    .style("fill", "none");

  vis
    .selectAll("path")
    .data(json.features)
    .enter()
    .append("path")
    .attr("d", path)
    .style("fill", "red")
    .style("stroke-width", "1")
    .style("stroke", "black");
});
```

## 线性渐变

### svg 渐变

举例如上。

### Css 背景线性渐变

如何画利用 css 提供的渐变画一个网格呢？可以利用`linear-gradient`来完成：

```css
background-color: #269;
background-image: linear-gradient(white 2px, transparent 2px), linear-gradient(90deg, white
      2px, transparent 2px);
background-size: 100px 100px;
background-position: -2px -2px;

background-image: linear-gradient(
  #f6f0cf 12.5%,
  #f6f0cf 25%
); // 在宽度12.5%-25%有色条
```

更多参数，可以查看[MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/linear-gradient)和[入坑线性渐变 linear-gradient](https://www.jianshu.com/p/98d576e1bff6)。想看更多例子，可以查看[CSS 网格背景](https://leaverou.github.io/css3patterns/)。

### border 渐变

border 也可以定义渐变，不过需要 IE11 以上支持。这里有一些[例子](https://css-tricks.com/gradient-borders-in-css/)。

```css
.g {
  border-image: linear-gradient(to bottom, red, rgba(0, 0, 0, 0)) 1 100%;
}
```

## cavans 相关

在实践中，我仅仅是依赖 ThreeJs 库实现了一个简单的 3D 效果。但是在过程中，各种图形学概念扑面而来，因为不明了为什么，步履蹒跚。

### 三维世界里的三个矩阵

openGLES 中，物体会在五个空间中变换:
![](/post-images/coordinate_systems.png)

> 物体本身拥有一个坐标系，叫本地坐标系。把物体放到世界坐标系中，采用了`模型矩阵`，就是执行缩放、平移、旋转操作的过程。此时物体就具有了世界坐标系。再加入`视图矩阵`，包括视点坐标，观察点坐标和方向。现在只差最后一步–投影矩阵，物体就可以呈现出来了。目前显示设备都是二维平面的，所以需要`投影矩阵`来转换物体。投影矩阵通常分为平行投影和透视投影。

![](/post-images/coordinate_systems_1.jpg)
基于此，一个顶点转换到我们的屏幕上要经历过 MVP 这三个矩阵，在实际矩阵运算中，则为 PVM 的形式。

```c++
gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
gl_Position = projectionMatrix * viewMatrix * modelMatrix * vec4( position, 1.0 );
```

### 着色器（shader）

`shader`是一个用 GLSL 编写的小程序，在 GPU 上运行。又分为顶点着色器（vertexShader）和片元着色器（fragmentShader）。

- 顶点着色器对顶点实现了一种通用的可编程方法。
- 顶点着色器的输出数据是 varying 变量，在图元光栅化阶段，这些 varying 值为每个生成的片元进行计算，并将结果作为片元着色器的输入数据。从分配给每个顶点的原始 varying 值来为每个片元生成一个 varying 值的机制叫做插值。
- `顶点着色器`先运行；它接收 attributes，计算和操作每个顶点的位置，并传递额外的数据（varyings）给片段着色器。
- `片元着色器`后运行；它设置绘制到屏幕上的每个单独的片段（像素）颜色。

着色器中有三种类型的变量：

- Uniforms 是所有顶点都具有相同的值的变量。比如灯光，雾，和阴影贴图就是被储存在 uniforms 中的数据。uniforms 可以通过顶点着色器和片段着色器来访问。
- Attributes 与每个顶点关联的变量。例如，顶点位置，法线和顶点颜色都是存储在 attributes 中的数据。attributes`只可以在顶点着色器`访问。
- Varyings 是从顶点着色器传递到片段着色器的变量。对于每一个片段，每一个 varying 的值将是相邻顶点值的平滑插值。

在 ThreeJs 中，内置了许多属性，包含在[`WebGLProgram`](https://threejs.org/docs/index.html#api/zh/renderers/webgl/WebGLProgram)里。

### 噪声

水波、云彩，如果想要实现这样的效果，我们应该怎么做？其实，这些都是[`噪声`](https://thebookofshaders.com/11/?lan=ch)的一种形式。

### 阿里 webgl 的一些胶卷

![](/post-images/webgl-1.png)
![](/post-images/webgl-2.png)
![](/post-images/webgl-3.png)
![](/post-images/webgl-4.png)
![](/post-images/webgl-5.png)

### 参考

- [海量着色器例子](https://www.shadertoy.com/)
- [ThreeJS 着色器打造震撼海洋动画背景](https://zhuanlan.zhihu.com/p/54898150)
- [Three.js 几何体的那些事儿](https://srtian96.gitee.io/blog/2019/01/15/Three.js%E5%87%A0%E4%BD%95%E4%BD%93%E7%9A%84%E9%82%A3%E4%BA%9B%E4%BA%8B%E5%84%BF/)
- [GLSL 内建变量](https://blog.csdn.net/hgl868/article/details/7876246)
- [对 OpenGLES 中的空间变换的理解](https://blog.csdn.net/srk19960903/article/details/77970630)
- [WebGL 矩阵变换总结（模型矩阵，视图矩阵，投影矩阵）](https://blog.csdn.net/weixin_37683659/article/details/79830278)
- [学习 WebGL 之深入了解 Shader](https://www.jianshu.com/p/8d297451ac38)
- [WebGL 三维透视投影](https://webglfundamentals.org/webgl/lessons/zh_cn/webgl-3d-perspective.html)
- [OpenGL ES 基础篇](https://www.jianshu.com/p/438de5a40855)
