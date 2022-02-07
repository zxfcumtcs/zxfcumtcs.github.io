---
layout: post
title: Swift Protocol 背后的故事(上)
date: 2022-02-01 10:06:08
tags:

- iOS
- Swift
- Protocol

---

我们将从实践技巧、实现原理两个方面对 Swift Protocol 展开深入讨论。

本文作为上篇主要介绍实践技巧，以一个 Protocol 相关的编译错误为引，通过实例对 Type Erasure、Opaque Types 、Generics 以及 Phantom Types 做了较详细的讨论。它们对于写出更优、更雅的 Swift 代码有一定的帮助。

<!--more-->

©原创文章，转载请注明出处！

# 引

---

Swift 推崇面向协议编程 (POP, Protocol Oriented Programming)，因此 Protocol 在 Swift 中就显得尤为重要。

但本文要讨论的既不是 Protocol 的使用，也不是 POP。

我们的讨论从一个编译错误开始：

> Protocol 'Equatable' can only be used as a generic constraint because it has Self or associated type requirements.

对于 Swift 开发者来说上面这个编译错误应该不陌生。

其字面意思不难理解：含有 `Self` 或关联类型的协议只能用作泛型约束，不能单独作为类型使用。

Why？

*因为 Swift 是类型安全的语言 (type-safe language)。*

why？

上面这个解释是句『 正确的废话 』，没有说到点子上。

下面我们以一个 Demo 为基础展开今天的讨论 ([GitHub - zxfcumtcs/MarkdownDemo: Swift Protocol Demo](https://github.com/zxfcumtcs/MarkdownDemo))：
![](/img/MarkdownDemo.png)

如上图，MarkdownEditor 是一个 Markdown 格式的编辑器。

为了处理不同的 Markdown 格式，我们定义了协议 `MarkdownBuilder`， 其作为公开接口曝露给业务方：

```swift
public protocol MarkdownBuilder: Equatable, Identifiable {
  var style: String { get }
  func build(from text: String) -> String
}
```

> 由于有判等需求，`MarkdownBuilder` 继承了 `Equatable` 协议。

如果我们直接将 `MarkdownBuilder` 作为类型使用，如：`var builder: MarkdownBuilder` ，就会报上面的错误。

因为，Equatable 有 **Self requirements**：要求 `==` 操作符的两个参数 `lhs` 与 `rhs` 的类型必须相同 (注意是准确的类型，而不是说只要遵守 `Equatable` 即可)。

```swift
public protocol Equatable {
  static func == (lhs: Self, rhs: Self) -> Bool
}
```

假如，允许有 `Self requirements` / `Associated Type` 的 Protocol 作为类型使用，就会出现以下情况，而编译器却无能为力：

```swift
let lhs: Equatable = 1           // Int
let rsh: Equatable = "1"         // String
lhs == rsh                       // ?!, 不同类型的值可以判等
```

对于 `Associated Type` 也是同样的道理：

```swift
// 用于校验电话号码是否合法
// 由于电话号码可以有多种表达格式
// 抽取了协议，并实现了Int、String两种格式
//
protocol PhoneNumberVerifier {
  associatedtype Phone
  func verify(_ model: Phone) -> Bool
}

struct IntPhoneNumberVerifier: PhoneNumberVerifier {
  func verify(_ model: Int) -> Bool {
    // do some verify
  }
}

struct StrPhoneNumberVerifier: PhoneNumberVerifier {
  func verify(_ model: String) -> Bool {
    // do some verify
  }
}

let verifiers: [PhoneNumberVerifier] = [...]
verifiers.forEach { verifier in
  verifier.verify(???) // 这里的参数怎么传？Int? String? 编译器无法保证类型安全
}
```

说这么多，归根结底是因为 Protocol 是运行时特性，而其附带的 `Self requirements` / `Associated Type` 却需要在编译时保证。其结果必定凉凉~

> Generics 是编译期特性，在编译时就能明确泛型的具体类型，故有 `Self requirements`/`Associated Type` 的 Protocol 只能作为其约束使用。

# Type Erasure

---

回到上节提到的 Markdown 编辑器：MarkdownEditor，我们实现了4种格式的 MarkdownBuilder：

```swift
extension MarkdownBuilder {
  public var id: String { style }
}

// 斜体
//
fileprivate struct ItalicsBuilder: MarkdownBuilder {
  public var style: String { "*Italics*" }

  public func build(from text: String) -> String { "*\(text)*" }
}

// 粗体
//
fileprivate struct BoldBuilder: MarkdownBuilder {
  public var style: String { "**Bold**" }

  public func build(from text: String) -> String { "**\(text)**" }
}

// 删除线
//
fileprivate struct StrikethroughBuilder: MarkdownBuilder {
  public var style: String { "~Strikethrough~" }

  public func build(from text: String) -> String { "~\(text)~" }
}

// 超链接
//
fileprivate struct LinkBuilder: MarkdownBuilder {
  public var style: String { "[Link](link)" }

  public func build(from text: String) -> String { "[\(text)](https://github.com)"}
}
```

`struct MarkdownView: View`是整个 Demo 的主界面，需要在其中存储所有支持的 Markdown Builder，以及当前选中的 Builder。

所以，我们不加思索地写下了以下代码：

```swift
struct MarkdownView: View {
  private let allBuilders: [MarkdownBuilder]
  private var selectedBuilders: [MarkdownBuilder]
}
```

结果可想而知！

怎么办？

Generics？ 在这里似乎行不通！

将所有支持的 Builder 逐个定义出来？

太蠢了！且不符合『 OCP 』原则。

此时，就需要用到本节的主角：Type Erasure (类型擦除)。

> Type Erasure 是一项通用技术，并非 Swift 特有，核心思想是在编译期擦除 (转换) 原有类型，使其对业务方不可见。
> 
> 有多种方式可以实现 Type Erasure，如：Boxing、Closures 等。

在 MarkdownEditor 中，我们通过 Boxing 实现 Type Erasure，简单讲就是对原有类型做一次封装 (Wrapper)：

```swift
public struct AnyBuilder: MarkdownBuilder {

  public let style: String
  public var id: String { "AnyBuilder-\(style)" }

  private let wrappedApply: (String) -> String

  public init<B: MarkdownBuilder>(_ builder: B) {
    style = builder.style
    wrappedApply = builder.build(from:)
  }

  public func build(from text: String) -> String {
    wrappedApply(text)
  }

  public static func == (lhs: AnyBuilder, rhs: AnyBuilder) -> Bool {
    lhs.id == rhs.id
  }
}
```

几个关键点：

- `AnyBuilder` 实现了 `MarkdownBuilder`协议，(一般情况下 Wrapper 都需要实现待封装的协议)；
- 其 `init` 是泛型方法，并将参数传递过来的 `style`、`build(from:)` 存储下来；
- 在其自身的`build(from:)`方法中直接调用存储的 `wrappedApply`，其本身相当于一个转发代理。

同时，扩展 `MarkdownBulider`：

```swift
public extension MarkdownBuilder {
  func asAnyBuilder() -> AnyBuilder {
    AnyBuilder(self)
  }
}
```

现在，我们就可以愉快地在 `MarkdownView` 中使用 `AnyBuilder` 了：

```swift
struct MarkdownView: View {
  private let allBuilders: [AnyBuilder]  
  private var selectedBuilders: [AnyBuilder]
}
```

由于有上面的 `MarkdownBuilder` 扩展，可以通过 2 种方式生成 `AnyBuilder` 实例：

- `BoldBuilder().asAnyBuilder()`
- `AnyBuilder(BoldBuilder())`

> 在 Swift 标准库中有大量通过 Boxing 实现的 Type Erasure ，如： `AnySequence`、`AnyHashable`、`AnyCancellable`等等。
> 
> 以 Any 为前缀的几乎都是。

# Opaque Types

---

如果，我们准备将 MarkdownEditor 做成一个独立的三方库，并且除了 `MarkdownBuilder` 协议，不打算曝露任何其他的实现细节以增加其灵活性。

即，`ItalicsBuilder`、`BoldBuilder`、`StrikethroughBuilder` 以及 `LinkBuilder` 都是库私有的。

如何做？

又一次不加思索地写下了以下代码：

```swift
public func italicsBuilder() -> MarkdownBuilder {
  ItalicsBuilder()
}

public func boldBuilder() -> MarkdownBuilder {
  BoldBuilder()
}

public func strikethroughBuilder() -> MarkdownBuilder {
  StrikethroughBuilder()
}

public func linkBuilder() -> MarkdownBuilder {
  LinkBuilder()
}
```

我们希望通过 public func 为业务方创建相应的 Builder 实例，同时以接口的方式返回。

理想丰满，现实骨感！

同样的错误在等着你！

怎么办？

轮到本节主角 Opaque Types 登场了！

简单讲，Opaque Types 就是让函数/方法的返回值是协议，而不是具体的类型。

> A function or method with an opaque return type hides its return value’s type information. 
> Instead of providing a concrete type as the function’s return type, the return value is described in terms of the protocols it supports.

几个关键点：

- 关键字 `some`，需在返回协议类型前添加 `some` 关键词，如：
  *`public func regularBuilder() -> some MarkdownBuilder`*
  而不是 `public func regularBuilder() -> MarkdownBuilder`

- Opaque Types 与直接返回协议类型的最大区别是：
  
  - Opaque Types 只是对使用方(人)隐藏了具体类型细节，编译器是知道具体类型的；
  
  - 而直接返回协议类型，则是运行时行为，编译器是无法知道的；
  
  - 如下代码，编译器是明确知道 `italicsBuilder` 方法的返回值类型是 `ItalicsBuilder`，但方法调用方却只知道返回值遵守了 `MarkdownBuilder` 协议。从而也就达到了隐藏实现细节的目的；
    
    ```swift
    public func italicsBuilder() -> some MarkdownBuilder {
      ItalicsBuilder()
    }
    ```
  - 正是由于编译器需要明确确定 Opaque Types 背后的真实类型，故不能在 Opaque Types 方法中返回不同的类型值，如下面这样是不允许的 (Opaque Types 属于编译期特性)：
    
    ```swift
    public func italicsBuilder() -> some MarkdownBuilder {
      if ... {
        return ItalicsBuilder()
      }
      else {
        return BoldBuilder()
      }
    }
    ```

好了，现在我们知道只需在上述不加思索写出的代码中加入 `some`  关键字即可，不再赘述。

> 在 SwiftUI 中，大量使用到  Opaque Types。甚至可以说 Opaque Types 是为 SwiftUI 而生的。

# **Phantom Types**

---

Phantom Types 本身与本文讨论的内容相关性不大，作为相似的概念，我们简单介绍一下。

Phantom Types 也非 Swift 特有的，属于一种通用编码技巧。

Phantom Types 没有严格的定义，一般表述是：出现在泛型参数中，但没有被真正使用。

如下代码中的 `Role` (例子来自 [How to use phantom types in Swift](https://www.hackingwithswift.com/plus/advanced-swift/how-to-use-phantom-types-in-swift))，它只出现在泛型参数中，在 `Employee` 实现中并未使用：

```swift
struct Employee<Role>: Equatable {
    var name: String
}
```

What?

Phantom Types 有何用？

*用于对类型做进一步的强化。*

Employee 可能有不同的角色，如：Sales、Programmer 等，我们将其定义为空 enum：

```swift
enum Sales { }
enum Programmer { }
```

由于 `Employee` 实现了 `Equatable`，可以在两个实例间进行判等操作。

但判等操作明显只有在同一种角色间进行才有意义：

```swift
let john = Employee<Sales>.init(name: "John")
let sea = Employee<Programmer>.init(name: "Sea")

john == sea
```

正是由于 Phantom Types 在起作用，上述代码中的判等操作编译无法通过：

Cannot convert value of type 'Employee<Programmer>' to expected argument type 'Employee<Sales>'

> 将 Phantom Types 定义成空 enum，使其无法被实例化，从而真正满足 Phantom Types 语义。
> 
> 由于 Swift 没有 NameSpacing 这样的关键字，故通常用空 enum 来实现类似的效果，如 Apple Combine Framework 中的 `Publishers`:
```swift
public enum Publishers {}
```
> 然后在 extension 中添加具体 Publisher 类型的定义，如： 
```swift
extension Publishers {
  struct First<Upstream>: Publisher where Upstream: Publisher {
    ...
  }
}
```
> 从而，可以通过 `Publishers.First` 的方式引用具体的 Publisher。
> 
> 关于适当使用命名空间的好处在：[Five powerful, yet lesser-known ways to use Swift enums](https://www.swiftbysundell.com/articles/powerful-ways-to-use-swift-enums/) 中有一段精彩描述：
> 
> Using the above kind of namespacing can be a great way to add clear semantics to a group of types without having to manually attach a given prefix or suffix to each type’s name.
> 
> So while the above `First` type *could* instead have been named `FirstPublisher` and placed within the global scope, the current implementation makes it publicly available as `Publishers.First` — which both reads really nicely, and also gives us a hint that `First` is just one of many publishers available within the `Publishers` namespace. 
> 
> It also lets us type `Publishers.` within Xcode to see a list of all available publisher variations as autocomplete suggestions.

# 小结

---

Swift 作为 POP (Protocol Oriented Programming) 的提倡者，Protocol 的地位自然十分重要，Swift 赋于其强大能力。

同时，Swift 又是类型安全的，因此对于带有 `Self requirements` / `Associated Type` 的 Protocol 在使用上又有一定的限制。

结合实例，本文主要介绍了如何通过 Type Erasure、Opaque Types 以及 Generics 等方式解决上述限制。

在 [Opaque Return Types and Type Erasure](https://www.raywenderlich.com/24942207-opaque-return-types-and-type-erasure) 这篇文章中作者分别从库的开发者 (Liam)、编译器 (Corrine)、使用方 (Abbie) 的视角分析了他们是否了解 Protocols、Opaque Types、Generics 以及 Type Erasure 背后的私密：

![](/img/Protocols_OpaqueTypes_Generics_TypeErasure.png)

如上图：

- Protocols：
  
  - 协议本身具有隐藏实现细节以及运行时实例化的特性，故编译器、使用方无法知道其背后对应的真实类型；
  - 但，作为库的开发者 (代码是他写的)，明确知道 Protocol 背后可能对应的所有真实类型。

- Opaque Types：
  
  - 同 Protocols，库的开发者肯定是知道的；
  - 由于 Opaque Types 限制只能对应一种真实类型，并在编译期需明确，故编译器是知道的；
  - 对于使用方来说，他们看到的还是隐藏了细节的 Protocol。

- Generics：
  
  - 泛型是将类型决定权让给使用方的，故库的开发者是不知道真实类型的，而使用方知道；
  - 泛型属于编译期行为，故编译器能明确知道泛型对于的真实类型。

- Type Erasure：
  
  - 类型擦除属于使用方行为，用于规避编译错误等，故只有使用方知道。

# 参考资料

[swift-evolution · Opaque Result Types](https://github.com/apple/swift-evolution/blob/master/proposals/0244-opaque-result-types.md)

[OpaqueTypes](https://docs.swift.org/swift-book/LanguageGuide/OpaqueTypes.html)

[Different flavors of type erasure in Swift](https://www.swiftbysundell.com/articles/different-flavors-of-type-erasure-in-swift/#closures-to-the-rescue)

[Opaque Return Types and Type Erasure](https://www.raywenderlich.com/24942207-opaque-return-types-and-type-erasure)

[Phantom types in Swift](https://www.swiftbysundell.com/articles/phantom-types-in-swift/)

[How to use phantom types in Swift](https://www.hackingwithswift.com/plus/advanced-swift/how-to-use-phantom-types-in-swift)

[swift/TypeMetadata.rst at main · apple/swift · GitHub](https://github.com/apple/swift/blob/main/docs/ABI/TypeMetadata.rst#protocol-metadata)

[swift/TypeLayout.rst at main · apple/swift · GitHub](https://github.com/apple/swift/blob/main/docs/ABI/TypeLayout.rst)

[Swift Type Metadata](https://speakerdeck.com/kateinoigakukun/swift-type-metadata?slide=48)

[Understanding Swift Performance · WWDC2016](https://developer.apple.com/videos/play/wwdc2016/416/)

[Swift.org - Whole-Module Optimization in Swift 3](https://www.swift.org/blog/whole-module-optimizations/)