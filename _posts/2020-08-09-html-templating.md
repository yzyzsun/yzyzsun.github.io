---
title: HTML 模板语言纵览
author: 孙耀珠
tags: 领域专用语言
render_with_liquid: false
---

前端开发的本质，是把结构化的数据映射到 HTML。HTML 本身是静态的，因此模板引擎应运而生，接下了动态生成 HTML 的任务，直到近年来在前后端分离的浪潮下被面面俱到的前端框架所兼并。本文试图梳理出模板语言的主流范式，不过注意本文并非按照时间线编排，如果要还原历史的话，应该是 PHP (1995) → Zope 2 (1998) → JSTL (2002) → Django (2005) → Haml (2006) → Mustache (2009) → AngularJS (2010)。

<!--more-->

- 目录
{:toc}

## PHP 风格

虽然 PHP 早已是一门通用编程语言了，不过它最早是作为 HTML 模版引擎而出现的。从它现在的全称「超文本预处理器」也可以想象出，PHP 代码可以嵌入到 HTML 中，在用户请求该网页时，后端预先执行 PHP 代码并生成插入了运行结果的 HTML。下面便是一个最简单的例子：

```php
<html>
  <head>
    <title>Personal Home Page</title>
  </head>
  <body>
    <?php echo "Hello {$world}"; ?>
    <?= "Hello {$world}" ?>
  </body>
</html>
```

包裹在 `<?php … ?>` 标签里面的便是服务器要执行的 PHP 代码，直接 `echo` 一个表达式可以简写为 `<?= … ?>`。像这种在 HTML 里用特殊标签插入后端脚本的做法，从互联网诞生开始就相当普遍，至今仍屡见不鲜，本文称其为「PHP 风格」。这些 PHP 风格的模板引擎大同小异，区别主要在于内嵌语言用什么、代码块用什么标签包裹。

几乎每一门后端编程语言都有自己的 PHP 风格的模板引擎，因为在 HTML 直接嵌入后端脚本对于后端开发者来说最容易上手，没有任何学习上的负担。这些模板引擎中较为知名的有：

- 微软公司推出的 **ASP**，默认脚本语言为 VBScript；
- 昇阳公司推出的 **JSP**，相当于 ASP 的 Java 版本；
- Ruby on Rails 框架默认使用的 **eRuby**；
- JavaScript 也有类似的 **EJS**，博客框架 Hexo 在用。

JSP / eRuby / EJS 都继承了 ASP 的习惯，使用 `<% … %>` 来包裹脚本，还有 `<%= … %>` 渲染表达式结果等其他便利的标签。

## Mustache

PHP 风格的模板引擎虽然历史悠久，但在设计上有一个比较明显的问题：代码逻辑和 HTML 模板混杂在一起。因此，GitHub 的联合创始人 Chris Wanstrath 发明了广为人知的 Mustache。Mustache 的语法非常简洁，没有任何显式的控制流语句，完全由数据驱动，因而自称 logic-less。它不与任何编程语言耦合，几乎所有主流语言都有 Mustache 模板引擎的实现。

Mustache 有两种基本的标签形式：一种是像 `{{variable}}` 这样渲染变量的值，另一种则是像 `{{#section}} … {{/section}}` 这样的区块。根据键值的不同，区块隐含四种语义：

- 如果是假值或者空列表，就完全不渲染；
- 如果既不是假值也不是列表，就会渲染一次；
- 如果是非空列表，就渲染列表长度次；
- 如果是函数，则会以区块包裹的原始文本调用该函数。

另外还有 `{{^inverted}} … {{/inverted}}` 与正常的区块相反，如果是假值或空列表则渲染一次，否则不渲染。下面是 Mustache 模板的一个典型用例：

```html
<h1>{{header}}</h1>
{{#items}}
  {{#first}}
    <li><strong>{{name}}</strong></li>
  {{/first}}
  {{#link}}
    <li><a href="{{url}}">{{name}}</a></li>
  {{/link}}
{{/items}}
{{^items}}
  <p>The list is empty.</p>
{{/items}}
```

假设我们的输入数据是用下面这个 JSON 表示的：

```json
{
  "header": "Colors",
  "items": [
    {"name": "Red", "first": true, "url": "#red"},
    {"name": "Green", "link": true, "url": "#green"},
    {"name": "Blue", "link": true, "url": "#blue"}
  ]
}
```

那么 Mustache 便会渲染出如下 HTML：

```html
<h1>Colors</h1>
<li><strong>Red</strong></li>
<li><a href="#green">Green</a></li>
<li><a href="#blue">Blue</a></li>
```

虽然 Mustache 的设计小而美，但实际使用起来难免捉襟见肘。**Handlebars** 是对 Mustache 语言的扩展，最大的区别在于它引入了辅助函数。值得一提的是，其内置的 `#if` `#unless` `#each` `#with` 辅助函数明确了 Mustache 区块的隐式语义，譬如前面例子中的：

- `{{#items}} … {{/items}}` 可以显式写成 `{{#each items}} … {{/each}}`；
- `{{#first}} … {{/first}}` 可以显式写成 `{{#if first}} … {{/if}}`；
- `{{^items}} … {{/items}}` 可以显式写成 `{{#unless items}} … {{/unless}}`。

## Django 风格

Mustache 完全去除了代码逻辑，而 Handlebars 又稍稍加回了一些；不过更多的模板引擎出于实用性考量，不吝于引入更多逻辑，但也不愿复杂到直接内嵌后端脚本，换句话说就是试图在 Mustache 和 PHP 风格之间寻找平衡。如果要给这些中庸的模板引擎选个代表，最早为人所知的应该是 Django Template Language（以下简称 DTL），实际上它的出现要早于 Mustache。

与先前的话术稍有不同，DTL 将渲染表达式的 `{{ variable }}` 称为变量，将控制流程的 `{% tag %}` 称为标签，其内置了二十多个标签，包括常用的 `for` `if` `elif` `else` 等等。DTL 最大的特色是过滤器，譬如 `{{ list | length }}` 能够获取列表的长度、`{{ text | escape | linebreaks }}` 能先将文本转义再把换行符替换成 HTML 标签等等，大约有六十个过滤器内置其中。下面是一段 Django 模板语言的简单示例：

```django
<h1>{% block title %}{% endblock %}</h1>
<ul>
{% for user in users %}
  <li><a href="{{ user.url }}">{{ user.username }}</a></li>
{% endfor %}
</ul>
```

后来，Flask 的作者 Armin Ronacher 参考 DTL 的设计实现了独立于后端框架的 **Jinja** 模板引擎；而 Mozilla 提供了一个 JavaScript 上的实现 **Nunjucks**；Shopify 在 Ruby 上也有十分相似的 **Liquid** 模板引擎，并被用于 GitHub Pages 默认的静态站点生成器 Jekyll。

## 模板属性语言

上述三种风格，其实都可以归类于往 HTML 里面插各种 HTML 语法以外的 `<% … %>` `{{ … }}`，那么还有没有别的方式嵌入动态内容呢？有一种有趣的设计叫做「模板属性语言」(TAL)，也就是说我们把动态内容写在正常 HTML 标签的自定义属性里。TAL 最大的好处是简化了开发者和设计师的协作，因为 TAL 能直接加在设计原型上，加上之后仍然是照常显示的 HTML，不经后端渲染直接用浏览器打开也不会感知到动态代码的存在。

最早提出 TAL 的是 Python 编写的 Zope 2，其模板引擎 **Zope Page Templates** 使用了一系列 `tal:` 属性来引入动态内容。如今较为纯粹的例子是 Java 上的模板引擎 **Thymeleaf**，自称「自然模板」。下面是自然模板的一个示例，其中 `th:text` 会替换掉标签内的原有内容、`th:each` 会进行迭代：

```html
<table>
  <thead>
    <tr>
      <th th:text="#{msgs.headers.name}">Name</th>
      <th th:text="#{msgs.headers.price}">Price</th>
    </tr>
  </thead>
  <tbody>
    <tr th:each="prod: ${allProducts}">
      <td th:text="${prod.name}">Oranges</td>
      <td th:text="${#numbers.formatDecimal(prod.price, 1, 2)}">0.99</td>
    </tr>
  </tbody>
</table>
```

## 标签库

既然能自定义 HTML 属性，那么可不可以自定义 HTML 标签呢？JSP 标准标签库（JSR-52: JSTL）便实践了这一想法，虽然自定义标签不再有自然模板的好处，但写起来会更方便不少。JSTL 定义了 `<c:if test="${age >= 20}">` `<fmt:message key="i18n">` `<sql:query … >` `<x:parse … >` 等四类标签，在属性上还可以使用表达式语言（JSR-341: EL）来插入动态内容，就像前述 `<c:if>` 中的 `${age >= 20}` 那样。当然，JSP 也允许用户定义 JSTL 以外的自定义标签，这不禁让我们联想起了如今的 **Web Components**：

```javascript
class PopUpInfo extends HTMLElement {
  constructor() {
    super();
    …… // write element functionality in here
  }
}
customElements.define('popup-info', PopUpInfo);
```

以上代码便可以创建一个自定义标签 `<popup-info>`，而该元素的行为和语义均可由用户自行决定。 React / Angular / Vue 等前端框架非常提倡这种可复用的组件，不过它们提供了更高层的抽象，让自定义组件更易写易用。

## Haml

前面的模板语言说到底都还是 HTML 的超集，而 Haml 则完全抛弃了 HTML 原有的语法，走向了截然不同的方向。Haml 全称 HTML 抽象标记语言，由 Sass 之父 Hampton Catlin 发明。Haml 的写法有点像 CSS selector，譬如 `%p.sample#welcome Hello, World!` 会被渲染为 `<p class="sample" id="welcome">Hello, World!</p>`。Haml 有 `=` 和 `-` 前缀分别用来渲染表达式结果和控制流程，另外它跟 Python / Haskell 一样采用了越位规则，也就是说以缩进来界定文档结构。

不过 Haml 需要在每个标签前面写 `%` 还是有点麻烦的，JavaScript 上的 **Pug**（原名 Jade）对其进行了一些语法上的改进，后来又出口转内销，**Slim** 把相似的语法带回了 Ruby：

```slim
doctype html
html
  head
    title Slim Examples
    link rel="icon" type="image/png" href=file_path("favicon.png")
  body
    #content
      p This example shows you what a basic Slim file looks like.
      - if items.any?
        table#items
          - items.each do |item|
            tr
              td.name = item.name
              td.price = item.price
      - else
        p No items found. Please add some inventory.
          Thank you!
    div id="footer"
      = render "footer"
      | Copyright &copy; #{year} #{author}
```

说句题外话，我在用 Spring 写网站的时候曾经一度很困惑，过去不支持 Java 注解的时候，大家是如何忍受手写 XML 配置文件的呢？然而我开始写 Thymeleaf 模板的时候突然意识到，我自己对于手写 HTML 不也习以为常了吗？XML 配置文件正逐渐被 YAML / TOML 等新兴格式所取代，HTML 模板的未来又会如何呢？

## 结语

虽然上述六种分类特意将 HTML 模板语言的范式孤立开来，但如今流行的前端框架往往集成了多种范式，譬如 Angular 和 Vue 都支持 Django 风格的插值和管道 `{{ interpolation | pipe }}`，而写在 HTML 属性上的指令 `<p *ngIf="true">` `<p v-if="true">` 则类似于模板属性语言，众所周知它们也都支持自定义标签的组件化开发。这里我有意忽略了 React 的 JSX：在 JSX 中 JS 反客为主，HTML 组件变成了 JS 代码的一部分，恕我不算它是 HTML 模板语言了。

总而言之，本文力求归纳了主流的 HTML 模板范式，但 Web 开发毕竟不是我的主业，行文难免有所疏漏，但愿不会贻笑大方。
