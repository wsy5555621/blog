---
title: 灰色
date: 2020-04-04 23:29:35
tags:
  - filter
  - css
  - grayscale
---

今年的清明是个悲伤的日子，灰蒙蒙的天，尚未散去的新冠肺炎。这一天，大家在 10 点一起默哀 3 分钟，汽车、火车、舰船鸣笛，防空警报鸣响。愿逝者安息，生者坚强。

<!--more-->

互联网有着自己的悼念方式，在今天，各大网站、App都变成了灰色，以表达悼念之情。如何快速的实现呢？

### filter（滤镜）
CSS3中的滤镜是实现这个效果的首选。它包含了诸多属性，例如：
- blur(px) 高斯模糊
- brightness(%) 亮度
- contrast(%) 对比度
- grayscale(%) 灰度

```css
.wrap {
  /* Element must "hasLayout"! */
  zoom: 1;
  filter: progid:DXImageTransform.Microsoft.BasicImage(grayscale=1); /* IE6-8 */
  filter: gray; /* IE6-IE9 */
  -webkit-filter: grayscale(100%);
  -moz-filter: grayscale(100%);
  -ms-filter: grayscale(100%);
  -o-filter: grayscale(100%);
  filter: grayscale(100%); /* Chrome 19+, Safari 6+, Safari 6+ iOS */

  /* Firefox 10+, Firefox on Android */
  filter: url("data:image/svg+xml;utf8,&lt;svg xmlns=\'http://www.w3.org/2000/svg\'&gt;&lt;filter id=\'grayscale\'&gt;&lt;feColorMatrix type=\'matrix\' values=\'0.3333 0.3333 0.3333 0 0 0.3333 0.3333 0.3333 0 0 0.3333 0.3333 0.3333 0 0 0 0 0 1 0\'/&gt;&lt;/filter&gt;&lt;/svg&gt;#grayscale");
}
```

但是IE10、IE11、Edge12这样是不生效的。简单搜索了一下，有一些解决方案：

* 通过meta声明使用ie9。`<meta content="IE=9" http-equiv="X-UA-Compatible">`
* 使用cavans，转换每个像素，适用于图片，且资源有同域限制。卡、慢、耗浏览器资源，不如直接贴图。
* 利用SVG滤镜。
* 引用grayscale.js。可以参考`/pdf/cross-browser-grayscale-ie11.zip`


[ie10-how-to-apply-grayscale-filter](https://stackoverflow.com/questions/14813142/internet-explorer-10-how-to-apply-grayscale-filter)
[彩色转灰度算法](https://github.com/aooy/blog/issues/4)
[RGB 转为灰度值的心理学公式 Gray = 0.30R + 0.59G + 0.11B 是怎么来的？](https://www.zhihu.com/question/22039410)
[IE 6~8使用其独有的滤镜](http://www.ruanyifeng.com/blog/2010/03/cross-browser_css3_features.html)