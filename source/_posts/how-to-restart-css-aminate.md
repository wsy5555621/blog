---
title: 重启CSS动画？
date: 2020-02-11 17:37
tags: 
- css
- animate
- reflow
---
使用了一个CSS动画，只执行一次。现在想要在特定的场景下，重新执行一遍此动画。怎么做？

<!--more-->

## 场景
现在，我们有这么一个简单的CSS，在初次渲染的时候，做了一个渐进动画。

``` html
<html>
  <h1 id="logo" class="run-animation">
    Fancy Text (click to restart)
  </h1>
</html>

<style>
@keyframes my-animation {        
  from {
    opacity : 0;
    left : -500px;
  }
  to {
    opacity : 1;
    left : 0;
  }      
}

.run-animation {
  position: relative;
  animation: my-animation 2s ease;
}

#logo {
  text-align: center;
  font-family : Palatino Linotype, Book Antiqua, Palatino, serif;
}
</style>
```

## 方式
1. 我们当然可以把这个元素删了，然后再把它增加回去。
2. 借助reflow的相关技巧来达到目的。例如，添加如下代码。
``` html
<script>
// This changes everything
"use strict";

// retrieve the element
var element = document.getElementById("logo");

// reset the transition by...
element.addEventListener("click", function(e){
  e.preventDefault;
  
  // -> removing the class
  element.classList.remove("run-animation");
  
  // -> triggering reflow /* The actual magic */
  // without this it wouldn't work. Try uncommenting the line and the transition won't be retriggered.
  // This was, from the original tutorial, will no work in strict mode. Thanks Felis Phasma! The next uncommented line is the fix.
  // element.offsetWidth = element.offsetWidth;
  
  void element.offsetWidth;
  
  // -> and re-adding the class
  element.classList.add("run-animation");
}, false);
</script>

```

实际操作中我们会发现，如果没有`void element.offsetWidth;`这段代码，这个方法是不生效的。可明明改变元素的class也是一种引发reflow的方法啊？我们借助Chrome，可以发现在删除class和增加class时，浏览器会设置一个重新计算样式的任务，然后继续执行代码；在遇上读取元素尺寸时，则直接执行了一次reflow。
![](/post-images/reflow-1.png)
class改变不是立即执行reflow的相关操作，这应该算是一个性能的优化，以防一个task中存在太多这样的操作，牺牲流畅度。当我们仅删除再增加class时，浏览器真正执行reflow的时候，发现什么都没有改变，甚至会直接跳过reflow这个步骤。
![](/post-images/reflow-2.png)

3. [restart-css-animation](https://css-tricks.com/restart-css-animation/)这篇文章中还有一个使用`webkitAnimationPlayState`的方法。



## 什么会引起回流
抄一段会引发reflow的操作：
* insert, remove or update an element in the DOM
* modify content on the page, e.g. the text in an input box
* move a DOM element
* animate a DOM element
* take measurements of an element such as offsetHeight or getComputedStyle
* change a CSS style
* change the className of an element
* add or remove a stylesheet
* resize the window
* scroll
* changing the font
* activation of CSS pseudo classes such as :hover (in IE the activation of the pseudo class of a sibling)
* setting a property of the style attribute

[Debugging CSS & Render Performance](https://www.youtube.com/watch?v=gqc88qWuiI4)
