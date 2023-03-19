---
layout: post
title: Swift 最佳实践之 Enum
date: 2023-03-17 22:53:27
tags:
- iOS
- Swift
---

Swift 作为现代、高效、安全的编程语言，其背后有很多高级特性为之支撑。

『 Swift 最佳实践 』系列对常用的语言特性逐个进行介绍，助力写出更简洁、更优雅的 Swift 代码，快速实现从 OC 到 Swift 的转变。

该系列内容主要包括：

- [Optional](https://juejin.cn/post/7212101192746696760)

- Enum

- Closure

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

本文是系列文章的第二篇，介绍 Enum，内容主要包括 Swift Enum 高级特性以及典型应用场景。

Swift 赋以 Enum 非常强大的能力，C-like Enum 与之不可同日而语。

充分利用 Enum 特性写出更优雅、更安全的代码。

# Enum 特性

---

首先，简要罗列一下 Swift Enum 具备的特性：

## Value

C-like Enum 中每个 case 都关联一个整型数值，而 Swift Enum 更灵活：

- **默认，case 不关联任何数值**

- **可以提供类似 C-like Enum 那样的数值 (Raw Values)，** 但类型更丰富，可以是 Int、Float、String、Character
  
  ```swift
  enum Direction: String {
  
    case east  // 等价于 case east = "east"，
    case west
    case south
    case north = "n"
  }
  ```
  
  如上，对于 `String` 类型的 Raw Value，若没指定值，默认为 case name
  
  对于 `Int`/`Float` 类型，默认值从 0 开始，依次 +1
  
  通过 `rawValue` 属性可以获取 case 对应的 Raw Value
  
  ```swift
  let direction = Direction.east
  let value = direction.rawValue  // "east"
  ```
  
  对于 Raw-Values Enum，编译器会自动生成初始化方法：
  
  ```swift
  // 由于并不是所有输入的 rawValue 都能找到匹配的 case
  // 故，这是个 failable initializer，在没有匹配的 case 时，返回 nil 
  init?(rawValue: RawValueType)
  
  let direction = Direction.init(rawValue: "east")
  ```

- **还可以为 case 指定任意类型的关联值 (Associated-Values)**
  
  ```swift
  enum UGCContent {
  
    case text(String)
    case image(Image, description: String?)
    case audio(Audio, autoPlay: Bool)
    case video(Video, autoPlay: Bool)
  }
  
  let text = UGCContent.text("Hello world!")
  let video = UGCContent.video(Video.init(), autoplay: true)
  ```
  
  还可以为关联值提供默认值：
  
  ```swift
  enum UGCContent {
  
    case text(String = "Hello world!")
    case image(Image, description: String?)
    case audio(Audio, autoPlay: Bool = true)
    case video(Video, autoPlay: Bool = true)
  }
  
  let text = UGCContent.text()
  let content = UGCContent.video(Video.init())
  ```
  
  如下，即可以通过 `if-case-let`，也可以通过 `switch-case-let` 匹配 enum 并提取关联值：
  
  ```swift
  if case let .video(video,  autoplay) = content {
    print(video, autoplay)
  }
  
  switch content {
    case let .text(text):
      print(text)
    case let .image(image, description):
      print(image, description)
    case let .audio(audio, autoPlay):
      print(audio, autoPlay)
    case let .video(video, autoPlay):
      print(video, autoPlay)
  }
  ```

## First-class type

Swift Enum 作为「 First-class type 」，有许多传统上只有 Class 才具备的特性：

- 可以有计算属性 (computed properties)，当然了存储属性是不能有的，如：
  
  ```swift
  enum UGCContent {
  
    var description: String {
      switch self {
        case let .text(text):
          return text
        case let .image(_, description):
          return description ?? "image"
        case let .audio(_, autoPlay):
          return "audio, autoPlay: \(autoPlay)"
        case let .video(_, autoPlay):
          return "video, autoPlay: \(autoPlay)"
      }
    }
  }
  
  let content = UGCContent.video(Video.init())
  print(content.description)  
  ```

- 可以有实例方法/静态方法

- 可以有初始化方法，如：
  
  ```swift
  enum UGCContent {
  
    init(_ text: String) {
      self = .text(text)
    }
  }
  
  let text = UGCContent.init("Hi!")
  ```

- 可以有扩展 (extension)，也可以实现协议，如：
  
  ```swift
  extension UGCContent: Equatable {
  
    static func == (lhs: Self, rhs: Self) -> Bool {
      return false
    }
  }
  ```

## Recursive Enum

Enum 关联值类型可以是其枚举自身，称其为递归枚举 (Recursive Enum)。

如下，定义了 Enum 类型的链表接点 `LinkNode`，包含 2 个 case：

- `end`，表示尾部节点

- `link`，其关联值 `next` 的类型为 `LinkNode`

```swift
// 关键词 indirect 也可以放在 enum 前面
// indirect enum LinkNode<NodeType> {
enum LinkNode<NodeType> {

  case end(NodeType)
  indirect case link(NodeType, next: LinkNode)
}


func sum(rootNode: LinkNode<Int>) -> Int {

  switch rootNode {
    case let .end(value):
      return value
    case let .link(value, next: next):
      return value + sum(rootNode: next)
  }
}


let endNode = LinkNode.end(3)
let childNode = LinkNode.link(9, next: endNode)
let rootNode = LinkNode.link(5, next: childNode)

let sum = sum(rootNode: rootNode)
```

## Iterating over Enum Cases

有时，我们希望遍历 Enum 的所有 cases，或是获取第一个 case，此时 `CaseIterable` 派上用场了：

```swift
public protocol CaseIterable {

    /// A type that can represent a collection of all values of this type.
    associatedtype AllCases : Collection = [Self] where Self == Self.AllCases.Element

    /// A collection of all values of this type.
    static var allCases: Self.AllCases { get }
}
```

可以看到，`CaseIterable` 协议有一个静态属性：`allCases`。

对于没有关联值 (Associated-Values) 的枚举，当声明其遵守 `CaseIterable` 时，会自动合成 `allCases` 属性：

```swift
enum Weekday: String, CaseIterable {
  case sunday, monday, tuesday, wednesday, thursday, friday, saturday

  /* 自动合成的实现
  static var allCases: Self.AllCases { [sunday, monday, tuesday, wednesday, thursday, friday, saturday] }
  */
}

// sunday, monday, tuesday, wednesday, thursday, friday, saturday
let weekdays = Weekday.allCases.map{ $0.rawValue }.joined(separator: ", ")
```

对于有关联值的枚举，不会自动合成 `allCases`，因为关联值没法确定

![](/img/CaseIterable-Enum-Error.png)

此时，需要手动实现 `CaseIterable` 协议：

```swift
enum UGCContent: CaseIterable {

  case text(String = "ugc")
  case image(Image, description: String? = nil)
  case audio(Audio, autoPlay: Bool = false)
  case video(Video, autoplay: Bool = true)

  static var allCases: [UGCContent] {
    [.text(), image(Image("")), .audio(Audio()), .video(Video())]
  }
}
```

## Equatable

没有关联值的枚举，默认可以执行判等操作 (`==`)，无需声明遵守 `Equatable` 协议：

```swift
let sun = Weekday.sunday
let mon = Weekday.monday

sum == Weekday.sunday // true
sum == mon // false
```

对于有关联值的枚举，若需要执行判等操作，需显式声明遵守 `Equatable` 协议：

![](/img/Enum-Equatable-Error.png)

```swift
// 由于 NodeType：Equatable
// 故，系统会为 LinkNode 自动合成 static func == (lhs: Self, rhs: Self) -> Bool
// 无需手写 == 的实现，只需显式声明 LinkNode 遵守 Equatable 即可
enum LinkNode<NodeType: Equatable>: Equatable {
  case end(NodeType)
  indirect case link(NodeType, next: LinkNode)
}
```

# 应用

-----

关联值可以说极大丰富了 Swift Enum 的使用场景，而 C-like Enum 限于只是个 Int 型值，只能用于一些简单的状态、分类等。

因此，我们需要转变思维，善用 Swift Enum。对于一组相关的「值」、「状态」、「操作」等等，都可以通过 Enum 封装，附加信息用 Associated-Values 表示。

## 标准库中的 Enum

 Enum 在 Swift 标准库中有大量应用，典型的如：

- Optional，在 「 Swift 最佳实践之 Enum 」中有详细介绍，不再赘述
  
  ```swift
  @frozen public enum Optional<Wrapped> : ExpressibleByNilLiteral {
  
      /// The absence of a value.
      ///
      /// In code, the absence of a value is typically written using the `nil`
      /// literal rather than the explicit `.none` enumeration case.
      case none
  
      /// The presence of a value, stored as `Wrapped`.
      case some(Wrapped)
  }
  ```

- Result，用于封装结果，如网络请求、方法返回值等
  
  ```swift
  @frozen public enum Result<Success, Failure> where Failure : Error {
  
      /// A success, storing a `Success` value.
      case success(Success)
  
      /// A failure, storing a `Failure` value.
      case failure(Failure)
  }
  ```

- Error Handle，如 `EncodingError`、`DecodingError`：
  
  ```swift
  public enum DecodingError : Error {
  
      /// An indication that a value of the given type could not be decoded because
      /// it did not match the type of what was found in the encoded payload.
      ///
      /// As associated values, this case contains the attempted type and context
      /// for debugging.
      case typeMismatch(Any.Type, DecodingError.Context)
  
      /// An indication that a non-optional value of the given type was expected,
      /// but a null value was found.
      ///
      /// As associated values, this case contains the attempted type and context
      /// for debugging.
      case valueNotFound(Any.Type, DecodingError.Context)
  
      /// An indication that a keyed decoding container was asked for an entry for
      /// the given key, but did not contain one.
      ///
      /// As associated values, this case contains the attempted key and context
      /// for debugging.
      case keyNotFound(CodingKey, DecodingError.Context)
  
      /// An indication that the data is corrupted or otherwise invalid.
      ///
      /// As an associated value, this case contains the context for debugging.
      case dataCorrupted(DecodingError.Context)
  }
  ```

- Never，是一个没有 case 的枚举，用于表示一个方法永远不会正常返回
  
  ```swift
  /// The return type of functions that do not return normally, that is, a type
  /// with no values.
  ///
  /// Use `Never` as the return type when declaring a closure, function, or
  /// method that unconditionally throws an error, traps, or otherwise does
  /// not terminate.
  ///
  ///     func crashAndBurn() -> Never {
  ///         fatalError("Something very, very bad happened")
  ///     }
  @frozen public enum Never {
    // ...
  }
  ```

## 实践中的应用

- **Error Handle**
  
  > Swift enumerations are particularly well suited to modeling a group of related error conditions, with associated values allowing for additional information about the nature of an error to be communicated.
  > 
  > ([Documentation-errorhandling](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/errorhandling/#Representing-and-Throwing-Errors))
  
  正如 Apple 官方文档所言，定义错误模型是 Enum 的典型应用场景之一，如上节提到的 `EncodingError`、`DecodingError`。
  
  将不同的错误类型定义为 case，错误相关信息以关联值的形式附加在相应 case 上。
  
  在著名网络库 [Alamofire](https://github.com/Alamofire/Alamofire) 中也有很好的应用，如：
  
  ```swift
  public enum AFError: Error {
      /// The underlying reason the `.multipartEncodingFailed` error occurred.
      public enum MultipartEncodingFailureReason {
          /// The `fileURL` provided for reading an encodable body part isn't a file `URL`.
          case bodyPartURLInvalid(url: URL)
          /// The filename of the `fileURL` provided has either an empty `lastPathComponent` or `pathExtension.
          case bodyPartFilenameInvalid(in: URL)
          /// The file at the `fileURL` provided was not reachable.
          case bodyPartFileNotReachable(at: URL)
  
          // ...
      }
  
      /// The underlying reason the `.parameterEncoderFailed` error occurred.
      public enum ParameterEncoderFailureReason {
          /// Possible missing components.
          public enum RequiredComponent {
              /// The `URL` was missing or unable to be extracted from the passed `URLRequest` or during encoding.
              case url
              /// The `HTTPMethod` could not be extracted from the passed `URLRequest`.
              case httpMethod(rawValue: String)
          }
  
          /// A `RequiredComponent` was missing during encoding.
          case missingRequiredComponent(RequiredComponent)
          /// The underlying encoder failed with the associated error.
          case encoderFailed(error: Error)
      }
  
      // ...
  }
  ```

- **命名空间**
  
  命名空间有助于提升代码结构化，Swift 中命名空间是隐式的，即以模块 (Module) 为边界，不同的模块属于不同的命名空间，无法显式定义命名空间 (没有 `namespace` 关键词)。
  
  我们可以通过 no-case Enum 创建自定义(伪)命名空间，实现更小粒度的代码结构化
  
  > 为什么是用 Enum，而不是 Struct 或 Class？
  > 
  > 原因在于，没有 case 的 Enum 是无法实例化的
  > 
  > 而 Struct、Class 一定是可以实例化的
  
  如，Apple 的 Combine 库用 Enum 定义了命名空间 `Publishers`、`Subscribers`：
  
  ![](/img/Namespace-Publishers.png)
  
  ![](/img/Publishers-Namespace-First.png)
  
  如上，通过命名空间 `Publishers` 将相关功能组织在一起，代码更加结构化
  
  也不需要在每个具体类型前重复添加前缀/后缀，如：
  
  - `MulticastPublisher` --> `Publishers.Multicast` 
  
  - `SubscribeOnPublisher` --> `Publishers.SubscribeOn`
  
  - `FirstPublisher` --> `Publishers.First`

- 定义常量
  
  思想跟上面提到的命名空间是一样的，可以将一组相关常量定义在一个 Enum 中，如：
  
  ```swift
  class TestView: UIView {
    enum Dimension {
      static let left = 18.0
      static let right = 18.0
      static let top = 10
      static let bottom = 10
    }
  
    // ...
  }
  
  enum Math {
    static let π = 3.14159265358979323846264338327950288
    static let e = 2.71828182845904523536028747135266249
    static let u = 1.45136923488338105028396848589202744
  }
  ```

- API Endpoints
  
  App/Module 内网络请求 (API) 模型也可以用 Enum 来定义，API 参数通过关联值绑定
  
  著名网络库 [Moya](https://github.com/Moya/Moya) 就是基于这个思想：
  
  ```swift
  public enum GitHub {
    case zen
    case userProfile(String)
    case userRepositories(String)
  }
  
  extension GitHub: TargetType {
      public var baseURL: URL { URL(string: "https://api.github.com")! }
      public var path: String {
          switch self {
          case .zen:
              return "/zen"
          case .userProfile(let name):
              return "/users/\(name.urlEscaped)"
          case .userRepositories(let name):
              return "/users/\(name.urlEscaped)/repos"
          }
      }
      public var method: Moya.Method { .get }
  
      public var task: Task {
          switch self {
          case .userRepositories:
              return .requestParameters(parameters: ["sort": "pushed"], encoding: URLEncoding.default)
          default:
              return .requestPlain
          }
      }
  }
  
  let provider = MoyaProvider<GitHub>()
  provider.request(.zen) { result in
    // ...
  }
  ```
  
  如上，将 GitHub 相关的 API (`zen`、`userProfile`、`userRepositories`) 封装在 enum `GitHub` 中。
  
  最终通过 `provider.request(.zen)` 的方式发起请求。
  
  > 强烈建议大家读一下 [Moya](https://github.com/Moya/Moya) 源码

- UI State
  
  可以将页面各种状态封装在 Enum 中：
  
  ```swift
  enum UIState<DataType, ErrorType> {
    case loading
    case empty
    case loaded(DataType)
    case failured(ErrorType)
  }
  ```

- Associated-Values case 可以当作函数用
  
  一般用于函数式 `map`、`flatMap`，如 Array、Optional、Combine 等：
  
  ```swift
  func uiState(_ loadedData: String?) -> UIState<String, Error> {
    // 等价于 loadedData.map { UIState.loaded($0) } ?? .empty
    loadedData.map(UIState.loaded) ?? .empty
  }
  
  // loadedTmp 本质上就是个函数：(String) -> UIState<String, Error>
  let loadedTmp = UIState<String, Error>.loaded)
  ```
  
  ![](/img/CaseAsFunction.png)

# 小结

Swift Enum 功能非常强大，具备很多传统上只有 Class 才有的特性。

关联值进一步极大丰富了 Enum 的使用场景。

对于一组具有相关性的 「值」、「状态」、「操作」等，都可以用 Enum 封装。

鼓励优先考虑使用 Enum。

# 参考资料

[Documentation-enumerations](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/enumerations/)

[GitHub - Moya/Moya: Network abstraction layer written in Swift](https://github.com/Moya/Moya)

[GitHub - Alamofire/Alamofire: Elegant HTTP Networking in Swift](https://github.com/Alamofire/Alamofire)

https://www.swiftbysundell.com/articles/powerful-ways-to-use-swift-enums/

https://appventure.me/guides/advanced_practical_enum_examples/introduction.html
