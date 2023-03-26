---
layout: post
title: Swift 最佳实践之 Optional
date: 2023-03-12 11:33:00
tags:
- iOS
- Swift
---

Swift 作为现代、高效、安全的编程语言，其背后有很多高级特性为之支撑。

『 Swift 最佳实践 』系列对常用的语言特性逐个进行介绍，助力写出更简洁、更优雅的 Swift 代码，快速实现从 OC 到 Swift 的转变。

该系列内容主要包括：

- Optional

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

> ps. 本系列不是入门级语法教程，需要有一定的 Swift  基础

本文是系列文章的第一篇，主要介绍 Optional。

Optional 作为 Swift 使用频率最高的特性之一，也是安全性的基石。除了常规的 `if`、`guard` 外还有不少高级特性，如：map、flatMap、Optional Pattern 等，通过它们可以写出更简洁、优雅的代码。

<!--more-->

<!--more-->

# Overview

---

可选类型 (Optional) 可以说是现代高级语言的标配，如：Kotin，Java，Javascript，Dart 等，Swift 也不类外。

在各种支持 Optional 的语言中，相关语法也非常类似。

定义 Optional 类型，最常用的语法是「 Type? 」，如：

```swift
let age: Int? = 20
```

「 Type? 」是语法糖，其完整语法是：`Optional<Type>`

如上 `age`的完整定义是：

```swift
let age: Optional<Int> = Optional.some(20)
```

可以看到，可选类型 `Optional` 是个「独立类型」，`Int` 与 `Int?` 有着本质的区别，不是一回事。

# Optional

---

```swift
@frozen public enum Optional<Wrapped> : ExpressibleByNilLiteral {

    /// The absence of a value.
    ///
    /// In code, the absence of a value is typically written using the `nil`
    /// literal rather than the explicit `.none` enumeration case.
    case none

    /// The presence of a value, stored as `Wrapped`.
    case some(Wrapped)

    /// Creates an instance that stores the given value.
    public init(_ some: Wrapped)

    // ...
}
```

如上，`Optional` 是个泛型 Enum，含有 2 个 case：

- `none` ：代表「空」，即 nil
  
  ```swift
  let age: Int? = nil
  ```
  
  等价于：
  
  ```swift
  let age: Optional<Int> = .none    // Optional.none
  ```

- `some` ：代表「非空」，关联具体的值
  
  ```swift
  let age: Int? = 20
  ```
  
  等价于：
  
  ```swift
  let age: Optional<Int> = .some(20)
  ```
  
  或：
  
  ```swift
  let age: Optional<Int> = .init(20)
  ```

## map

实现如下方法，加载 data：

你会怎么做？

```swift
func loadData(url: URL?) -> Data?
```

首先想到的：

```swift
func loadData(url: URL?) -> Data? {
  guard let url else {
    return nil
  }

  return try? Data.init(contentsOf: url)
}
```

还可以写得更优雅、简洁些：

```swift
func loadData(url: URL?) -> Data? {
  try? url.map { try Data.init(contentsOf: $0)}
}
```

`map` ？！

没错，Optional 实现了 `map` 方法：

```swift
public func map<U>(_ transform: (Wrapped) throws -> U) rethrows -> U? {
  switch self {
  case .some(let y):
    return .some(try transform(y))
  case .none:
    return .none
  }
}
```

通过 `map` 源码可以看到：

- 若 optional 不为 nil，执行闭包 `transform`

- 否则返回 nil

## flatMap

实现如下方法，将 String 类型的 url 转换成 URL：

```swift
func transformURL(_ url: String?) -> URL?
```

现学现用，愉快地写下：

```swift
func transformURL(_ url: String?) -> URL? {
  url.map { URL(string: $0) }
}
```

抱歉，编译错误：

`Value of optional type 'URL?' must be unwrapped to a value of type 'URL'`

原因在于，在方法 `transformURL` 中，根据类型推演，闭包 `map.transform` 的返回值类型为 `URL`，而非 `URL?`

那只能老老实实的：

```swift
func transformURL(_ url: String?) -> URL? {
  guard let url else {
    return nil
  }

  return URL(string: url)
}
```

非也！

```swift
func transformURL(_ url: String?) -> URL? {    
  url.flatMap { URL(string: $0) }
}
```

如上，Optional 还实现了 `flatMap`：

```swift
public func flatMap<U>(_ transform: (Wrapped) throws -> U?) rethrows -> U? {
  switch self {
  case .some(let y):
    return try transform(y)
  case .none:
    return .none
  }
}
```

`flatMap` 与 `map` 唯一的区别就是其 `transform` 闭包返回的泛型类型的可选类型

总之，`map`、`flatMap` 是 2 个非常方便的方法，可以写出更简洁优雅的代码。

## Equatable

```swift
extension Optional : Equatable where Wrapped : Equatable {
  public static func == (lhs: Wrapped?, rhs: Wrapped?) -> Bool {
    switch (lhs, rhs) {
    case let (l?, r?):
      return l == r
    case (nil, nil):
      return true
    default:
      return false
    }
  }
}
```

由于 `Optional` 有条件的 (`Wrapped : Equatable`) 实现了 `Equatable` 协议，故可以对两个符合条件的 `Optional` 类型直接进行判等操作

该运算符还可用于 non-optional 与 optional 间，其中 non-optional value 先自动转换成 optional，再进行判等操作，如：

```swift
let num: Int = 1
let numOptional: Int? = 1

if num == numOptional {
    // here!
}
```

## Optional Pattern

Pattern-Matching Operator (~=)，模式匹配运算符，主要用于 `case` 匹配

由于 Optional 实现了该运算符，故可以通过 `case` 语句判断 Optional 是否为 `nil`：

```swift
func transformURL(_ url: String?) -> URL? {
  switch url {
    case let url?:    // 等价于 case let .some(url):
      return URL(string: url)
    case nil:
      return nil
  }    
}
```

上面这种写法与直接用 `if`/`guard` 判断相比并没什么优势，更别说与 `flatMap` 版本相比了。

但对于一些组合的判断即十分方便，如上面提到的 `==` 运算符的实现：

```swift
public static func == (lhs: Wrapped?, rhs: Wrapped?) -> Bool {
  switch (lhs, rhs) {
  case let (l?, r?):
    return l == r
  case (nil, nil):
    return true
  default:
    return false
  }
}
```

如果用 `if` 判断改写：

```swift
public static func == (lhs: Wrapped?, rhs: Wrapped?) -> Bool {
  if let lhs, let rhs {
    return lhs == rhs
  }
  else if lhs == nil, rhs == nil {
    return true
  }
  else {
    return false
  }
}
```

很明显，`if` 版本的可读性比 `switch...case` 版本差。

`case` 不仅可以用于 `switch` ，还可以用于 `if`、`for`等，如：

```swift
let numOptional: Int? = 0
if case .none = numOptional {  // 等价于 if numOptional == nil
    // ...
}

if case let num? = numOptional {  //等价于 if let num = numOptional
    // ...
}
```

如下，打印 `scores` 中所有及格的分数，你会如何实现？

```swift
let scores: [Int?] = [90, nil, 80, 100, 40, 50, 60]
```

通过 `for case let` 可以写出很优雅的代码：

```swift
// 等价于 for case let .some(score) in scores where score >= 60
for case let score? in scores where score >= 60 {
  print(score)
}
```

> 用函数式形式也可以写出很优雅的代码
> 
> ```swift
> let passingScores = scores.compactMap { score in
>   score.flatMap {
>     $0 >= 60 ? $0 : nil
>   }
> }
> ```

> 关于 Pattern-Matching 的详细介绍将在后续专题文章中展开

# `if let` shorthand

---

本文所有关于 `if let` 的示例是否感觉有点怪，如：

```swift
public static func == (lhs: Wrapped?, rhs: Wrapped?) -> Bool {
  if let lhs, let rhs {
    // ...
  }
  // ...
}
```

其中，`if let` 我们熟悉的写法是不是：

```swift
if let lhs = lhs, let rhs = rhs {
    // ...
}
```

上述 2 种写法是等价的

第一个版本是 Swift 5.7 实现的新特性，用于简化「 Optional binding 」：[swift-evolution/0345-if-let-shorthand](https://github.com/apple/swift-evolution/blob/main/proposals/0345-if-let-shorthand.md)

对于所有条件控制语句都适用：

```swift
if let foo { ... }
if var foo { ... }

else if let foo { ... }
else if var foo { ... }

guard let foo else { ... }
guard var foo else { ... }

while let foo { ... }
while var foo { ... }
```

# 小结

Optional 本质上是 Enum，因此 `Int` 与 `Int?` 有着本质的区别。

Enum Optional 上定义了许多便捷的操作方法，如：`map`、`flatMap` 等，充分利用它们可以写出更优雅的代码。

Swift 5.7 简化了 Optional binding，`if let foo = foo {}` --> `if let foo {}`。

# 参考资料

[swift/Optional.swift](https://github.com/apple/swift/blob/aa3e5904f8ba8bf9ae06d96946774d171074f6e5/stdlib/public/core/Optional.swift)

[swift-evolution/0345-if-let-shorthand](https://github.com/apple/swift-evolution/blob/main/proposals/0345-if-let-shorthand.md)
