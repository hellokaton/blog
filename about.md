---
layout: page
title: 关于
permalink: about.html
image: /public/images/redflag.jpg
order: 5
---

{% if site.author.photo %}
![{{ site.author.name }}]({{ site.author.photo | prepend: site.cdnurl }}){:.me}
{% endif %}

`biezhi` 是我一直在使用的英文 ID，因为之前购买了 `biezhi.me` 的域名就一直在使用。
博客只有在想起来 && 有时间的时候可能会写一篇。

目前还没什么其他可说的，如果想看我的代码可以去我的 [Github](https://github.com/biezhi){:target="_blank"}，想看我的视频教程可以去 [Youtube](https://www.youtube.com/channel/UCmlhPmTdqYhRWwWZWSIBwGw){:target="_blank"} 频道。

## 编程理念

仰慕「优雅编码的艺术」，追崇实践 + 理论得真知。

## 做过什么

- 2018：翻译 [Docker Curriculum](https://docker-curriculum.biezhi.me/) 指南
- 2018：发布 [examples](https://examples.codesofun.com/){:target="_blank"} 常用代码分享博客
- 2018：创建 20DaysOfCode 开发者训练计划
- 2018：创建 「代码真香」Youtube 频道
- 2018：开源 [profit](https://github.com/biezhi/profit){:target="_blank"}：在线打赏系统
- 2018：开源 [gitmoji](https://github.com/biezhi/gitmoji-plugin){:target="_blank"}：Git 提交表情插件
- 2018：开源 [excel-plus](https://github.com/biezhi/excel-plus){:target="_blank"}：Excel 操作库
- 2018：开源 [eve](https://github.com/biezhi/eve){:target="_blank"}：一个简单的命令行新闻客户端
- 2018：开源 [anima](https://github.com/biezhi/anima){:target="_blank"}：小而美的数据库操作库
- 2018：发布 [elves](https://github.com/biezhi/elves){:target="_blank"}：爬虫框架的设计和实现
- 2018：发布 [learn-java8](https://github.com/biezhi/learn-java8){:target="_blank"} Java 8 视频课程
- 2017：发布 [bye-2017](https://github.com/biezhi/bye-2017){:target="_blank"}：年终总结统计
- 2017：开源 [geekbb](https://github.com/biezhi/geekbb){:target="_blank"}：极简程序员论坛
- 2017：开源 [mrpc](https://github.com/kongzhongfinance/mrpc){:target="_blank"}：分布式服务治理框架
- 2017：开源 [tale](https://github.com/otale/tale){:target="_blank"}：美观方便的博客系统
- 2016：开源 [wechat-api](https://github.com/biezhi/wechat-api){:target="_blank"}：微信机器人 SDK
- 2015：开源 [blade](https://github.com/lets-blade/blade){:target="_blank"}：高性能简洁优雅的 MVC 框架

## 版权说明

我坚信着开放、自由和乐于分享是推动计算机技术发展的动力之一。所以本站所有内容均采用[署名 4.0 国际（CC BY
4.0）](http://creativecommons.org/licenses/by/4.0/deed.zh)创作共享协议。通俗地讲，只要在使用时署名，那么使用者可以对本站所有内容进行转载、节选、二次创作，并且允许商业性使用。

## 其他

我的博客使用 Jekyll 搭建，源码托管在 [Github](https://github.com/biezhi/blog){:target="_blank"}。如果你有什么自认为伟大的想法或者想对我说的请发送邮件至 `biezhi.me#gmail.com`，注意逻辑清晰，表明来意，否则不回复。

<!-- Add Disqus Comments -->
{% if site.disqus %}
{% include disqus.html %}
{% endif %}