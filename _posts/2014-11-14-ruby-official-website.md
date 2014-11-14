---
layout: post
title: Ruby 官方网站加载过慢的解决方法
category: Tech

---

一直觉得 [Ruby 官方网站](https://www.ruby-lang.org/) 加载得非常慢，在 Chrome 中停止加载的话仍能正常阅读，但在 Safari 中停止加载只能得到一片空白。

为了探明真相，我想到了浏览器的开发者工具中有录制时间线功能，从时间线中可以看出究竟是什么拖慢了加载的进度。

![屏幕快照](/images/ruby-official-website-00.png)

<!--more-->

这下原因就显而易见了，因为 Google 被墙导致 **Google Web Fonts** 无法加载，即截屏中无响应的 `https://fonts.googleapis.com/css`。实际上其他使用 Google API 的网站还有很多，包括但不限于 [Ruby on Rails](http://rubyonrails.org/)、[Jekyll](http://jekyllrb.com/)、[Hexo](http://hexo.io/)、[Stack Overflow](https://stackoverflow.com/) 以及 [WordPress 博客](https://wordpress.org/plugins/disable-google-fonts/)。

如果使用的浏览器是 Chrome 或 Firefox，可以安装 [gooreplacer](http://liujiacai.net/gooreplacer/) 插件将 Google 的资源替换为中科大的镜像。在 Safari 上目前没有类似的扩展，不过可以用 AdBlock Plus 屏蔽 Google API 来解决问题。向 ABP 添加自定义过滤规则：

* `http*://*googleapis.com/*`
* `http*://fonts.gstatic.com/*`
* `http*://themes.googleusercontent.com/*`
* `http*://*google-analytics.com/*`
