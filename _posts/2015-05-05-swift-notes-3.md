---
title: Swift 学习笔记（三）
author: 孙耀珠
tags: 编程语言
---

* 目录
{:toc}

## 与 Objective-C API 交互

Swift 3 对 Objective-C API 做了大规模的修改，使之更清晰、更适应于 Swift。Swift Evolution 的三个提案 [SE-0005](https://github.com/apple/swift-evolution/blob/master/proposals/0005-objective-c-name-translation.md)、[SE-0006](https://github.com/apple/swift-evolution/blob/master/proposals/0006-apply-api-guidelines-to-the-standard-library.md)、[SE-0059](https://github.com/apple/swift-evolution/blob/master/proposals/0059-updated-set-apis.md) 都对 API 的设计提出了不少改进，并最终形成了一份 [API Design Guidelines](https://swift.org/documentation/api-design-guidelines/)。

### 初始化

- 使用 Swift 语法调用 Objective-C 的构造器时，方法名中的 `init` 和 `initWith` 前缀会被截去，其余各部分依次变为构造器的参数名，不过其中多余的成分会被删掉。同时不再需要调用 `alloc` 方法。
- 为了一致性和便捷性，Objective-C 中的工厂方法（factory methods）也被映射成为 Swift 中的构造器。
- 在 Objective-C 中可能返回 `nil` 的构造器，引入 Swift 时被定义为了可失败构造器。

```objc
// Objective-C
UITableView *myTableView = [[UITableView alloc] initWithFrame:CGRectZero style:UITableViewStyleGrouped];
UIColor *color = [UIColor colorWithRed:0.5 green:0.0 blue:0.5 alpha:1.0];
```

```swift
// Swift
let myTableView = UITableView(frame: CGRect.zero, style: .grouped)
let color = UIColor(red: 0.5, green: 0.0, blue: 0.5, alpha: 1.0)
```

### 方法和属性

- 在 Swift 中调用 Objective-C 对象的方法和属性时，使用点语法。
- Objective-C 的方法移植到 Swift 中时，原方法名的第一部分作为新的方法名和第一个参数标签，其余部分依次作为后面的参数标签，当然一些多余的成分会被删掉。

<!--more-->

### `id` 兼容性

- Objective-C 中的 `id` 可以指向任何类型的对象，在 Swift 中它会与 `Any` 类型桥接。[^id]
- 当 `Any` 桥接到 `id`，会在编译时和运行时进行 **universal bridging conversion**：
  - 类（class）是最容易处理的，因为它在 Swift 和 Objective-C 中都存在；
  - 可桥接的值类型如 `String`，会借助 `_ObjectiveCBridgeable` 协议桥接到相对应的 Objective-C 类如 `NSString`；
  - 不可桥接的值类型会被封装在一个不可变类的实例中，其类名和功能不会暴露在 Objective-C 中。
- 当 `id` 桥接到 `Any`，运行时会自动将其转换为类的引用或值类型，这被称为 **ambivalent dynamic casting**。
- `AnyObject` 允许在不进行类型转换的情况下调用任何 Objective-C 的方法和属性，这个行为类似于隐式解析可选类型。与 Objective-C 相同，若方法或属性不存在将触发运行时错误，但这可以使用可选链来避免，如 `myObject.character?(at: 5)`。

[^id]: 在 [Swift 3.0](https://github.com/apple/swift-evolution/blob/master/proposals/0116-id-as-any.md) 以前，`id` 是被桥接到 `AnyObject` 的。

### 可空性和可选类型

- 在 Objective-C 中以 `__nonnull` 标注的类型声明将被引入为 Swift 中的**非可选类型**。
- 以 `__nullable` 标注的类型声明将被引入为**可选类型**。
- 没有为空性标注的类型声明将被引入为**隐式解析可选类型**。

### 闭包

- Swift 中的闭包和 Objective-C 中的代码块是互相兼容的，所以可以将闭包直接传入一个以代码块为参数的函数。

```objc
// Objective-C
void (^completionBlock)(NSData *) = ^(NSData *data) { … }
```

```swift
// Swift
let completionBlock: (NSData) -> Void = { data in … }
```

- 然而闭包和代码块有一个关键性的不同，闭包中的变量是可修改的，而不像代码块那样使用值拷贝，即 Swift 闭包中用到的变量相当于在 Objective-C 的变量声明前面加了 `__block`。

### Swift 类型兼容性

- 如果希望 Swift API 在 Objective-C 中可用，可以在前面加上 `@objc` 或是 `@objc(name)` ；相反也可以使用 `@nonobjc` 使其不可用。
- 如果在 Swift 中定义的类继承自 `NSObject` 或者其他 Objective-C 类，编译器将自动添加 `@objc` 属性，同时类中所有的属性和方法也会加上该属性，除非其访问级别为私有。另外当使用 `@IBOutlet` / `@IBAction` / `@NSManaged` 属性时，`@objc` 也会被自动加上。
- Swift 中的枚举类型只有使用 `Int` 作为原始值才能使用 `@objc` 属性。

### Selector

- Selector 是一个指向 Objective-C 方法名的类型，在 Swift 中它被表示为 `Selector` 结构体，可以通过 `#selector` 表达式构造它，如 `#selector(NSString.lowercased(with:))`，这可以被 `perform(_:)` 等方法使用。

### Key 和 Key Path

- Key 是用来识别一个对象属性（property）的字符串，而 Key Path 则是用来表示对象属性序列的点分隔的字符串。
- 与 selector 类似，在 Swift 中可以使用 `#keyPath` 表达式来构造一个经过编译器检查的 key 或 key path，如 `#keyPath(Member.friends.name)`。这可以被 KVC 方法如 `value(forKey:)` / `value(forKeyPath)` 以及 KVO 方法如 `addObserver(_:forKeyPath:options:context:)` 使用。


## 利用 Objective-C 的特性编写 Swift 类

### Interface Builder

- 在 Swift 中使用 outlet 和 action 时，需要在属性和方法前加上 `@IBOutlet` 和 `@IBAction`，并将它们声明为隐式解析可选类型。
- 对于一个继承自 `UIView` 或 `NSView` 的自定义视图，可以在类定义前加上 `@IBDesignable`，则该视图将在 IB 的画布上实时渲染。
- 同时可以为类的属性添加 `@IBInspectable`，这样便能在 IB 的监视器面板中编辑这些属性，并且这里的编辑将覆盖代码中对属性的设置。

### Core Data Managed Object

- Core Data 为 `NSManagedObject` 子类的属性提供了底层存储和实现，所以需要在 Core Data 数据模型相应的属性前面加上 `@NSManaged`，以告诉编译器该属性的存储和实现将在运行时提供，类似于 Objective-C 中的 `@dynamic`。

## 与 Cocoa 框架共处

- 在 Swift 3.0 中，提案 [SE-0086](https://github.com/apple/swift-evolution/blob/master/proposals/0086-drop-foundation-ns.md) 去掉了大部分 `Foundation` 类型名中的 `NS`，不过仍有一些例外保留着 `NS` 前缀：
  - Objective-C 特有的或是与 Objective-C 运行时有内在联系的类：`NSObject` / `NSAutoreleasePool` / `NSException` / `NSProxy` 等；
  - 平台限定的类，这些类虽然位于 `Foundation` 但实际上应当属于 `UIKit` / `AppKit` 这些更高层次的框架：`NSUserNotification` / `NSBackgroundActivity` / `NSXPCConnection` 等；
  - 在 Swift 中有值类型的等价物的类，提案 [SE-0069](https://github.com/apple/swift-evolution/blob/master/proposals/0069-swift-mutability-for-foundation.md) 大大扩充了这一名单：`NSString` / `NSDictionary` / `NSURL` 等；
  - 在不远的将来计划拥有值类型等价物的类：`NSAttributedString` / `NSRegularExpression` / `NSPredicate`。
- `Foundation` 中还定义了很多枚举和常量，在 Swift 中它们会成为相关类型的嵌套类型，如 `NSJSONReadingOptions` 会成为 `JSONSerialization.ReadingOptions`。

### 字符串

- 在 Swift 中应当尽量使用值类型的 `String`，避免引用类型的 `NSString` / `NSMutableString`。因为值类型可以使用 Swift 原生的 `let` / `var` 来控制对象内容是否可变，而引用类型则需要使用不同的类来实现。
- 在 Objective-C 中，会使用 `NSLocalizedString` 一系列不同的宏来对字符串进行本地化，而在 Swift 中这被简化为一个单独的函数 `NSLocalizedString(key:tableName:bundle:value:comment:)`，其中 `tableName` / `bundle` / `value` 参数已提供默认值。

### 数字

- `Int` / `Double` / `Bool` 等算数类型均可用 `as` [^bridge] 安全地桥接到 `NSNumber`，但反过来需要使用 `as?` / `as!`，因为 `NSNumber` 可能代表多种类型。

[^bridge]: 在 Swift 3.0 以前，像这样的桥接是自动进行的，不需要 `as`，但提案 [SE-0072](https://github.com/apple/swift-evolution/blob/master/proposals/0072-eliminate-implicit-bridging-conversions.md) 废除了隐式桥接。

### 集合类型

- `NSArray` / `NSSet` / `NSDictionary` 桥接到 Swift 时会转换为 `[Any]` / `Set<AnyHashable>` / `[AnyHashable: Any]`。[^collection]
- 从 [Xcode 7.0](https://developer.apple.com/library/prerelease/mac/documentation/DeveloperTools/Conceptual/WhatsNewXcode/Articles/xcode_7_0.html#//apple_ref/doc/uid/TP40015242-SW3) 开始，Objective-C 允许指定集合类型的元素类型，以方便与 Swift 桥接，如 `NSArray<NSData *>*` 将桥接到 `[Data]`。

[^collection]: 原本这三者是被转换为 `[AnyObject]` / `Set<NSObject>` / `[NSObject: AnyObject]` 的，得益于 [SE-0116](https://github.com/apple/swift-evolution/blob/master/proposals/0116-id-as-any.md) 和 [SE-0131](https://github.com/apple/swift-evolution/blob/master/proposals/0131-anyhashable.md) 有了现在的桥接方式。

### Core Foundation

- Swift 在引入 Core Foundation 类型时，编译器会自动去掉类名的 `Ref` 后缀，因为 Swift 的类一定是引用类型的。另外，`CFTypeRef` 会重新映射到 `AnyObject`。

### 统一的日志记录

- 在 macOS 10.12 / iOS 10.0 / watchOS 3.0 / tvOS 10.0 及更高版本的操作系统上，可以使用 `os.log` 模块中的 `os_log(_:dso:log:type:_:)` 函数。

## 采用 Cocoa 设计模式

- Cocoa 设计模式的主要内容包括：委托（delegation）、惰性初始化（lazy initialization）、错误处理（error handling）、键值观察（key-value observing）、撤销（undo）、目标-动作（target-action）、单例（singleton）、自省（introspection）、序列化（serializing）、本地化（localization）、自动释放池（autorelease pools）、API 可用性（API availability）、处理命令行参数（processing command-line arguments）。

### 委托

- 在 Swift 和 Objective-C 中，委托通常都是以协议的形式来表示。委托的过程通常包括：检查委托者是否非空、检查方法是否被实现、若满足上述条件则调用这个方法，这些可以通过 Swift 中的可选链和可选绑定来简便地实现。

### 错误处理

- 在 Objective-C 中可能抛出错误的方法接受一个 `NSError **` 作为参数，而在 Swift 中该参数被替换为 `throws` 关键字。[^error]
- 如果原 Objective-C 方法返回值为 `BOOL`，表示方法调用是否成功，那么引入 Swift 后其返回类型被改为 `Void`。

[^error]: 在 [Swift 2.0](https://developer.apple.com/swift/blog/?id=29) 以前，Swift 的错误报告沿用了 Objective-C 传参的模式。在 Swift 中可以定义一个 `NSError?` 类型的可选变量，并在变量前加上 `&` 运算符使其成为 `NSErrorPointer` 对象作为 error 参数。当自定义一个以 `NSErrorPointer` 为参数的函数时，为对象的 `memory` 属性赋予 `NSError` 对象即可。

### 单例模式

- 单例模式使用一个供全局访问的共享实例，以完成需要统一接入点的任务，如播放音效或发送 HTTP 请求。
- 在 Objective-C 中，可以用 `dispatch_once` 函数保证构造器只被执行一次。

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

- 而在 Swift 中，可以直接使用类属性，这保证了它只被惰性初始化一次。如果希望在调用构造器后做其他初始化工作，可以写一个立即执行的闭包。

```swift
class Singleton {
    static let sharedInstance: Singleton = {
        let instance = Singleton()
        // Setup
        return instance
    }()
}
```

### 自省

- 在 Objective-C 中，可以使用 `isKindOfClass:` 来检测对象的类型，使用 `conformsToProtocol:` 来检测对象是否遵守协议。而在 Swift 中可以通过 `is` 运算符实现以上功能，且能通过 `as?` / `as!` 向下转型。
- 在 Swift 3 中可以使用 `type(of:)` 来获取一个对象的动态类型，在 [SE-0096](https://github.com/apple/swift-evolution/blob/master/proposals/0096-dynamictype.md) 之前这是通过对象的 `dynamicType` 属性获取的。


### 命令行参数

- 可以通过 `Process.arguments` 来访问命令行参数，这和 `NSProcessInfo.processInfo().arguments` 等价。

## 与 C API 交互

### 原始类型（Primitive Types）

- Swift 为 C 语言的原始类型提供了等价的 Swift 类型，但它们不会隐式转换为 Swift 原有的数字类型。

| C 类型               | Swift 类型          |
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

### 枚举

- Swift 会导入所有用 `NS_ENUM` / `NS_OPTIONS` 宏标记的 C 样式枚举，同时枚举成员的名称前缀会被自动截断，例如 `UITableViewCellStyleDefault` 在 Swift 中将是 `UITableViewCellStyle` 的成员 `.default`。

### 指针

- Swift 尽可能避免了对指针的直接访问，但当需要直接操作内存时仍有多种指针类型可供使用。

| C 语法           | Swift 语法                               |
| -------------- | -------------------------------------- |
| const T *      | UnsafePointer\<T\>                     |
| T *            | UnsafeMutablePointer\<T\>              |
| T * const *    | UnsafePointer\<T\>                     |
| T * __strong * | UnsafeMutablePointer\<T\>              |
| T **           | AutoreleasingUnsafeMutablePointer\<T\> |
| RetT (*)(ArgT) | @convention(c) (ArgT) -> RetT          |

- 根据提案 [SE-0107](https://github.com/apple/swift-evolution/blob/master/proposals/0107-unsaferawpointer.md)，从 Swift 3.0 开始，可以指向任何类型的指针 `Unsafe(Mutable)Pointer<Void>` 被 `Unsafe(Mutable)RawPointer` 取代。
- 根据提案 [SE-0055](https://github.com/apple/swift-evolution/blob/master/proposals/0055-optional-unsafe-pointers.md)，从 Swift 3.0 开始上述指针在原代码未标注 `__nullable` / `__nonnull` 时为隐式解析可选类型，而 `__nonnull` 对应的非可选类型不能赋值为 `nil`。
- 以**常量指针**为参数的函数可以接受以下值：
    - 常量指针、变量指针或自动释放指针；
    - `String`，如果 `T` 为 `Int8` / `UInt8` 则字符串将自动转换为 UTF-8；
    - `inout T`，即相应类型值前加 `&` 取地址；
    - 数组 `[T]`。
- 以**变量指针**为参数的函数可以接受以下值：
    - 变量指针；
    - `inout T`，即相应类型值前加 `&` 取地址；
    - `inout [T]`，即数组前加 `&` 得到指向起始元素的指针。
- 以**自动释放指针**为参数的函数可以接受以下值：
    - 自动释放指针；
    - 原始类型的拷贝前加 `&`。

### 全局变量

- C 和 Objective-C 源文件中的全局变量将被自动导入 Swift。

### 预处理指令

- Swift 编译器中不包含预处理器，因此预处理指令（preprocessor directives）没有被引入 Swift。
- 在 Swift 中可以使用构建配置（build configurations）进行条件编译，构建配置包括字面值 `true` 和 `false`，命令行标志，以及平台测试函数。其中命令行标志可通过 `swift -D FLAG` 指定。

| 平台测试函数  | 有效参数                             |
| ------- | -------------------------------- |
| os()    | macOS, iOS, watchOS, tvOS, Linux |
| arch()  | x86_64, arm, arm64, i386         |
| swift() | >=x.x                            |

```swift
#if arch(arm) || arch(arm64)
print("Using ARM code")
#elseif arch(x86_64)
print("Using 64-bit x86 code")
#else
print("Using general code")
#endif
```


## 在同一个工程中使用 Swift 和 Objective-C

- Swift 对 Objective-C 的兼容性支持在同一个工程中同时使用两种语言，因此可以用这种叫做 mix and match 的特性来开发基于混合语言的应用。

### 在同一个 App Target 中导入

- 将 Objective-C 代码导入到 Swift 时，需要依赖 Objective-C 的桥接头文件（bridging header）。当添加 Swift 文件到现有的 Objective-C 应用（或反之）时，Xcode 会自动创建这些头文件，名为 `ProductModuleName-Bridging-Header.h`。在桥接头文件中，可以 `#import` 任何想暴露给 Swift 的头文件。
- 将 Swift 代码导入 Objective-C 时，需要 `#import` Xcode 自动生成的头文件，它声明了所有 Swift 中定义的接口。该文件可以看作 Swift 代码的 umbrella header，名为 `ProductModuleName-Swift.h`。

|                | 导入到 Swift                                | 导入到 Objective-C                       |
| -------------- | ---------------------------------------- | ------------------------------------- |
| Swift 代码       | 不需要 import                               | `#import "ProductModuleName-Swift.h"` |
| Objective-C 代码 | 不需要 import，但需要 Objective-C bridging header | `#import "Header.h"`                  |

### 在同一个 Framework Target 中导入

- 首先确认已经将 `Build Settings > Packaging > Defines Module` 设置为 `Yes`。
- 要将 Objective-C 代码导入到 Swift 中，需要在 Objective-C 的 umbrella header（即 `ProductModuleName.h`）中 `#import` 相应的头文件。
- 要将 Swift 代码导入到 Objective-C 中，只需 `#import` Xcode 自动生成的头文件，名为 `<ProductName/ProductModuleName-Swift.h>`。

|                | 导入到 Swift                                | 导入到 Objective-C                          |
| -------------- | ---------------------------------------- | ---------------------------------------- |
| Swift 代码       | 不需要 import                               | `#import <ProductName/ProductModuleName-Swift.h>` |
| Objective-C 代码 | 不需要 import，但需要 Objective-C umbrella header | `#import "Header.h"`                     |

### 导入外部框架

- 导入到 Swift 时使用 `import FrameworkName`
- 导入到 Objective-C 时使用 `@import FrameworkName;`

### 在 Objective-C 中使用 Swift

- 将 Swift 导入 Objective-C 之后，便可以用 Objective-C 的语法访问 `@objc` 修饰的类和协议。
- Swift 的独有特性无法在 Objective-C 中使用，包括：泛型、元组、枚举、结构体、顶层函数、全局变量、类型别名、可变参数、嵌套类型、柯里化函数（curried functions）。
- 为避免循环引用，不要把 Swift 导入到 Objective-C 头文件中，但可以用 `@class SwiftClass;` 前向声明（forward declare）一个 Swift 类来使用它。
- Objective-C 的类不可以继承自 Swift。

### Product Module Name

- 默认的 Product Module Name 跟 Product Name 相同，不过如果其中如果有非字母数字的字符，它们将会被 `_` 替代。
- 也可以在 `Build Settings > Packaging > Product Module Name` 中自定义名称。


> **\<Prev\>** [Swift 学习笔记（二）](/swift-notes-2/)
