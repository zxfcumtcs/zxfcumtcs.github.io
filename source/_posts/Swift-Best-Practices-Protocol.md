---
layout: post
title: Swift 最佳实践之 Protocol
date: 2023-03-26 21:30:23
tags:
- iOS
- Swift
---

Swift 作为现代、高效、安全的编程语言，其背后有很多高级特性为之支撑。

『 Swift 最佳实践 』系列对常用的语言特性逐个进行介绍，助力写出更简洁、更优雅的 Swift 代码，快速实现从 OC 到 Swift 的转变。

该系列内容主要包括：

- [Optional](https://juejin.cn/post/7212101192746696760)

- [Enum](https://juejin.cn/post/7212143399036813349)

- [Closure](https://juejin.cn/post/7214665617310892092)

- Protocol

- Generic

- Property Wrapper

- Structured Concurrent

- Result builder

- Error Handle

- Advanced Collections (Asyncsequeue/OptionSet/Lazy)

- Expressible by Literal

- Pattern Matching

- Metatypes(.self/.Type/.Protocol)

> ps. 本系列不是入门级语法教程，需要有一定的 Swift 基础

本文是系列文章的第四篇，介绍  Protocol。



# Overview

---

Protocol 基本语法在此就不赘述，不熟悉的话建议看看官方文档：[Documentation · Protocols](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/)



Swift 号称 POP，为 Protocol 提供了强大的能力，如 Extension、Default Implementation 等，深受开发者喜爱。

将 Protocol 用作类型时，需要装箱且方法是动态派发，有一定的性能损耗。

借助 Associated Type 使得 Protocol 更加灵活，但又不失类型安全，值得拥有👍。

对，本文主要就是讨论👆这些话题 。

# Extension & Default Implementation

---

## Extension

Swift Protocol 可以像 Class/Structure/Enum 那样提供 Extension。

在 Extension 中可以实现方法、计算属性等：

- 既可以在 Extension 中实现 Protocol 要求的方法

- 还可以实现未出现在 Protocol 定义中的方法

## Default Implementation

在扩展中实现协议要求的方法 (requirement of protocol)，即为其提供默认实现。从而实现该协议的 Class/Structure/Enum 就可以直接使用默认实现：

```swift
protocol ProtocolDemo {
  func foo()
}

extension ProtocolDemo {
  func foo() {
    print("foo: in Extension")
  }
}

// 正是由于 ProtocolDemo 提供了 foo 的默认实现
// 故，ImplementDemo 可以不实现 foo，编译没问题
//
struct ImplementDemo: ProtocolDemo {}
```

## Quiz

来个小测验 🫣，如下这段代码的结果是❓

- Compiler Error？

- Crash？

- ?

![](/img/protocol-extension-implementation.png)

正确答案 😇：

**bar: in Extension**

**foo: in ImplementDemo**

**bar: in ImplementDemo**

**foo: in ImplementDemo**

**这个呢？🥱🧐**

Compiler Error❓ Crash❓

![](/img/protocol-extension-implementation-1.png)

正确答案：

**bar: in Extension**

**foo: in ImplementDemo**

**bar: in Extension**

**foo: in ImplementDemo**

**Why❓🫢 🫥**

对于 `foo` 的行为应该好理解，就是具体实现 (`ImplementDemo.foo`) 覆盖默认实现。

对于 `bar` 的行为就不那么好理解了 🤯，但似乎可以总结出一些规律。

对于不是协议要求的方法 (non-requirement of protocol)，即只在 protocol extension 中定义的方法：

- 对于用协议类型定义的实例 (如，`protocolDemo`)，调用的**一定是 protocol extension 中的实现**

- 对于用具体类型定义的实例 (如，`implementDemo`)，情况较复杂：
  
  - 若具体类型也实现了该方法 (如，`ImplementDemo.bar`)，则调用的是该实现
  
  - 否则调用 protocol extension 中的实现
  
  > ⚡️⚡️⚡️ 避免重写 Protocol extension 中定义的 non-requirement 方法！

😢 🤯 🤔 😵‍💫 🤢❓❗️

`protocolDemo` 与 `implementDemo` 有何不同❓

不都是 `ImplementDemo` 的实例吗❓

这就需要好好说说「协议作为类型」用了。

# Existential Type

---

如上节所述，Protocol 可以作为类型用，用于定义变量：

```swift
var protocolDemo: ProtocolDemo
```



继续做题吧 🤮

分别用 Class、Struct 实现了 `ProtocolDemo` 协议：

![](/img/protocol-implement-size.png)

正确答案：

**protocolClassDemo = 40**

**classDemo = 8**

**protocolStructDemo = 40**

**structDemo = 48**

这个呢❓😵‍💫 🙈，给 `ProtocolDemo` 加了只能用于 Class 的限制 (`AnyObject`)：

![](/img/protocol-implement-class-size.png)

正确答案：

**protocolClassDemo = 16**

**classDemo = 8**

**有没有感受到「 Protocol as Type 」的复杂性！😱**

「 Protocol as Type 」有个专有名称:「 *Existential Type* 」

> Protocols don’t actually implement any functionality themselves. Nonetheless, you can use protocols as a fully fledged types in your code. Using a protocol as a type is sometimes called an *existential type*, which comes from the phrase “**there exists a type *T* such that *T* conforms to the protocol**”.
> 
> -- [Swift Docs · Protocols](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/#Protocols-as-Types)

在 [Swift Protocol 背后的故事(理论)](https://juejin.cn/post/7061891716931911687) 中，我们详细介绍了「 Existential Type 」背后的实现机制：

![](/img/ExistentialContainer-Desc.png)

简单总结一下：

- Existential Type 与对应的 Normal Type (如：`protocolClassDemo` 与 `classDemo`) 根本不是一回事

- 对于 Existential Type，Swift 对其做了一层封装 (**Existential Container**)，作为 Protocol  的「模型」

- non-class constraint protocol 与 class constraint protocol (`: AnyObject`) 的 Existential Container 不一样
  
  ```swift
  // for non-class constraint protocol
  //
  struct OpaqueExistentialContainer {
    void *fixedSizeBuffer[3];
    Metadata *type;
    WitnessTable *witnessTables[NUM_WITNESS_TABLES];
  };
  ```
  
  ```swift
  // for class constraint protocol
  //
  struct ClassExistentialContainer {
    HeapObject *value;
    WitnessTable *witnessTables[NUM_WITNESS_TABLES];
  };
  ```

正是由于，从 Normal Type 到 Existential Type 需要做一次封装转换 (装箱)，在性能上有一定的损耗。

```swift
// normal type -> protocol type
let protocolClassDemo: ProtocolDemo = ClassDemo()
```

同时，通过 Existential Type 发起的方法调用都是动态派发 (在 extension 中定义的 non-requirement 方法除外)，也有一定的性能损耗。

**总之，Existential Type 有性能损耗，需要尽量避免使用❗️⚡️：**

- 用泛型代替 Existential Type：
  
  ```swift
  protocol Animal {
    func eat()
  }
  
  struct Farm {
    // 从使用角度看 genericsFeed 与 existentialFeed 效果是一样的
    // 性能上 genericsFeed 更优
    // 但 existentialFeed 更简洁，写起来更方便
    func genericsFeed<T>(_ animal: T) where T: Animal {
      animal.eat()
    }
  
    func existentialFeed(_ animal: Animal) {
      animal.eat()
    }
  }
  ```

-  正是由于 Existential Type 使用太方便了，经常有意无意的就用上了
  
  为此，Swift 5.6 引入了 `any` 关键字 ([swift-evolution/0335-existential-any](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fapple%2Fswift-evolution%2Fblob%2Fmain%2Fproposals%2F0335-existential-any.md "https://github.com/apple/swift-evolution/blob/main/proposals/0335-existential-any.md")) 用于标记「Existential Type」：
  
  ```swift
  //                     👇
  let protocolClassDemo: any ProtocolDemo = ClassDemo()
  ```
  
  主要目的是显式提醒开发人员正在使用『 Existential Type 』
  
  目前 `any` 还不是强制的，但从 Swift 6.0 开始将强制使用 `any`，否则编译报错。因此，尽早用上 `any`，以免后期升级成本过高。
  
  > 关于对 `any` 的介绍可以参看之前的文章：[Swift Protocol 背后的故事(Swift 5.6/5.7)](https://juejin.cn/post/7115050654170628127/)

有个大胆的想法 🤓：能否*既有泛型的性能又有『 Existential Types 』的简洁？*

答案是肯定的，那就是：「 **Opaque Type** 」、「 **Opaque Parameter** 」：

简单讲，就是当 Protocol 作为类型时，可以在其前面加上 `some` 关键字，如：

```swift
struct Farm {  
  func genericsFeed<T>(_ animal: T) where T: Animal {
    animal.eat()
  }

  // some Animal 可以理解为一个匿名的具体类型
  // 并且该类型实现了 Animal 协议
  //                       👇
  func someFeed(_ animal: some Animal) {
    animal.eat()
  }
}

// 如上，`someFeed` 在性能上与 `genericsFeed` 无任何差别，但更简洁、可读性更好
// Opaque Parameter 可以理解为泛型的简化版本
```

再来猜个题吧 😂

![](/img/protocol-implement-class-some-size.png)

正确答案：

**protocolClassDemo = 8**

**classDemo = 8**

> 关于 Opaque Types 的详细介绍请参看 [Swift Protocol 背后的故事(实践)](https://juejin.cn/post/7061874172560949279)

> 关于 Opaque Parameter 的详细介绍请参看 [Swift Protocol 背后的故事(Swift 5.6/5.7)](https://juejin.cn/post/7115050654170628127/)

> `some` 关键字需要在 (iOS 13 & Swift 5.1/Swift 5.7) 以上才可以用 🐶😭，`any` 只需 Swift 5.6 即可 👍

# Associated Type

---

从一个小任务开始 🧐：定义一个 Collection Protocol (`BetterCollection`)，要求：

- 支持 「 增、删、查 」

- Collection 中所有元素类型必须一致

看似很简单：

```swift
protocol BetterCollection {
  mutating func append(_ element: ❓)
  mutating func remove(_ element: ❓)
  subscript(i: Int) -> ❓ { get }
}
```

Collection 中元素的类型怎么写❓❗️

`Any` 🤔❓

无法满足「 **Collection 中所有元素类型必须一致** 」的要求！🥲

此时，就需要 Associated Type 登场了：

```swift
protocol BetterCollection {
  associatedtype Element    // 👈
  mutating func append(_ element: Element)
  mutating func remove(_ element: Element)
  subscript(i: Int) -> Element { get }
}
```

- `associatedtype` 关键字用于定义 Associated Type

- Associated Type 可能理解为是一个类型占位符，其具体值则在实现该协议时确定
  
  ```swift
  class MyElement {}
  
  struct MyArray: BetterCollection {
    typealias Element = MyElement  // 👈，可省略
    var elements: [MyElement] = []
    mutating func append(_ element: MyElement) {}
    mutating func remove(_ element: MyElement) {}
    subscript(i: Int) -> MyElement { MyElement() }
  }
  ```
  
  正常情况下，在实现协议时通过 `typealias associatedType = ***` 确定associatedtype 的具体值
  
  但得益于 Swift 强大的类型推演能力，一般情况下不用显式写 `typealias associatedType = ***`

不错，任务圆满完成✌️

but，从语义上说 `mutating func remove(_ element: Element)` 方法要求 `Element` 实现 `Equatable` 协议，即可以判等(`==`)

问题不大，可以通过给 Associated Type 添加约束来实现

## Adding Constraints to an Associated Type

```swift
protocol BetterCollection {
  associatedtype Element: Equatable  // 👈，要求 Element 实现 Equatable 协议
  mutating func append(_ element: Element)
  mutating func remove(_ element: Element)
  subscript(i: Int) -> Element { get }
}
```

此时，如果具体类型不满足 associatedtype 的约束，当然通不过编译了：

![](/img/associatedType-constraints-error.png)

这样就完美了🤓：

![](/img/BetterCollection-associatedtype-equatable.png)

别高兴的太早，还有新任务 🧐

给 `BetterCollection` 增加批量 append 元素的接口：

```swift
mutating func append(contentsOf elements: ❓)  // 参数 contentsOf 的类型？
```

上述 `append` 参数 `contentsOf` 的类型应该满足：

- 实现 `BetterCollection` 协议

- 元素类型须一致 (`contentsOf`中元素类型与 self 中元素类型要相同)

有两种实现方式：

- Generic + Associated Type Constraints
  
  ```swift
  protocol BetterCollection {  
    mutating func append<T: BetterCollection>(contentsOf elements: T) where T.Element == Element
  }
  
  struct MyArray: BetterCollection {  
    mutating func append<T>(contentsOf elements: T) where T : BetterCollection, MyElement == T.Element {}
  }
  ```

- Self requirements
  
  ```swift
  protocol BetterCollection {
    mutating func append(contentsOf elements: Self)
  }
  
  struct MyArray: BetterCollection {
    //                                          👇
    mutating func append(contentsOf elements: MyArray) {}
  }
  ```




从上述不同版本 `MyArray.append` 的定义可以看出它们间还是有一定区别的：

- Generic 版本只要求参数 `contentsOf` 的类型实现 `BetterCollection` 协议且两者的 associatedtype 相同即可

- Self requirements 版本要求参数 `contentsOf` 的类型与 Self 相同
  
  > 关于 Self requirements，最著名的应用恐怕就是：
  > 
  > ```swift
  > public protocol Equatable {
  >   // 判等的 2 个类型必须相同
  >   static func == (lhs: Self, rhs: Self) -> Bool
  > }
  > ```

应该说，Generic 版本更灵活，应用场景更广，但写起来有点麻烦，有没有更简洁的版本？😉

答案是肯定的

## Primary Associated Type

为了解决上一小节提到的 Generic 版本复杂的问题，Swift 5.7 引入了「 Primary Associated Type 」的概念 ([swift-evolution/0346-light-weight-same-type-syntax](https://github.com/apple/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md))

通过「 Primary Associated Type 」可以改写上面的 Generic 版本，代码更简洁：

```swift
//                          👇
protocol BetterCollection<Element> {
  associatedtype Element: Equatable
  //                                                                👇
  mutating func append(contentsOf elements: some BetterCollection<Element>)
}

struct MyArray: BetterCollection {
  //                                                                  👇
  mutating func append(contentsOf elements: some BetterCollection<MyElement>) {}
}
```

关于「 Primary Associated Types 」的详细介绍可以参看 [Swift Protocol 背后的故事(Swift 5.6/5.7) ](https://juejin.cn/post/7115050654170628127/#heading-5)

至此，`BetterCollection` 似乎很「完美」了：

```swift
protocol BetterCollection<Element> {
  associatedtype Element: Equatable
  mutating func append(_ element: Element)
  mutating func remove(_ element: Element)
  subscript(i: Int) -> Element { get }

  mutating func append(contentsOf elements: some BetterCollection<Element>)
}
```

but，`MyArray` 不怎么「完美」🙈，其中的元素只能是 `MyElement` 类型！

## Generic + Associated Type

可以将 `MyArray` 定义为 Generic，再联合 Protocol Associated Type，完美 ✌️：

```swift
// 将泛型类型 T 与 BetterCollection 的 associatedtype 相绑定
// 
struct MyArray<T: Equatable>: BetterCollection {

  var elements: [T] = []
  mutating func append(_ element: T) {}

  mutating func remove(_ element: T) {
    let index = elements.firstIndex { $0 == element }
    guard let index else {
      return
    }
    elements.remove(at: index)
  }

  subscript(i: Int) -> T { elements[i] }

  mutating func append(contentsOf elements: some BetterCollection<T>) {}
}
```

> 将有 associatedtype 或 Self-requirements 的协议作为类型用时可能会遇到点麻烦🫢⚡️
> 
> 比如这样的编译错误：
> 
> 「 Swift 5.7 」: Member 'append' cannot be used on value of type 'any BetterCollection'; consider using a generic constraint instead
> 
> 「 Swift ~5.6」: Protocol 'BetterCollection' can only be used as a generic constraint because it has Self or associated type requirements.
> 
> 详细信息请参看：[Swift Protocol 背后的故事(实践)](https://juejin.cn/post/7061874172560949279)、[Swift Protocol 背后的故事(Swift 5.6/5.7)](https://juejin.cn/post/7115050654170628127#heading-4)

# Other

---

## Automatically Synthesized Protocol Implementation

Swift 在满足一定条件时，会自动合成某些协议的实现，即不用手动写实现了 👍：

- `Equatable`：
  
  - Struct 的所有存储属性都(自动/手动)实现了 `Equatable`
  
  - 没有关联值的 Enum
  
  - Enum 关联值的类型(自动/手动)实现了 `Equatable`

- `Hashable`：
  
  - Struct 的所有存储属性都(自动/手动)实现了 `Hashable`
  
  - 没有关联值的 Enum
  
  - Enum 关联值的类型(自动/手动)实现了 `Hashable`

- `Codable`：
  
  - Class/Struct 的所有存储属性都(自动/手动)实现了 `Codable`
  
  - 没有关联值的 Enum
  
  - Enum 关联值的类型(自动/手动)实现了 `Codable`

> `Equatable`、`Enum` 只对 Struct、Enum 会自动合成
> 
> `Codable` 对 Class、Struct、Enum 都会自动合成

## Class-Only-Protocols

有些 Protocol 要求其实现必须是 Class

如，需要弱引用的 delegate：

```swift
weak var delegete: Delegate?
```

在定义协议时可以加上 `AnyObject`：

```swift
protocol Delegate: AnyObject {}
```

## Constraints

在 Protocol 中有各种不同的 Constraints，如：

- associatedtype constraint，在定义 associatedtype 时指定约束：
  
  ```swift
  associatedtype Element: Equatable
  ```

- constraints between associatedtypes，当有多个 associatedtypes 时，可以在它们间指定约束，如 Swift 标准库中 `Sequence` 的定义：
  
  ```swift
  public protocol Sequence<Element> {
  
      /// A type representing the sequence's elements.
      associatedtype Element where Self.Element == Self.Iterator.Element
  
      /// A type that provides the sequence's iteration interface and
      /// encapsulates its iteration state.
      associatedtype Iterator : IteratorProtocol
  }
  ```

- method constraint，在声明方法时指定约束：
  
  ```swift
  mutating func append<T: BetterCollection>(contentsOf elements: T) where T.Element == Element
  ```

- Self requirements，要求相同的实现类型：
  
  ```swift
  static func == (lhs: Self, rhs: Self) -> Bool
  ```

- extension constraint，有条件的对 Protocol extension：
  
  ```swift
  extension BetterCollection where Element: Codable {
   // ... 
  }
  ```

- extension constraint on Self，在扩展协议时还可以对 Self 添加约束
  
  ```swift
  extension BetterCollection where Self: UIView {}
  ```

- primary associatedtype constraint，通过 primary associatedtype 可以简化 extension constraint 的写法：
  
  ```swift
  extension BetterCollection<Codable> {
    // ...
  }
  
  // Equivalent to:
  
  extension BetterCollection where Element: Codable {
   // ... 
  }
  ```

- inherit constraint，有条件的继承：
  
  ```swift
  protocol CodableBetterCollection: BetterCollection where Element: Codable {}
  
  struct MyArray<T: Equatable & Codable>: CodableBetterCollection {
   // ...
  }
  ```

- conditionally conforming to a protocol，有条件的实现某个协议
  
  如下代码编译报错，原因在于泛型类型 `T` 并没有实现 `Codable` 协议
  
  ![](/img/MyArray-Codable-Error.png)
  
  给 `MyArray` 的扩展加个约束即可：
  
  ```swift
  extension MyArray: Codable where T: Codable {}
  ```

# 小结

Swift Protocol 能力非常强大，通过 extension 提供协议的默认实现，有助于开发效率的提升。

将 Protocol 用于类型时，由于需要装箱 (Existential Container) 以及方法需要动态派发，有一定的性能损耗。通过 Generic 或 Opaque Types(`some`) 可以避免性能问题。

Associated Type 提升了 Protocol 的灵活性，但又不失类型安全。

Associated Type + Generic 更是如虎添翼💥🔥



# 参考资料

[Swift Protocol 背后的故事(实践)](https://juejin.cn/post/7061874172560949279)

[Swift Protocol 背后的故事(理论)](https://juejin.cn/post/7061891716931911687)

[Swift Protocol 背后的故事(Swift 5.6/5.7)](https://juejin.cn/post/7115050654170628127)

[Documentation · Protocols](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/)

[Documentation · Generics · Associated-Types](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/generics/#Associated-Types)

[swift-evolution/0346-light-weight-same-type-syntax](https://github.com/apple/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md)

[swift-evolution/0358-primary-associated-types-in-stdlib](https://github.com/apple/swift-evolution/blob/main/proposals/0358-primary-associated-types-in-stdlib.md)

[Getting started with associated types in Swift Protocols](https://www.avanderlee.com/swift/associated-types-protocols/)
