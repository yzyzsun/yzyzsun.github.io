---
layout: post
title: Swift 学习笔记（一）
category: Tech
author: 孙耀珠
---

苹果自收购 NeXT 公司开始便使用 Objective-C 作为主力开发语言，至今已有十多年了，若是从这门语言发明之日算起更是超过三十年；而在这期间微软推出了 C# 和 F#，谷歌推出了 Go 和 Dart，Mozilla 推出了 Rust……虽然 Objective-C 随着 OS X 和 iOS 的迅速发展而越来越火，但相比之下它的语言设计已经落后于时代了，于是在这个大背景下 Swift 诞生了，开发者正是 LLVM / Clang 之父 Chris Lattner。

Swift 仍然是一门静态类型的语言，不过它拥有很多现代的语言特性，譬如类型推断、泛型、元组、更优雅的闭包等等，同时也有 Playgrounds 这样便利的交互式编程环境。Swift 非常强调安全性，不论是随处可见的可选类型、继承时复杂的构造规则，还是赋值没有返回值、控制流不能省略花括号，都是为了代码安全而考虑。另外 Swift 终于丢掉了 C 语言的包袱，放弃了指针，`switch` 语句不再需要 `break`，整型溢出会抛出运行时错误等等。

我们能够看出它本身还是构建在 Objective-C 的基础之上，两者能够很方便地交互和共存，Cocoa / Cocoa Touch 的 API 也是共通的。Swift 的语法目前仍在不断改进，从 [Swift-Evolution Proposal Status](https://apple.github.io/swift-evolution/) 可见一斑，我也会根据最新的文档及时更新这三篇学习笔记。*（Updated: 2016-10-17）*

* 目录
{:toc}

## 数据类型

### 整数

- 在 32 位平台上，`Int` / `UInt` 和 `Int32` / `UInt32` 长度相同。
- 在 64 位平台上，`Int` / `UInt` 和 `Int64` / `UInt64` 长度相同。
- 字面量前缀：二进制为 `0b`，八进制为 `0o`，十六进制为 `0x`。

<!--more-->

### 浮点数

- `Float` 为 32 位浮点数；`Double` 为 64 位浮点数，浮点数字面量会被自动推断为 `Double`。
- `1.25e2` 表示 1.25×10<sup>2</sup>；`0xFp2` 表示 15×2<sup>2</sup>。
- 加减乘除运算严格检查左右操作数类型是否相同，不会进行隐式类型转换，因此 `Int` / `UInt` / `Double` / `Float` / `CGFloat` 之间进行运算时需要强制类型转换，可以使用 `Int()` 等构造器完成。

### 元组（Tuples）

- 元组内的值可以是任意不同类型。
- 可以通过点语法来访问元组中的单个元素，下标从零开始，如 `tuple.0`。
- 在定义时可以给元素命名，命名后便可通过名字来获取元素的值。

```swift
let status = (code: 200, message: "OK")
print("Code: \(status.code), message: \(status.message)")
```

### 可选类型（Optionals）

- 可选类型相当于一个特殊的枚举类型：成员 `None` 表示值为 `nil`；成员 `Some` 则可以通过 `!` 来**强制解析**（forced unwrapping）获取值，或是通过 `?` 构成**可选链**（optional chaining）。对 `nil` 进行强制解析会触发运行时错误 `EXC_BAD_INSTRUCTION`，而可选链不会。
- 当可选链中有可选值为 `nil` 时整条链失败并返回 `nil`，但不会触发运行时错误；若成功则返回一个相应的可选类型。
- 在 `if` 和 `while` 语句中使用**可选绑定**（optional binding）可以判断可选类型是否包含值，若包含则将值赋给临时常量或变量，可使用 `where` 来判断额外条件。[^binding]

[^binding]: 从 [Swift 1.2](https://developer.apple.com/library/ios/releasenotes/DeveloperTools/RN-Xcode/Chapters/xc6_release_notes.html#//apple_ref/doc/uid/TP40001051-CH4-SW6) 开始，`if-let` / `while-let` 语句支持多个可选绑定，且可选绑定可以接在布尔条件后面用 `,` 隔开。从 [Swift 3.0](https://github.com/apple/swift-evolution/blob/master/proposals/0099-conditionclauses.md) 开始，逗号只用于 condition clause 间的分割，不再用于 condition clause 内的分割，于是可选绑定不再限定位置、但前面的 `let` 不可省略了，原来接在 `let` 后面使用的 `where` 关键字也不再需要了。

```swift
if let a = foo(), let b = bar(), a < b && b < 42 {
    // Do something
}
```

- 如果在第一次被赋值之后可以确定一个可选类型总会有值，可以采用**隐式解析可选类型**（implicitly unwrapped optionals），声明时将类型后面的 `?` 改为 `!`，则之后获取值时将不需要解析。

```swift
var optionalString: String? // nil
var possibleString: String? = "233"
print(possibleString!)
var assumedString: String! = "666"
print(assumedString)
```


## 运算符

### 赋值

- 赋值运算不返回任何值，以防止赋值号被错用为等号，但同时也导致 `x = y = z` 是不合法的。
- 如果赋值的右边是一个元组，其元素可以被分解开来，如 `(x, y, _) = (1, 2, 3)`。

### 溢出

- 整数溢出会触发运行时错误，但如果要像 C 一样允许溢出，可以使用溢出运算符 `&+` `&-` `&*`。[^overflow]

[^overflow]: 溢出运算符 `&/` `&%` 在 [Swift 1.2](https://developer.apple.com/library/ios/releasenotes/DeveloperTools/RN-Xcode/Chapters/xc6_release_notes.html#//apple_ref/doc/uid/TP40001051-CH4-SW3) 中被移除。

### 求余

- 求余运算 `a % b` 的结果跟 `a` 的符号相同，而跟 `b` 的符号无关。这与 C / Java / JavaScript 等语言是一致的，一般称这样的运算为**求余**（remainder）。
- 而 Python / Ruby 等语言 `%` 运算结果的符号只与 `b` 相同，一般称其为**求模**（modulo）。[^modulo]

[^modulo]: [Modulo operation - Wikipedia](https://en.wikipedia.org/wiki/Modulo_operation)

### 空合运算符（Nil Coalescing Operator）

```swift
a ?? b
// is equal to
a != nil ? a! : b
```

- `a` 必须是可选类型，且 `b` 要与 `a` 所存储值的类型一致。

### 区间运算符

- 闭区间运算符 `a...b` 表示 [a, b]
- 半开区间运算符 `a..<b` 表示 [a, b)


## 字符串和字符

- 与 C 语言不同，Swift 无论字符串还是字符都使用双引号 `"` 而不用单引号 `'`。
- Swift 的 String 是**值类型**，当其进行赋值操作或在函数中传递时会进行值拷贝。
- 字符串之间可以通过 `+` 连接，将字符连接到字符串尾部可以使用 `append()` 方法。通过构造器也可以使用字符数组来创建一个字符串。
- 在字符串中插值可以使用 `"\()"`。

### Unicode

- 在字符串字面量中，**Unicode scalar value** 可以表示为 `\u{n}`，其中 `n` 可以为 1-8 位的十六进制数。
- 目前 Unicode 编码共 21 位，Unicode scalars (code points) 的范围包括：
    - **基本多文种平面**（Basic Multilingual Plane, Plane 0）：[`U+0000`, `U+D7FF`] ∪ [`U+E000`, `U+FFFF`]；
    - **多文种补充平面**（Supplementary Multilingual Plane, Plane 1）：[`U+10000`, `U+1FFFF`]；
    - **表意文字补充平面**（Supplementary Ideographic Plane, Plane 2）：[`U+20000`, `U+2FFFF`]；
    - 尚未使用的第三至十三平面：[`U+30000`, `U+DFFFF`]；
    - **特别用途补充平面**（Supplementary Special-purpose Plane, Plane 14）：[`U+E0000`, `U+EFFFF`]；
    - **私人使用区**（Private Use Area, Plane 15-16）：[`U+F0000`, `U+10FFFF`]；
    - 但不包括 UTF-16 **代理对**（surrogate pair）的码位：[`U+D800`, `U+DFFF`]。
- 分别可以通过字符串的 `utf16` / `utf8` / `unicodeScalars` / `characters` 属性来访问其 UTF-8 / UTF-16 / Unicode Scalars / 字符集合。
- 注意 Swift 的字符类型表示一个**扩展字形集群**（extended grapheme cluster），可以包含多个 Unicode scalars，例如一对 Unicode scalars `"\u{65}\u{301}"` 与单个 Unicode scalar `\u{E9}` 均表示**单个**字符「é」。
- `str.characters.count` 可以获得字符串中的字符个数。因为一个字符占用的空间可能不同，所以需要使用特殊的 `String.Index` 类型作为下标获取字符串指定位置的字符，如 `str[str.index(after: str.startIndex)]` 和 `str[str.index(str.endIndex, offsetBy: -7)]`。[^stringindex]
- 而 NSString 其实是用 UTF-16 编码的码元（code units）组成的数组，相应的 `length` 属性的值是其包含的码元个数，而不是字符个数。[^unicode]

![Swift String Views](/images/swift-string-views.png)

[^stringindex]: [Swift 2](https://developer.apple.com/swift/blog/?id=30) 和 [Swift 3](https://github.com/apple/swift-evolution/blob/master/proposals/0065-collections-move-indices.md) 分别对字符串的下标访问方式做出了不小的改动。

[^unicode]: [NSString 与 Unicode - objc中国](http://objccn.io/issue-9-1/)


## 集合类型

- 集合类型（Collection types）包括数组（Array）、集合（Set）[^set] 和字典（Dictionary），其存储值类型必须相同，由泛型（generic）实现。
- 集合类型均由结构体实现，为**值类型**。
- 获取元素个数可访问其 `count` 属性，判断是否为空可以用 `isEmpty` 属性。

[^set]: [Swift 1.2](https://developer.apple.com/library/ios/releasenotes/DeveloperTools/RN-Xcode/Chapters/xc6_release_notes.html#//apple_ref/doc/uid/TP40001051-CH4-SW6) 引入了原生的 `Set` 类型，与原先的 `NSSet` 桥接。

### 数组

- 数组类型可以表示为 `Array<Element>`，简写为 `[Element]`。
- 创建空数组可用 `[Element]()`，以重复的值创建数组可用 `Array(repeating:count:)`。
- 可以用 `insert(_:at:)` / `append(_:)` / `remove(at:)` / `removeLast()` 来插入和删除元素。
- 如果数组下标越界或为负数，会直接触发运行时错误。

### 集合

- 集合类型可以表示为 `Set<Element>`。`Element` 必须是可哈希的，即遵循 `Hashable` 协议。
- 创建空数组可用 `Set<Element>()`，可用数组字面量来初始化集合 `var groups: Set = ["AKB48", "SKE48", "NMB48", "HKT48", "NGT48", "STU48"]`。
- 分别用 `insert(_:)` / `remove(_:)` / `contains(_:)` 方法来插入、移除、判断元素在集合中。
- `union(_:)` / `intersection(_:)` / `subtracting(_:)` / `symmetricDifference(_:)` 方法分别会创建两个集合的并集、交集、差集、对称差；另有 `formUnion(_:)` / `formIntersection(:_)` / `subtract(_:)` / `formSymmetricDifference(_:)` 会直接在原集合上进行修改。[^setalgebra]
- `isSubset(of:)` / `isSuperset(of:)` / `isDisjoint(with:)` 方法分别用来判断子集、超集、互斥。

[^setalgebra]: [Swift 3.0](https://github.com/apple/swift-evolution/blob/master/proposals/0059-updated-set-apis.md) 按照 API Design Guidelines 对集合代数的接口做了调整。

### 字典

- 字典类型可以表示为 `Dictionary<Key, Value>`，简写为 `[Key: Value]`。`Key` 类型必须是可哈希的。
- 访问字典可以使用 `dic["key"]`，返回值为可选类型，键不存在即返回 `nil`。
- 遍历字典可用 `for (key, value) in dict { … }` 或单独遍历 `dict.keys` 和 `dict.values`。


## 控制流

- 所有控制流都不需要条件外侧的圆括号，但不可以省略语句体的花括号。

### 循环语句

- `for i in 0..<10` 中的 `i` 是一个每轮循环开始时自动赋值的常量，因此不需要提前声明。
- `for-in` 若不需要知道循环变量的值，可用 `_` 代替变量名。
- C 样式 `for` 循环已于 [Swift 3.0](https://github.com/apple/swift-evolution/blob/master/proposals/0007-remove-c-style-for-loops.md) 被废除，不过当然 `while` 和 `repeat-while` [^repeat] 循环仍然存在。
- 可以在循环语句或下面提到的 `switch` 语句前放置一个标签 `label:`，则可以用 `continue label` 或 `break label` 来跳过它。

[^repeat]: `repeat-while` 原为 `do-while`，Swift 2.0 之后 `do` 关键字被用于错误处理。

### Switch

- `switch` 语句必须是完备的，即在各 `case` 分支不能涵盖所有情况时，最后要有 `default` 分支。如果能匹配多个 `case`，那么只会执行第一个匹配的分支。
- `switch` 不存在隐式的贯穿，即不需要在 `case` 分支结束时写 `break`；不过如果一定要像 C 那样贯穿到下一个 `case`，可以用 `fallthrough` 关键字。 
- 每个 `case` 必须包含至少一条语句，所以两个 `case` 连着写会编译错误。如果是需要一次处理多种情况，可以在单个 `case` 中把多个表达式用逗号分开；如果是什么都不做，要写个 `break`。
- `case` 的表达式可以是区间或者元组，另可使用 `_` 来匹配所有可能的值。
- `case let` 允许将匹配的值绑定到临时值（value bindings），并可以使用 `where` 来判断额外条件。

```swift
switch somePoint {
case (0, 0):
    print("At the origin")
case (let distance, 0), (0, let distance):
    print("On an axis, \(distance) from the origin")
case (-2...2, -2...2):
    print("Inside the box")
case let (x, y) where x == y:
    print("(\(x), \(y)) is on the line x == y"
default:
    print("Just some arbitrary point")
}
```

### 模式匹配 [^pattern]

[^pattern]: [Pattern Matching, Part 4 – Crunchy Development](http://alisoftware.github.io/swift/pattern-matching/2016/05/16/pattern-matching-4/)

- Swift 在进行 `case` 的匹配时，实际上使用了 `~=` 运算符，譬如为区间的匹配定义了 `static func ~=(pattern: Range<Bound>, value: Bound) -> Bool`。因此，我们也可以为自定义类型定义 `~=` 运算符。
- 当 `switch` 处理一个可选值时，可以在 `case` 中使用 `x?` 作为语法糖来表示 `.Some(x)`。
- 当只需要匹配一条 `case` 时，可以使用 `if case let x = y { … }` 来代替 `switch y { case let x: … }`，类似的还有 `guard case let`，后面都可以接 `where` 判断。
- 使用 `for case` 可以只遍历相应 `if case` 匹配成功的元素，也可以后接 `where` 判断，实际上使用 `for … where` 而不带 `case` 依然是合法的。

```swift
for case let (title?, kind) in mediaList.map({ ($0.title, $0.kind) }) where title.hasPrefix("Harry Potter") {
    print("- [\(kind)] \(title)")
}
```

### Guard [^guard]

[^guard]: `guard` 语句于 [Swift 2.0](https://developer.apple.com/swift/blog/?id=29) 后被引进。

- `guard … else { … }` 类似于只有 `else` 分支的 `if` 语句。
- 如果条件满足则跳过花括号的内容，并且可选绑定的赋值对当前代码块的剩下部分依然有效。
- 如果条件不满足，`else` 分支必须退出当前代码块，譬如使用 `return` / `break` / `continue` / `throw` / `fatalError()`。

### 检查 API 可用性

- 在 `if` 或 `guard` 语句中可以判断当前平台版本（包括 `macOS` / `iOS` / `watchOS` / `tvOS`），以验证 API 目前是否可用。[^available]
- 最后一个参数 `*` 是不可省略的，表示在未指定的平台上，其版本与最低部署目标相同。

```swift
if #available(macOS 10.12, iOS 10, *) {
    // Use macOS Sierra and iOS 10 APIs
} else {
    // Fall back to earlier macOS and iOS APIs
}
```

[^available]: `#availale()` 于 [Swift 2.0](https://developer.apple.com/swift/blog/?id=29) 后被引进。


## 函数

### 参数与返回值

- 无参函数在定义和调用时需要写一对空括号。
- 无返回值函数在定义时不需要写 `-> Type`，实际上它返回了一个特殊的值 `Void`，这是一个空的元组即 `()`。
- 可以使用元组类型让函数返回多个值。

### 参数名称

```swift
func join(_ s1:String, to s2: String, joiner: String = " ") -> String {
    return s1 + joiner + s2
}
join("hello", to: "world", joiner: ", ")
join("hello", to: "world")
```

- 上述代码中的 `s1` / `s2` 部分为**参数名称**（parameter name），在函数内部使用；`_` / `to` 部分为**参数标签**（argument label），在调用函数时使用，以加强可读性。
- 若不指定参数标签，则参数标签与参数名称相同，也可以使用 `_` 忽略参数标签。此项规则现在也适用于方法和构造器。[^parameter]
- 在参数类型后加 `...` 可定义**可变参数**（variadic parameters），调用时可以传入不确定数量的参数，在函数内这将被当做这个类型的一个数组，但一个函数至多只能有一个可变参数。
- 函数参数默认是常量，如果需要修改参数在函数外的实际值，可以定义**输入输出参数**（in-out Parameters）。首先需要在参数的类型前加关键字 `inout`，其次调用时传入的变量前要加 `&`。

[^parameter]: 在 Swift 1.x 时代，函数默认没有外部参数名（现称参数标签），方法除了第一个参数其他默认都有外部参数名，而构造器所有参数都有外部参数名。在 Swift 2.0 中，函数改用了方法的外部参数名规则；在 [Swift 3.0](https://github.com/apple/swift-evolution/blob/master/proposals/0046-first-label.md) 之后，函数和方法都统一成了构造器的参数标签规则。

### 函数类型

- 函数类型可以表示为诸如 `(Int) -> Int` 的形式，既无参数也无返回值的函数类型为 `() -> Void`。从 [Swift 3.0](https://github.com/apple/swift-evolution/blob/master/proposals/0066-standardize-function-type-syntax.md) 开始，即使是单参数函数也不能省略参数两边的圆括号。
- 函数的类型现在只与参数类型和返回值有关，因此函数作为变量时不需要书写函数标签，函数标签不再是类型的一部分。[^functiontype]
- 函数在 Swift 中是一等公民（first-class citizen），可以作为参数类型和返回类型。
- 函数可以被定义在别的函数体内，这被称为**嵌套函数**（nested function）。嵌套函数对全局是不可见的，但可以被它的封闭函数（enclosing function）返回从而被外界使用。

[^functiontype]: [Swift 3.0](https://github.com/apple/swift-evolution/blob/master/proposals/0111-remove-arg-label-type-significance.md) 之前，函数标签是类型的一部分，且函数变量在被重新赋值时会保留原来的参数标签，从而有奇怪的表现。

## 闭包

- Swift 的闭包与 Objective-C 的代码块（blocks）以及其他语言的 lambdas 函数类似。Swift 的闭包是**引用类型**。
- 全局函数是一个有名字但不会捕获任何值的闭包。
- 嵌套函数是一个有名字并可以捕获其封闭函数域内值的闭包。
- 闭包表达式是一个利用轻量级语法所写的可以捕获其上下文中变量或常量值的匿名闭包。
- 当闭包作为参数传给函数时，若闭包的调用发生在函数返回之后，则称这是一个**逃逸闭包**（escaping closure）。闭包参数默认是不逃逸的 [^escaping]，如果要允许闭包逃逸，可以在参数类型前加上 `@escaping`，但这也意味着在闭包中 `self.` 将不可省略。

[^escaping]: 在 [Swift 3.0](https://github.com/apple/swift-evolution/blob/master/proposals/0103-make-noescape-default.md) 之前，闭包参数默认是可以逃逸的，因此只有 `noescape` 关键字没有 `escaping` 关键字。

```swift
// Closure expression syntax
reversed = names.sorted(by: { (s1: String, s2: String) -> Bool in
    return s1 > s2
})

// Inferring type from context
reversed = names.sorted(by: { s1, s2 in return s1 > s2 })

// Implicit return from single-expression closures
reversed = names.sorted(by: { s1, s2 in s1 > s2 })

// Shorthand argument names
reversed = names.sorted(by: { $0 > $1 })

// Trailing closures
reversed = names.sorted { $0 > $1 }

// Operator methods
reversed = names.sorted(by: >)
```

## 枚举

- 枚举类型是**一等公民**，它采用了很多传统上只被类所支持的特性，例如实例方法、计算属性、遵守协议等。
- 枚举与 Swift 中其他类型名一样，应当首字母大写；而枚举的 `case` 成员应当首字母小写 [^enumcase]。
- 与 C 语言不同，Swift 的枚举成员在被创建时不会被赋予一个默认的整数值。

[^enumcase]: 在 Swift 3.0 之前枚举的 `case` 成员为首字母大写。

### 相关值（Associated Values）

- 枚举可以存储任何类型的相关值，且每个成员的类型可以各不相同，这类似于 C 语言中的联合（union）。

```swift
enum Barcode {
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

### 原始值（Raw Values）

- 枚举成员也可以被预先填充为同一类型的原始值，这与 C 语言中的枚举（enum）类似。当整型被用于原始值时，如果其他枚举成员没有值整数会自动递增。
- 枚举成员的 `rawValue` 属性可以获取其原始值，而枚举的可失败构造器接受 `rawValue` 参数并返回一个可选类型。[^rawvalue]

[^rawvalue]: 原来的 `fromRaw()` / `toRaw()` 方法在 [Swift 1.1](https://developer.apple.com/library/ios/releasenotes/DeveloperTools/RN-Xcode/Chapters/xc6_release_notes.html#//apple_ref/doc/uid/TP40001051-CH4-DontLinkElementID_60) 中被现在的新语法取代。

```swift
enum Plant: Int {
    case mercury = 1, venus, earth, mars, jupiter, saturn, uranus, neptune
}

let earthOrder = Planet.earth.rawValue
let somePlanet = Planet(rawValue: 7)!
```


> **\<Next\>** [Swift 学习笔记（二）](/swift-notes-2/)
