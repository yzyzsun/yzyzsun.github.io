---
title: Swift 学习笔记（一）
author: 孙耀珠
tags: 编程语言
---

苹果自收购 NeXT 公司开始便使用 Objective-C 作为主力开发语言，至今已有近二十年了；而在这期间，各大科技公司都如火如荼地设计着更为现代的语言，譬如微软推出了 C# 和 F#、谷歌推出了 Go 和 Dart、Mozilla 推出了 Rust……虽然 Objective-C 随着 OS X 和 iOS 的迅速发展而越来越火，但相比之下它的语言设计已经落后于时代了，于是在这个大背景下 Swift 诞生了，开发者正是 LLVM / Clang 之父 Chris Lattner。

Swift 仍然是一门静态类型、面向对象的语言，不过它拥有很多现代的语言特性，譬如类型推断、代数数据类型、模式匹配、泛型等等，同时也有 Playgrounds 这样便利的交互式编程环境。Swift 非常强调安全性，不论是随处可见的可空类型、继承时复杂的构造规则，还是赋值没有返回值、控制流不能省略花括号，都是为了代码安全而考虑。另外 Swift 终于丢掉了 C 语言的包袱，语言层面上移除了指针，`switch` 语句不再需要 `break`，整型溢出会抛出运行时错误等等。

就目前来讲，用 Swift 进行 iOS 开发依然离不开 Objective-C 运行时，不过两者能够很方便地交互和共存，从前所有 Objective-C 撰写的 API 均可供 Swift 使用。Swift 的语法目前仍在不断改进，从 [Swift Evolution](https://apple.github.io/swift-evolution/) 可见一斑；目前这三篇学习笔记已从初版更新到了 Swift 5.0（2019-03-25），这是第一个 ABI 稳定的版本，也就是说从 Swift 5.0 开始二进制接口将向下兼容。[^abi]

[^abi]: [ABI Stability and More — Swift Blog](https://swift.org/blog/abi-stability-and-more/)

<!--more-->

* 目录
{:toc}

## 数据类型

### 整数

- 在 32 位平台上，`Int` / `UInt` 和 `Int32` / `UInt32` 长度相同。
- 在 64 位平台上，`Int` / `UInt` 和 `Int64` / `UInt64` 长度相同。
- 字面量前缀：二进制为 `0b`，八进制为 `0o`，十六进制为 `0x`。

### 浮点数

- `Float` 为 32 位浮点数；`Double` 为 64 位浮点数，浮点数字面量会被自动推断为 `Double`。
- 十进制 `1.25e2` 表示 1.25×10²；十六进制（类似 IEEE 754）`0xFp2` 表示 15×2²。
- 加减乘除运算严格检查左右操作数类型是否相同，不会进行隐式类型转换，因此 `Int` / `UInt` / `Double` / `Float` / `CGFloat` 等类型之间进行运算时需要显式类型转换。

### 元组

- 元组类型是任意各种类型的有序组合。可以通过点语法来访问元组中的单个元素，下标从零开始，如 `tuple.0`。也可以在定义时给元素命名，命名后便可通过名字来获取元素的值。

```swift
let status = (code: 200, message: "OK")
print(status.code, status.message)
```

### 可空类型

- 可空类型（Optional type）是一种特殊的泛型枚举：成员 `.none` 表示没有值，即 `nil`；成员 `.some(Wrapped)` 表示包含值，可以通过 `!` 来**强制解包**（forced unwrapping）获取值，或是通过 `?` 构成**可空链式调用**（optional chaining）。对 `nil` 进行强制解包会触发运行时错误，而可空链式调用则不会；当可空链式调用中有可空值为 `nil` 时整条链失败并返回 `nil`，若成功则返回一个相应的可空类型。

```swift
var s: String? // = nil
s!.count // Fatal error!
s?.count // : Int? = nil
s = ""
s!.count // : Int = 0
s?.count // : Int? = 0
```

- 在 `if` 和 `while` 语句中可以使用**可空绑定**（optional binding）判断可空类型是否包含值，若包含则为真并将值赋给局部常量或变量，可空绑定与其他条件判断可用逗号隔开。[^binding]

[^binding]: 最开始 `if-let` / `while-let` 的可空绑定和布尔表达式是用 `where` 关键字隔开的，Swift 3.0（[SE-0099](https://github.com/apple/swift-evolution/blob/master/proposals/0099-conditionclauses.md)）开始统一改用逗号分割，且不再限定它们的前后位置。

```swift
if let a = x, let b = y, a < b {
    // optional binding
}
if x != nil && y != nil && a < b {
    let a = x!, b = y!
    // equivalent form
}
```

- 如果确信一个可空类型的变量在后续使用中一定不为空，可以采用**隐式解包可空类型**（implicitly unwrapped optional type）[^iuo] 简化其操作：只要在声明时将类型后面的 `?` 改为 `!`，则之后的使用默认强制解包。

```swift
var s: String! = "Must not be nil!"
s.count // = s!.count
```

[^iuo]: 隐式解包可空值原本所属的 `ImplicitlyUnwrappedOptional<T>` 类型已于 Swift 4.2（[SE-0054](https://github.com/apple/swift-evolution/blob/master/proposals/0054-abolish-iuo.md)）被废除，取而代之的实现是为普通的可空类型加上 `@_autounwrapped`。


## 基本运算

### 赋值

- 赋值运算不返回任何值，以防止赋值号被错用为等号，但同时也导致 `x = y = z` 是不合法的。
- 如果赋值的右边是一个元组，其元素可以被分解开来，如 `(x, y, _) = (1, 2, 3)`。
- C 语言中的 `++` 和 `--` 运算符已于 Swift 3.0（[SE-0004](https://github.com/apple/swift-evolution/blob/master/proposals/0004-remove-pre-post-inc-decrement.md)）被废除，应当改用 `+=1` 和 `-=1`。

### 溢出

- 整数溢出会触发运行时错误，但如果要像 C 语言一样允许溢出，可以使用溢出运算符 `&+` `&-` `&*`。[^overflow]

[^overflow]: 因为除法溢出运算符 `&/` `&%` 的行为比较迷惑，因此已在 [Swift 1.2](https://developer.apple.com/library/archive/documentation/Xcode/Conceptual/RN-Xcode-Archive/Chapters/xc6_release_notes.html#//apple_ref/doc/uid/TP40016994-CH4-SW3) 中被移除。

### 求余

- 求余运算 `a % b` 的结果跟 `a` 的符号相同，而跟 `b` 的符号无关。这与 C / Java / JavaScript 等语言是一致的，一般称这样的运算为取余（remainder）；
- 而 Python / Ruby 等语言 `%` 运算结果的符号只与 `b` 相同，一般称其为取模（modulo）。

### 空合运算符

```swift
a ?? b // nil coalescing operator
a != nil ? a! : b // equivalent form
```

- `a` 必须是可空类型，且 `b` 要与 `a` 所存储值的类型一致。

### 区间运算符

- 闭区间运算符 `a...b` 表示 [a, b]；
- 半开区间运算符 `a..<b` 表示 [a, b)；
- 单侧无界区间运算符 `a...` / `...a` / `..<a` 分别表示 [a, +∞) / (−∞, a] / (−∞, a)。[^range]

[^range]: 单侧无界区间运算符于 Swift 4.0（[SE-0172](https://github.com/apple/swift-evolution/blob/master/proposals/0172-one-sided-ranges.md)）引入，用于简化合集类型的下标访问。


## 字符串和字符

- Swift 无论字符串还是字符都使用双引号而不用单引号，多行字符串字面量则可以放在三个双引号内。字符串之间可以通过 `+` 连接，将字符连接到字符串尾部可以使用 `append()` 方法，在字符串中插值（interpolation）可以使用 `"\()"`。
- 因为一个字符占用的空间可能不同，所以需要使用特殊的 `String.Index` 类型作为下标获取字符串指定位置的字符，如 `string[string.index(string.endIndex, offsetBy: -7)]`。[^index]

[^index]: Swift 3.0（[SE-0065](https://github.com/apple/swift-evolution/blob/master/proposals/0065-collections-move-indices.md)）对合集类型及其下标访问进行了一次大改，本来由索引类型负责的遍历工作转交给合集类型本身，这样索引对象就不必再持有对合集对象的引用了。比如要取合集对象 `c` 的第二个元素的索引，原来是  `c.startIndex.successor()`，现在是 `c.index(after: c.startIndex)`。

### Unicode

- Unicode 是目前世界通行的字符编码标准，其字符集共定义了 U+0000 至 U+10FFFF 共 17×2¹⁶ 个码位。Swift 支持 Unicode 的三种编码方案 UTF-8 / UTF-16 / UTF-32，分别对应字符串的 `utf8` / `utf16` / `unicodeScalars` 三个属性。
- 在字符串字面量中，Unicode 码位 U+xxxx 可以使用 `\u{xxxx}` 表示，其中 `xxxx` 可以为 1-8 位的十六进制数（但实际上 Unicode 目前最多 6 位）。
- Swift 的字符表示一个**扩展字素群**（extended grapheme cluster），可以包含多个 Unicode 码位，例如单个字符「é」可由两个码位 `"\u{65}\u{301}"` 表示。
- 而以前遗留下来的 `NSString` 本质上是 UTF-16 编码的码元数组，相应的 `length` 属性是其包含的码元个数，因此很可能会与 `String` 的 `count` 属性大小不同。[^unicode]

[^unicode]: [NSString 与 Unicode — ObjC 中国](https://objccn.io/issue-9-1/)

![Swift String Views](https://developer.apple.com/swift/blog/images/swift-string-views_2x.png)


## 合集类型

- 合集类型（Collection types）包括数组（Array）、集合（Set）[^set] 和字典（Dictionary），其存储的元素类型必须相同；实际上，上一章节介绍的字符串 [^string] 也遵循 `Collection` 协议。合集类型均由结构体通过泛型实现，为**值类型**。
- 根据合集类型协议的要求，`count` 属性能获取其元素个数，`isEmpty` 属性能判断是否为空，下标访问既可以使用索引也可以使用索引区间（即 `Range<Self.Index>`）等等。

[^set]: [Swift 1.2](https://developer.apple.com/library/archive/documentation/Xcode/Conceptual/RN-Xcode-Archive/Chapters/xc6_release_notes.html#//apple_ref/doc/uid/TP40016994-CH4-SW6) 引入了原生的 `Set` 类型，与原先的 `NSSet` 桥接。

[^string]: 在 Swift 1.x 中字符串遵循 `CollectionType` 协议，相当于字符数组；但 [Swift 2.0](https://developer.apple.com/swift/blog/?id=30) 取消了这一设定，认为 Unicode 字符串与字符数组有明显差异，原因包括字符串拼接后可能有字符合并、字符串相等的判断不是逐元素比较而是标准等价；然而 Swift 4.0（[SE-0163](https://github.com/apple/swift-evolution/blob/master/proposals/0163-string-revision-1.md)）又重新评估认为之前的改动弊大于利，字符串再次遵循 `Collection` 协议。

### 数组

- 数组类型可以表示为 `Array<Element>`，简写为 `[Element]`。
- 创建空数组可用 `[Element]()`，以重复的值创建数组可用 `Array(repeating:count:)`。
- 可以用 `insert(_:at:)` / `append(_:)` / `remove(at:)` / `removeLast()` / `firstIndex(of:)` 来插入、删除、查找元素。
- 如果数组下标越界或为负数，会直接触发运行时错误。

### 集合

- 集合类型可以表示为 `Set<Element>`，其中 `Element` 必须是可哈希的，即遵循 `Hashable` 协议。
- 创建空数组可用 `Set<Element>()`，亦可用数组字面量来初始化集合 `var groups: Set = ["AKB48", "SKE48", "NMB48", "HKT48", "NGT48", "STU48"]`。
- `insert(_:)` / `remove(_:)` / `contains(_:)` 方法分别用来插入、删除、查找元素。
- `isSubset(of:)` / `isSuperset(of:)` / `isDisjoint(with:)` 方法分别用来判断子集、超集、互斥。
- `union(_:)` / `intersection(_:)` / `subtracting(_:)` / `symmetricDifference(_:)` 方法分别会创建两个集合的并集、交集、差集、对称差；另有 `formUnion(_:)` / `formIntersection(:_)` / `subtract(_:)` / `formSymmetricDifference(_:)` 会直接在原集合上进行修改。[^setalgebra]

[^setalgebra]: Swift 3.0（[SE-0059](https://github.com/apple/swift-evolution/blob/master/proposals/0059-updated-set-apis.md)）按照 [API Design Guidelines](https://swift.org/documentation/api-design-guidelines/#name-according-to-side-effects) 中无副作用用名词、有副作用用动词的方针，对 `SetAlgebra` 协议中的方法名进行过调整，比如将原来的 `subtract` / `subtractInPlace` 改成了现在的 `subtracting` / `subtract`。

### 字典

- 字典类型可以表示为 `Dictionary<Key, Value>`，简写为 `[Key: Value]`，其中 `Key` 必须是可哈希的。
- 访问字典可以使用 `dict[key]`，返回值为可空类型，键不存在即返回 `nil`。新增、修改键值亦可使用下标，而删除键值只需 `dict[key] = nil`。
- 遍历字典可用 `for (key, value) in dict { … }` 或单独遍历 `dict.keys` 和 `dict.values`。


## 控制流

- 所有控制流都不需要条件外侧的圆括号，但不可以省略语句体的花括号。

### 循环语句

- `for i in 0..<10` 中的 `i` 是一个每轮循环开始时自动生成的局部常量，因此不需要提前声明。
- C 样式的 `for` 循环已于 Swift 3.0（[SE-0007](https://github.com/apple/swift-evolution/blob/master/proposals/0007-remove-c-style-for-loops.md)）被废除，当然传统的 `while` 和 `repeat-while` [^repeat] 循环仍然存在。
- 可以在循环语句（或 `if` / `switch`）前放置一个标签 `label:`，则可以用 `break label` 来中断特定循环（或条件分支），或用 `continue label` 跳过特定循环的当前轮。

[^repeat]: `repeat-while` 原为 `do-while`，Swift 2.0 之后 `do` 关键字被改用于错误处理。

### Switch

- `switch` 语句必须是完备的，如果各 `case` 分支不能涵盖所有情况时，最后要有 `default` 分支。如果能匹配多个 `case`，那么只会执行第一个匹配的分支。
- `switch` 不存在隐式的贯穿，即不需要在 `case` 分支结束时写 `break`；不过如果一定要像 C 语言那样贯穿到下一个 `case`，可以用 `fallthrough` 关键字。 
- 每个 `case` 必须包含至少一条语句，所以两个 `case` 连着写会编译错误。如果是需要一次处理多种情况，可以在单个 `case` 中把多个表达式用逗号分开；如果是什么都不做，要写个 `break`。
- `case let` 允许将匹配的值绑定到局部常量，并可以使用 `where` 来判断额外条件。

```swift
switch point {
case (0, 0):
    print("At the origin.")
case (_, 0), (0, _):
    print("On an axis.")
case (-2...2, -2...2):
    print("Inside a 4x4 box.")
case let (x, y) where x < y:
    print("(\(x), \(y)) is above the line y = x.")
default:
    print("Just some arbitrary point.")
}
```

### 模式匹配

- Swift 在进行 `case` 的匹配时，实际上使用了 `~=` 运算符，譬如为区间的匹配定义了 `static func ~=(pattern: Range<Bound>, value: Bound) -> Bool`。因此，我们也可以为自定义类型定义 `~=` 运算符。
- 当只需要匹配一条 `case` 时，可以使用 `if case let x = y { … }` 来代替 `switch y { case let x: … }`，类似的还有 `guard case let`，后面都可以接 `where` 判断。[^pattern]
- 使用 `for case` 可以只遍历相应 `if case` 匹配成功的元素，也可以后接 `where` 判断，实际上使用 `for … where` 而不带 `case` 依然是合法的。[^pattern]

[^pattern]: [模式匹配第四弹：if case, guard case, for case — Crunchy Development](http://swift.gg/2016/06/06/pattern-matching-4/)

```swift
for case let (title, kind) in mediaList.map({ ($0.title, $0.kind) }) where title.hasPrefix("Harry Potter") {
    print("- [\(kind)] \(title)")
}
```

### Guard

- `guard … else { … }` 类似于只有 `else` 分支的 `if` 语句。
- 如果条件满足则跳过花括号里的内容，并且 `guard let` 可空绑定对当前代码块的剩下部分依然有效。
- 如果条件不满足，`else` 分支必须退出当前代码块，譬如使用 `return` / `break` / `continue` / `throw` / `fatalError()`。

### API 可用性

- 在 `if` 或 `guard` 语句中可以使用 `#available` 检查当前操作系统版本（包括 macOS / iOS / watchOS / tvOS）和 Swift 版本 [^available]，以验证 API 目前是否可用。类似地，也可以在各种声明前加 `@available`。
- 最后一个参数 `*` 是不可省略的，表示在未指定的平台上，其版本与最低部署目标相同。

```swift
if #available(macOS 10.12, iOS 10, *) {
    // Use macOS Sierra and iOS 10 APIs
} else {
    // Fall back to earlier macOS and iOS APIs
}
```

[^available]: 操作系统版本检查于 [Swift 2.0](https://developer.apple.com/swift/blog/?id=29) 引入，Swift 版本检查则 Swift 3.1（[SE-0141](https://github.com/apple/swift-evolution/blob/master/proposals/0141-available-by-swift-version.md)）才引入，之前得用预处理指令 `#if swift(>= N)`。


## 函数

### 参数与返回值

- 无参函数在定义和调用时不能省略括号。无返回值函数在定义时不需要写 `-> Type`，实际上它返回了一个特殊的值 `Void`，这是一个空的元组即 `()`。
- 可以使用元组类型让函数返回多个值。

### 参数名称

```swift
func join(_ s1:String, to s2: String, joiner: String = " ") -> String {
    return s1 + joiner + s2
}
join("hello", to: "world", joiner: ", ")
join("hello", to: "world")
```

- 上述代码中的 `s1` / `s2` 部分为**参数名称**（parameter names），在函数内部使用；`_` / `to` 部分为**参数标签**（argument labels），在调用函数时使用，以加强可读性。若不指定参数标签，则参数标签与参数名称相同，也可以使用 `_` 忽略参数标签。此项规则也适用于方法和构造器。[^parameter]
- 在参数类型后加 `...` 可定义**可变参数**（variadic parameters），调用时可以传入不确定数量的参数，在函数内该参数将作为数组使用。一个函数至多只能有一个可变参数。
- 函数参数默认是常量，如果需要修改参数在函数外的实际值，可以定义**输入输出参数**（in-out parameters）。首先需要在参数的类型前加关键字 `inout`，其次调用时传入的变量前要加 `&`。

[^parameter]: 在 Swift 1.x 时代，函数默认没有外部参数名，方法除了第一个参数其他都有外部参数名，而构造器所有参数都有外部参数名；在 Swift 2.0 中，函数改用了方法的外部参数名规则；Swift 3.0（[SE-0046](https://github.com/apple/swift-evolution/blob/master/proposals/0046-first-label.md)）开始，函数和方法都统一成了构造器的外部参数名规则，并改称参数标签。

### 函数类型

- 函数类型可以表示为诸如 `(Int, Int) -> Int` 的形式，既无参数也无返回值的函数类型为 `() -> Void`。函数类型只与参数类型和返回值有关，参数标签不是类型的一部分。[^label]
- 函数是 Swift 语言的**一等公民**，可以作为参数类型和返回类型。
- Swift 允许定义**嵌套函数**，嵌套函数只在其作用域中可见，但也可以由其外层函数返回从而被外界使用。

[^label]: Swift 3.0（[SE-0111](https://github.com/apple/swift-evolution/blob/master/proposals/0111-remove-arg-label-type-significance.md)）之前，参数标签是类型的一部分，有无参数标签的函数类型会建立子类型关系，导致参数和返回类型相同但参数标签不同的函数变量在互相赋值时有奇怪的表现。


## 闭包

- 广义来讲 Swift 中有三种闭包，而狭义的闭包是指最后一种：
  - 全局函数是一个有名字但不会捕获任何值的闭包；
  - 嵌套函数是一个有名字并可以捕获其外层函数作用域中值的闭包；
  - 闭包表达式是一个可以捕获外界值的匿名闭包，与 Objective-C 的代码块以及其他语言的 lambda 表达式类似。
- 在 Swift 中，当一个闭包作为参数传入，但它在函数返回后才被执行，则认为它是一个特殊的**逃逸闭包**（escaping closure）。闭包参数默认是不逃逸的 [^escaping]，因此需要在逃逸闭包的参数类型前加上 `@escaping`，同时这也意味着闭包内的 `self.` 将不可省略。

[^escaping]: 以前闭包参数默认是可以逃逸的，只有 `@noescape` 没有 `@escaping`；Swift 3.0（[SE-0103](https://github.com/apple/swift-evolution/blob/master/proposals/0103-make-noescape-default.md)）将其改为默认不逃逸以简化编译器相关算法，并且 `@escaping` 可以通过 Xcode 的静态检查自动添加。

```swift
// Closure expression syntax
reversed = names.sorted(by: { (s1: String, s2: String) -> Bool in
    return s1 > s2
})
// Inferring type from context
reversed = names.sorted(by: { s1, s2 in return s1 > s2 })
// Implicit returns from single-expression closures
reversed = names.sorted(by: { s1, s2 in s1 > s2 })
// Shorthand argument names
reversed = names.sorted(by: { $0 > $1 })
// Trailing closures
reversed = names.sorted { $0 > $1 }
// Operator methods
reversed = names.sorted(by: >)
```


## 枚举

- 枚举类型也是 Swift 语言的**一等公民**，它支持了很多传统上类独有的特性，例如实例方法、计算属性、遵循协议等。
- 枚举与其他类型名一样，应当首字母大写；而枚举的 `case` 成员应当首字母小写 [^enumcase]。

[^enumcase]: 以前枚举的 `case` 成员是首字母大写的，但 Swift 3.0 推出的 [API Design Guidelines](https://swift.org/documentation/api-design-guidelines/#follow-case-conventions) 规定除类型和协议外其余一律首字母小写。

### 关联值

- 枚举成员可以关联任意各种类型的值，且每个成员关联的类型和数量可以各不相同。这扩展了 C 语言中的联合（union），相当于在 Swift 中实现了代数数据类型，当然模式匹配亦适用于此。

```swift
enum Barcode { // Associated Values
    case upc(Int, Int, Int, Int)
    case qrCode(String)
}

var productBarcode = Barcode.upc(8, 85909, 51226, 3)
productBarcode = .qrCode("ABCDEFGHIJKLMNOP")

switch productBarcode {
case let .upc(numberSystem, manufacturer, product, check):
    print("UPC: \(numberSystem), \(manufacturer), \(product), \(check).")
case let .qrCode(productCode):
    print("QR code: \(productCode).")
}
```

- Swift 还支持**递归枚举**，在 `enum` 或 `case` 前加上 `indirect` 关键字即可提醒编译器为递归结构生成必要的中间层。

```swift
enum ArithmeticExpression { // Recursive Enumeration
    case number(Int)
    indirect case addition(ArithmeticExpression, ArithmeticExpression)
    indirect case multiplication(ArithmeticExpression, ArithmeticExpression)
}
```

### 原始值

- 枚举成员也可以被预先填充为同一类型的原始值，这与 C 语言中的枚举（enum）类似。
- 当整型被用于原始值时，没有初始化的成员默认为上个成员的值加一，第一个成员则默认为 0；当字符串被用于原始值时，没有初始化的成员默认值为自身的名字。
- 枚举成员的 `rawValue` 属性可以获取其原始值，而枚举的构造器接受 `rawValue` 参数并返回一个可空类型。

```swift
enum Plant: Int { // Raw Values
    case mercury = 1, venus, earth, mars, jupiter, saturn, uranus, neptune
}

let earthOrder = Planet.earth.rawValue // = 3
let somePlanet = Planet(rawValue: 7)! // = .uranus
```


> **\<Next\>** [Swift 学习笔记（二）](/swift-notes-2/)
