---
title: "Ruby: collect, detect, inject, reject, select"
author: 孙耀珠
tags: 编程语言
---

在比较 Ruby 和 Python 的时候，很多人会说 Python 是一门简约的语言，而 Ruby 是一门魔幻的语言。之所以说 Ruby 魔幻，一方面是因为神奇的元编程和甜甜的语法糖，另一方面是在 Ruby 中总有不止一种方法去做一件事（[There's more than one way to do it](https://en.wikipedia.org/wiki/There%27s_more_than_one_way_to_do_it)），循环便是其中一例。

如果写过主流的结构化编程语言，那么一定对 for 循环非常熟悉吧。Pascal 继承了 ALGOL 风格的 for 循环，由初值、终值以及可选的步长组成；C 语言则创造了现在广为人知的三段式 for 循环，被 Java / JavaScript / PHP 等主流语言所沿用；而 Python / Swift 等语言使用 for-in 语句结合 range 也能实现相同的功能：

```
for i := 0 to n-1 do ……         ⍝ Pascal
for (int i = 0; i < n; ++i) ……  ⍝ C
for i in range(0, n): ……        ⍝ Python
for i in 0..<n { …… }           ⍝ Swift
```

而在 Ruby 的世界中，有一些有趣的方法可以取代 for 循环：

``` ruby
n.times { |i| …… }
0.upto(n-1) { |i| …… }
(0...n).each { |i| …… }
```

不过 Ruby 并没有激进地删掉 for 关键字，实现了 `each` 方法的对象都能以 for-in 语句进行迭代。像上述 `times` / `upto` / `each` 这样的方法在 Ruby 中被称为**迭代器**（iterator），其实现方式与后来 Python / JavaScript 等语言中的**生成器**（generator）相似，但使用方式更接近于函数式编程中广泛采用的**高阶函数**（higher-order function）。

<!--more-->

``` ruby
def fibonacci(n)
  x, y = 1, 1
  n.times do
    yield x
    x, y = y, x + y
  end
end
fibonacci(10) { |x| puts x }
```

在这段代码中，我们首先定义了一个斐波那契数列的迭代器，接着用它打印了数列的前十项。一个方法被称为迭代器的充要条件是其使用了 `yield` 关键字，这也意味着在调用该方法时需要紧跟着一个形如 `{ …… }` 或 `do …… end` 的**代码块**（block）。迭代器在执行到 `yield` 语句时会将控制权暂时移交给代码块，`yield` 的值将成为代码块的参数，代码块执行完之后迭代器将重获控制权并继续执行下去，如此反复便达到了迭代的效果。而这个代码块实际上是 Ruby 中一种特殊的**闭包**（closure），它只能跟在方法后面而不能单独存在，如果希望存储或是传递闭包，则需将其转换 proc 或 lambda（前者的行为类似于代码块、后者类似于方法）。

我们可以注意到，迭代器的做法与绝大多数结构化编程语言的区别是：以普通的方法调用取代了特殊的控制语句。那么我们有没有可能进一步用 OOP 的特性消灭掉所有控制流的语法呢？其实这早在 1970 年代初就被 Smalltalk 实现了：Smalltalk 没有 if / while / for 语句，一切控制流都被动态派发的**消息传递**所取代。首先以 if 为例：

``` smalltalk
result := a > b
    ifTrue: [ 'greater' ]
    ifFalse: [ 'less or equal' ].
```

Smalltalk 中的条件判断是向 `Boolean` 对象发送 `ifTrue:ifFalse:` 消息，而其子类 `True` 和 `False` 分别以相反的方式实现对该消息的响应：`True` 对象只执行 `ifTrue:` 的代码块，而 `False` 对象只执行 `ifFalse:` 的代码块。因为 Smalltalk 的消息是动态派发的，所以 `Boolean` 对象如何响应消息直到运行时才会决定，这样一来条件语句就被 Smalltalk 的多态特性完美地取代了。

而 Smalltalk 的条件循环则是向一个返回布尔值的代码块发送 `whileTrue:` 消息，利用上述的 `ifTrue:` 即可递归实现。下面是一个 while 的例子：

``` smalltalk
[ i > 0 ] whileTrue: [
    Transcript showCr: i asString.
    i := i - 1
].
```

而 for 系列任务则交给 `timesRepeat:` / `to:by:do:` / `do:` 等来完成，这些消息跟开头提到的 Ruby 方法极其相似，可以看到 Ruby 从 Smalltalk 中借鉴了相当多的想法：

``` smalltalk
n timesRepeat: [ …… ].
0 to: n-1 do: [ :i | …… ].
#(1 2 3) do: [ :i | …… ].
```

另外十分有趣的是，常用的 map / reduce / filter 系列函数，在 Smalltalk 的 `Iterable` 类中都起了 -ect 后缀的名字，简直是强迫症的福音。Ruby 也将其继承到了 `Enumerable` 模块中，但凡实现了 `each` 方法的类都可以将其混入（mixin）。下面则是 Ruby 中这些方法的 [Reference](http://ruby-doc.org/core/Enumerable.html)：

* 目录
{:toc}

## #collect (#map)

``` 
collect { |obj| block } → array
collect → an_enumerator
```

对每个元素执行一次 block，返回一个由所有 block 返回值构成的新数组。

``` ruby
(1..4).collect { |x| x**2 }
#=> [1, 4, 9, 16]
```

## #detect (#find)

``` 
detect(ifnone = nil) { |obj| block } → obj or nil
detect(ifnone = nil) → an_enumerator
```

找出第一个使 block 返回 true 的元素，若未找到则返回 nil。如果指定了参数，那么未找到时调用 `ifnone.call` 并返回其结果。

``` ruby
(1..10).detect { |x| x % 4 == 0 }
#=> 4
```

## #inject (#reduce)

``` 
inject(initial, sym) → obj
inject(sym) → obj
inject(initial) { |memo, obj| block } → obj
inject { |memo, obj| block } → obj
```

通过二元运算将所有元素规约为一个对象，二元运算可以通过 block 或是方法、运算符的 symbol 来指定。如果指定的是 block，则上一轮规约结果和当前元素会分别作为 block 参数传入，而返回值将是本轮的规约结果；如果指定的是 symbol，则每轮规约结果是 `memo.sym(obj)`。如果没有指定初值，则第一个元素会被作为初值使用。

``` ruby
(1..4).inject(0) { |sum, x| sum + x**2 }
#=> 30
(1..100).inject(:+)
#=> 5050
```

## #reject

``` 
reject { |obj| block } → array
reject → an_enumerator
```

筛选出所有使 block 返回 false 的元素。

``` ruby
(1..10).reject { |x| x % 4 == 0 }
#=> [1, 2, 3, 5, 6, 7, 9, 10]
```

## #select (#find_all)

``` 
select { |obj| block } → array
select → an_enumerator
```

筛选出所有使 block 返回 true 的元素。

``` ruby
(1..10).select { |x| x % 4 == 0 }
#=> [4, 8]
```
