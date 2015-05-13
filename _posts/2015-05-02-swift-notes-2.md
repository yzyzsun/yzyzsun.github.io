---
layout: post
title: Swift 学习笔记（二）
category: Tech
---

## 类和结构体

* 类和结构体有以下共同点：属性（properties）、方法（methods）、下标（subscripts）、构造器（initializers）、扩展（extensions）、协议（protocols）。
* 另外类有以下独有特性：继承（inheritance）、类型转换（type casting）、析构器（deinitializers）、自动引用计数（ARC）。
* 类是**引用类型**，可通过 `===` / `!==` 判断是否同一实例；而结构体是**值类型**。[^valuetype]

[^valuetype]: [结构体和值类型 - objc中国](http://objccn.io/issue-16-2/)


## 属性

### 存储属性（Stored Properties）

* 存储属性即存储在类和结构体实例中的变量或常量，可以通过点语法（dot syntax）访问和赋值。
* 存储属性必须在定义时或是在构造器中被赋初值（常量也可在构造器中赋初值），否则应将其定义为可选类型。
* 如果创建了一个结构体的实例并赋给一个常量，则无法修改实例的任何属性，包括变量存储属性。而类为引用类型，故仍可修改。
* 在变量属性声明前使用 `lazy` 可以标记一个**延迟存储属性**，该属性将在第一次被访问时才会创建。

### 计算属性（Computed Properties）

* 计算属性不直接存储值，而是提供一个 getter 来获取值，一个可选的 setter 来间接设置其他属性的值，亦通过点语法来访问和赋值。
* 如果 `set` 没有提供参数名，则可以使用默认名称 `newValue`。
* 只读计算属性的声明可以去掉 `get` 关键字和花括号，但仍需要被声明为变量属性。

<!--more-->

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
        set {
            origin.x = newValue.x - (size.width / 2)
            origin.y = newValue.y - (size.height / 2)
        }
    }
}
```

### 属性观察器（Property Observers）

* 属性观察器包括两种：`willSet` 在设置新的值之前调用，`didSet` 在新的值被设置之后立即调用。
* 定义观察器的语法与 getter / setter 类似，`willSet` 默认参数名为 `newValue`，`didSet` 默认参数名为 `oldValue`。
* 当为存储属性设置默认值或是在构造器中为其赋值时，属性观察器不会被触发。

### 全局变量和局部变量

* 计算属性和属性观察器的语法也可用于全部和局部变量。
* 全局的常量和变量是**惰性求值**的，跟延迟存储属性相似；而局部变量是**及早求值**的。

### 类型属性（Type Properties）

* 实例属性属于特定的实例，各个实例之间的属性各自独立。而类型属性属于类型本身，不管有多少个实例该属性都是唯一的。
* 类型属性使用 `static` 关键字，值类型（结构体、枚举）可以定义存储属性和计算属性，但类只能定义计算属性。
* 在类中，可以使用 `class` 关键字以允许子类重写其类型属性。[^static]

[^static]: 从 [Swift 1.2](https://developer.apple.com/library/ios/releasenotes/DeveloperTools/RN-Xcode/Chapters/xc6_release_notes.html#//apple_ref/doc/uid/TP40001051-CH4-SW6) 开始，`static` 在类中被定义成了 `class final` 的同义词。


## 方法

### 实例方法

* 与函数不同，对于方法的第二个及后续参数，Swift 默认添加与局部参数名相同的外部参数名。同时也可以使用 `#` 为第一个参数添加外部参数名，或是使用 `_` 忽略外部参数名。
* 访问属性一般不必显式写出 `self`，除非方法的参数名与属性名重名时等情况。
* 一般情况下，值类型的属性不可以在实例方法中被修改。但可以在实例方法前加上关键字 `mutating`，这样便可以修改属性或是为 `self` 赋一个新实例，修改后的新实例将会自动取代原实例。

### 类型方法

* 类型方法的关键字与类型属性相同，为 `static` 和 `class`。


## 下标

* 可以在类、结构体和枚举中定义下标，即可通过 `[]` 访问和赋值。
* 与 getter 和 setter 相似，只读下标可以省略 `get` 和花括号。
* 下标的参数可以使用可变参数和变量参数，但不允许输入输出参数和设置参数默认值。

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

* 可以通过 `class subclass: superclass {...}` 来实现继承，子类将可以调用父类的属性、方法和下标。
* 子类可以使用关键字 `override` 重写父类的属性、方法和下标，重写版本的名称、类型（参数、返回值）要与被继承版本完全相同。[^overload] 重写时可以通过 `super` 来调用父类的属性、方法和下标。
* 可以将一个继承来的只读属性重写为读写属性，但反过来不行。
* 可以在属性重写中添加属性观察器，但不可以为常量属性或只读属性添加（不可同时添加 setter 和属性观察器）。
* 属性、方法和下标可以使用 `final` 关键字防止被重写，也可以用 `final class` 使类不可以被继承，否则会导致编译错误。

[^overload]: **重写**（override）与**重载**（overload）不同，重载的函数仅名称相同，参数彼此不同，不需要用关键字标识。


## 构造器

* Swift 会为每个构造参数自动生成一个跟内部名相同的外部名，可以用 `_` 覆盖。
* 对于所有属性已提供默认值且未定义构造器的结构体和基类，Swift 自动生成了一个无参的默认构造器。而满足此条件的结构体同时也有**逐一成员构造器**（memberwise initializers），如 `Monitor(width: 1440, height: 900)`。
* 可以通过一个立即执行的闭包来设置属性的默认值，如 `let property: Type = { return value }()`。

### 类的继承

* 类构造器分为**指定构造器**（designated initializers）和**便利构造器**（convenience initializers），每个类至少拥有一个指定构造器（可通过继承满足），它不需要关键字修饰，便利构造器则需要关键字 `convenience`。
* 指定构造器首先保证该类所有属性被初始化，该过程中不能调用任何实例方法、不能读取实例属性的值、也不能引用 `self`，初始化完成后必须向上调用父类构造器，接着为继承的属性赋予新值。
* 便利构造器必须先调用该类的其他构造器，再为属性赋予新值。
* 子类默认不会继承父类的构造器。但有两个例外：若子类没有定义任何指定构造器，它将自动继承所有父类的指定构造器；若子类提供了所有父类指定构造器的实现（可通过继承），它将自动继承所有父类的便利构造器。
* 使用关键字 `required` 可以标记一个**必要构造器**，子类必须实现该构造器（可通过继承），且不需要加关键字 `override`。

### 可失败构造器（Failable Initializers）

* `init?()` 或 `init!()` 可以定义一个可失败构造器，它将创建一个可选类型以应对初始化失败的情况。[^failable]
* 若要使初始化失败，可以在构造器内 `return nil`，不过类构造器需要在所有属性被赋初值后才能 `return nil`。
* 可失败构造器可以调用一个不可失败构造器，但反过来不行。
* 子类可以将父类的可失败构造器重写为不可失败构造器，但反过来不行。

[^failable]: 可失败构造器于 [Swift 1.1](https://developer.apple.com/swift/blog/?id=17) 后被引入。


## 析构器

* 当类的实例释放时如果需要进行额外的清理，可以用 `deinit {...}` 定义一个析构器。
* 不论子类是否定义了析构器，最后都会自动调用父类的析构器。


## 自动引用计数

* 将实例赋值给属性、常量或者变量时会建立一个**强引用**，强引用会阻止 ARC 释放实例，有时会导致强引用循环。
* 为解决强引用循环，可以使用关键字 `weak` 声明一个**弱引用**，或使用 `unowned` 声明**无主引用**，它们都不会阻止 ARC 释放实例。
* 弱引用必须是一个可选类型的变量，当其引用的实例被销毁后会被赋值为 `nil`。而无主引用的值不会被改变，但在实例被销毁后访问它会触发运行时错误。
* 为解决闭包和类实例之间的强引用循环，可以定义闭包的捕获列表。捕获列表形如 `[unowned self, weak instance]`，放置在闭包开始处。


## 类型转换

* 在用类实例初始化集合类型时，将自动推断出其类型为它们共同的父类。
* 用类型检查操作符 `is` 可以检查是否为某类型（或其父类）的实例，抑或是否遵循某协议。
* 使用 `as?` 或者 `as!` 可以尝试将实例向下转型（downcast）为原类型的子类。
* `AnyObject` 可以表示任何类（class）的实例，`Any` 可以表示任何类型（包括函数类型）的实例。
* 可以在 `switch` 语句中使用 `is` 和 `as` 来判断 `AnyObject` 或 `Any` 类型的值。

```swift
switch thing {
case 0 as Int:
   println("zero as an Int")
case 0 as Double:
   println("zero as a Double")
case let someDouble as Double where someDouble > 0:
   println("a positive double value of \(someDouble)")
case is Double:
   println("some other double value that I don't want to print")
case let stringConverter as String -> String:
   println(stringConverter("Michael"))
default:
   println("something else")
}
```


## 嵌套类型

* 可以在类型定义的花括号内嵌套定义类、结构体和枚举，甚至可以多级嵌套，访问时使用点语法。


## 扩展

* 扩展即向一个已有的类、结构体或枚举类型添加新功能，包括在没有权限获取源代码的情况下进行扩展（retroactive modeling）。扩展与 Objective-C 中的分类（category）相似，但不同的是扩展没有名字。使用 `extension Type: Protocol {...}` 来声明一个扩展。
* 扩展可以向已有类型添加计算型的实例属性和类型属性，但不可以添加存储属性或属性观察器。
* 对于所有属性已提供默认值且未定义构造器的结构体，扩展构造器时可以调用其默认构造器和成员逐一构造器；对于已有的类，可以添加新的便利构造器，但不可以添加指定构造器和析构器。
* 另外，扩展也可以添加新的实例方法、类型方法、下标和嵌套类型。


## 协议

* 协议用于声明其遵守者必须实现的属性、方法、下标、构造器等，协议通过 `protocol InheritingProtocol: SomeProtocol {...}` 定义，类通过 `class SomeClass: SuperClass, SomeProtocal, AnotherProtocol {...}` 来声明其遵守协议。
* 协议中声明的属性需要指定其只读还是可读写：在声明后加上 `{ get }` 表示只读，存储属性和计算属性都能满足要求；`{ get set }` 表示可读写，变量存储属性和读写计算属性可以满足要求。
* 协议可以作为类型使用，所有该协议遵守者的实例都符合该类型，亦可通过该协议类型的数组遍历调用协议规定的方法。
* **委托**（Delegation）是一种设计模式，它允许类或结构体将一些需要负责的功能委托给其他类型的实例，可以定义协议来封装这些需要被委托的方法。
* 可以在协议的继承列表前面加上 `class,` 来定义**类专属协议**，当试图让结构体或枚举适配该协议会导致编译错误。
* 可以使用 `protocol<SomeProtocol, AnotherProtocol>` 来将多个协议合成为一个临时协议。
* 可以在声明前加上 `optional` 以标识这是一个**可选协议要求**，该协议类型的实例调用时这个可选要求时，可以在其名称后加 `?` 检查它是否被实现，如 `optionalMethod?(args)`。[^protocol] 可选协议要求只能用于被 `@objc` 修饰的协议，且这样的协议只能被类遵守。

[^protocol]: [文档原文](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Protocols.html#//apple_ref/doc/uid/TP40014097-CH25-ID267)「Optional property requirements, and optional method requirements that return a value, will always return an optional value of the appropriate type when they are accessed or called, to reflect the fact that the optional requirement may not have been implemented.」疑有误，可选方法被调用后并不会返回一个可选类型，而是方法其本身是可选的，即非 `(Type) -> Type?` 而是 `((Type) -> Type)?`。


## 泛型

* 可以在函数、类、结构体和枚举名称后使用类型参数 `<T>` 来定义泛型函数和泛型类型。
* 如果需要限定泛型，可以为类型参数添加**类型约束**，同时可以在后面紧跟 `where` 语句为关联类型声明额外的约束，如 `<T: SomeProtocol, U: SomeClass where T.ItemType: Equatable>`。
* 当定义一个协议时，可以声明**关联类型**，以提供一个类型的占位名，如 `typealias ItemType`，我们不需要知道 `ItemType` 的实际类型是什么，在实现时它会被自动推断出来。


## 权限控制

* `public` 表示可以被任何源文件访问，即作为公开的 API。
* `internal` 表示只能被本模块（framework / app bundle）中的源文件访问，这是所有实体的**默认访问级别**。
* `private` 表示只能在当前源文件中使用，以隐藏某些功能的实现细节。
* 一个 `public` 类的所有成员默认为 `internal` 级别，以防止模块内部使用的实体被默认公开。
* 子类的访问级别不得高于父类，常量、变量和属性自身的级别不能高于其类型所设的级别。


## 运算符函数

### 运算符重载

* 二元运算符，或称**中置**（infix）运算符，重载时不需要关键字修饰，跟普通函数相似，如 `func + (left: Vector, right: Vector) -> Vector {...}`。
* 一元运算符分为**前置**运算符和**后置**运算符，分别需要用关键字 `prefix` 和 `postfix` 修饰。
* 组合赋值运算符重载时需要将左参数设置为 `inout`，而赋值号和三目条件运算符是不可重载的。

### 自定义运算符

* 首先需要声明一个自定义运算符，格式为 `*fix operator ∙ {}`，其中 `*` 代表 `in` / `pre` / `post`。定义时的语法与重载相同。
* 在声明的花括号中可以为中置运算符定义结合性和优先级，如 `{ associativity left precedence 150 }`。结合性可以是 `left` / `right` / `none`，默认值 `none` 表示它不能和同级运算符写在一起；优先级默认为 `100`，即与三目运算符同级。[^precedence]

[^precedence]: Swift 的默认优先级列表参见 [The Swift Programming Language: Expressions](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Expressions.html#//apple_ref/doc/uid/TP40014097-CH32-ID383)。

---

[\<Prev\> Swift 学习笔记（一）](/swift-notes-1/) | [Swift 学习笔记（三）\<Next\>](/swift-notes-3/)
