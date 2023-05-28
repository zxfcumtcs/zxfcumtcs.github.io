---
layout: post
title: Swift 最佳实践之 Advanced Collections
date: 2023-05-20 10:39:10
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

- [Property Wrapper](https://juejin.cn/post/7222189908429275173)

- [Error Handling](https://juejin.cn/post/7232924659960037436)

- Advanced Collections

- Pattern Matching

- Metatypes(.self/.Type/.Protocol)

- High Performance

> ps. 本系列不是入门级语法教程，需要有一定的 Swift 基础

本文是系列文章的第八篇，主要介绍 Swift 中一些高级集合类型，如：OptionSet、LazySequence、Range、ArraySlice、Substring 等。

# Overview

---

充分利用 OptionSet、LazySequence、Range、ArraySlice、Substring 等集合的特性可以写出更优雅、高效的代码。

进一步了解其背后的实现机制对代码健壮性也十分重要。

本文将对这些集合的常用特性以及背后实现机制展开详细介绍。

# OptionSet

---

任务：设计一个方法给 View 设置圆角，要求可以通过参数指定哪些角需要设成圆角(LeftTop/RightTop/LeftBottom/RightBottom)

接口该如何设计？

关键是需要设成圆角的「角」怎么传递？

enum？似乎不合适

此正是 `OptionSet` 的用武之地

```swift
//                            👇
public struct RectCorner: OptionSet {
  public let rawValue: Int

  public init(rawValue: Int) {
    self.rawValue = rawValue
  }

  //       👇
  public static let leftTop = RectCorner(rawValue: 1 << 0)
  public static let rightTop = RectCorner(rawValue: 1 << 1)

  public static let leftBottom = RectCorner(rawValue: 1 << 2)
  public static let rightBottom = RectCorner(rawValue: 1 << 3)

  public static let all: RectCorner = [.leftTop, .rightTop, .leftBottom, .rightBottom]
}

extension UIView {
  func corner(_ corners: RectCorner, radius: CGFloat) -> UIView {
    if corners.contains(.leftTop) {
      // ...
    } else if corners.contains(.rightTop) {
      // ...
    } else if corners.contains(.leftBottom) {
      // ...
    } else if corners.contains(.rightBottom) {
      // ...
    }
  }
}

view.corner([.leftTop, .rightBottom], radius: 5)
```

- `OptionSet` 是个协议
  
  ```swift
  public protocol OptionSet : RawRepresentable, SetAlgebra {
      associatedtype Element = Self
      init(rawValue: Self.RawValue)
  }
  ```
  
  `SetAlgebra` 中定义了所有跟 `Set` 相关的操作：
  
  ```swift
  public protocol SetAlgebra<Element> : Equatable, ExpressibleByArrayLiteral {
      associatedtype Element
      init()
  
      func contains(_ member: Self.Element) -> Bool
      func union(_ other: Self) -> Self
      func intersection(_ other: Self) -> Self
      func symmetricDifference(_ other: Self) -> Self
  
      mutating func insert(_ newMember: Self.Element) -> (inserted: Bool, memberAfterInsert: Self.Element)
      mutating func remove(_ member: Self.Element) -> Self.Element?
      mutating func update(with newMember: Self.Element) -> Self.Element?
      mutating func formUnion(_ other: Self)
      mutating func formIntersection(_ other: Self)
  
      // ...
  }
  ```

- 其中的 `rawValue` 一般为 `Int` (`FixedWidthInteger`)
  
  > 如果 `rawValue` 不是 `FixedWidthInteger`，则需要手动实现 `SetAlgebra` 协议中的：`init`、`formUnion`、`formIntersection`、`formSymmetricDifference` 方法。
  > 
  > 在 OptionSet extension 中有条件地(Self.RawValue: FixedWidthInteger)实现了它们：
  > 
  > ```swift
  > extension OptionSet where Self.RawValue : FixedWidthInteger {
  >   @inlinable public init()
  > 
  >   @inlinable public mutating func formUnion(_ other: Self)
  >   @inlinable public mutating func formIntersection(_ other: Self)
  >   @inlinable public mutating func formSymmetricDifference(_ other: Self)
  > }
  > ```

- 将所有 option 分别定义为：`static let`

- 可以通过 `[]` 的形式定义 option 的组合，如：[.leftTop, .rightTop]

## enum vs. OptionSet

- enum — 用于表示一组「互斥」关系

- OptionSet — 用于表示「组合」关系

需要注意的是，「互斥」、「组合」与使用场景相关，如上用于表示 4 个方位的「LeftTop、RightTop、LeftBottom、RightBottom」：

- 在 `func corner(_ corners: RectCorner, radius: CGFloat) -> UIView` 场景下就是组合关系，应用 `OptionSet` 定义之

- 如在表示某个物体当前所在位置时，应用 `enum`，因为同一个物体不可能同时位于 2 个不同的位置上

## NS_OPTIONS

OC 下的 `NS_OPTIONS` 在 Swift 下会转成 `OptionSet`，如：

```objectivec
typedef NS_OPTIONS(NSInteger, Direction) {
  DirectionNone = 0,
  DirectionLeft = 1 << 0,
  DirectionRight = 1 << 1,
  DirectionTop = 1 << 2,
  DirectionBottom = 1 << 3
};
```

在 Swift 下：

```swift
public struct Direction : OptionSet, @unchecked Sendable {

    public init(rawValue: Int)

    public static var left: Direction { get }
    public static var right: Direction { get }
    public static var top: Direction { get }
    public static var bottom: Direction { get }
}
```

## 应用

### JSONEncoder

Swift 标准库中控制 JSON 解码输出格式的 `OutputFormatting`：

```swift
open class JSONEncoder {

    /// The formatting of the output JSON data.
    public struct OutputFormatting : OptionSet, Sendable {
        public let rawValue: UInt
        public init(rawValue: UInt)

        public static let prettyPrinted: JSONEncoder.OutputFormatting
        public static let sortedKeys: JSONEncoder.OutputFormatting
        public static let withoutEscapingSlashes: JSONEncoder.OutputFormatting
    }
}
```

### UIView

UIView 相关的大量从 `NS_OPTIONS` 转换过来的：

```swift
public struct AutoresizingMask : OptionSet, @unchecked Sendable {
    public init(rawValue: UInt)

    public static var flexibleLeftMargin: UIView.AutoresizingMask { get }
    public static var flexibleWidth: UIView.AutoresizingMask { get }
    public static var flexibleRightMargin: UIView.AutoresizingMask { get }
    public static var flexibleTopMargin: UIView.AutoresizingMask { get }
    public static var flexibleHeight: UIView.AutoresizingMask { get }
    public static var flexibleBottomMargin: UIView.AutoresizingMask { get }
}
```

### SwiftUI

SwiftUI layout 时描述「边」：

```swift
@frozen public enum Edge : Int8, CaseIterable {
    /// An efficient set of `Edge`s.
    @frozen public struct Set : OptionSet {
        public let rawValue: Int8
        public init(rawValue: Int8)

        public static let top: Edge.Set
        public static let leading: Edge.Set
        public static let bottom: Edge.Set
        public static let trailing: Edge.Set
        public static let all: Edge.Set
        public static let horizontal: Edge.Set
        public static let vertical: Edge.Set
    }
}
```

### Moya

[GitHub - Moya](https://github.com/Moya/Moya) 中用于表示网络请求的哪些部分将打到 log 里：

```swift
public extension NetworkLoggerPlugin.Configuration {
    struct LogOptions: OptionSet {
        public let rawValue: Int
        public init(rawValue: Int) { self.rawValue = rawValue }

        /// The request's method will be logged.
        public static let requestMethod: LogOptions = LogOptions(rawValue: 1 << 0)
        /// The request's body will be logged.
        public static let requestBody: LogOptions = LogOptions(rawValue: 1 << 1)
        /// The request's headers will be logged.
        public static let requestHeaders: LogOptions = LogOptions(rawValue: 1 << 2)
        /// The request will be logged in the cURL format.
        public static let formatRequestAscURL: LogOptions = LogOptions(rawValue: 1 << 3)
        /// The body of a response that is a success will be logged.
        public static let successResponseBody: LogOptions = LogOptions(rawValue: 1 << 4)
        /// The body of a response that is an error will be logged.
        public static let errorResponseBody: LogOptions = LogOptions(rawValue: 1 << 5)

        //Aggregate options
        /// Only basic components will be logged.
        public static let `default`: LogOptions = [requestMethod, requestHeaders]
        /// All components will be logged.
        public static let verbose: LogOptions = [requestMethod, requestHeaders, requestBody,
                                                 successResponseBody, errorResponseBody]
    }
}
```

## Alamofire

[GitHub - Alamofire](https://github.com/Alamofire/Alamofire) 中将下载好的文件移到指定位置时执行的操作：

- 要不要创建中间目录

- 要不要删除老的文件

```swift
public class DownloadRequest: Request {
    /// A set of options to be executed prior to moving a downloaded file from the temporary `URL` to the destination
    /// `URL`.
    public struct Options: OptionSet {
        /// Specifies that intermediate directories for the destination URL should be created.
        public static let createIntermediateDirectories = Options(rawValue: 1 << 0)
        /// Specifies that any previous file at the destination `URL` should be removed.
        public static let removePreviousFile = Options(rawValue: 1 << 1)

        public let rawValue: Int

        public init(rawValue: Int) {
            self.rawValue = rawValue
        }
    }
}
```

# LazySequence

---

```swift
let someValues = ["1", "3", "8", "5", "8", "10", "7", "6"]

let _ =
someValues.map {
  print("In map: ", $0)
  return Int($0) ?? 0       // 👈, 若此处操作非常耗时⚡️
}.first {
  print("In first-where:", $0)
  return $0 > 3
}
```

如上代码，先将 `String --> Int`，再找到第一个大于 `3` 的数

整个执行过程：

```swift
In map:  1
In map:  3
In map:  8
In map:  5
In map:  8
In map:  10
In map:  7
In map:  6
In first-where: 1
In first-where: 3
In first-where: 8
```

- 先将整个 [String] --> [Int]
- 在结果 [Int] 中找到第一个大于 3 的数

![](/img/Array_Map.png)

其实，真正需要执行 `map(String --> Int)` 操作的就前 3 个数："1"、"3"、"8"，因为 "8" 满足要求。

如果 `map` 操作非常耗时❓

有没有优化措施，减少无谓的 `map` 操作？

当然有了：

```swift
//                  👇
let _ = someValues.lazy.map {
  print("In lazy map:", $0)
  return Int($0) ?? 0
}.first {
  print("In lazy first-where:", $0)
  return $0 > 3
}
```

其执行流程：

```swift
In lazy map: 1
In lazy first-where: 1
In lazy map: 3
In lazy first-where: 3
In lazy map: 8
In lazy first-where: 8
```

![](/img/Array-LazySequence-LazyMapSequence.png)

可以看到，其执行流程与普通的 array-map 有很大的区别：

- `Array`、`Dictionary` 等 Collections 都提供了计算属性 `lazy` ，将其转成 `LazySequence`
  
  ```swift
  @frozen public struct Array<Element> {
          /// A sequence containing the same elements as this sequence,
      /// but on which some operations, such as `map` and `filter`, are
      /// implemented lazily.
      @inlinable public var lazy: LazySequence<Array<Element>> { get }
  }
  ```
  
  ```swift
  @frozen public struct Dictionary<Key, Value> where Key : Hashable {
          /// A sequence containing the same elements as this sequence,
      /// but on which some operations, such as `map` and `filter`, are
      /// implemented lazily.
      @inlinable public var lazy: LazySequence<Dictionary<Key, Value>> { get }
  }
  ```
  
  ![](/img/LazySequence-SomeVaules.png)

- LazySequence，其中的 `map`、`filter` 等操作是「懒」执行
  
  > A sequence containing the same elements as a `Base` sequence, but on which some operations such as `map` and `filter` are implemented lazily.
  > 
  > -- [LazySequence - Apple Developer Documentation](https://developer.apple.com/documentation/swift/lazysequence)
  
  ```swift
  @frozen struct LazySequence<Base> where Base : Sequence
  ```
  
  对于 `LazySequence` 本身我们不用太关注，很少直接用它，一般都是通过 Array、Dictionary 的 `lazy` 属性间接获取

- 对 `LazySequence` 类型的实例做 `map` 操作返回的是 `LazyMapSequence` 类型的结果
  
  ![](/img/LazyMapSequence.png)
  
  可以简单的理解为，`LazyMapSequence` 存储的是 `map-closure`，而不是 `map` 执行后的结果：
  
  ![](/img/Array-LazySequence-LazyMapSequence.png)

## 应用

可以看到，LazySequence 与普通的 Collection 在执行流程上有很大的差异，在正常开发中应避免使用，除非有性能问题。

另外，只有在取部分结果的场景下才有意义，如 `first`、`first(where:)`、`prefix` 等。

在 [GitHub - BetterCodable](https://github.com/marksands/BetterCodable) 中做 Lossless Decode 时用到 lazy：

![](/img/Codable-LosslessValueCodable.png)

![](/img/LosslessDefaultStrategy.png)

看一个有意思的问题：

找出最小的 5 个数：

- 它们都大于 100

- 它们的立方根都为整数

老实巴交：

```swift
var results = [Int]()
var curValue = 1
while curValue * curValue * curValue <= 100 {
  curValue += 1
}

(1...5).forEach { _ in
  results.append(curValue * curValue * curValue)
  curValue += 1
}
```

小聪明：

```swift
let lazyResults = (1...)
   .lazy
   .map { $0 * $0 * $0 }
   .filter({ $0 > 100} )

let results = Array(lazyResults.prefix(5))
```

# Range

---

Swift Range 提供了非常强大的能力，充分利用其特性可以写出非常优雅的代码。

Swift Range 有三类：

- Closed Range -- `a...b`

- Half-open Range -- `a..<b`

- One-sided Range -- `a...`、`...b`、`..<b`

![](/img/Range.png)

如上图：

- 不同类型的 Range 都实现了 `RangeExpression` 协议
  
  其中最重要的方法就是 `contains`，判断某个值是否在指定区间内

- 不仅 `Int` 可以用于表示 range，所有实现了 `Comparable` 协议的类型都可以
  
  如，`Double`、`String` (`1.0...14.0`、`"a"..."z"`)

## 应用

- 有效取值区间校验
  
  如：ph 值有效取值区间为 0.0~14.0
  
  ```swift
  func isVaildPH(_ value: Double) -> Bool {
    (0.0...14.0).contains(value)   // value >= 0.0 && value <= 14.0
  }
  ```
  
  校验 http status code：
  
  ```swift
  func isSuccessHTTPStatusCode(_ statusCode: Int) -> Bool {
    (200..<300).contains(statusCode)
  }
  ```

- `for...in`、`forEach`
  
  ```swift
  for index in 1..<7 {
    // ...
  }
  
  (1...31).forEach { i in
    // ...
  }
  ```

- `switch`
  
  ```swift
  switch score {
  case 0..<60:
    print("Failed.")
  case 60..<85:
    print("OK.")
  default:
    print("Good!")
  }
  ```

- Slicing collections
  
  ```swift
  var someValues = ["1", "3", "8", "5", "8", "10", "7", "6"]
  
  var prefixValues = someValues[...3]   // ["1", "3", "8", "5"]
  var prefixToVaules = someValues[..<3] // ["1", "3", "8"]
  var suffixValues = someValues[4...]   // ["8", "10", "7", "6"]
  var middleValues = someValues[1...5]  // ["3", "8", "5", "8", "10"]
  ```
  
  需要注意的 `prefixValues`、`prefixToVaules`、`suffixValues`、`middleValues` 的类型是 `ArraySlice` 而非 `Array`

# ArraySlice

---

如上，对 `Array` 做切片得到的结果类型不是 `Array` 而是 `ArraySlice`：

```swift
// A slice of an Array, ContiguousArray, or ArraySlice instance.
//
@frozen public struct ArraySlice<Element> {}
```

- `ArraySlice` 具有与 `Array` 相同的接口，故能用 `Array` 的地方都能用 `ArraySlice`
  
  也就是在使用时不必过多在意具体类型是`Array` 还是  `ArraySlice` 

- 由 `Array` 生成 `ArraySlice` 时，并没有发生内存的 alloc、copy 等操作，`ArraySlice` 完全共享 `Array` 的数据
  
  ![](img/ArraySlice.png)

由于 `Array` 与 `ArraySlice` 共享数据：

- 不建议长时间持有 `ArraySlice` (不要作为 `class`、`struct` 的属性，只应作为局部变量使用)，因为 slice 会强持有整个 array，可能会出现内存问题
  
  对于需要长时间持有的，应将 slice 转成 array:
  
  ```swift
  let newStorage = Array(middleValues)
  ```
  
  > Long-term storage of `ArraySlice` instances is discouraged. A slice holds a reference to the entire storage of a larger array, not just to the portion it presents, even after the original array’s lifetime ends. Long-term storage of a slice may therefore prolong the lifetime of elements that are no longer otherwise accessible, which can appear to be memory and object leakage.
  > 
  > -- [ArraySlice - Apple Developer Documentation](https://developer.apple.com/documentation/swift/arrayslice#Slices-Are-Views-onto-Arrays)

- slice 与 array 共享相同的 index，slice 起始 index 不一定是 0 💥⚡️
  
  如下，`middleValues` index 从 1 开始，而不是 0
  
  (`var middleValues = someValues[1...5]`)
  
  ![](/img/middlevalues-index.png)
  
  ![](/img/middlevalues-outofbounds.png)
  
  因此，对 `slice` 做下标相关操作时需格外谨慎，相关操作应该基于 `ArraySlice` 提供的 `startIndex`、`endIndex`：
  
  `slice` 的有效取值区间为：`startIndex..<endIndex`
  
  ![](/img/middlevalues-startindex.png)

# Substring

---

`String` 与 `Substring` 的关系非常类似于 `Array` 与 `ArraySlice`：

```swift
// When you create a slice of a string, a `Substring` instance is the result.
//
@frozen public struct Substring : Sendable {}
```

- 共享存储

- 共享 Index

- `Substring` 具有与 `String` 相同的接口

```swift
let greeting = "Hello, world!"
let index = greeting.firstIndex(of: ",") ?? greeting.endIndex  // 5

// beginning、ending is instance of `Substring`
//
let beginning = greeting[..<index]  // "Hello"
let ending = greeting[greeting.index(after: index)...]  // " world!"

let wIndex = ending.firstIndex(of: "w") ?? ending.endIndex  // 7

// Convert the result to a String for long-term storage.
let newString = String(beginning)
```

# RawRepresentable

---

```swift
public protocol RawRepresentable<RawValue> {
    associatedtype RawValue

    init?(rawValue: Self.RawValue)
    var rawValue: Self.RawValue { get }
}
```

> With a `RawRepresentable` type, you can switch back and forth between a custom type and an associated `RawValue` type without losing the value of the original `RawRepresentable` type. Using the raw value of a conforming type streamlines interoperation with Objective-C and legacy APIs and simplifies conformance to other protocols, such as `Equatable`, `Comparable`, and `Hashable`.
> 
> The `RawRepresentable` protocol is seen mainly in two categories of types: enumerations with raw value types and option sets.
> 
> -- [RawRepresentable - Apple Developer Documentation](https://developer.apple.com/documentation/swift/rawrepresentable)

- `RawRepresentable` 表示自定义类型与 Raw value (`Int`、`String` ...) 间可以「 无损互转 」
  
  - 通过 `init?(rawValue: Self.RawValue)` 可以将 Raw value 转成自定义类型对象
  
  - 通过 `rawValue` 属性可以获取自定义对象对应的 raw value

- 2 个主要应用场景：带 raw value 的 enum、OptionSet
  
  - 关于 Raw value enum 详见 [Swift 最佳实践之 Enum](https://juejin.cn/post/7212143399036813349#heading-1)

# Expressible by Literal

---

```swift
var someInt: Int = 1
var someString: String = "Hello"
var someArray: Array<String> = ["Hello", "world!"]
var someDictionary: Dictionary<String, Int> = ["Key": 1]
```

大家有感觉到这些赋值奇怪吗？🤔

在 Swift 中 `Int`、`String`、`Array`、`Dictionary` 都是 `struct`，如：

```swift
@frozen public struct Int
```

其初始化不应该是要调用相应的 `init` 方法吗？

```swift
var someInt: Int = Int(1)
var someString: String = String("Hello")
var someArray: Array<String> = Array(["Hello", "world!"])
var someDictionary: Dictionary<String, Int> = Dictionary(dictionaryLiteral: ("Key", 1))
```

怎么直接给它们赋值了相关的字面量 (Literal)❓

Swift 为了简化初始化赋值过程，提供了一系列的协议用于「字面量赋值」：

```swift
public protocol ExpressibleByIntegerLiteral {
    associatedtype IntegerLiteralType : _ExpressibleByBuiltinIntegerLiteral
    init(integerLiteral value: Self.IntegerLiteralType)
}


public protocol ExpressibleByStringLiteral : ExpressibleByExtendedGraphemeClusterLiteral {
    associatedtype StringLiteralType : _ExpressibleByBuiltinStringLiteral
    init(stringLiteral value: Self.StringLiteralType)
}


// ExpressibleByArrayLiteral
// ExpressibleByDictionaryLiteral
// ExpressibleByBooleanLiteral
// ExpressibleByNilLiteral
```

实现了相关 Protocol 后，就可以用相应的「字面量」赋值

没错，`Int`、`String`、`Array`、`Dictionary` 分别实现了上述 Protocol

在实际开发中，我们也可以按需实现这些协议，如：

```swift
extension Date: ExpressibleByStringLiteral {
    public init(stringLiteral value: String) {
      let dateFormatter = ISO8601DateFormatter()
      dateFormatter.formatOptions = [.withFullDate]
      self = dateFormatter.date(from: value) ?? Date.now
    }
}

//                     👇
let date: Date = "2023-05-28"
print(date)   // 2023-05-28 00:00:00 +0000
```

如上，我们扩展了 `Date`，并实现了 `ExpressibleByStringLiteral`

从此，可以直接将 String 类型的「字面量」赋给 `Date`！

> OptionSet 类型之所以可以用 Array 形式的初始化，也是因为其实现了 ExpressibleByArrayLiteral 协议，如：
> 
> ```swift
> public static let all: RectCorner = [.leftTop, .rightTop, .leftBottom, .rightBottom]
> ```

# 参考资料

[OptionSet - Apple Developer Documentation](https://developer.apple.com/documentation/swift/optionset)

[ArraySlice - Apple Developer Documentation](https://developer.apple.com/documentation/swift/arrayslice)

[String - Apple Developer Documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/stringsandcharacters/#String-Indices)

[LazySequence - Apple Developer Documentation](https://developer.apple.com/documentation/swift/lazysequence)

[How and when to use Lazy Collections in Swift - SwiftLee](https://www.avanderlee.com/swift/lazy-collections-arrays/)

https://www.objc.io/blog/2018/03/20/lazy-infinite-sequences/

[Expressible literals in Swift explained by 3 useful examples - SwiftLee](https://www.avanderlee.com/swift/expressible-literals/)

[How OptionSet works inside the Swift Compiler](https://swiftrocks.com/how-optionset-works-inside-the-swift-compiler)
