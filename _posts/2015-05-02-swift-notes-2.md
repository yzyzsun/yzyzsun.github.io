---
title: Swift 学习笔记（二）
author: 孙耀珠
tags: 编程语言
---

* 目录
{:toc}

## 结构体和类

- 结构体（Structure）和类（Class）有以下共同特性：属性（property）、方法（method）、下标（subscript）、构造器（initializer）、扩展（extension）、协议（protocol）；另外类还有以下独有特性：继承（inheritance）、类型转换（type casting）、析构器（deinitializer）、自动引用计数（ARC）。
- 结构体是**值类型**，当其进行赋值操作或参数传递时会发生值拷贝，当然编译器也会在此基础上做 copy-on-write 优化；而类是**引用类型**，相当于通过指针间接访问，因此不会发生值拷贝，但同时 `let` 关键字和参数传递时不再能保证其指向的内容不被修改。[^valuetype]
- 对于类的实例来说，相等不意味着恒等，我们可以通过 `===` / `!==` 来判断是不是同一个对象。

[^valuetype]: 对结构体和值类型的进一步讨论可以参见 [ObjC 中国](http://objccn.io/issue-16-2/)，对于是选用结构体还是类可以参见 [官方文档](https://developer.apple.com/documentation/swift/choosing_between_structures_and_classes)（优先选用结构体）。

<!--more-->

## 属性

### 存储属性

- 存储属性（Stored property）即存储在结构体或类实例中的变量或常量，其合并了 Objective-C 中属性（property）和实例变量（instance variable）的概念。
- 存储属性必须在定义中或是在构造器中被赋初值，否则应将其定义为可空类型。
- 如果一个结构体的实例是常量，则无法修改其任何属性；而类的实例常量仍可修改其变量属性。
- 使用 `lazy var` 可以定义一个**惰性存储属性**，该属性将在首次访问时才计算初值。因为实例方法在实例初始化完成前是不可调用的，所以使用惰性存储属性可以绕开这一规则延后调用实例方法为属性设初值。

### 计算属性

- 计算属性（Computed property）不直接存储值，而是提供一个 getter 和一个可选的 setter 来间接获取和设置其他属性的值。因为计算属性的值是不确定的，所以一定要声明为 `var`。
- 只读计算属性的定义可以省略 `get` 关键字直接写在第一层大括号里。
- 如果 `set` 没有提供参数名，则可使用默认名称 `newValue`。

```swift
struct AlternativeRect {
    var origin = Point()
    var size = Size()
    var center: Point {
        get {
            let centerX = origin.x + size.width / 2
            let centerY = origin.y + size.height / 2
            return Point(x: centerX, y: centerY)
        }
        set(newCenter) {
            origin.x = newCenter.x - size.width / 2
            origin.y = newCenter.y - size.height / 2
        }
    }
}
```

### 属性观察器

- 属性观察器（Property observer）可以监视存储属性值的变化，分为两种：一种是事先响应的 `willSet`，一种是事后响应的 `didSet`。
- 观察器的语法与 getter / setter 类似，`willSet` 默认参数名为 `newValue`，`didSet` 默认参数名为 `oldValue`。
- 在实例初始化阶段为属性设初值以及 ARC 将弱引用设为 `nil` 时不会触发属性观察器。

### 全局和局部变量

- 计算属性和属性观察器的语法也可用于全局和局部变量。
- 全局常量总是**惰性求值**的，而局部变量总是**及早求值**的。

### 类型属性

- 前面介绍的属性是实例属性，还有一种特殊的属性叫类型属性（type property）。实例属性存储在各个实例中，同一类型的不同实例之间属性值是各自独立的；而类型属性属于类型本身，不管有多少个实例该属性都是一样的。
- 类型属性可使用 `static` 关键字定义；对于类的计算属性，可将 `static` 改为 `class` 以允许子类重写其实现，`static` 就相当于 `final class`。
- 跟全局变量相似，类型属性也是惰性求值的，因此可以方便地创建单例。


## 方法

### 实例方法

- 在实例方法中访问属性和方法一般不必显式写出 `self.`，除非局部变量与属性重名、在逃逸闭包中访问属性和方法等情况。
- 值类型（结构体和枚举）默认不可以在实例方法中被修改，但可以在实例方法前加上 `mutating` 关键字，这样便可以修改属性或是直接为 `self` 赋一个新值，修改后的新实例将会自动取代原实例。对于一个值类型的常量，`mutating` 方法都是不可调用的。

### 类型方法

- 类型方法的机制与类型属性相同，关键字亦为 `static` 和 `class`。


## 下标

- 可以在类、结构体和枚举中定义下标，即可通过 `[]` 访问和赋值。
- 下标的语法与 getter 和 setter 类似，只读下标可以省略 `get` 关键字，`set` 默认参数名为 `newValue`。
- 下标的参数可以使用可变参数，但不允许输入输出参数或设置参数默认值。

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

- 可以通过 `class Subclass: Superclass { … }` 的语法来实现继承，Swift 不允许多重继承。
- 子类可以使用 `override` 关键字重写父类的属性、方法和下标，重写版本的方法签名（名称、参数标签、参数类型、返回值类型）应与父类版本完全相同。[^overload] 重写时可以通过 `super` 来访问父类的属性、方法和下标。
- 不论继承来的是存储属性还是计算属性，都可以用一个 getter 和一个可选的 setter 来重写。可以将一个继承来的只读属性重写为读写属性，但反过来不行。
- 不论继承来的是存储属性还是计算属性，都可以添加属性观察器。没有必要同时重写 setter 和添加观察器，因为观察器的代码都可以直接放在 setter 里。
- 属性、方法和下标可以使用 `final` 关键字防止被重写，也可以在整个类的定义前加 `final` 使其不可以被继承。

[^overload]: 重写（override）不是重载（overload），重载的方法仅名称和参数标签相同，通过参数类型进行重载决议，不需要用关键字标识，跟继承也没有必然联系。


## 类型转换

- 类型检查运算符 `is` 可以检查是否为某类型的实例，抑或是否遵循某协议。
- 类型转换运算符 `as` 可以进行编译器确保成功的类型转换，譬如从子类到父类的向上转型（upcasting）[^casting]、指定字面量的类型（如 `1 as Float`）、Swift 值类型与 Objective-C 类型的桥接转换（如 `string as NSString`）[^bridging] 等。
- 对于运行时才能确定是否成功的向下转型（downcasting）[^casting]，可以使用 `as?` 返回一个可空值或者使用 `as!` 强制解包。
- Swift 中有两个特殊的类型：`Any` 可以表示任何类型（type）的实例，`AnyObject` 可以表示任何类（class）的实例。

[^casting]: 这里的类型转换（casting）与形如 `Int(3.14)` 的类型转换（conversion）不同：前者只是向编译器重新描述了原对象的类型，而后者是依赖于构造器生成了一个新的对象。

[^bridging]: 在 Swift 3.0（[SE-0072](https://github.com/apple/swift-evolution/blob/master/proposals/0072-eliminate-implicit-bridging-conversions.md)）之前，这种桥接转换是隐式进行的，不需要 `as`。

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


## 构造器

- 对于所有属性均有初值且未自定义构造器的结构体和类，Swift 会自动生成一个无参的**默认构造器**。而所有未自定义构造器的结构体同时也有一个默认的**成员逐一构造器**（memberwise initializer），形如 `Monitor(width: 2560, height: 1600)` 来初始化所有属性。
- 对于初始化过程相对复杂的属性，除了自定义构造器之外，也可以通过一个立即执行的闭包（类似 JS 中的 IIFE）来赋值，形如 `let property: T = { return value }()`。

### 类的构造规则

![](https://docs.swift.org/swift-book/_images/initializerDelegation01_2x.png)

- 类的构造器分为两种：**指定构造器**（designated initializer）是主要的构造器，它负责初始化所有属性并调用父类的构造器以完成构造链，每个类至少拥有一个指定构造器；**便利构造器**（convenience initializer）是次要的构造器，它将委托指定构造器完成初始化，需要额外的 `convenience` 关键字标识。
  - 指定构造器第一步必须保证本类引入的所有属性被初始化，该过程中不能调用任何实例方法、不能读取实例属性的值、也不能引用 `self`；第二步必须向上调用父类构造器，自底向上完成构造链；第三步的时候实例已经初始化完成了，所以可进行任何自定义操作，这个过程是自顶向下即先父类后子类的。
  - 便利构造器必须先直接或间接调用本类的指定构造器，再进行任何自定义操作。
- 无论是把父类的指定构造器重写成子类的指定构造器还是便利构造器，都需要加上 `override` 关键字。另一方面，根据构造规则子类无法访问父类的便利构造器，因此即使跟父类的便利构造器方法签名相同严格来讲也不算重写，所以不需要加 `override`。
- 子类默认不会继承父类的构造器，但有两个例外：
  - 如果子类没有定义任何指定构造器，它将自动继承父类的所有指定构造器；
  - 如果子类提供了父类所有指定构造器的实现（可通过继承），它将自动继承父类的所有便利构造器。

```swift
class Person {
    var name: String
    init(name: String) {
        self.name = name
    }
    convenience init() {
        self.init(name: "Anonymous")
    }
}
class Idol: Person {
    var group: String
    init(name: String, group: String) {
        self.group = group
        super.init(name: name)
    }
    override convenience init(name: String) {
        self.init(name: name, group: "Solo")
    }
}
```

- 使用 `required` 关键字可以标识一个**必要构造器**，即子类必须实现这个构造器（可通过继承），且不需要加 `override` 关键字。

### 可失败构造器

- `init?` 或 `init!` 可以定义可失败构造器（failable initializer），它将创建一个可空值以应对初始化失败的情况。当无法完成初始化时，在构造器内 `return nil` 即可。
- 子类可以将父类的可失败构造器重写为非可失败构造器（对父类构造器创建的可空值强制解包），但反过来不行。


## 析构器

- 当类的实例释放时如果需要进行额外的清理，可以用 `deinit { … }` 定义一个析构器。
- 不论子类有没有定义析构器，最后都会自动调用父类的析构器。


## 自动引用计数

- Swift 使用引用计数（reference counting）进行类实例的内存管理，因为 Objective-C 2.0 以前需要手动增减计数，所以现在自动化的方式被称为自动引用计数（ARC）。不过相比于追踪式垃圾回收（tracing garbage collection），引用计数的一大问题在于互相持有引用的对象会形成环从而无法释放，因此 Swift 引入了强弱引用的概念。
  - 类实例的赋值默认会建立一个**强引用**，所有引用方式中只有强引用会阻止 ARC 释放资源；
  - 在变量声明前加上 `weak` 关键字表示一个**弱引用**，其类型必须是可空类型，当其引用的类实例被释放后会被赋值为 `nil`；
  - 在变量和常量声明前加上 `unowned` 关键字表示一个**非持有引用**，它不会被设为 `nil`，但在其引用的类实例被释放后访问会触发运行时错误。
- 当一个类拥有一个闭包类型的属性，且该闭包捕获了该类的实例时，也会形成一个环。为了解决这个问题，我们可以在闭包的参数表前面显式指定捕获列表，譬如 `[unowned self, weak delegate = self.delegate!]`。


## 协议

- 协议用于声明其遵循者必须实现的属性、方法、下标、构造器等要求，协议亦可以继承。声明一个子类遵循协议时，应当先写其父类再列出协议，形如 `class Subclass: Superclass, FirstProtocol, AnotherProtocol`。
  - 协议中声明的属性只需要指定其读写能力：在声明后加上 `{ get }` 表示可读，任何属性都能满足要求；`{ get set }` 表示可读写，只有变量存储属性和读写计算属性可以满足要求。
  - 如果希望值类型也能顺利遵循协议，则应在所有可能对实例进行修改的方法要求前加上 `mutating` 关键字，而引用类型不受此限制。
  - 非 `final` 类在实现构造器要求时，需要将其标识为 `required`。

```swift
protocol Togglable {
    var isOn: Bool { get }
    mutating func toggle()
}
```

- 如果不希望结构体或枚举遵循协议，可以让该协议继承于 `AnyObject`。
- 将协议作为类型使用时，可以用 `&` 来组合多个协议表示这些都遵循的类型。[^composition]
- 为了与 Objective-C 兼容，Swift 还可以声明可选要求，此时协议及其可选要求都必须以 `@objc` 修饰，且这样的协议只能被 `@objc` 类（包括继承于 Objective-C 类的类）遵循。[^optional] 协议的可选要求用 `optional` 关键字标识，若应用于方法要求则方法类型本身是可空类型，即 `((T) -> T)?`，调用这个方法时应在方法名和括号之间加 `?` 或 `!`。

[^composition]: 在 Swift 3.0（[SE-0095](https://github.com/apple/swift-evolution/blob/master/proposals/0095-any-as-existential.md)）之前，这被表示为 `protocol<P1, P2>`。

[^optional]: 如 [SE-0070](https://github.com/apple/swift-evolution/blob/master/proposals/0070-optional-requirements.md) 所述，可选要求能被两种 Swift 已有的特性取代：一是通过协议扩展来提供某些要求的默认实现，二是通过协议继承把某些要求拆分到另一个协议里去。因此保留可选要求只是为了与 Objective-C 兼容。


## 扩展

- Swift 的扩展（extension）与Objective-C 中的分类（category）类似，可以向一个已有的类、结构体和枚举添加新的功能，但扩展不能重写已有的功能。
  - 扩展可以添加计算属性，但不可以添加存储属性或向已有属性添加观察器。
  - 对于已有的类，扩展可以添加新的便利构造器，但不可以添加指定构造器和析构器。
  - 对于结构体，因为一旦自定义构造器就得不到默认的成员逐一构造器，因此可以把所有自定义构造器都挪到扩展中。
  - 用 `extension Type: Protocol { … }` 语法可以扩展 `Type` 遵循 `Protocol` 协议，这经常用于拆分一个类型对多个协议的实现，让代码更具可读性。
  - 另外，扩展也可以添加新的实例方法、类型方法、下标、嵌套类型。
- **协议扩展**是对一个已有的协议进行扩展，既可以为遵循该协议的类型添加计算属性、方法、下标、构造器的实现，也可以为已有的协议要求提供默认实现。

```swift
extension Collection where Element: Equatable {
    func allEqual() -> Bool {
        return reduce(true) { $0 && $1 == first }
    }
}
```


## 泛型

- 在函数和类型名称后加上类型参数 `<T>` 可以定义泛型函数和泛型类型，亦可以在此基础上添加**类型约束**，如 `<T: SomeClass, U: SomeProtocol>`。
- 协议有时也需要类似的泛型特性，为此我们可以使用**关联类型**（associated type），这些占位符的实际类型会在其遵循者实现协议要求时自动推断出来。
- 与泛型的类型约束类似，关联类型可以使用**泛型 `where` 从句**（generic `where` clause）进行约束，可以指定关联类型应遵循什么协议，或者与其他什么类型相同。

```swift
protocol Container {
    associatedtype Item
    mutating func append(_ item: Item)
    var count: Int { get }
    subscript(i: Int) -> Item { get }

    associatedtype Iterator: IteratorProtocol where Iterator.Element == Item
    func makeIterator() -> Iterator
}
protocol ComparableContainer: Container where Item: Comparable {}
```


## 错误处理

[^error]: Swift 2.0 以前的语法还不支持错误处理，对错误的处理跟 Objective-C 一样是传入 `NSError` 对象指针作为输出参数来进行的，相关讨论可以参见 [Swifter.tips](https://swifter.tips/error-handle/)。
[^unwinding]: Swift 错误处理的实现被称为隐式手动传播（implicit manual propagation），即由编译器生成手动传播错误（如 POSIX 的错误码、Objective-C 的错误参数等方式）的代码，对此的进一步讨论可以参见 [官方文档](https://github.com/apple/swift/blob/master/docs/ErrorHandlingRationale.rst#implementation-design)。

- 错误处理在 Swift 中拥有语言层面的支持  [^error]，不过与其他语言不同的是 Swift 错误处理的实现并非栈回退（stack unwinding）[^unwinding]，因此其开销相对较小。
- Swift 中的错误用遵循 `Error` 协议的对象表示，通常会使用枚举类型，抛出则用 `throw` 语句。
- Swift 中有 4 种处理错误的方式：将错误传播到调用本函数的代码、用 `do-catch` 处理错误、将错误视为可空值、断定错误不会发生。因为抛出错误会改变程序的流程，所以应该在所有可能抛出错误的函数、方法和构造器的调用前加上 `try` 关键字。
  - 对于一个会传播错误的函数，需要在参数列表和返回值的箭头之间加上 `throws` 关键字，这样才能在函数中自由使用 `throw` 和 `try`。
  - 用 `do-catch` 语句可以处理一大块代码的错误，`catch` 对错误的捕获类似模式匹配，默认分支的错误对象将绑定到 `error`，未匹配的错误将继续向外传播。错误一旦抛出，`do` 中剩下的代码将不再执行，`catch` 捕获并处理完错误后接着执行整个 `do-catch` 后面的代码。
  - 将 `try` 替换为 `try?` 可将结果转换成可空值，一旦出错则结果为 `nil`。
  - 将 `try` 替换为 `try!` 可以无视错误强制执行，一旦出错将触发运行时错误。

```swift
do {
    try shakeHands(with: "Nanase Nishino", in: "Nogizaka46")
    print("I have just shaken hands with my favorite idol!")
} catch HandshakeError.noTicket {
    print("You do not have a handshake ticket.")
} catch HandshakeError.incorrectGroup(let correctGroup) {
    print("The group you specify is incorrect. She is a member of \(correctGroup) instead.")
} catch {
    print("Unexpected error: \(error).")
}
```

- 使用 `defer` 语句可以确保在离开当前代码块前执行特定的清理操作，包括抛出错误或是 `return` / `break` 的情况。如果一个代码块有多个 `defer` 语句，则它们最后会被逆序执行。

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

### 断言

- 在调试中我们常常希望检查某些条件是否成立，但又没有必要为此抛出错误，这时我们可以使用断言。断言函数的代码只会在 `Debug` 编译配置下执行，其签名如下：

```swift
func assert(_ condition: @autoclosure () -> Bool,
              _ message: @autoclosure () -> String = String(),
                   file: StaticString = #file,
                   line: UInt = #line)
```

- 其中 `@autoclosure` 能够将一句表达式自动包装成一个无参的闭包，使用该特性能够延迟表达式（即闭包返回值）的计算，譬如 `Release` 编译配置下就不会进行计算。[^autoclosure]
- `#file` / `#line` 相当于 C 语言中的宏 `__FILE__` / `__LINE__`，其名始于 Swift 2.2（[SE-0028](https://github.com/apple/swift-evolution/blob/master/proposals/0028-modernizing-debug-identifiers.md)）。

[^autoclosure]: [Building assert() in Swift, Part 1: Lazy Evaluation — Swift Blog, Apple Developer](https://developer.apple.com/swift/blog/?id=4)

### 致命错误

- 如果在某些场景下  [^fatalerror] 遇到不可恢复的错误需要立即中止程序，我们可以使用 `fatalError` 函数，其签名如下：

```swift
func fatalError(_ message: @autoclosure () -> String = String(),
                     file: StaticString = #file,
                     line: UInt = #line) -> Never
```

- 返回类型 `Never` 表示这个函数一定不会正常返回，Swift 3.0（[SE-0102](https://github.com/apple/swift-evolution/blob/master/proposals/0102-noreturn-bottom-type.md)）之前是在函数上用 `@noreturn` 属性来标识。

[^fatalerror]: [fatalError — Swifter.tips](http://swifter.tips/fatalerror/)

## 访问控制

- Swift 的访问控制模型基于**模块**和**源文件**两个概念，其中模块是指代码分发的独立单元，如一款应用或一个框架。Swift 目前共有 5 种访问级别，从高（宽松）到低（严格）分别为：[^levels]
  - `open` 表示可以被任何模块的源文件访问，并可以在任何地方被继承和重写，只适用于类和可被重写的类成员。
  - `public` 表示可以被任何模块的源文件访问，但只能在本模块内被继承和重写。
  - `internal` 表示只能被本模块内的源文件访问，这是**默认访问级别**。
  - `fileprivate` 表示只在当前源文件中可见。
  - `private` 表示只在其外层声明和当前源文件中该声明的扩展内可见。
- Swift 的访问级别遵循一个基本原则：一个实体不能基于更低访问级别的实体来定义，譬如变量自身的访问级别不能高于其类型、函数的访问级别不能高于其参数或返回值的类型等等。另外有一些规则需要特别注意：
  - 自定义类的成员的访问级别默认与类相同，但 `open` / `public` 类的成员默认还是 `internal` 级别，以防公开 API 的内部接口也被默认公开。
  - 子类的访问级别不得高于父类，但子类重写的成员可以拥有比父类版本更高的访问级别。
- 可以为 setter 指定比 getter 更低的访问级别，譬如 `public private(set) var`。
- 只要在被测试模块的编译选项中开启测试，并使用 `@testable import` 来导入模块，则可以在单元测试中访问所有 `internal` 级别的实体。

[^levels]: Swift 一开始只有 3 种访问级别：`public` / `internal` / `private`，其中 `public` 与现在的 `open` 含义相同，`private` 与现在的 `fileprivate` 含义相同。Swift 3.0 时 [SE-0117](https://github.com/apple/swift-evolution/blob/master/proposals/0117-non-public-subclassable-by-default.md) 和 [SE-0025](https://github.com/apple/swift-evolution/blob/master/proposals/0025-scoped-access-level.md) 分别引入了 `open` 和 `fileprivate`，Swift 4.0 时 [SE-0169](https://github.com/apple/swift-evolution/blob/master/proposals/0169-improve-interaction-between-private-declarations-and-extensions.md) 将 `private` 的可见范围延伸到了扩展。


## 运算符

| Operator                                                | Associativity | Precedence Group   |
| ------------------------------------------------------- | ------------- | ------------------ |
| `!` `~` `+` `-`                                         | -             | (prefix)           |
| `<<` `>>`                                               | -             | BitwiseShift       |
| `*` `/` `%` `&*` `&`                                    | Left          | Multiplication     |
| `+` `-` `&+` `&-` `|` `^`                               | Left          | Addition           |
| `..<` `...`                                             | -             | RangeFormation     |
| `is` `as` `as?` `as!`                                   | Left          | Casting            |
| `??`                                                    | Right         | NilCoalescing      |
| `<` `<=` `>` `>=` `==` `!=` `===` `!==` `~=`            | -             | Comparison         |
| `&&`                                                    | Left          | LogicalConjunction |
| `||`                                                    | Left          | LogicalDisjunction |
| `?:`                                                    | Right         | Ternary            |
| `=` `*=` `/=` `%=` `+=` `-=` `<<=` `>>=` `&=` `|=` `^=` | Right         | Assignment         |

### 运算符重载

- 双目运算符，或称中置运算符，实现的语法与普通函数或类型方法相同。[^operator]
- 单目运算符分为前置运算符和后置运算符，分别表示为 `prefix func` 和 `postfix func`。
- 现有运算符中 `=` 和 `?:` 不可被重载，另外复合赋值运算符的第一个参数应为 `inout`。

[^operator]: 从 Swift 3.0（[SE-0091](https://github.com/apple/swift-evolution/blob/master/proposals/0091-improving-operators-in-protocols.md)）开始，运算符不仅可以被定义为全局函数，还可以是类型方法，且后者更为提倡。

```swift
extension Vector2D {
    static func + (left: Vector2D, right: Vector2D) -> Vector2D {
        return Vector2D(x: left.x + right.x, y: left.y + right.y)
    }
}
```

### 自定义运算符

- 对于尚不存在的运算符，在实现之前需要首先进行声明，语法为 `{in,pre,post}fix operator ×: PrecedenceGroup`。
- 双目运算符的优先级和结合性都是由其声明的优先级组决定的，若不指定则优先级组默认为 `DefaultPrecedence`，这个组的优先级仅高于三目运算符，且没有结合性。[^precedence]
- 单目运算符没有优先级组的概念，优先级：后置 > 前置 > 中置。

[^precedence]: 在 Swift 3.0（[SE-0077](https://github.com/apple/swift-evolution/blob/master/proposals/0077-operator-precedence.md)）之前，运算符优先级是用整数表示的，譬如默认优先级是三目运算符的 `100`。例程中的 `**` 运算符，按照原来的语法会声明为 `infix operator ** { precedence 160 associativity right }`。

```swift
precedencegroup ExponentiationPrecedence {
    associativity: right
    higherThan: MultiplicationPrecedence
}
infix operator **: ExponentiationPrecedence
extension Double {
    static func ** (left: Double, right: Double) -> Double {
        return pow(left, right)
    }
}
```


> **\<Prev\>** [Swift 学习笔记（一）](/swift-notes-1/)  
> **\<Next\>** [Swift 学习笔记（三）](/swift-notes-3/)
