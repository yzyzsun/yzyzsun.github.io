---
title: Swift 学习笔记（三）
author: 孙耀珠
tags: 编程语言
---

* 目录
{:toc}

## Objective-C API

为了让 Swift 继承 Objective-C 的成熟生态，苹果开发了 Clang Importer 将 Objective-C API 引入 Swift，并自动进行了若干命名转换。而 Swift 3 更是对命名原则做了大规模的调整，使之更清晰、更适应于 Swift，详见 [SE-0005](https://github.com/apple/swift-evolution/blob/master/proposals/0005-objective-c-name-translation.md)、[SE-0006](https://github.com/apple/swift-evolution/blob/master/proposals/0006-apply-api-guidelines-to-the-standard-library.md) 两份提案，并且最终形成了一份 [API Design Guidelines](https://swift.org/documentation/api-design-guidelines/)。

### 初始化

- 使用 Swift 调用 Objective-C 类的构造器时，方法名中的 `init` 和 `initWith` 前缀会被截去，其余各部分依次变为构造器的参数名。`alloc` 方法不必再手动调用，Swift 会自己处理内存分配。
- 简洁起见，Objective-C 类的工厂方法也被映射成了 Swift 的便利构造器。
- 在 Objective-C 中可能返回 `nil` 的构造器，在 Swift 中会被映射为可失败构造器。

```objc
// Objective-C
UITableView *myTableView = [[UITableView alloc] initWithFrame:CGRectZero style:UITableViewStyleGrouped];
UIColor *color = [UIColor colorWithRed:0.5 green:0.0 blue:0.5 alpha:1.0];
```

```swift
// Swift
let myTableView = UITableView(frame: .zero, style: .grouped)
let color = UIColor(red: 0.5, green: 0.0, blue: 0.5, alpha: 1.0)
```

<!--more-->

### 属性

- Objective-C 中使用 `@property` 语法定义的属性会被映射为 Swift 中的属性；而不带参数且有返回值的方法虽然在 Objective-C 可以用点语法调用，但在 Swift 中仍然是普通的方法。
- Objective-C 属性声明中形如 `(attribute)` 的特性在 Swift 中会特殊处理：
  - `readonly` 会被映射为 Swift 中带 `{ get }` 的计算属性；
  - Swift 的 `weak` / `unowned(unsafe)` 关键字分别对应原来的 `weak` / `unsafe_unretained`；Swift 没有基本类型的概念，不考虑 `assign`；Swift 的值类型赋值默认进行拷贝，引用类型可以添加 `@NSCopying`，皆对应于 `copy`。
  - Objective-C 中的属性默认拥有原子性，即会加锁以防止多线程同时访问；而 Swift 语言并没有该特性，即不保证原子性。Objective-C 中的 `atomic` 和 `nonatomic` 将不会反映在 Swift 的属性声明上，但其 Objective-C 实现仍然会保证其原子性。

### 可空性

- Objective-C 中的裸指针均可为 `NULL` 或 `nil`，前者为 `(void *) 0` 后者为 `(id) 0`；而 Swift 中所有类型默认是不可空的，并有更安全的可空类型来处理可空性。为了弥合语言间的不协调，Xcode 6.3（[官方博客](https://developer.apple.com/swift/blog/?id=25)）为 Objective-C 引入了新的语法：
  - 类型声明默认为 `null_unspecified`，将被映射为**隐式解包可空类型**；
  - `nullable` 将被映射为**可空类型**；
  - `nonnull` 将被映射为**非可空类型**。

### `id`

- Objective-C 定义了 `typedef struct objc_object *id`，即 `id` 可以指向任意 Objective-C 类的对象，与 Swift 中的 `AnyObject` 类似。不过为了充分发挥 Swift 值类型的特性，Swift 3（[SE-0116](https://github.com/apple/swift-evolution/blob/master/proposals/0116-id-as-any.md)，[官方博客](https://developer.apple.com/swift/blog/?id=39)）将其桥接范围扩展到了 `Any`，因此引入了一些额外的桥接规则。
- 为了将 `Any` 桥接到 `id`，编译器引入了**通用桥接转换**（universal bridging conversion）：
  - 引用类型的类在 Swift 和 Objective-C 中都存在，所以能直接转换；
  - 可桥接的值类型如 `String`，会借助 `_ObjectiveCBridgeable` 协议转换到对应的 Objective-C 类如 `NSString`；
  - 不可桥接的值类型会被封装在一个不可变类的实例中，我们不期待在 Objective-C 中能使用它，只求它能保证 `id` 兼容性、能在语言间往返即可。
- 将 `id` 桥接到 `Any` 则需要利用运行时**模棱两可的动态转换**（ambivalent dynamic casting），因为无法预先得知 Objective-C 对象是希望被桥接到值类型还是引用类型，因此会在动态类型转换时再决定。譬如 `NSString` 对象在桥接后既可以 `as? String` 也可以 `as? NSString`。
- 另外，`AnyObject` 允许在不进行类型转换的情况下调用任何 Objective-C 的方法和属性。因为动态方法查找失败会触发运行时错误，所以 Swift 用可空类型对其进行了包装，`AnyObject` 上的方法调用行为类似于隐式解包可空值，可空链式调用等特性亦可适用。

```swift
let myObject: AnyObject = NSDate()
myObject.character(at: 5) // Unrecognized selector error!
myObject.character?(at: 5) // : unichar? = nil
```

### 闭包

- Objective-C 中的代码块和 Swift 中的闭包是互相兼容的：

```objc
// Objective-C
void (^completionBlock)(NSData *) = ^(NSData *data) { … }
```

```swift
// Swift
let completionBlock: (Data) -> Void = { data in … }
```

- 然而闭包和代码块有一个关键性的不同：闭包中的变量是可修改的，而不像代码块那样使用值拷贝。换句话说，Swift 闭包捕获的变量相当于都在 Objective-C 的变量声明中加了 `__block`。

### 判等

- Objective-C 只有一种等号，而 Swift 有两种：相等（`==`）和相同（`===`）。
  - Swift 让所有继承自 `NSObject` 的类遵循了 `Equtable` 协议，其 `==` 运算符的默认实现直接返回了 Objective-C `isEqual:` 方法的结果；
  - 全局定义的 `func === (lhs: AnyObject?, rhs: AnyObject?) -> Bool` 会判断对象指针是否相等。

### Selector

- Selector 可以在运行时表示 Objective-C 方法名，类似于成员函数指针。
- `Selector` 结构体可以用字符串构造，不过对于字面量 Swift 2.2（[SE-0022](https://github.com/apple/swift-evolution/blob/master/proposals/0022-objc-selectors.md)）引入了一种更安全的做法：通过 `#selector` 表达式来构造，编译器会检查方法是否存在。

```swift
let string: NSString = "Sapientia et Virtus"
let selector = #selector(NSString.lowercased(with:))
if let result = string.perform(selector, with: Locale.current) {
  print(result.takeUnretainedValue()) // result : Unmanaged<AnyObject>
}
```

### Key / Key Path

- Key 可以在运行时表示 Objective-C 对象的属性，而 Key Path 还能表示多级的链式路径。
- Key Path 常常用于**键值编码**（KVC）和**键值观察**（KVO）：前者可以间接访问任意对象的属性，如 `value(forKeyPath:)` 和 `setValue(_:forKeyPath:)`；后者用于观察任意对象属性的修改，如 `addObserver(_:forKeyPath:options:context:)`。
- Key Path 本质上就是以点分隔的字符串，不过 Swift 3（[SE-0062](https://github.com/apple/swift-evolution/blob/master/proposals/0062-objc-keypaths.md)）和 Swift 4（[SE-0161](https://github.com/apple/swift-evolution/blob/master/proposals/0161-key-paths.md)）分别引入了新的构造方式：
  - 通过 `#keyPath` 表达式来构造，如 `#keyPath(Member.name)`，经编译器验证后会变成字符串 `"Member.name"`；
  - 形如 `\Member.name` 的表达式则会生成 `KeyPath` 对象，保留了各种类型信息，还可以通过 `member[keyPath: \.name]` 的语法来访问属性。


## Cocoa 框架

- Swift 3（[SE-0086](https://github.com/apple/swift-evolution/blob/master/proposals/0086-drop-foundation-ns.md)）去掉了 Foundation 框架大部分类型名中的 `NS` 前缀，除了一些例外：
  - 与 Objective-C 联系十分紧密的类：`NSObject` `NSAutoreleasePool` `NSException` 等；
  - 用于特定平台的类，它们虽然位于 Foundation 但其实应当属于 AppKit / UIKit 这些更高层的框架：`NSUserNotification` `NSBackgroundActivity` `NSXPCConnection` 等；
  - 在 Swift 中有值类型等价物的类，详见 [SE-0069](https://github.com/apple/swift-evolution/blob/master/proposals/0069-swift-mutability-for-foundation.md)：`NSString` `NSDictionary` `NSURL` 等。
- Foundation 中还定义了很多枚举和常量，在 Swift 中它们会成为相关类型的嵌套类型，如 `NSJSONReadingOptions` 会成为 `JSONSerialization.ReadingOptions`。

### 字符串

- 在 Swift 中应当尽量使用值类型的 `String`，避免引用类型的 `NSString` / `NSMutableString`。因为值类型可以使用 Swift 原生的 `let` / `var` 来控制对象内容是否可变，而引用类型则需要使用不同的类来实现。
- Objective-C 中有四种 `NSLocalizedString` 开头的宏来对字符串进行本地化，而 Swift 中这被简化为了单个函数 `NSLocalizedString(_:tableName:bundle:value:comment:)`，其中后三个参数有默认值。

### 数字

- `Int` / `Double` / `Bool` 等数字类型均可用 `as` 安全地桥接到 `NSNumber`，但反过来需要使用 `as?` 或 `as!`，因为 `NSNumber` 可以表示多种类型。

### 合集类型

- `NSArray` / `NSSet` / `NSDictionary` 这三种合集类型，一开始是桥接到 `[AnyObject]` / `Set<NSObject>` / `[NSObject: AnyObject]`。得益于 Swift 3 的两份提案 [SE-0116](https://github.com/apple/swift-evolution/blob/master/proposals/0116-id-as-any.md)、[SE-0131](https://github.com/apple/swift-evolution/blob/master/proposals/0131-anyhashable.md)，现在已桥接到 `[Any]` / `Set<AnyHashable>` / `[AnyHashable: Any]`。
- [Xcode 7.0](https://developer.apple.com/library/archive/documentation/Xcode/Conceptual/RN-Xcode-Archive/Chapters/xc7_release_notes.html#//apple_ref/doc/uid/TP40016994-CH5-SW46) 为 Objective-C 引入了轻量级的泛型，允许指定合集的元素类型，以方便与 Swift 桥接，如 `NSArray<NSData*>*` 将桥接到 `[Data]`。

### Core Foundation

- 在 Swift 中，Core Foundation 所有类型的 `Ref` 后缀都会自动删掉，因为 Swift 的类一定是引用类型的。另外，可以指向任意 Core Foundation 类型的 `CFTypeRef` 桥接到了 `AnyObject`。
- 对于已经标注过的 CF API，Swift 会自动进行内存管理，不再需要 `CFRetain` 或 `CFRelease`；否则需要手动进行标注或是手动管理 `Unmanaged` 对象，详见 [NSHipster 的文章](https://nshipster.cn/unmanaged/)。

### 日志记录

- 在 macOS 10.12 / iOS 10.0 / watchOS 3.0 / tvOS 10.0 及更高版本的平台上，统一日志记录系统于 `os.log` 模块提供了 `os_log` 函数来记录日志，以此取代 Foundation 中的 `NSLog`。


## Cocoa 设计模式

### 委托

- 不论是 Swift 还是 Objective-C，委托均由协议来表达。被委托类型遵循委托协议并实现了一系列委托方法，而委托对象则会保存一个被委托类型的实例，并在各种事件发生时委托其处理。

```swift
class MyDelegate: NSObject, NSWindowDelegate {
  func window(_ window: NSWindow, willUseFullScreenContentSize proposedSize: NSSize) -> NSSize {
    return proposedSize
  }
}
myWindow.delegate = MyDelegate()
if let fullScreenSize = myWindow.delegate?.window(myWindow, willUseFullScreenContentSize: mySize) {
  print(NSStringFromSize(fullScreenSize))
}
```

### 错误处理

- 按照 Objective-C 的惯例，可能产生错误的方法接受一个 `NSError**` 作为输出参数，并返回 `BOOL` 值表示方法调用是否成功。
- 在 Swift 1 时代，这些 Objective-C API 没有什么变动，依然需要传入 `NSError` 对象指针。
- 在 Swift 2 时代，引入了语言原生的错误处理机制，所有错误参数被替换为 `throws` 关键字，返回类型从 `BOOL` 改为 `Void`。

### 单例

- 单例模式使用一个全局共享的实例，以提供资源或服务的统一接入点。
- 在 Objective-C 中，可以用 `dispatch_once` 保证单例只被创建一次。

```objc
+ (instancetype)sharedInstance {
    static id _sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _sharedInstance = [[self alloc] init];
    });
    return _sharedInstance;
}
```

- 在 Swift 中，可以直接使用惰性初始化的类型属性。如果调用构造器之后还要做其他初始化工作，可以写一个立即执行的闭包（[IIFE](https://en.wikipedia.org/wiki/Immediately_invoked_function_expression)）。

```swift
class Singleton {
    static let sharedInstance: Singleton = {
        let instance = Singleton()
        // Setup…
        return instance
    }()
}
```

### 内省

- 内省（Introspection）即运行时类型检查，Objective-C 中的 `isKindOfClass:` 和 `conformsToProtocol:` 方法、Swift 中的 `is` 和 `as?` 运算符均属于此类。
- 另外，从 Swift 3 开始可以用 `type(of:)` 来获取一个对象的动态类型（[SE-0096](https://github.com/apple/swift-evolution/blob/master/proposals/0096-dynamictype.md) 之前是对象的 `dynamicType` 属性）。

### 序列化

- 在 Swift 中，遵循 `Codable` 协议的类型即可使用 `JSON{En,De}coder` 和 `PropertyList{En,De}coder` 进行序列化和反序列化，处理自定义类型详见[官方文档](https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types)。

### 命令行参数

- 可以通过 `CommandLine.arguments` 来访问命令行参数，这和 `ProcessInfo.processInfo.arguments` 等价。


## C API

### 基本类型

- Swift 为 C 语言的基本类型提供了等价的类型，但它们不会隐式转换为 Swift 原生的数字类型。

| C                  | Swift             |
| ------------------ | ----------------- |
| bool               | CBool             |
| char, signed char  | CChar             |
| unsigned char      | CUnsignedChar     |
| short              | CShort            |
| unsigned short     | CUnsignedShort    |
| int                | CInt              |
| unsigned int       | CUnsignedInt      |
| long               | CLong             |
| unsigned long      | CUnsignedLong     |
| long long          | CLongLong         |
| unsigned long long | CUnsignedLongLong |
| wchar_t            | CWideChar         |
| char16_t           | CChar16           |
| char32_t           | CChar32           |
| float              | CFloat            |
| double             | CDouble           |

### 指针

- Swift 尽可能避免了直接使用指针，但仍然提供了多种指针类型以操作内存地址：

| C         | Swift                     |
| --------- | ------------------------- |
| const T * | UnsafePointer\<T\>        |
| T *       | UnsafeMutablePointer\<T\> |

- 对于类对象而言：

| C              | Swift                                  |
| -------------- | -------------------------------------- |
| T * const *    | UnsafePointer\<T\>                     |
| T * __strong * | UnsafeMutablePointer\<T\>              |
| T **           | AutoreleasingUnsafeMutablePointer\<T\> |

- 从 Swift 3（[SE-0107](https://github.com/apple/swift-evolution/blob/master/proposals/0107-unsaferawpointer.md)）开始，指向未知类型内存的指针是：

| C            | Swift                   |
| ------------ | ----------------------- |
| const void * | UnsafeRawPointer        |
| void *       | UnsafeMutableRawPointer |

- 以**常量指针**为参数的函数可以接受以下值：
    - 常量指针、变量指针或自动释放指针；
    - （如果 `T` 为 `Int8` / `UInt8`）字符串，以 UTF-8 编码；
    - `inout T`，即相应类型值前加 `&`；
    - 数组 `[T]`。
- 以**变量指针**为参数的函数可以接受以下值：
    - 变量指针；
    - `inout T`；
    - `inout [T]`。
- 以**自动释放指针**为参数的函数可以接受以下值：
    - 自动释放指针；
    - `inout T`，不过传递的指针指向一个回写临时缓冲区。

### 枚举

- Swift 会将所有用 `NS_ENUM` 宏标记的 C 枚举导入为 Swift 枚举，`NS_OPTION` 导入为遵循 `OptionSet` 的结构体和一系列类型属性，而没有用宏标记的 C 枚举则导入为遵循 `RawRepresentable` 的结构体和一系列全局变量。
- [Xcode 10.0](https://developer.apple.com/documentation/macos_release_notes/macos_mojave_10_14_release_notes/appkit_release_notes_for_macos_10_14) 引入了相仿的 `NS_TYPED_ENUM` 宏，Swift 会将一系列全局常量导入为遵循 `RawRepresentable` 的结构体和一系列类型属性，详见[官方文档](https://developer.apple.com/documentation/swift/objective-c_and_c_code_customization/grouping_related_objective-c_constants)。

### 预处理指令

- 因为 Swift 编译器不包含预处理器，所以 Swift 没有预处理指令（preprocessor directives）。
- 不过 Swift 仍支持条件编译，编译条件包括字面值 `true` / `false`、条件编译标志（`swift -D <#flag#>`）和平台条件函数：

| 平台条件函数 | 有效参数                         |
| ------------ | -------------------------------- |
| os()         | macOS, iOS, watchOS, tvOS, Linux |
| arch()       | x86_64, arm, arm64, i386         |
| swift()      | >=x.x                            |

```swift
#if arch(arm) || arch(arm64)
print("Using ARM code")
#elseif arch(x86_64)
print("Using 64-bit x86 code")
#else
print("Using general code")
#endif
```

- 顺便一提，因为没有宏系统来支持元编程，Swift 团队维护了一个名叫 GYB（Generate Your Boilerplate）的轻量级模板工具，详见 [NSHipster 的文章](https://nshipster.cn/swift-gyb/)。


## Swift / Objective-C 混编

### 应用内混编

|                  | 导入到 Swift                         | 导入到 Objective-C                    |
| ---------------- | ------------------------------------ | ------------------------------------- |
| Swift 代码       | 不需要导入语句                       | `#import "ProductModuleName-Swift.h"` |
| Objective-C 代码 | 不需要导入语句，但需要配置桥接头文件 | `#import "Header.h"`                  |

- 将 Swift 代码导入 Objective-C，只需 `#import` Xcode 为 Swift 自动生成的头文件，它声明了所有 Swift 中定义的公开接口，如果已经配置了桥接头文件则亦会声明所有内部接口。
- 将 Objective-C 代码导入到 Swift，需要在配置好的 Objective-C 桥接头文件（文件名为 `ProductModuleName-Bridging-Header.h`）中 `#import` 相关头文件。

### 框架内混编

|                  | 导入到 Swift                         | 导入到 Objective-C                                |
| ---------------- | ------------------------------------ | ------------------------------------------------- |
| Swift 代码       | 不需要导入语句                       | `#import <ProductName/ProductModuleName-Swift.h>` |
| Objective-C 代码 | 不需要导入语句，但依赖框架的伞头文件 | `#import "Header.h"`                              |

- 将 Swift 代码导入到 Objective-C，只需 `#import` Xcode 为 Swift 自动生成的头文件，它声明了所有 Swift 中定义的公开接口。
- 将 Objective-C 代码导入到 Swift，需要在框架的 Objective-C 伞头文件（文件名为 `ProductModuleName.h`）中 `#import` 相关头文件，当然它们也就成为了公开接口。

### 小提示

- 为避免循环引用，千万别把 Swift 导入到 Objective-C 头文件中，但可以用 `@class MyClass; @protocol MyProtocol;` 来前向声明 Swift 的类或协议。
- 上文多次提到的 product module name，默认与用户设定的 product name 相同，不过其中的非字母数字字符会被下划线替代。

### `@objc`

- 将 Swift 导入到 Objective-C 之后，便可访问标为 `@objc` 或 `@objc(<#name#>)` 的 API。
- Swift 的独有特性不可以被标为 `@objc`：泛型、元组、非 `Int` 原始值的枚举、结构体、全局函数、全局变量、类型别名、可变参数、嵌套类型、柯里化的函数。
- 从 Swift 4（[SE-0160](https://github.com/apple/swift-evolution/blob/master/proposals/0160-objc-inference.md)）开始，继承自 `NSObject` 的类及其属性和方法不再自动标为 `@objc`，现在仍然自动标为 `@objc` 的情况只剩下几种确保语义一致性的情况：
  - 该类继承自 Objective-C 中定义的类；
  - 其声明重写了父类中的 `@objc` 声明；
  - 其声明满足了一项 `@objc` 协议的要求；
  - 已被标为 `@IBAction` / `@IBOutlet` / `@IBDesignable` / `@IBInspectable` / `@GKInspectable` / `@NSManaged`。
- 可以用 `@objcMembers` 让一个类本身、其扩展、其子类、其子类的扩展都被自动标为 `@objc`，另外也有 `@nonobjc` 显式取消隐式的 `@objc`。
- 在 Objective-C 中调用的 Swift API 必须支持**动态派发**（dynamic dispatch），但这并不意味着在 Swift 中编译器不会将 `@objc` 方法优化成静态派发。如果一定要使用 Objective-C 运行时中的 [KVO](https://nshipster.cn/key-value-observing/)、[Method Swizzling](https://nshipster.cn/method-swizzling/) 等动态特性，得用 `@objc dynamic` 来强制方法进行动态派发。
- Objective-C 的类不可以继承自 Swift 的类。


> **\<Prev\>** [Swift 学习笔记（二）](/swift-notes-2/)
