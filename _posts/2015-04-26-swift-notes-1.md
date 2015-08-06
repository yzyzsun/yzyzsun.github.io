---
layout: post
title: Swift 学习笔记（一）
category: Tech
---

Swift 吸收了不少动态语言的语法，比如类型推断、元组、闭包等等；不过能够看出它本身还是建立在 Objective-C 的基础上的，Foundation / Cocoa / UIKit 的 API 基本是共通的。Swift 的语法目前仍在不断改进，从 [The Swift Programming Language: Document Revision History](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/RevisionHistory.html#//apple_ref/doc/uid/TP40014097-CH40-ID459) 可见一斑。


## 数据类型

### 整数

* 在 32 位平台上，`Int` / `UInt` 和 `Int32` / `UInt32` 长度相同。
* 在 64 位平台上，`Int` / `UInt` 和 `Int64` / `UInt64` 长度相同。
* 字面量前缀：二进制为 `0b`，八进制为 `0o`，十六进制为 `0x`。

### 浮点数

* `Float` 为 32 位浮点数，`Double` 为 64 位浮点数。
* `1.25e2` 表示 1.25×10^2 ；`0xFp2` 表示 15×2^2 。

### 元组（Tuples）

* 元组内的值可以是任意不同类型。
* 可以通过点语法来访问元组中的单个元素，下标从零开始，如 `tuple.0`。
* 在定义时可以给元素命名，命名后便可通过名字来获取元素的值。

```swift
let http200Status = (statusCode: 200, description: "OK")
println("Code: \(http200Status.statusCode), message: \(http200Status.description)")
```

<!--more-->

### 可选类型（Optionals）

* 可选类型相当于一个特殊的枚举类型：成员 `None` 表示值为 `nil`；成员 `Some` 则可以通过 `!` 来**强制解析**（forced unwrapping）获取值，或是通过 `?` 构成一个**可选链**（optional chaining）。对 `nil` 进行强制解析会触发运行时错误 `EXC_BAD_INSTRUCTION`，而可选链不会。
* 当可选链中有可选值为 `nil` 时整条链失败并返回 `nil`，但不会触发运行时错误；若成功则返回一个相应的可选类型。
* 在 `if` 和 `while` 语句中使用**可选绑定**（optional binding）可以判断可选类型是否包含值，若包含则将值赋给临时常量或变量，可使用 `where` 来判断额外条件。[^binding]

[^binding]: 从 [Swift 1.2](https://developer.apple.com/library/ios/releasenotes/DeveloperTools/RN-Xcode/Chapters/xc6_release_notes.html#//apple_ref/doc/uid/TP40001051-CH4-SW6) 开始，`if-let` / `while-let` 语句支持多个可选绑定，且可选绑定可以接在布尔条件后面用 `,` 隔开。

```swift
if let a = foo(), b = bar() where a < b {
    statements
}
```

* 如果在第一次被赋值之后可以确定一个可选类型总会有值，可以采用**隐式解析可选类型**（implicitly unwrapped optionals），声明时将类型后面的 `?` 改为 `!`，则之后获取值时将不需要解析。

```swift
var optionalString: String? // nil
var possibleString: String? = "233"
println(possibleString!)
var assumedString: String! = "666"
println(assumedString)
```


## 运算符

### 赋值

* 赋值运算不返回任何值，以防止赋值号被错用为等号，但同时也导致 `x = y = z` 是不合法的。
* 如果赋值的右边是一个元组，其元素可以被分解开来，如 `(x, y, _) = (1, 2, 3)`。

### 溢出

* 整数溢出会触发运行时错误，但如果要像 C 一样允许溢出，可以使用溢出运算符 `&+ &- &*`。[^overflow]

[^overflow]: 溢出运算符 `&/`、`&%` 在 [Swift 1.2](https://developer.apple.com/library/ios/releasenotes/DeveloperTools/RN-Xcode/Chapters/xc6_release_notes.html#//apple_ref/doc/uid/TP40001051-CH4-SW3) 中被移除。

### 求余

* 求余运算 `a % b` 的结果跟 `a` 的符号相同，而跟 `b` 的符号无关。这与 C / Java / Pascal 等语言是一致的，一般称这样的运算为**求余**（remainder）。
* 而 Python / Ruby 等语言 `%` 运算结果的符号只与 `b` 相同，一般称其为**求模**（modulo）。[^modulo]
* Swift 中也可以对浮点数进行求余运算。

[^modulo]: [Modulo operation - Wikipedia](https://en.wikipedia.org/wiki/Modulo_operation)

### 空合运算符（Nil Coalescing Operator）

```swift
a ?? b // a 必须是可选类型，b 要与 a 存储值的类型一致
a != nil ? a! : b
```

### 区间运算符

* 闭区间运算符 `a...b` 表示 [a, b]
* 半开区间运算符 `a..<b` 表示 [a, b)


## 字符串和字符

* Swift 的 String 是**值类型**，当其进行赋值操作或在函数中传递时会进行值拷贝。
* 字符串之间可以通过 `+` 连接，将字符连接到字符串尾部可以使用 `append()` 方法。
* 在字符串中插值可以使用 `"\()"`。

### Unicode

* 在字符串字面量中，**Unicode 标量值**（Unicode scalar value）可以表示为 `\u{n}`，其中 `n` 可以为 1-8 位的十六进制数。
* 目前 Unicode 编码共 21 位，标量值的范围包括：
  * **基本多语言平面**（Basic Multilingual Plane）：[`U+0000`, `U+D7FF`] ∪ [`U+E000`, `U+FFFF`]；
  * **辅助平面**（Supplementary Planes）：[`U+10000`, `U+10FFF`]；
  * 但不包括 UTF-16 **代理对**（surrogate pair）的码位：[`U+D800`, `U+DFFF`]。
* 分别可以通过字符串的 `utf8` / `utf16` / `unicodeScalars` 属性来访问其 UTF-8 / UTF-16 / Unicode Scalars 表示。
* 调用全局函数 `count()` 可以获得字符串中的字符数，但需注意 Swift 的字符类型表示一个**扩展字形集群**（extended grapheme cluster），例如一对 Unicode 标量 `"\u{65}\u{301}"` 与单个 Unicode 标量 `\u{E9}` 均表示单个字符 é。
* 而 NSString 其实是用 UTF-16 编码的码元（code units）组成的数组，相应地 `length` 属性的值是其包含的码元个数，而不是字符个数。[^unicode] 因此在 Swift 的 String 类型中这个属性名为 `utf16Count`。[^collectiontype]

[^unicode]: [NSString 与 Unicode - objc中国](http://objccn.io/issue-9-1/)

[^collectiontype]: 从 [Swift 2.0](https://developer.apple.com/swift/blog/?id=30) 开始字符串不再遵循 `CollectionType` 协议，这意味着它与 NSString 的实现不相一致。


## 集合类型

* 集合类型（collection types）包括数组（Array）、集合（Set）[^set] 和字典（Dictionary），其存储值类型必须相同，由泛型（generic）实现。
* 集合类型均由结构体实现，为**值类型**。
* 获取元素个数可访问其 `count` 属性。

[^set]: [Swift 1.2](https://developer.apple.com/library/ios/releasenotes/DeveloperTools/RN-Xcode/Chapters/xc6_release_notes.html#//apple_ref/doc/uid/TP40001051-CH4-SW6) 引入了原生的 `Set` 类型，与原先的 `NSSet` 桥接。

### 数组

* 数组类型可以表示为 `Array<SomeType>`，简写为 `[SomeType]`。
* 创建空数组可用 `[SomeType]()`，以重复的值创建数组可用 `Array(count:repeatedValue:)`。

### 集合

* 集合类型可以表示为 `Set<SomeType>`。`SomeType` 必须是可哈希的，即遵循 `Hashable` 协议。
* 创建空数组可用 `Set<SomeType>()`，可用数组字面量来初始化集合 `var groups: Set = ["AKB48", "SKE48", "NMB48", "HKT48"]`。
* 分别用 `insert(_:)` / `remove(_:)` / `contains(_:)` 方法来插入、移除、判断元素在集合中。
* `union(_:)` / `subtract(_:)` / `intersect(_:)` / `exclusiveOr(_:)` 方法分别表示并集、差集、交集、对称差。

### 字典

* 字典类型可以表示为 `Dictionary<Key, Value>`，简写为 `[Key: Value]`。`Key` 类型必须是可哈希的。
* 遍历字典可用 `for (key, value) in dict {...}` 或单独遍历 `dict.keys` 和 `dict.values`。


## 控制流

* 所有控制流不需要条件外侧的圆括号，但不可以省略语句体的花括号。

### 循环语句

* 若不需要知道循环变量的值，可用 `_` 代替变量名。
* 除了 `for-in` 循环，Swift 仍提供 C 样式 `for` 循环，三个表达式用分号隔开，但不需要加圆括号。`while` 和 `repeat-while`（原为 `do-while`，现 `do` 关键字被用于错误处理）循环仍然存在。

```swift
for _ in 0..<10 {
    statements
}
for var i = 0; i < 10; ++i {
    statements
}
```

### 条件语句

* `switch` 语句必须是完备的，即在各 `case` 分支不能涵盖所有情况时，最后要有 `default` 分支。如果能匹配多个 `case`，那么只会执行第一个匹配的分支。
* `switch` 不存在隐式的贯穿，即不需要在 `case` 分支中写 `break` 语句。
* 每个 `case` 必须包含至少一条语句，所以两个 `case` 连着写会编译错误。这时可以在单个 `case` 中把多个表达式用逗号分开（亦可分行写）。
* `case` 的表达式可以是区间或者元组，另可使用 `_` 来匹配所有可能的值。
* `case` 允许将匹配的值绑定到临时常量或变量（value bindings），以及使用 `where` 来判断额外条件。

```swift
switch somePoint {
case (0, 0):
    println("(0, 0) is at the origin")
case (_, 0):
    println("(\(somePoint.0), 0) is on the x-axis")
case (0, _):
    println("(0, \(somePoint.1)) is on the y-axis")
case (-2...2, -2...2):
    println("(\(somePoint.0), \(somePoint.1)) is inside the box")
case let (x, y) where x == y:
    println("(\(x), \(y)) is on the line x == y"
case let (x, y):
    println("(\(x), \(y)) is just some arbitrary point")
}
```

### 控制转移语句

* 可以在循环语句和 `switch` 语句前放置一个标签 `label:`，则可以用 `continue label` 或 `break label` 来跳过特定的循环。
* 在 `switch` 中可以用 `fallthrough` 继续执行下一个 `case` 的代码，这和 C 语言的特性相似。

### 提前退出

* `guard ... else {...}` 类似于只有 `else` 分支的 `if` 语句。[^guard]
* 如果条件满足则跳过花括号的内容，并且可选绑定的赋值对当前代码块的剩下部分依然有效。
* 如果条件不满足，`else` 分支必须退出当前代码块，譬如使用 `return`, `break`, `continue` 或抛出错误。

[^guard]: `guard` 语句于 [Swift 2.0](https://developer.apple.com/swift/blog/?id=29) 后被引进。

### 检查 API 可用性

* 在 `if` 或 `guard` 语句中可以判断当前平台版本（包括 `OSX`, `iOS` 和 `watchOS`），以验证 API 目前是否可用。[^available]
* 最后一个参数 `*` 表示在未指定的平台上，其版本与最低部署目标相同。

```swift
if #available(OSX 10.11, iOS 9, *) {
    // Use OS X El Capitan and iOS 9 APIs
} else {
    // Fall back to earlier OS X and iOS APIs
}
```

[^available]: `#availale()` 于 [Swift 2.0](https://developer.apple.com/swift/blog/?id=29) 后被引进。


## 函数

### 参数与返回值

* 无参函数在定义和调用时需要写一对空括号。
* 无返回值函数在定义时不需要写 `-> returnType`，实际上它返回了一个特殊的值 `Void`，这是一个空的元组即 `()`。
* 可以使用元组类型让函数返回多个值。

### 参数名称

```swift
func join(string s1:String, toString s2: String, joiner: String = " ") -> String {
    return s1 + joiner + s2
}
join(string: "hello", toString: "world", joiner: ", ")
```

* 上述代码中的 `s1` 等为**局部参数名**（local parameter name），在函数内部使用；`string` 等为**外部参数名**（external parameter name），在调用函数时使用，以加强可读性。
* 当两者相同时，可只写一次并加上 `#` 前缀。
* 当函数调用时 `joiner` 的值未被指定，函数会使用默认值。当未给带默认值的参数提供外部参数名时，局部名会自动作为外部名。
* 在参数类型后加入 `...` 可定义**可变参数**（variadic parameters），调用时可以传入不确定数量的参数，在函数内这将被当做这个类型的一个数组。
* 函数参数默认是常量，修改参数值会导致编译错误。若要将其当做可修改的副本使用，可在参数名前加上关键字 `var`。
* 如果需要修改参数在函数外的实际值，可以定义**输入输出参数**（in-out Parameters）。首先需要在参数前加关键字 `inout`，其次调用时传入的变量前要加 `&`。

### 函数类型

* 函数类型可以表示为诸如 `(Int) -> Int` 的形式，既无参数也无返回值的函数类型为 `() -> ()`。
* 函数在 Swift 中是一等公民（first-class citizen），可以作为参数类型和返回类型。
* 函数可以被定义在别的函数体内，这被称为**嵌套函数**（nested function）。嵌套函数对全局是不可见的，但可以被它的封闭函数（enclosing function）返回从而被外界使用。


## 闭包

* Swift 的闭包与 Objective-C 的代码块（blocks）以及其他语言的 lambdas 函数类似。Swift 的闭包是**引用类型**。
* 全局函数是一个有名字但不会捕获任何值的闭包。
* 嵌套函数是一个有名字并可以捕获其封闭函数域内值的闭包。
* 闭包表达式是一个利用轻量级语法所写的可以捕获其上下文中变量或常量值的匿名闭包。

```swift
// Closure expression syntax
reversed = sorted(names, { (s1: String, s2: String) -> Bool in
    return s1 > s2
})

// Inferring type from context
reversed = sorted(names, { s1, s2 in return s1 > s2 })

// Implicit return from single-expression closures
reversed = sorted(names, { s1, s2 in s1 > s2 })

// Shorthand argument names
reversed = sorted(names, { $0 > $1 })

// Trailing closures
reversed = sorted(names) { $0 > $1 }

//Operator functions
reversed = sorted(names, >)
```


## 枚举

* 枚举类型是**一等公民**，它采用了很多传统上只被类所支持的特性，例如实例方法、计算属性、遵守协议等。枚举定义的类型与 Swift 中其他类型一样，名字必须首字母大写。
* 与 C 不同，Swift 的枚举成员在被创建时不会被赋予一个默认的整数值。

### 相关值（Associated Values）

* 枚举可以存储任何类型的相关值，且每个成员的类型可以各不相同。

```swift
enum Barcode {
    case UPCA(Int, Int, Int)
    case QRCode(String)
}

var productBarcode = Barcode.UPCA(8, 85909_51226, 3)
productBarcode = .QRCode("ABCDEFGHIJKLMNOP")

switch productBarcode {
case let .UPCA(numberSystem, identifier, check):
    println("UPC-A with value \(numberSystem), \(identifier), \(check).")
case let .QRCode(productCode):
    println("QR Code with value of \(productCode).")
}
```

### 原始值（Raw Values）

* 枚举成员也可以被预先填充为同一类型的原始值，当整型被用于原始值时，如果其他枚举成员没有值整数会自动递增。
* 枚举成员的 `rawValue` 属性可以获取其原始值，而枚举的可失败构造器接受 `rawValue` 参数并返回一个可选类型。[^rawvalue]

[^rawvalue]: 原来的 `fromRaw()` / `toRaw()` 方法在 [Swift 1.1](https://developer.apple.com/library/ios/releasenotes/DeveloperTools/RN-Xcode/Chapters/xc6_release_notes.html#//apple_ref/doc/uid/TP40001051-CH4-DontLinkElementID_60) 中被现在的新语法取代。

```swift
enum Plant: Int {
    case Mercury = 1, Venus, Earth, Mars, Jupiter, Saturn, Uranus, Neptune
}

let earthsOrder = Planet.Earth.rawValue
let possiblePlanet = Planet(rawValue: 7)
```


> **\<Next\>** [Swift 学习笔记（二）](/swift-notes-2/)
