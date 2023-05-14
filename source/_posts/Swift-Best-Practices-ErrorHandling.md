---
layout: post
title: Swift 最佳实践之 Error Handling
date: 2023-05-13 09:53:28
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

- Error Handling

- Advanced Collections (Asyncsequeue/OptionSet/Lazy)

- Expressible by Literal

- Pattern Matching

- Metatypes(.self/.Type/.Protocol)

- High Performance

> ps. 本系列不是入门级语法教程，需要有一定的 Swift 基础

本文是系列文章的第七篇，介绍在 Swift 中如何处理错误 ⚡️❌。

# Overview

---

合理、优雅地处理错误对于 App 的体验至关重要，也是对开发人员基本功的考验。

然而在实际项目中对错误的处理不甚理想，要么完全忽视错误、要么以「错误的方式」处理错误🧐。

如：常见的白屏、执行某个操作 (点击按钮) 没任何反应可能都是对错误处理不当引起的，用户体验不佳。

Swift 有清晰、完善的错误处理机制帮助开发人员处理错误，本文将对此展开详细介绍。



# Kinds of Error

---

程序运行过程中可能出现的错误千奇百怪，大概可以分为 3 类：

- 代码缺陷，雅称「bug 🐞」，如代码逻辑上的错误、➗ 0、越界等
  
  此类问题，除了发版 fix，别无他法

- 非法性输入，主要指用户输入的内容非法、方法的入参非法等

- 运行时错误，主要指程序运行过程中出现的错误，如：网络超时、文件不存在、数据库链接异常、Server 响应内容错误等

本文讨论的主要是后 2 类错误。

根据错误严重程度 (错误发生后程序是否可以继续运行) 以及复杂性，Swift 提供了多种不同的处理方式：

- Optional

- Result

- throws

- assert

- precondition

- fatalError

# Optional

---

对于一些简单的错误场景，如果用 throws 那套流程处理可能有点重，此时 `Optional` 可能是一个不错的选择。

如：`String` --> `Int` 的转换，并不是所有的 String 都能转成对应的 Int，转换失败时可以返回 `nil`。

Swift 在设计 Initializer 时也考虑到此类情况，即*初始化可能会失败*，并提供了 [Failable Initializers](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/initialization/#Failable-Initializers) 机制，在无法完成初始化时可以选择返回 `nil`，代表初始化失败。

定义 Failable Initializer 只需在 `init` 关键字后加上 `?` 即可 (`init?`)，如：

```swift
struct Animal {
  let species: String
  // 👇
  init?(species: String) {
    //                          👇
    if species.isEmpty { return nil }
    self.species = species
  }
}
```

Swift `Int` 提供的从 `String` 转换 `Int` 的 init 也是 Failable Initializer：

```swift
struct Int {
  public init?(_ description: String)
}
```

可以看到通过返回 `Optional` 处理错误时，无法传递错误信息，只知道失败了，不知道为什么失败。

有时错误原因也很重要，可以给用户提供适当的反馈 (对于用户体验也很重要)。

# Modeling Errors

---

为了表示错误，封装错误信息，Swift 提供了 `Error` 协议：

```swift
public protocol Error : Sendable {}
```

可以看到 `Error` 是个空协议，主要用于标记错误模型，后文讲到的 Result、throws 都是基于 `Error` 类型。

> public protocol Sendable {} 
> 
> Sendable 也是个空协议，主要用于并发下的线程安全，详情可参看「 [Swift 新并发框架之 Sendable](https://juejin.cn/post/7076741945820872717/) 」。

由于 `Error` 是个空协议，任何 class、struct、enum 都可以声明实现它。

## Using Enumerations as Errors

> Swift's enumerations are well suited to represent simple errors. Create an enumeration that conforms to the `Error` protocol with a case for each possible error. If there are additional details about the error that could be helpful for recovery, use associated values to include that information.
> 
> -- [Error - Apple Developer Documentation](https://developer.apple.com/documentation/swift/error#Using-Enumerations-as-Errors)

正如 [Swift 最佳实践之 Enum](https://juejin.cn/post/7212143399036813349#heading-8) 一文所讲，通过 enum 定义错误模型是一个非常好的选择：

- 通过 case 定义不同的错误状态

- 具体错误信息通过 associated values 附加在相应的 case 上

如，Swift 标准库定义的解码错误模型：

```swift
public enum DecodingError : Error {
  public struct Context : Sendable {
    public let codingPath: [CodingKey]
    public let debugDescription: String
    public let underlyingError: (Error)?

    public init(codingPath: [CodingKey], debugDescription: String, underlyingError: (Error)? = nil)
  }

  case typeMismatch(Any.Type, DecodingError.Context)
  case valueNotFound(Any.Type, DecodingError.Context)
  case keyNotFound(CodingKey, DecodingError.Context)
  case dataCorrupted(DecodingError.Context)
}
```

## Including More Data in Errors

> Sometimes you may want different error states to include the same common data, such as the position in a file or some of your application's state. When you do, use a structure to represent errors.
> 
> [Error - Apple Developer Documentation](https://developer.apple.com/documentation/swift/error#Including-More-Data-in-Errors)

如果不同的错误状态间有相同的数据需要关联，那 struct 也是一种选择。

如，XML 解析失败，无论是哪种失败原因，都需要附带出错的行号、列号，则可以将`XMLParsingError` 定义为 struct：

```swift
struct XMLParsingError: Error {
  enum ErrorKind {
    case invalidCharacter
    case mismatchedTag
    case internalError
  }

  let line: Int
  let column: Int
  let kind: ErrorKind
}
```

# Result

---

定义方法 `loadImage`：通过 url 加载对应的图片，如何设计？

```swift
func loadImage(url: String, completion: (UIImage?, Error?) -> Void)
```

如上，`loadImage` 肯定是个异步方法，故通过 closure 返回结果。

由于加载可能会失败，故 `completion` 用了 2 个 Optional 参数分别表示成功(`UIImage?`)、失败(`Error?`)

上述设计如何？🤔

不咋地！🫣

问题主要出在 2 个 Optional 参数上，那有没有更好的方式❓

为了解决此类问题，Swift 5 引入了新类型 `Result`：

```swift
/// A value that represents either a success or a failure, including an
/// associated value in each case.
@frozen public enum Result<Success, Failure> where Failure : Error {

  /// A success, storing a `Success` value.
  case success(Success)

  /// A failure, storing a `Failure` value.
  case failure(Failure)

  @inlinable public func map<NewSuccess>(_ transform: (Success) -> NewSuccess) -> Result<NewSuccess, Failure>
  @inlinable public func mapError<NewFailure>(_ transform: (Failure) -> NewFailure) -> Result<Success, NewFailure> where NewFailure : Error
  @inlinable public func flatMap<NewSuccess>(_ transform: (Success) -> Result<NewSuccess, Failure>) -> Result<NewSuccess, Failure>
  @inlinable public func flatMapError<NewFailure>(_ transform: (Failure) -> Result<Success, NewFailure>) -> Result<Success, NewFailure> where NewFailure : Error
  @inlinable public func get() throws -> Success
}
```

可以看到，`Result` 是个 enum：

- `case success(Success)`，表示成功，具体信息存储在关联值中

- `case failure(Failure)`，表示失败，失败信息也存储在关联值中

通过 `Result`，`loadImage` 方法可以改写为：

```swift
func loadImage(url: String, completion: (Result<UIImage, Error>) -> Void)
```

相比之下，`Result` 优势主要有：

- 更结构化的表达，将结果统一封装在一个 enum 内，而不是 2 个离散的 Optional

- `Result` 提供了 `map`、`flatMap` 等便捷的方法

- 对错误处理带有一定的强制性⚡️、提示性⚠️
  
  - 调用方拿到的是个 enum，在 switch 时对 2 个 case 都需要处理，而对于 2 个离散的 Optional 完全可以选择忽略
  
  - 对于定义方，在出错时必须使用 `.failure`，而不能随意的对 Optional Error 赋个 `nil`

**`Result` 主要用于异步方法，那同步方法呢？🤔**

# Throwing Errors

---

在 Swift 中，对于同步方法的错误处理，除了上面提到的用 Optional 表示，还可以对外显式声明⚡️：「该方法运行过程中可能会抛出错误 ❌」。

如下，`readFile` 通过在方法定义中添加 `throws` 关键字表明其可能会抛出错误：

```swift
//                                   👇
func readFile(atPath path: String) throws -> String
```

用 `throws` 声明的方法称为：*throwing method*

> 方法定义中没有添加 `throws` 关键字的，一定不会抛出错误。

*throwing method* 内部在发生错误时可以通过 `throw` 抛出错误，如：

```swift
struct FileHandler {
  let fileManager: FileManager = .default

  enum FileHandleError: Error {
    case invalidPath
    case notExists
    case unreadable
    case `nil`
  }

  func readFile(atPath path: String) throws -> String {
    guard !path.isEmpty else {
      // 👇
      throw FileHandleError.invalidPath
    }

    guard fileManager.fileExists(atPath: path) else {
      throw FileHandleError.notExists
    }

    guard fileManager.isReadableFile(atPath: path) else {
      throw FileHandleError.unreadable
    }

    guard let data = fileManager.contents(atPath: path) else {
      throw FileHandleError.nil
    }

    let decoder = JSONDecoder()
    return try decoder.decode(String.self, from: data)
  }
}
```

# Propagating Errors

---

在很多语言中，如：C++、Java、Python 等，都有异常传播的概念 (Exception Propagation)。

然而，Swift 对 Error 的传递与其他语言异常传播的实现机制有较大区别，故 Swift 极力避免使用 Exception Propagation 这一词。

但是，对于开发者来说，Swift Error 传递与 Exception Propagation 在理解、使用上并无太大差异。

下面的讨论我们还是基于 Swift Error Propagation 这个术语展开。

什么是 Error Propagation 呢 🤨❓

如下图：

- Error 沿着方法调用链 **「逆向」** 传播

- Error Propagation 起于 throwing method，终于 non-throwing method

- non-throwing method 是不能抛出 Error 的

![](/img/PropagatingErrors.png)

对于 throwing method 的调用需加上关键字 `try` (或变体 `try!`、`try?`)，如：

```swift
//             👇
let content = try readFile(atPath: "path")
```

由于可能会抛出错误，故 `try` 调用只能出现在以下 2 个地方：

- throwing method 内，此时 Error 将向上传播，如：
  
  ```swift
  //                  👇
  func loadConfig() throws -> String {
    //👇
    try readFile(atPath: "path.config")
  }
  
  func parseConfig() throws -> Config {
    let configStr = try loadConfig()
    return Config(configStr)
  }
  
  // error propagation chain: readFile -> loadConfig -> parseConfig -> ...
  ```

- `do...catch` 中，一般是错误传播的终点站



> OC 中通过 in-out params 来传递 Error「`(NSError **)error`」，如：
> 
> ```objectivec
> - (nullable NSArray<NSString *> *)contentsOfDirectoryAtPath:(NSString *)path error:(NSError **)error
> ```
> 
> 很明显，OC 的 Error Propagation 没有任何约束性，调用方完全可以忽略🫣

## rethrows

我们知道 [Closure](https://juejin.cn/post/7214665617310892092) 本质上是匿名函数，故闭包也可以 throw error。

```swift
//                                 👇                     👇
func configModel(transformer: () throws -> ConfigModel) throws -> ConfigModel {
  try transformer()
}
```

如上，`configModel` 方法本身不会抛出错误，但其入参 `transformer` 「 *可能* 」会抛出错误，导致 `configModel` 也必须被定义为 `throwing method`❗️

好像没问题🤔❓❗️



调用 `configModel` 方法必须按 Propagating Errors 来处理：

```swift
do {
  try configModel {
    return ConfigModel()   // 👈, no any errors are thrown！
  }
} catch {
  // ...
}
```

如上，传给 `configModel` 的 Closure (`transformer`) 100% 不会抛出错误，但也必须按 Propagating Errors 来调用 `configModel`❗️

合理吗？🤔🤔

`rethrows` 正是用于优化此问题的：

```swift
//                                                         👇
func configModel(transformer: () throws -> ConfigModel) rethrows -> ConfigModel {
  try transformer()
}
```

***对于方法本身不会抛出错误，但入参可能会抛出错误的情况***，可以用 `rethrows` 来声明方法：即对外宣称：***「我本身不会抛出任何错误，但传入的 Closure 有可能！」***

由于参数闭包是调用方传入，其肯定知道闭包会不会抛错误了🤓

故，对于不会抛出错误的闭包，可以像普通方法那样调用 `rethrows` 方法：

```swift
configModel {  // 👈, no need `try`
   ConfigModel()
}
```

对于 rethrows method，只要入参 Closure 不会抛出错误，就可以像普通方法那样进行调用❗️

## Converting Errors to Optional Values

正如上文所述，Optional 有时也可以用于处理错误，`try?` 正是用于将 Error 转成 Optional，如：

```swift
let content = try? readFile(atPath: "path")

// Equivalent to

let content: String?
do {
  content = try readFile(atPath: "path")
} catch {
  content = nil
}
```

- 通过 `try?` 调用方法，返回值为 Optional

- 当调用方法抛出错误时，`try?` 将吞下 error，转而返回 `nil` (Error Propagation 就此终止)



## Disabling Error Propagation

如果能 ***100%*** 确定调用的 throwing function 不会抛出错误，也可以使用 `try!` 来终止 Error Propagation，如：

```swift
//             👇
let content = try! readFile(atPath: "path")  // content 类型为 String
```

当然了，如果不幸此时 `readFile` 还是抛出了错误，那等待你的将是 ~~Cash~~ Crash💥

在实际开发中应该禁止使用 `try!`❗️⚡️



# Handling Errors

---

错误虽可向上传播，但终归是要处理的 🫣，不可能无限传递下去 (卑微的打工人🥲)

Swift 通过 `do...catch` 来捕获并处理错误：

```swift
do {
  try <#expression#>
  <#statements#>
} catch <#pattern 1#> {
  <#statements#>
} catch <#pattern 2#> where <#condition#> {
  <#statements#>
} catch <#pattern 3#>, <#pattern 4#> where <#condition#> {
  <#statements#>
} catch {
  // matches any error
  // binds the error to a local constant named error
  <#statements#>
}
```

-  `try` 调用用  `do-block`  包起来

- 通过 `catch` + `PatternMatching` 可以匹配特定类型的 Error
  
  > Swift 中 Pattern Matching 功能强大，且无处不在，后续有个专题讨论之

- 没有指定 Pattern Matching 的 catch 将匹配任意类型的 Error，并将 Error 绑定到名为 `error` 的本地常量上

如下，在 `loadConfig` 方法中通过 `do...catch` 捕获来自 `readFile` 的错误，并全部处理掉：

```swift
// loadConfig is non-throwing function
// do...catch handle all error
//
func loadConfig() -> String? {
  do {
    //     👇
    return try readFile(atPath: "path.config")
  } catch FileHandleError.notExists {    // 👈
    print("file not exists!")
    return nil
  } catch FileHandleError.unreadable {   // 👈
    print("file is unreadable!")
    return nil
  } catch {
    //      👇
    print(error)
    return nil
  }
}
```

通过 `do...catch` 也可以只处理部分错误类型，其他类型的错误继续向上传播，如：

```swift
// loadConfig is throwing function
// it only handles FileHandleError type
// other error types continue to propagate upwards 
//                  👇
func loadConfig() throws -> String {
  do {
    return try readFile(atPath: "path.config")
  } catch let error as FileHandleError {  // 👈，so also：catch is FileHandleError
    print(error)
    return ""
  }
}

```

# Clean-up actions

---

由于 Error Propagation 会改变正常的执行流程，在退出当前流程时可能有些清理工作需要执行，如：释放内部、关闭文件、断开 DB 等。

大多数语言通过 `try...catch...finally` 中的 `finally` 来执行清理工作，如：

```java
try {
  int[] array = new int[1];
  int i = array[2];
  
  // this statement will not be executed
  // as the above statement throws an exception
  System.out.println("in try block!");
}
catch(Exception e) {
  System.out.println("exception has been caught!");
}
finally {  // 👈
  System.out.println("in finally block!");
}
```

Swift 则是通过 `defer` 来处理清理工作，如：

```swift
func processFile(filename: String) throws {
  let file = open(filename)
  defer {   // 👈
      close(file)
  }
  
  while let line = try file.readline() {
      // Work with the file.
  }
  
  // close(file) is called here, at the end of the scope.
}

```

- `defer` 语句在执行流离开当前作用域时执行

- `defer` 不仅可以用于 Error handle 的清理工作中，正常执行流程也可以使用，通常用于需要配对执行的操作中，如：open-close、lock-unlock、alloc-release 等，如：
  
  ```swift
  {
    lock.lock()
    defer {
      lock.unlock()
    }
  
    // Do something under the protection of the lock ...
  }
  ```

- 在 `defer` 内部不允许任何可能会将控制流提前转移出 `defer-block` 的行为，如：return、break、throw (不允许在 `defer-block` 内抛出错误) 等
  
  ```swift
  defer {
    print("in defer block")
  
    // ❌ Errors cannot be thrown out of a defer body
    throw FileHandleError.invalidPath
    
    lock.unlock()
  }
  
  defer {
    print("in defer block")
  
    // ❌ 'return' cannot transfer control out of a defer statement
    return
  }
  ```

# Errors and Polymorphism

---

根据[「 **LSP** - Liskov Substitution Principle 」](https://juejin.cn/post/6844904003638132744#heading-10)：

- 基类中 non-throwing method，不能在子类中重载为 throwing method
  
  ```swift
  class Base {
    func test() {}
  }
  
  class Sub: Base {
    override
    func test() throws {}  // ❌ Cannot override non-throwing instance method with throwing instance method
  }
  ```

- 基类中 throwing method，在子类中可以重载为 non-throwing method
  
  ```swift
  class Base {
    func test() throws {}
  }
  
  class Sub: Base {
    override 
    func test() {}  // ✅
  }
  ```

- 同理，Protocol 中定义是 non-throwing method，那实现时一定不能是 throwing method，反之则可以
  
  ```swift
  protocol SomeProtocol {
    func test()
  }
  
  class SomeClass: SomeProtocol { // ❌ Type 'SomeClass' does not conform to protocol 'SomeProtocol'
    func test() throws {}
  }
  ```
  
  ```swift
  protocol SomeProtocol {
    func test() throws  // 👈
  }
  
  class SomeClass: SomeProtocol {
    func test() {}  // ✅
  }
  ```

# assert、precondition、fatalError

- assert，俗称断言，只在开发环境 (debug) 下生效，主要用于在开发阶段检查不应该发生的错误，如：对入参的合法性校验，以便尽早发现问题
  
  assert 可以理解为是对接口语义的补充，如下，ph 值的有效取值范围为 0~14 (无法在接口定义中表达出来)，可以通过 assert 的方式表达：
  
  ```swift
  func setPH(ph: Double) {
    assert(ph >= 0 && ph <= 14.0, "The valid value range of ph value is 0 to 14!")
  
    //...
  }
  ```
  
  由于其只在开发环境下有效，故还需要通过如上文所讲的 Optional、throw 等方式处理生产环境的问题

- precondition，与 assert 的区别在于其在生产环境 (release) 下也有效 ⚡️

- fatalError，无条件的终止程序的运行 ❌

> precondition、fatalError 一般用于无法恢复的严重错误、严重的安全问题 ❌
> 
> 作为卑微的应用开发者，是没有权利使用  precondition、fatalError 的！🥹



# 小结

---

合理、优雅地处理错误不是一件容易的事情，几点建议：

- 可恢复的错误用 Optional、Result、throws 处理

- 不可恢复的错误用 precondition、fatalError 处理 ⚡️

- assert 用于开发环境中提前发现不应出现的错误，可以增强接口语义

- **在接口定义中将错误语义表达出来，让调用方清晰地知道该接口可能出现的错误**
  
  - ***同步方法用 Optional、throws***
  
  - ***异步方法用 Closure + Result***



错误会沿调用链逆向传播，在合适的地方捕获并处理错误也是一件非常重要的事情，遇到过在底层网络请求中处理网络错误并弹 toast 的例子❗️

错误处理同样要遵守「单一职责」原则，只处理自己份内的错误，不该处理的错误往上抛！



# 参考资料

[Swift · Docs - Error Handling](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/errorhandling/)

[Swift · Docs - Error Handling Rationale and Proposal](https://github.com/apple/swift/blob/7c3e480218b05bc08dea382cf7a4f2384309fae0/docs/ErrorHandlingRationale.md)

[Error - Apple Developer Documentation](https://developer.apple.com/documentation/swift/error)

[Swift Error Handling Implementation](https://www.mikeash.com/pyblog/friday-qa-2017-08-25-swift-error-handling-implementation.html)

[Swift Error Handling Strategies: Result, Throw, Assert, Precondition and FatalError](https://www.vadimbulavin.com/swift-error-handling-with-result-throw-assert-precondition-and-fatalerror/)

[Understanding, Preventing and Handling Errors in Swift](https://khawerkhaliq.com/blog/swift-error-handling/)

[Failable Initializers](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/initialization/#Failable-Initializers)
