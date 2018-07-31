---
title: Google Web Fonts 无法加载的解决方法
category: Dev
author: 孙耀珠
---

如果没有翻墙，大家一定会发现 [Ruby 官方网站](https://www.ruby-lang.org/) 加载得非常慢，在 Chrome 中停止加载的话仍能正常阅读，但在 Safari 中停止加载只能得到一片空白。

为了探明真相，我们可以调出浏览器的开发者工具，从时间线中可以看出究竟是什么拖慢了加载的进度：

![屏幕快照](/images/ruby-official-website.png)

<!--more-->

这下原因就显而易见了，因为 Google 被墙导致 **Google Web Fonts** 无法加载，即截屏中无响应的 `https://fonts.googleapis.com/css`。实际上其他使用 Google API 的网站还有很多，包括但不限于 [Ruby on Rails](http://rubyonrails.org/)、[Jekyll](http://jekyllrb.com/)、[Hexo](http://hexo.io/)、[Stack Overflow](https://stackoverflow.com/) 以及 [WordPress 博客](https://wordpress.org/plugins/disable-google-fonts/)。

最直接的解决方案当然是翻墙，不过针对这个问题也有另外的方法：如果使用的浏览器是 Chrome 或 Firefox，可以安装 [gooreplacer](http://liujiacai.net/gooreplacer/) 插件将 Google 的资源替换为中科大的镜像；Safari 则可以使用 [Redirector](http://lanceli.github.io/redirector/) 扩展以自定义重定向规则，譬如

```
fonts.googleapis.com,fonts.proxy.ustclug.org
ajax.googleapis.com,ajax.proxy.ustclug.org
themes.googleusercontent.com,google-themes.proxy.ustclug.org
fonts.gstatic.com,fonts-gstatic.proxy.ustclug.org
```

