---
title: winodw.open，crash all
date: 2019-12-16 10:35:36
tags: 
- window.open
- chrome links & process
- target="_blank"
---
嘻嘻，标题夸张了点。这篇文章主要是记录一下，使用window.open来模拟`a`标签`target="_blank"`时，碰到的一个父、子tab页面同时卡死的问题。
<!--more-->

## 前因后果
### 需求
这里我们的需求是实现一个符合用户习惯的`a`标签，直接点击的时候，本窗口转跳；按下`ctrl`/`command`再点击的时候，打开一个新的tab页。普通的`a`标签可以实现上述需求，但由于项目使用了`react-router`（或者是单页框架），如果直接使用`a`标签+路径进行转跳，每次都重载资源，显然效果不佳。所以在一开始，我们直接使用`点击`事件和`window.open`来处理这一切。

``` javascript
onClick={e => {
  e.preventDefault();
  const shouldOpenNew = e.metaKey || e.ctrlKey;
  if (shouldOpenNew) {
    // 即使加了noopener也无法解决性能问题，只能解决安全问题
    window.open(`/web/3?xxx=${record.xxx}`, '_blank', 'noopener');
    return;
  }
  router.push({
    pathname: '/web/3',
    query: { xxx: record.xxx },
  });
}
```
### 现象
似乎一切都很顺利，没想到实际测试的时候出了点意外。这段代码在直接点击的时候没有问题，但是在打开新tab页的时候，却发生了父、子tab页同时奔溃的问题。奔溃后，无提示、无
经过观察，carsh都是由子tab引发的，且在子tab页打开一段时间后发生的。对子tab页面进行排查时，console里在奔溃前无明显的未捕获错误；从network信息中发现存在5条并发请求一直处于`pending`状态，penging着penging着就奔溃了。向提供服务的后端同学确认，这一批请求出了点问题，无法返回数据，修复后，我们页面奔溃的问题也解决了。
那么，真的时由于这5条请求超时导致页面crash的吗？为什么子页面奔溃会导致父页面也奔溃呢？
经过一番查询，发现一些有用的信息：
* window.open打开的页面，将与打开它的页面共享一个线程（ps:本文是针对chrome中的现象，其他浏览器未测试是否有相似现象）。在子窗口可以耗尽资源，引发父窗口奔溃。
* chrome不同的标签页面使用不同进程和线程，但是有个例外，通过a标签的target="_blank"属性，或者window.open(url)在新窗口中打开页面, 会与父窗口共用进程和线程。为什么呢？还是因为opener。因为opener里有DOM信息。两个进程中同时hold住了DOM信息，在多进程下很难道控制，所以干脆就放在一个进程里了。（[参考](https://imweb.io/topic/584cd0459be501ba17b10aaa)）
* opener全局对象引起的安全问题。使用a标签的target="_blank"属性，或者window.open(url)在新窗口中打开页面时，子页面的opener是父页面*窗口对象*。虽然有同源限制，但可以通过window.opener.location = newURL来重写父页面的url，即使与父窗口的页面不同域。

怀疑是因为后端超时、而前端未处理空值引起的。

### 解决
虽然Crash的原因是由后端服务超时+前端异常未处理引起的，但是一番折腾发现winodw.open有这么多问题需要解决。
因此，这里采用了一个折中方案：
1. 采用`a`标签。解决window.open打开引起的共用线程性能问题。
2. 普通点击时，阻止默认事件，采用js跳转路由。
3. 按下`ctrl`/`command`再点击时，我们不阻止默认事件，也不使用额外的js，让浏览器的默认行为打开这个链接。
4. 同时，我们设置rel为*noopener noreferrer*。每次都打开新的线程，解决性能问题和安全问题。

``` javascript
export const renderJp2Details = (d, record) => {
  const { xxx } = record;
  const props = {
    style: { cursor: 'pointer' },
    href: `/web/3?xxx=${xxx}`,
    rel: 'noopener noreferrer',
  };

  return (
    <a>
      {...props}
      title={d}
      onClick={e => {
        const shouldOpenNew = e.metaKey || e.ctrlKey;
        if (shouldOpenNew) {
          return false; // 让a标签的默认行为来打开
        }
        // 使用js来转跳
        e.preventDefault();
        router.push({
          pathname: '/web/3',
          query: { xxx },
        });
        return false;
      }}
    >
      {d}
    </a>
  );
};
```

## 知识点
### window.open、target="_blank"打开的tab，与父页面公用线程
* 安全问题，如上所诉。
* 性能问题，如上所诉，共用线程，可能耗尽资源造成奔溃。
* 使用时，如果是a标签要在新窗口中打开，添加noopener属性；如果是js中打开新窗口，手动将新窗口的opener置为null；

### rel的noopener与noreferrer
* *noopener* If this feature is set, the newly-opened window will open as normal, except that it will not have access back to the originating window (via Window.opener — it returns null). In addition, the window.open() call will also return null, so the originating window will not have access to the new one either.  This is useful for preventing untrusted sites opened via window.open() from tampering with the originating window, and vice versa.
* *noreferrer* If this feature is set, the request to load the content located at the specified URL will be loaded with the request's referrer set to noreferrer; this prevents the request from sending the URL of the page that initiated the request to the server where the request is sent. In addition, setting this feature also automatically sets noopener.
* [参考](https://developer.mozilla.org/en-US/docs/Web/API/Window/open)

### 如何知道点击时ctrl或command是否按下
* MouseEvent.ctrlKey
* MouseEvent.metaKey
* MouseEvent.altKey
* MouseEvent.shiftKey

### network的线程和tab的线程是同一个吗
* network线程是独立的线程吧。看pdf目录下的有关内容。

## 参考
[when-using-window-open-if-the-new-window-freeze-so-too-does-the-parent](https://stackoverflow.com/questions/34957480/when-using-window-open-if-the-new-window-freeze-so-too-does-the-parent)
[how-to-detect-command-shift-click-for-os-x-in-javascript](https://stackoverflow.com/questions/10910554/how-to-detect-command-shift-click-for-os-x-in-javascript)
[when-should-i-use-rel-noreferrer](https://stackoverflow.com/questions/50773152/when-should-i-use-rel-noreferrer)