---
layout: post
title: 使用 pjax 技术为博客提速
tags: ['博客']
---

相信很多人都听说过 `ajax` 这个词，那你听过还有 `pjax` 吗？使用这个技术可以让我们的站点访问速度看起来飞快，我的博客也在使用，下面我将给你介绍它是什么？如何实现的、以及如何在你的站点里使用。

<!-- more -->

## 什么是 pjax？

![pjax]({{ "/public/images/2018/10/pjax.png" | prepend: site.cdnurl }} "pjax")

ajax 可以实现网站的局部刷新，但是 ajax 不会修改网站的 URL，它主要是为了异步的获取某个资源（可能是一段 JSON 或者一个页面）。很多人可能都听过目前流行的 `SPA`（Single Page Application）开发，它可以实现加载单个 HTML 页面并在用户与应用交互时动态更新该页面的内容，页面不会刷新；切换页面时 URL 会发生改变，这样比起刷新整个页面的体验要好的多，一般使用 MVVM 框架来实现这种单页应用，比如流行的 Vue.js。

而我们今天介绍的是 pjax，pjax 的全称是 `pushState + ajax`，它其实是 JQuery 的一个插件，你可以了解一下 [jquery-pjax](https://github.com/defunkt/jquery-pjax){:target="_blank"} 这个项目。[pushState](https://developer.mozilla.org/zh-CN/docs/Web/API/History_API){:target="_blank"} 是 HTML5 出现的 API，可以实现将要访问的请求压入一个历史堆栈中，pjax 本质上是对 ajax 的封装，通过 `pushState + ajax` 实现页面的缓存和本地存储，让我们看起来页面没有刷新，URL 却会发生改变，你可以点击我的博客链接看到。

## 实现原理

关于 pjax 的源码你可以在 [Github](https://github.com/defunkt/jquery-pjax/blob/master/jquery.pjax.js){:target="_blank"} 查看，下面我们通过原生 API 实现一个简易版的 pjax（_不懂编程的同学这段可以略过_）。

### 使用到的 API

- `history.pushState`：将当前URL和history.state加入到history中，并用新的state和URL替换当前
- `window.onpopstate`：浏览器历史前进后退事件

```js
function load(url) {
    $.get(url, function (data) {
        $('#content').html(data);
    });
};

$('a').on('click', function (e) {
    e.preventDefault();
    var url = $(this).attr("href");
    var title = $(this).text();

    history.pushState({
        url: url,
        title: title
    }, title, url);

    document.title = title;
    load(url);
});

$(window).on('popstate', function (e) {
    var state = e.originalEvent.state;
    if (state !== null) {
        document.title = state.title;
        load(state.url);
    } else {
        document.title = '不知道什么鬼';
        $('#content').empty();
    }
});
```

[在线演示](https://codepen.io/biezhi/project/editor/XrMgdd){:target="_blank"}

![pjax 简易版]({{ "/public/images/2018/10/simple_pjax_demo.png" | prepend: site.cdnurl }} "pjax 简易版")

用户点击链接会修改当前 URL，通过 `history.pushState` 压入历史浏览堆栈，按下后退按钮后会触发 `window.popstate`，弹出上一次访问替换掉内容即可。

## 尝试一下

如果你使用前面提到的 pjax 可能会需要服务端支持，因为它是局部更新某一块内容，服务端不返回整个 HTML 文档，而我今天让大家实现博客站点的无刷新化，就用到了 [instantclick](http://instantclick.io/){:target="_blank"} 这个利器。

InstantClick 又是什么呢？这是一个 JavaScript 库，只有 `2.7kb`。它也是使用了 `pushState + Ajax` 的方式帮助我们优化网站体验，它说尽管我们的网络带宽是足够的，但网站速度并不快，因为网页加载速度的瓶颈在于 [延迟](https://www.igvita.com/2012/07/19/latency-the-new-web-performance-bottleneck/){:target="_blank"}。

我们在访问一个链接的时候首先鼠标会悬浮在 URL 上，它会利用这几百毫秒的时间去加载链接的内容，当你点击的时候就不会感觉到延迟（前提是你的服务器真的不是龟速），当然如果你想减少服务器的压力也可以在点击的时候才加载。

它对浏览器的支持如下：

| IE | Firefox | Chrome | Safari | Opera | iOS Safari | Android Browser | Chrome for Android |
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
| 10+ | 4.0+ | 5+ | 5.0+ | 11.5+ | 5.0+ | 4.4+ | 18+ |

### 安装 InstantClick

安装它非常简单，只需要在你的页面 `body` 标签结束前加入它的 JS 代码。

```html
...
<script src="instantclick.min.js" data-no-instant></script>
<script data-no-instant>InstantClick.init();</script>
</body>
</html>
```

这样就可以了，你可以体验 `instantclick` 给你带来的飞速提升了，带有 `data-no-instant` 属性的标签不会被预加载，它称之为黑名单。

## 使用说明

**预加载配置**

InstantClick 有不同的预加载选项，根据你的服务器情况进行选择。默认的预加载策略（mouseover）是鼠标移动到超链接上就会加载，如果你觉得这样服务器压力过大可以修改为点击按钮。

```js
InstantClick.init('mousedown');
```

如果你只是想修改一下延迟事件可以使用

```js
InstantClick.init(80);
```

当访问者将鼠标悬停你的超链接后，InstantClick 将根据你设置的时间延迟预加载，单位是毫秒。建议延迟是 `50ms ~ 100ms`。超过 `100ms` 实际上可能比 `mousedown` 要慢，小于 `50ms` 和 `mouseover` 差不多，我的博客托管在 Github，我使用了 `mousedown` 方式加载。

**回调事件**

InstantClick 会触发四个事件，方便为页面的生命周期提供钩子函数：

- `change`：页面已加载，类似 `DOMContentLoaded` 事件
- `fetch`：页面开始预加载
- `receive`：页面已预先加载
- `wait`：用户点击了链接，但页面尚未预加载

你可以使用 `InstantClick.on('事件名')` 来监听它们。

**修改进度条**

使用 InstantClick 后页面加载时顶部会有个蓝色的进度条，如果你不喜欢进度条可以隐藏它，在你的样式表中加入以下代码。

```css
#instantclick {
    display: none;
}
```

如果你只是想换个颜色，可以这样修改

```css
#instantclick-bar {
    background: #23333;
}
```

## 常见问题

**我使用了 instantclick 发现 [不蒜子](http://busuanzi.ibruce.info/){:target="_blank"} 不生效了**

你可以在 `onchange` 事件后加载脚本，

```js
InstantClick.on('change', function (isInitialLoad) {
    $.getScript("//busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js");
});
```

> 不蒜子是一款只需要 2 行代码帮助你实现站点访问量统计的工具，[使用说明](http://ibruce.info/2015/04/04/busuanzi/){:target="_blank"}

**我站点里有些链接是外链，不用预加载**

站点里可能有友情链接或者要跳出的链接，这些是不需要预加载的，你可以使用黑名单功能，在标签上加入 `data-no-instant` 属性即可。

```html
<a href="https://github.com/biezhi" data-no-instant>Github</a>
```

希望看完这篇文章可以帮助你优化博客加载速度，有什么问题可以留言回复。

## 参考资料

- [Pjax 是什么以及为什么推荐大家用](http://www.cnblogs.com/shihao/archive/2013/04/18/3028969.html){:target="_blank"}
- [PJAX 原理和使用](https://www.fanhaobai.com/2017/07/pjax.html){:target="_blank"}
- [instantclick](http://instantclick.io/){:target="_blank"}
