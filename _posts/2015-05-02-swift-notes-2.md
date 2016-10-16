---
layout: post
title: Swift 学习笔记（二）
category: Tech
author: 孙耀珠
---

## 类和结构体

- 类和结构体有以下共同点：属性（properties）、方法（methods）、下标（subscripts）、构造器（initializers）、扩展（extensions）、协议（protocols）。
- 另外类有以下独有特性：继承（inheritance）、类型转换（type casting）、析构器（deinitializers）、自动引用计数（ARC）。
- 类是**引用类型**，可通过 `===` / `!==` 判断是否同一实例；而结构体是**值类型**。[^valuetype]

[^valuetype]: [结构体和值类型 - objc中国](http://objccn.io/issue-16-2/)


## 属性

### 存储属性（Stored Properties）

- 存储属性即存储在类和结构体实例中的变量或常量，可以通过点语法（dot syntax）访问和赋值。
- 存储属性必须在定义时或是在构造器中被赋初值（常量也可在构造器中赋初值），否则应将其定义为可选类型。
- 如果创建了一个结构体的实例并赋给一个常量，则无法修改实例的任何属性，包括变量存储属性。而类为引用类型，故仍可修改。
- 在变量属性声明前使用 `lazy var` 可以标记一个**延迟存储属性**，该属性将在第一次被访问时才会创建。因为实例方法在初始化完成前是不可被调用的，所以使用 `lazy` 可以避开这一规则调用实例方法为属性设初始值。

<!--more-->

### 计算属性（Computed Properties）

- 计算属性不直接存储值，而是提供一个 getter 来获取值，一个可选的 setter 来间接设置其他属性的值，亦通过点语法来访问和赋值。
- 如果 `set` 没有提供参数名，则可以使用默认名称 `newValue`。
- 只读计算属性的声明可以去掉 `get` 关键字和花括号，但仍需要被声明为变量属性。

```swift
struct AlternativeRect {
    var origin = Point()
    var size = Size()
    var center: Point {
        get {
            let centerX = origin.x + (size.width / 2)
            let centerY = origin.y + (size.height / 2)
            return Point(x: centerX, y: centerY)
        }
        set(newCenter) {
            origin.x = newCenter.x - (size.width / 2)
            origin.y = newCenter.y - (size.height / 2)
        }
    }
}
```

### 属性观察器（Property Observers）

- 属性观察器包括两种：`willSet` 在设置新的值之前调用，`didSet` 在新的值被设置之后立即调用。
- 定义观察器的语法与 getter / setter 类似，`willSet` 默认参数名为 `newValue`，`didSet` 默认参数名为 `oldValue`。
- 当为存储属性设置默认值，或是于调用其他构造器前在构造器中为其赋值时，属性观察器不会被触发。

### 全局变量和局部变量

- 计算属性和属性观察器的语法也可用于全部和局部变量。
- 全局的常量和变量是**惰性求值**（lazy evaluation）的，跟延迟存储属性相似；而局部变量是**及早求值**（eager evaluation）的。

### 类属性（Type Properties）

- 实例属性从属于特定的实例，不同实例之间的属性值是各自独立的。而类属性属于类型本身，不管有多少个实例该属性都是唯一的。
- 与全局变量相似，类属性也是惰性求值的，因此可以用于创建单例。
- 类属性使用 `static` 关键字，值类型（结构体、枚举）可以定义存储属性和计算属性，但类只能定义计算属性。
- 在类中，可以使用 `class` 关键字以允许子类重写其类属性。[^static]

[^static]: 从 [Swift 1.2](https://developer.apple.com/library/ios/releasenotes/DeveloperTools/RN-Xcode/Chapters/xc6_release_notes.html#//apple_ref/doc/uid/TP40001051-CH4-SW6) 开始，`static` 在类中被定义成了 `class final` 的同义词。


## 方法

### 实例方法

- 访问属性一般不必显式写出 `self`，除非局部变量与属性重名、在逃逸闭包中访问属性等情况。
- 一般情况下，值类型的属性不可以在实例方法中被修改。但可以在实例方法前加上关键字 `mutating`，这样便可以修改属性或是为 `self` 赋一个新实例，修改后的新实例将会自动取代原实例。

### 类方法

- 类方法的关键字与类属性相同，为 `static` 和 `class`。


## 下标

- 可以在类、结构体和枚举中定义下标，即可通过 `[]` 访问和赋值。
- 与 getter 和 setter 相似，只读下标可以省略 `get` 和花括号。
- 下标的参数可以使用可变参数和变量参数，但不允许输入输出参数和设置参数默认值。

```swift
subscript(index: Int) -> Int {
    get {
        return values[index]
    }
    set {
        values[index] = newValue
    }
}
```


## 继承

- 可以通过 `class subclass: superclass { … }` 来实现继承，子类将可以调用父类的属性、方法和下标。
- 子类可以使用关键字 `override` 重写父类的属性、方法和下标，重写版本的名称、类型（参数、返回值）包括参数标签应与父类版本完全相同。[^overload] 重写时可以通过 `super` 来调用父类的属性、方法和下标。
- 提供新的 getter 和 setter 便可以重写存储和计算属性。可以将一个继承来的只读属性（包括存储属性）重写为读写属性，但反过来不行。
- 可以在属性重写中添加属性观察器，但不可以为常量属性或只读属性添加（不可同时添加 setter 和属性观察器）。
- 属性、方法和下标可以使用 `final` 关键字防止被重写，也可以用 `final class` 使类不可以被继承，否则会导致编译错误。

[^overload]: **重写**（override）与**重载**（overload）不同，重载的方法仅名称相同，而方法签名（method signature）不同，不需要用关键字标识。


## 构造器

- 对于所有属性已提供默认值且未定义构造器的结构体和基类，Swift 自动生成了一个无参的默认构造器。而满足此条件的结构体同时也有**逐一成员构造器**（memberwise initializers），如 `Monitor(width: 1440, height: 900)`。
- 可以通过一个立即执行的闭包来设置属性的默认值，如 `let property: Type = { return value }()`。

### 类的继承

- 类构造器分为**指定构造器**（designated initializers）和**便利构造器**（convenience initializers），每个类至少拥有一个指定构造器（可通过继承满足），它不需要关键字修饰，便利构造器则需要关键字 `convenience`。
- 指定构造器首先保证该类所有属性被初始化，该过程中不能调用任何实例方法、不能读取实例属性的值、也不能引用 `self`，初始化完成后必须向上调用父类构造器，接着为继承的属性赋予新值。
- 便利构造器必须先直接或间接调用该类的指定构造器，再为属性赋予新值。
- 子类默认不会继承父类的构造器。但有两个例外：若子类没有定义任何指定构造器，它将自动继承所有父类的指定构造器；若子类提供了所有父类指定构造器的实现（可通过继承），它将自动继承所有父类的便利构造器。
- 使用关键字 `required` 可以标记一个**必要构造器**，子类必须实现该构造器（可通过继承），且不需要加关键字 `override`。

### 可失败构造器（Failable Initializers）[^failable]

[^failable]: 可失败构造器于 [Swift 1.1](https://developer.apple.com/swift/blog/?id=17) 后被引入。

- `init?()` 或 `init!()` 可以定义一个可失败构造器，它将创建一个可选类型以应对初始化失败的情况。
- 若要使初始化失败，可以在构造器内 `return nil`，不过类构造器需要在所有属性被赋初值后才能 `return nil`。
- 可失败构造器可以调用一个不可失败构造器，但反过来不行。
- 子类可以将父类的可失败构造器重写为不可失败构造器，但反过来不行。

## 析构器

- 当类的实例释放时如果需要进行额外的清理，可以用 `deinit { … }` 定义一个析构器。
- 不论子类是否定义了析构器，最后都会自动调用父类的析构器。


## 自动引用计数

- 将类的实例赋值给属性、常量或者变量时会建立一个**强引用**，强引用会阻止 ARC 释放实例，有时会导致强引用循环。
- 为解决强引用循环，可以使用关键字 `weak` 声明一个**弱引用**，或使用 `unowned` 声明**无主引用**，它们都不会阻止 ARC 释放实例。
- 弱引用必须是一个可选类型的变量，当其引用的实例被销毁后会被赋值为 `nil`。而无主引用的值不会被改变，但在实例被销毁后访问它会触发运行时错误。
- 为解决闭包和实例之间的强引用循环，可以定义闭包的捕获列表。捕获列表形如 `[unowned self, weak instance]`，放在闭包的参数表前面。

## 错误处理 [^error]

[^error]: 错误处理于 [Swift 2.0](https://developer.apple.com/swift/blog/?id=29) 后被引入。

- 在 Swift 中，可以使用遵循 `Error` 协议的枚举类型来定义一系列错误。
- 对于一个可能抛出错误的函数或方法，需要在参数列表和返回值之间加上 `throws` 关键字，这样便可以在函数内使用 `throw` 语句。
- 当调用一个可能抛出错误的函数时，需要在前面加上 `try` 关键字；也可以使用 `try?` 将该函数转换为一个返回可选类型的函数，这将不再抛出错误而是返回 `nil`；或用 `try!` 来绕过错误处理强制执行，这时出错将直接触发运行时错误。
- 可以使用 `do-catch` 处理异常，`catch` 对错误的匹配类似于 `case`。错误一旦抛出，`do` 语句块中剩下的语句将不再执行，`catch` 捕获并处理完异常后直接执行整个 `do-catch` 语句块下面的代码。

```swift
do {
    try shakeHands(with: "Nanase Nishino", in: "Nogizaka 46")
} catch HandshakeError.noTicket {
    print("You do not have a handshake ticket.")
} catch HandshakeError.incorrectGroup(let correctGroup) {
    print("The group you specify is incorrect. She is a member of group \(correctGroup) instead.")
} catch {
    print("Another error happens.")
}
```

- 使用 `defer` 可以确保在离开当前代码块前执行特定语句，不论是因为抛出了错误，还是 `return` 或 `break`。且最后一个 `defer` 语句块将最先被执行，以此类推逆序执行。

```swift
func processFile(_ filename: String) throws {
    if exists(filename) {
        let file = open(filename)
        defer {
            close(file)
        }
        while let line = try file.readline() {
            // Work with the file.
        }
        // `close(file)` is called here, at the end of the scope.
    }
}
```

- 如果一个函数或方法只有在它的函数参数抛出错误时才抛出错误，则其声明时可以使用 `rethrows` 关键字。在这样的函数中，只能在 `catch` 的语句块中才能 `throw`，且 `catch` 只允许处理其函数参数抛出的错误。
- `rethrows` 方法可以重写 `throws` 方法，`rethrows` 方法也可以满足协议中 `throws` 方法的要求；但反过来不行。 

### 断言

- 在调试中我们常常希望检查某些条件是否成立，但又没有必要为此抛出错误，这时我们可以使用断言。断言只会在 Debug 编译时有效，其签名如下：

```swift
func assert(_ condition: @autoclosure () -> Bool,
              _ message: @autoclosure () -> String = default,
                   file: StaticString = #file,
                   line: UInt = #line)
```

- 其中 `@autoclosure` 能够将一句表达式自动封装成一个闭包，但此闭包必须是无参的。使用该特性能够延迟表达式（即闭包返回值）的计算。[^autoclosure]
- `#file` / `#line` 相当于 C 语言中的宏 `__FILE__` / `__FILE__`，在 [Swift 2.2](https://github.com/apple/swift-evolution/blob/master/proposals/0028-modernizing-debug-identifiers.md) 中启用现在的名字。

[^autoclosure]: [@autoclosure 和 ?? - Swifter.tips](http://swifter.tips/autoclosure/)

### Fatal Error

- 因为断言在 Release 编译时无效，因此如果遇到致命错误需要立即中止程序，我们可以使用 `fatalError` 函数，其签名如下：

```swift
func fatalError(_ message: @autoclosure () -> String = default,
                     file: StaticString = #file,
                     line: UInt = #line) -> Never
```

- 返回值 `Never` 表示这个函数一定不会正常返回。[^noreturn]
- 该函数可能的使用场景为：我们在父类中定义了一个抽象方法，需要交由子类去实现，为了防止程序错误调用父类未实现的方法，我们可以在此处使用 `fatalError`。[^fatalerror]

[^noreturn]: 在 [Swift 3.0](https://github.com/apple/swift-evolution/blob/master/proposals/0102-noreturn-bottom-type.md) 之前，这种行为是以 `@noreturn` 属性表示的。

[^fatalerror]: [fatalError - Swifter.tips](http://swifter.tips/fatalerror/)


## 类型转换

- 在用实例初始化集合类型时，将自动推断出其类型为它们共同的父类。
- 用类型检查操作符 `is` 可以检查是否为某类型（或其父类）的实例，抑或是否遵循某协议。
- 使用 `as?` 或者 `as!` 可以尝试将实例**向下转型**（downcast）为原类型的子类。
- `as` 亦可用于 Swift 与 Objective-C 桥接类型之间的相互转换，如 `String` 与 `NSString`。[^coercion] 这样的类型转换是绝对安全的，`as` 也不必加感叹号。
- `AnyObject` 可以表示任何类（class）的实例，`Any` 可以表示任何类型（包括函数类型）的实例。
- 可以在 `switch` 语句中使用 `is` 和 `as` 来判断类型。

[^coercion]: 这里的类型转换（casting）与形如 `String(str)` 的强制转换（coercion）不同，后者是依赖于构造器生成了一个新的对象，而前者只是向编译器重新描述了原对象的类型。

```swift
switch thing {
case 0 as Int:
   print("zero as an Int")
case 0 as Double:
   print("zero as a Double")
case let someDouble as Double where someDouble > 0:
   print("a positive double value of \(someDouble)")
case is Double:
   print("some other double value that I don't want to print")
case let stringConverter as (String) -> String:
   print(stringConverter("Michael"))
default:
   print("something else")
}
```

## 嵌套类型

- 可以在类型定义的花括号内嵌套定义类、结构体和枚举，甚至可以多级嵌套，访问时使用点语法。


## 扩展

- 扩展即向一个已有的类、结构体、枚举或协议添加新功能，包括在没有权限获取源代码的情况下进行扩展（retroactive modeling）。扩展与 Objective-C 中的分类（category）相似，但不同的是扩展没有名字。使用 `extension Type: Protocol { … }` 来声明一个扩展。
- 扩展可以向已有类型添加计算型的实例属性和类属性，但不可以添加存储属性或属性观察器。
- 对于所有属性已提供默认值且未定义构造器的结构体，扩展构造器时可以调用其默认构造器和成员逐一构造器；对于已有的类，可以添加新的便利构造器，但不可以添加指定构造器和析构器。
- 另外，扩展也可以添加新的实例方法、类方法、下标和嵌套类型。


## 协议

- 协议用于声明其遵守者必须实现的属性、方法、下标、构造器等，协议通过 `protocol InheritingProtocol: SomeProtocol { … }` 定义，类通过 `class SomeClass: SuperClass, SomeProtocal, AnotherProtocol { … }` 来声明其遵守协议。
- 协议中声明的属性需要指定其只读还是可读写：在声明后加上 `{ get }` 表示只读，存储属性和计算属性都能满足要求；`{ get set }` 表示可读写，变量存储属性和读写计算属性可以满足要求。
- 协议可以作为类型使用，所有该协议遵守者的实例都符合该类型，亦可通过该协议类型的数组遍历调用协议规定的方法。
- **委托**（Delegation）是一种设计模式，它允许类或结构体将一些需要负责的功能委托给其他类型的实例，可以定义协议来封装这些需要被委托的方法。
- 可以在协议的继承列表前面加上 `class,` 来定义**类专属协议**，当试图让结构体或枚举适配该协议会导致编译错误。
- 可以使用 `SomeProtocol & AnotherProtocol` 来组合多个协议。[^protocol]
- 为了与 Objective-C 兼容，Swift 提供了**可选协议要求**，此时协议及其可选要求都必须以 `@objc` 修饰，且这样的协议只能被类遵守。[^optional] 协议的可选要求用 `optional` 关键字标注，若用于方法，此时方法本身是可选类型，即 `((Type) -> Type)?`；调用时这个可选方法时，可以在其名称后加 `?` 检查它是否被实现，如 `optionalMethod?(args)`。
- 通过**协议扩展**不仅能够添加协议的一般功能，还可以提供协议的默认实现，并可使用 `where` 关键字对扩展进行约束。[^extension]

[^protocol]: 在 [Swift 3.0](https://github.com/apple/swift-evolution/blob/master/proposals/0095-any-as-existential.md) 之前，这被表示为 `protocol<SomeProtocol, AnotherProtocol>`。

[^optional]: 实际上，使用协议扩展来提供协议的默认实现可以取代可选协议要求，所以保留可选协议要求只是为了与 Objective-C 兼容。参考：[Swift Evolution SE-0070](https://github.com/apple/swift-evolution/blob/master/proposals/0070-optional-requirements.md)

[^extension]: 协议扩展于 [Swift 2.0](https://developer.apple.com/swift/blog/?id=29) 后被引入。

```swift
extension Collection where Iterator.Element: TextRepresentable {
    var textualDescription: String {
        let itemsAsText = self.map { $0.textualDescription }
        return "[" + itemsAsText.joined(separator: ",") + "]"
    }
}
```


## 泛型

- 可以在函数、类、结构体和枚举名称后使用类型参数 `<T>` 来定义泛型函数和泛型类型。
- 如果需要限定泛型，可以为类型参数添加**类型约束**，同时可以在声明的最后用 `where` 语句为关联类型声明额外的约束，如 `func f<T: SomeClass, U: SomeProtocol>(_ x: T, _ y: U) where T.ItemType: Equatable`。
- 当定义一个协议时，可以声明**关联类型**，以提供一个类型的占位名，如 `associatedtype ItemType` [^associatedtype]，我们不需要知道 `ItemType` 的实际类型是什么，在实现时它会被自动推断出来。

[^associatedtype]: 在 [Swift 2.2](https://github.com/apple/swift-evolution/blob/master/proposals/0011-replace-typealias-associated.md) 以前，关联类型借用的是 `typealias` 关键字，现在 `typealias` 只用于类型别名。

## 权限控制

- `open` 表示可以被任何源文件访问，并且可以在任何地方被继承和重写。[^open]
- `public` 表示可以被任何源文件访问，但只能在本模块（app bundle / framework）内被继承和重写。
- `internal` 表示只能被本模块中的源文件访问，这是所有实体的**默认访问级别**。
- `fileprivate` 表示只在当前源文件中可见。[^fileprivate]
- `private` 表示只在声明的作用域中可见。
- 变量自身的访问级别不能高于其类型所设的级别，函数的级别不能高于其参数类型或返回值类型的级别。
- 一个 `public` 类的所有成员默认为 `internal` 级别，以防止模块内部使用的实体被默认公开。
- 子类本身的访问级别不得高于父类，但子类重写的成员可以拥有比原来更高的访问级别。
- 可以为 setter 指定比 getter 更低的访问级别，譬如 `public private(set) var value`。
- 使用 `@testable import ModuleName` 可以让单元测试能够访问所有 `internal` 实体。

[^open]: 于 [Swift 3.0 (SE-0117)](https://github.com/apple/swift-evolution/blob/master/proposals/0117-non-public-subclassable-by-default.md) 加入，行为与以前的 `public` 相同。

[^fileprivate]: 于 [Swift 3.0 (SE-0025)](https://github.com/apple/swift-evolution/blob/master/proposals/0025-scoped-access-level.md) 加入，行为与以前的 `private` 相同。

## 运算符

| Operator                                 | Associativity | Precedence Group    |
| ---------------------------------------- | ------------- | ------------------- |
| `!`  `~`  `+`  `-`  (Prefix)             | -             | -                   |
| `<<`  `>>`                               | -             | Bitwise shift       |
| `*`  `/`  `%`  `&*`  `&`                 | Left          | Multiplication      |
| `+`  `-`  `&+`  `&-`  `|`  `^`           | Left          | Addition            |
| `..<`  `...`                             | -             | Range formation     |
| `is`  `as`  `as?`  `as!`                 | Left          | Casting             |
| `??`                                     | Right         | Nil coalescing      |
| `<`  `<=`  `>`  `>=`  `==`  `!=`  `===`  `!==`  `~=` | -             | Comparison          |
| `&&`                                     | Left          | Logical conjunction |
| `||`                                     | Left          | Logical disjunction |
| `?:`                                     | Right         | Ternary             |
| `=`  `*=`  `/=`  `%=`  `+=`  `-=`  `<<=`  `>>=`  `&=`  `|=`  `^=` | Right         | Assignment          |

### 运算符重载

- 二元运算符，或称**中置**（infix）运算符，重载时不需要关键字修饰，跟普通函数相似，如 `func + (left: Vector2D, right: Vector2D) -> Vector2D { … }`。[^operator]
- 一元运算符分为**前置**运算符和**后置**运算符，分别 `prefix func` 和 `postfix func` 修饰。
- 组合赋值运算符重载时需要将左参数设置为 `inout`，而 `=` 和 `?:` 是不可重载的。

[^operator]: 从 [Swift 3.0](https://github.com/apple/swift-evolution/blob/master/proposals/0091-improving-operators-in-protocols.md) 开始，运算符的实现不仅可以是全局函数，还可以是 `static func`，且后者更为提倡。

### 自定义运算符

- 首先需要声明一个自定义运算符，格式为 `{in,pre,post}fix operator ×: PrecedenceGroup`，实现这个运算符的语法与重载相同。
- 运算符的优先级和结合性是由它的优先级组决定的，若不指定优先级组则其属于 `DefaultPrecedence`，这个组的优先级仅高于三目运算符，且没有结合性，即它不能和同组运算符写在一起。[^precedence]

[^precedence]: 在 [Swift 3.0](https://github.com/apple/swift-evolution/blob/master/proposals/0077-operator-precedence.md) 之前，运算符优先级是用整数表示，譬如默认优先级是三目运算符的 `100`。

```swift
infix operator +-: AdditionPrecedence

extension Vector2D {
    static func +- (left: Vector2D, right: Vector2D) -> Vector2D {
        return Vector2D(x: left.x + right.x, y: left.y - right.y)
    }
}
```


> **\<Prev\>** [Swift 学习笔记（一）](/swift-notes-1/)  
> **\<Next\>** [Swift 学习笔记（三）](/swift-notes-3/)
