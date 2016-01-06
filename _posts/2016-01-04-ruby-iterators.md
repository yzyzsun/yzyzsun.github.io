---
layout: post
title: "Ruby: collect, detect, inject, reject, select"
category: Tech
---

在比较 Ruby 和 Python 的时候，很多人会说 Python 是一门简约的语言，而 Ruby 是一门魔幻的语言。之所以说 Ruby 魔幻，一方面是因为神奇的元编程和好吃的语法糖，另一方面是在 Ruby 中总有不止一种方法去做一件事（[There's more than one way to do it](https://en.wikipedia.org/wiki/There%27s_more_than_one_way_to_do_it)），循环便是其中一例。

如果你写过 C 语言，那么你一定很熟悉传统的 `for (int i = 0; i < n; ++i) ...`；或者在 Pascal 等语言里，它有更简洁的形式 `for i := 0 to n-1 do ...`；如果你还学过 Python，你可能会把它改写成 `for i in range(0, n): ...`。在 Ruby 中，虽然也有 `for i in 0...n` 的语法，但实际上大多数 Rubyist 都不会去用 for 这个关键字。譬如上面的例子，Ruby 通常是这样表达的：

``` ruby
n.times { |i| ... }
(0...n).each { |i| ... }
0.upto(n-1) { |i| ... }
```

当然相应地，Java / C++11 中的 `for (type x : array)` 或是其他语言中的 for-in 语句，在 Ruby 中只需要用 `array.each` 就能实现了。像 `times` / `each` / `upto` 这样的方法在 Ruby 中被称为**迭代器**（iterators），类似于通常所说的高阶函数（[higher-order functions](https://en.wikipedia.org/wiki/Higher-order_function)）。迭代器能够接受一个 block，并在适当的时候反复调用这个代码块；而这个 block 实际上是 Ruby 中一种特殊的闭包，它只能跟在方法后面而不能单独存在，如果希望存储或是传递闭包，需要使用 proc / lambda。

<!--more-->

Ruby 中的迭代器当然不止于此，十分有趣的是，常用的 map / reduce / filter 系列方法，在 Enumerable 模块中都起了 -ect 后缀的名字，简直是强迫症的福音。下面是它们的 Reference：

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

---

## Ruby - AKB48（篠田TeamA）

<embed width="720" height="430" quality="high" allowfullscreen="true" type="application/x-shockwave-flash" src="http://static.hdslb.com/miniloader.swf" flashvars="aid=3194009" pluginspage="https://get.adobe.com/flashplayer/">

