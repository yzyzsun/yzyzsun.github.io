---
title: Hello, World!
category: Diary
author: 孙耀珠
---

这是博客的第一篇文章，首先当然是说声欢迎来访～

我曾经用国外网络服务器商的免费空间搭建过 WordPress 博客，并绑定了 .tk 的免费域名。不过这些免费空间大多不太靠谱，一旦账号不活跃就很可能被停用；在国内不仅访问速度很慢，还有被墙的风险。于是不久我就放弃了原来的博客。

前几天，我偶然接触到了 Markdown 这个轻量级标记语言，发现其与维基百科所使用的 [Wiki Markup](https://en.wikipedia.org/wiki/Help:Wiki_markup) 异曲同工（不考虑后者复杂的模板功能）——比 HTML 简单、比 Microsoft Word / Apple Pages 纯粹，但同时也能满足平时富文本编辑的需要。自然而然地，我萌生了用 Markdown 写博客的想法。对于一个程序员来说，网站搭建在自己的服务器上这种可控的感觉当然是最好的，不过像博客这样的静态网站托管在 GitHub Pages 也是不错的选择，还省去了自己去维护的工夫。而静态网站生成工具我选择了 Jekyll，首先是因为自己对于 Ruby 更加熟悉，其次 GitHub Pages 原生支持 Jekyll，因此可以直接托管源码，不用像 Hexo 一样先在本地生成所有静态文件再推送到 GitHub 上。

<!--more-->

Jekyll 的官方文档写得非常清楚，有时间的话完全可以从零开始自己设计，而我是直接 fork 了 [Jekyll Now](https://github.com/barryclark/jekyll-now) 项目~~（主要是因为懒）~~。网上的相关资料非常多，我觉得以下几个很有帮助：

- [GitHub Pages](https://pages.github.com)
- [Jekyll • Simple, blog-aware, static sites](https://jekyllrb.com)
- [Markdown 語法說明](http://markdown.tw)
- [GitHub Flavored Markdown Spec](https://github.github.com/gfm/)
- [使用 GitHub, Jekyll 打造自己的免费独立博客 - on_1y 的博客](http://blog.csdn.net/on_1y/article/details/19259435)

至此便已经可以尽情享受 Jekyll 博客的乐趣了～
