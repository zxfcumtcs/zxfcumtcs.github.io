---
layout: post
title: Swift 最佳实践之 Property Wrapper
date: 2023-04-09 22:09:05
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

- [Protocol](https://juejin.cn/post/7217360688263036985)

- [Generics](https://juejin.cn/post/7219908995338731575)

- Property Wrapper

- Error Handling

- Advanced Collections (Asyncsequeue/OptionSet/Lazy)

- Expressible by Literal

- Pattern Matching

- Metatypes(.self/.Type/.Protocol)

> ps. 本系列不是入门级语法教程，需要有一定的 Swift 基础

本文是系列文章的第六篇，介绍在 Swift 5.1 引入的 Property Wrapper ([Swift-Evolution 0258 · Property Wrappers](https://github.com/apple/swift-evolution/blob/main/proposals/0258-property-wrappers.md))。

# 初识 Property Wrapper

---

> A property wrapper adds a layer of separation between code that manages how a property is stored and the code that defines a property. 
> 
> -- [Swift Dosc · Property-Wrappers](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/properties/#Property-Wrappers)

简单讲，Property Wrapper 是对属性的一层封装，隐藏与属性相关的逻辑细节，提高代码的复用性。

```swift
// 定义 Property Wrapper
// SomePropertyWrapper 可以是 struct、enum 或 class
//
@propertyWrapper
struct SomePropertyWrapper {
  var wrappedValue: Int
}

class SomeClass {
  // 使用 Property Wrapper
  //       👇
  @SomePropertyWrapper var a: Int = 1
}
```

如上，定义了一个最简单的 Property Wrapper `SomePropertyWrapper`，几个关键点：

- `@propertyWrapper`，用于定义 Property Wrapper

- Property Wrapper 的具体类型可以是 class、struct 或 enum (很少用)

- Property Wrapper 必须有一个名为 `wrappedValue` 的属性

- 使用时，通过 `@PropertyWrapperName` 标记相关属性

有何用❓

假如，我们要定义一个 struct，用于存储 RBG 色值：

```swift
struct RGB {
  let r: Int
  let g: Int
  let b: Int

  init(r: Int, g: Int, b: Int) {
    self.r = max(0, min(255, r))
    self.g = max(0, min(255, g))
    self.b = max(0, min(255, b))
  }
}
```

由于 `r`、`g`、`b` 的有效取值范围为：[0, 255]

故，显式实现了 `init` 方法，并对每个值做了保护

如果用 PropertyWrapper 实现：

```swift
@propertyWrapper
struct RGBValue {
  var value: Int = 0

  var wrappedValue: Int {
    get {
      value
    }
    set {
      value = max(0, min(255, newValue))
    }
  }
}

struct RGB {
  @RGBValue var r: Int
  @RGBValue var g: Int
  @RGBValue var b: Int
}
```

如上，对 `r`、`g`、`b` 取值保护的逻辑封装到了 Property Wrapper 中

有利于代码复用、逻辑封装

## 内幕

用 Property Wrapper 标记的属性 (如上述 `a`) 编译器会自动合成相关代码：

```swift
class SomeClass {
  @SomePropertyWrapper var a: Int = 1
}

// compiler synthesizes pseudo code ==>

class SomeClass {
  //          👇
  private var _a = SomePropertyWrapper(wrappedValue: 1)

  //  👇
  var a: Int {
    get {
      _a.wrappedValue
    }
    set {
      _a.wrappedValue = newValue
    }
  }
}
```

- 编译器会生成一个 PropertyWrapper 类型的「**存储属性**」(如上 `_a`)

- 用 PropertyWrapper 标记的属性，实际上是个「**计算属性**」，是对 PropertyWrapper `wrappedValue` 属性的代理

## Initial

如上节所述，PropertyWrapper 一定是在定义时完成初始化的，如：

```swift
@SomePropertyWrapper var a: Int    // no initial value

// ---->

private var _a = SomePropertyWrapper()
```

```swift
@SomePropertyWrapper var a: Int = 1    // has initial value

// ---->

private var _a = SomePropertyWrapper(wrappedValue: 1)
```

 如上，根据有没有提供初始值分别调用不同的 `init` 方法：

- 没提供初始值时，调用 PropertyWrapper 的 `init()` 方法

- 提供了初始值，则调用 `init(wrappedValue:)` 方法

故，需要确保对应的 `init` 方法存在，否则编不过

除了，`init()`、`init(wrappedValue:)` ，还可以提供更多自定义 init 方法，如：

```swift
@propertyWrapper
struct RGBValue {
  var value: Int = 0

  var minValue: Int
  var maxValue: Int

  init(minValue: Int, maxValue: Int) {
    self.minValue = minValue
    self.maxValue = maxValue
  }

  init(wrappedValue: Int, minValue: Int, maxValue: Int) {
    self.minValue = minValue
    self.maxValue = maxValue
    value = max(minValue, min(wrappedValue, maxValue))
  }

  var wrappedValue: Int {
    // ...
  }
}

struct RGB {
  // 调用 init(minValue: Int, maxValue: Int)
  //
  @RGBValue(minValue: 100, maxValue: 200) var r: Int

  // 调用 init(wrappedValue: Int, minValue: Int, maxValue: Int)
  //
  @RGBValue(wrappedValue: 10, minValue: 0, maxValue: 100) var g: Int

  // 调用 init(wrappedValue: Int, minValue: Int, maxValue: Int)
  // 注意 ⚡️：对于这种写法，wrappedValue: 必须是 init 的第一个参数
  //
  @RGBValue(minValue: 200, maxValue: 255) var b: Int = 2
}
```

## Projected Value

Property Wrapper 除了对外曝露 `wrappedValue`，还可以曝露一个值，称之为 **Projected Value**，如：

```swift
@propertyWrapper
struct RGBValue {
  var value: Int = 0

  //                     👇
  private(set) var projectedValue: Bool = false

  var wrappedValue: Int {
    get {
      value
    }
    set {
      projectedValue = newValue <= 255 && newValue >= 0
      value = max(0, min(255, newValue))
    }
  }
}

struct RGB {
  @RGBValue var r: Int
  @RGBValue var g: Int
  @RGBValue var b: Int

  func someFunc() {
    //      👇
    let x = $r
  }
}
```

- `projectedValue` 可以是任意类型

- `projectedValue` 可以是存储属性，也可以是计算属性

- 使用时，只需在原始属性名前加上`$` (如 `$r`)

对于 Projected Value 更常见的用法是，直接返回 `self`，并借此调用 Property Wrapper 提供的辅助能力，如：

```swift
@propertyWrapper
struct RGBValue {
  var value: Int = 0

  //                              👇
  var projectedValue: RGBValue { self }

  var wrappedValue: Int {
    get {
      value
    }
    set {
      value = max(0, min(255, newValue))
    }
  }

  //  👇
  var hex: String {
    String(format:"%02X", value)
  }
}

struct RGB {
  @RGBValue var r: Int
  @RGBValue var g: Int
  @RGBValue var b: Int

  func hexRGB() -> String {
    //    👇       👇       👇
    "#\($r.hex)\($g.hex)\($b.hex)"
  }
}
```

如上，`RGBValue` 将 `projectedValue` 定义为同类型的属性，并返回 `self`

同时，`RGBValue` 提供了转十六进制的辅助计算属性：`hex`

使用时，通过 `projectedValue` 就可以访问到 `hex` (`$r.hex`)

# 应用

---

关于 Property Wrapper 的应用，「只有想不到，没有做不到」，是一个充满想象力和创造力的地方！

## SwiftUI

SwiftUI 提供了大量的 Property Wrapper，可以说 Property Wrapper 是为 SwiftUI 而生，离开它们寸步难行，如：

- @State
- @Binding
- @StateObject
- @ObservedObject
- @EnvironmentObject
- ...

## Protected

线程安全是一个常见且处理繁碎、容易出错的问题

通过 Property Wrapper 可以很好地将这些逻辑封装起来，极大简化了业务上的处理

[GitHub - Alamofire](https://github.com/Alamofire/Alamofire) 中提供了相关的 Property Wrapper (代码略有删简)：

```swift
/// A thread-safe wrapper around a value.
@propertyWrapper
final class Protected<T> {
    private let lock = UnfairLock()
    private var value: T

    init(_ value: T) {
        self.value = value
    }

    var wrappedValue: T {
        get { lock.around { value } }
        set { lock.around { value = newValue } }
    }

    //                                  👇
    var projectedValue: Protected<T> { self }

    init(wrappedValue: T) {
        value = wrappedValue
    }

    func read<U>(_ closure: (T) throws -> U) rethrows -> U {
        try lock.around { try closure(self.value) }
    }

    @discardableResult
    func write<U>(_ closure: (inout T) throws -> U) rethrows -> U {
        try lock.around { try closure(&self.value) }
    }
}
```

对于需要线程安全保护的属性，在定义时只需加上 `@Protected` 即可，lock/unlock 之类的问题一律不用操心：

```swift
@Protected var validators: [() -> Void] = []
```

同时，将 `projectedValue` 指向 `self`，并提供了 `read`、`write` 辅助方法，可以将「一大块」代码保护起来，如：

```swift
//                  👇
$multipartFormData.read { multipartFormData in
  //                                        👇
  urlRequest.headers.add(.contentType(multipartFormData.contentType))
}
```

```swift
$mutableState.write { state in
  state.listenerQueue = queue
  state.listener = listener
}
```

## Codable

Swift 提供了原生的 JSON 解析能力 `Codable`，但也有一些限制，如不能提供默认值、Lossless value 转换 (如 JSON 里是个 `String`，但 Model 中是个 `Int`)、Array 解析时只要有一个元素解析失败整个 Array 解析就失败等。

这些问题都需要通过手动方式解决，不够友好

[GitHub - BetterCodable](https://github.com/marksands/BetterCodable) 通过 Property Wrapper 较好地解决了这些问题，如：

```swift
struct Response: Codable {
    @LosslessValue var sku: String
    @LosslessValue var isAvailable: Bool
}

let json = #"{ "sku": 12345, "isAvailable": "true" }"#.data(using: .utf8)!
let result = try JSONDecoder().decode(Response.self, from: json)

print(result) // Response(sku: "12355", isAvailable: true)
```

## User defaults

开发中有不少场景需要使用 `UserDefaults` 存储信息，相关的读写操作可以封装到 Property Wrapper 中，[如](https://github.com/apple/swift-evolution/blob/main/proposals/0258-property-wrappers.md#user-defaults)：

```swift
@propertyWrapper
struct UserDefault<T> {
  let key: String
  let defaultValue: T

  var wrappedValue: T {
    get {
      return UserDefaults.standard.object(forKey: key) as? T ?? defaultValue
    }
    set {
      UserDefaults.standard.set(newValue, forKey: key)
    }
  }
}

enum GlobalSettings {
  @UserDefault(key: "FOO_FEATURE_ENABLED", defaultValue: false)
  static var isFooFeatureEnabled: Bool

  @UserDefault(key: "BAR_FEATURE_ENABLED", defaultValue: false)
  static var isBarFeatureEnabled: Bool
}
```

## Clamping

如上介绍的 `RGBValue`，日常开发中有很多值有有效取值区间，如：RGB、age、 weekday、fps 等

可以将有效取值区间封装到一个 Property Wrapper 中，如：

```swift
@propertyWrapper
struct Clamping<WrappedValue: Comparable> {
  let range: ClosedRange<WrappedValue>
  var value: WrappedValue

  init(wrappedValue value: WrappedValue, _ range: ClosedRange<WrappedValue>) {
    self.value = value
    self.range = range
  }

  var wrappedValue: WrappedValue {
    get { value }
    set { value = min(max(range.lowerBound, newValue), range.upperBound)}
  }
}


struct RGB {
  @Clamping(0...100) var r: Int = 0
  @Clamping(0...255) var g: Int = 0
  @Clamping(100...255) var b: Int = 255
}
```

...

# 限制

用 Property Wrapper 标记的属性，有一些限制：

- 只能是 var，不能是 let

- 不能是 lazy

- 不能是 weak

# 小结

合理的封装 Property Wrapper ，可以提升代码的复用性，以及简化业务使用。

# 参考资料

[Swift Dosc · Property-Wrappers](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/properties/#Property-Wrappers)

[Swift-Evolution 0258 · Property Wrappers](https://github.com/apple/swift-evolution/blob/main/proposals/0258-property-wrappers.md)

[Swift Property Wrappers - NSHipster](https://nshipster.com/propertywrapper/)

[Property wrappers in Swift](https://www.swiftbysundell.com/articles/property-wrappers-in-swift/)

[What is a Property Wrapper in Swift](https://sarunw.com/posts/what-is-property-wrappers-in-swift/)
