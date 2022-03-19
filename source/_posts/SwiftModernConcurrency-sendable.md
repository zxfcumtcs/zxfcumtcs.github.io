---
layout: post
title: Swift 新并发框架之 Sendable
date: 2022-03-19 15:46:45
tags:
- iOS
- Swift
---
本文是 『 Swift 新并发框架 』系列文章的第三篇，主要介绍 Swift 5.6 引入的 Sendable。

<!--more-->

©原创文章，转载请注明出处！

本系列文章对 Swift 新并发框架中涉及的内容逐个进行介绍，内容如下：

- [Swift 新并发框架之 async/await](https://zxfcumtcs.github.io/2022/03/18/SwiftModernConcurrency-asyncawait/)

- [Swift 新并发框架之 actor](https://zxfcumtcs.github.io/2022/03/19/SwiftModernConcurrency-actor/)

- Swift 新并发框架之 Sendable

- Swift 新并发框架之 Task


# Overview
---

书接前文 ([『 Swift 新并发框架之 actor 』](https://zxfcumtcs.github.io/2022/03/19/SwiftModernConcurrency-actor/))，本文主要介绍 Sendable 为何物以及如何解决前文提到的那些问题。

```swift
/// The Sendable protocol indicates that value of the given type can
/// be safely used in concurrent code.
public protocol Sendable {}
```

`Sendable` 是一个空协议：

**用于向外界声明实现了该协议的类型在并发环境下可以安全使用，更准确的说是可以自由地跨 actor 传递。**

这属于一种 『 语义 』上的要求。

像 `Sendable` 这样的协议有一个专有名称：*『 Marker Protocols 』*，其具有以下特征：

- 具有特定的语义属性 (semantic property)，且它们是编译期属性而非运行时属性。
  
  如 `Sendable` 的语义属性就是要求并发下可以安全地跨 actor 传递；

- 协议体必须为空；

- 不能继承自 non-marker protocols (这其实是第 2 点的延伸)；

- 不能作为类型名用于 `is`、`as?`等操作
  
  如：x is Sendable，编译报错: Marker protocol 'Sendable' cannot be used in a conditional cast.

- 不能用作泛型类型的约束，从而使某类型遵守一个 non-marker protocol，如：
  
  ```swift
   protocol P {
     func test()
   }
  
   class A<T> {}
  
   // Error: Conditional conformance to non-marker protocol 'P' cannot depend on conformance of 'T' to non-marker protocol 'Sendable'
   extension A: P where T: Sendable {
     func test() {}
   }
  ```

我们知道，值语义 (Value semantics) 类型在传递时 (如作为函数参数、返回值等) 是会执行拷贝操作的，也就是它们跨 Actor 传递是安全的。故，这些类型隐式地自动遵守 `Sendable` 协议，如：

- 基础类型，`Int`、`String`、`Bool` 等；

- 不含有引用类型成员的 `struct`；

- 不含有引用类型关联值的 `enum`；

- 所含元素类型符合 `Sendable` 协议的集合，如：`Array`、`Dictionary` 等。

当然了，所有 actor 类型也是自动遵守 `Sendable` 协议的。

> 事实上是所有 actor 都遵守了 `Actor`协议，而该协议继承自 `Sendable`：

```swift
@available(macOS 10.15, iOS 13.0, watchOS 6.0, tvOS 13.0, *)
public protocol Actor : AnyObject, Sendable {
    nonisolated var unownedExecutor: UnownedSerialExecutor { get }
}
```

class 需要主动声明遵守 `Sendable` 协议，并有以下限制：

- class 必须是 `final`，否则有 Warning: Non-final class 'X' cannot conform to 'Sendable'; use ' @unchecked Sendable'

- class 的存储属性必须是 immutable，否则有 Warning: Stored property 'x' of 'Sendable'-conforming class 'X' is mutable

- class 的存储属性必须都遵守 `Sendable` 协议，否则 Warning: Stored property 'y' of 'Sendable'-conforming class 'X' has non-sendable type 'Y'

- class 的祖先类 (如有) 必须遵守 `Sendable` 协议或者是 `NSObject`，否则 Error: 'Sendable' class 'X' cannot inherit from another class other than 'NSObject'。

以上这些限制都很好理解，都是确保实现了 `Sendable` 协议的类数据安全的必要保障。

回到上面那个例子：

```swift
extension AccountManager {
  func user() async -> User {
    // Warning: Non-sendable type 'User' returned by implicitly asynchronous call to actor-isolated instance method 'user()' cannot cross actor boundary
    return await bankAccount.user()
  }
}
```

很明显，要消除例子中的 Warning，只需让 `User` 实现 `Sendable`协议即可。

就本例而言，`User`有 2 种改造方案：

- 由 class 改成 struct：
  
  ```swift
  struct User {
    var name: String
    var age: Int
  }
  ```

- 手动实现 `Sendable` 协议：
  
  ```swift
  final
  class User: Sendable {
    let name: String
    let age: Int
  }
  ```

回头想想，`Sendable` 对实现它的 class 的要求是不是太严格了 (final、immutable property) ？！

有点过于理想，有点不切实际

从并发安全的角度说，完全可以通过传统的串行队列、锁等机制保障。

此时，可以通过 `@unchecked` attribute 告诉编译器不进行 `Sendable` 语义检查，如：

```swift
// 相当于说 User 的并发安全由开发人员自行保证，不用编译器检查
class User: @unchecked Sendable {
  var name: String
  var age: Int
}
```

`Sendable` 作为协议只能用于常规类型，对于函数、闭包等则无能为力。

此时，就轮到 `@Sendable` 登场了。

# @Sendable

___

被 `@Sendable` 修饰的函数、闭包可以跨 actor 传递。

对于前文提到的例子：

```swift
extension BankAccount {
  func addAge(amount: Int, completion: (Int) -> Void) {
    age += amount
    completion(age)
  }
}

extension AccountManager {
  func addAge() async {
    // Wraning: Non-sendable type '(Int) -> Void' passed in implicitly asynchronous call to actor-isolated instance method 'addAge(amount:completion:)' cannot cross actor boundary
    await bankAccount.addAge(amount: 1, completion: { age in
      print(age)
    })
  }
}
```

只需对 `addAge` 方法的 `completion` 参数加上 `@Sendable` 即可：

```swift
func addAge(amount: Int, completion: @Sendable (User) -> Void)
```

总结一下，用`@Sendable` 修饰 Closure 真正意味着什么？

**其实是告诉 Closure 的实现者，该 Closure 可能会在并发环境下调用，请注意数据安全！**

*因此，如果对外提供的接口涉及 Closure (作为方法参数、返回值)，且其可能在并发环境下执行，就应用 `@Sendable`修饰。*

> 根据这一原则，actor 对外的方法如涉及 Closure，也应用 `@Sendable`修饰。

```swift
extension Task where Failure == Error {
  public init(priority: TaskPriority? = nil, operation: @escaping @Sendable () async throws -> Success)
}
```

如 `Task` 的 `operation` 闭包会在并发环境下执行，故用了 `@Sendable` 修饰。

当然编译器会对 @Sendable Closure 的实现进行各种合规检查：

- 不能捕获 actor-isolated 属性，否则 Error: Actor-isolated property 'x' can not be referenced from a Sendable closure；(原因也很简单，@Sendable Closure 可能会在并发环境下执行，这与 actor 串行保护数据有冲突)
  
  > 如果 @Sendable 闭包是异步的 (@Sendable () async )，则不受此限制。
  > 
  > 大家可以思考一下是为啥？

- 不能捕获 `var` 变量，否则 Error: Mutation of captured var 'x' in concurrently-executing code；

- 所捕获对象必须实现 Sendable 协议，否则 Warning: Capture of 'x' with non-sendable type 'X' in a `@Sendable` closure。

还记得 [Swift 新并发框架之 actor](https://zxfcumtcs.github.io/2022/03/19/SwiftModernConcurrency-actor/) 中最后那个 crash 吗：

```swift
extension User {
  func testUser(callback: @escaping () -> Void) {
    for _ in 0..<1000 {
      DispatchQueue.global().async {
        callback()
      }
    }
  }
}

extension BankAccount {
  func test() {
    let user = User.init(name: "Tom", age: 18)
    user.testUser {
      let b = self.balances[1] ?? 0.0
      self.balances[1] = b + 1
      print("i = \(0), \(Thread.current), balance = \(String(describing: self.balances[1]))")
    }
  }
}
```

这个 crash 该如何 fix 呢？

可以思考一下~

```swift
extension User {
  // 由于 callback 会在并发环境下执行，故用 `@Sendable` 修饰
  // 一般情况下，@Sendable closure 都是异步的，否则受限于 @Sendable 的规则无法捕获 Actor-isolated property
  func test(callback: @escaping @Sendable () async -> Void) {
    for _ in 0..<1000 {
      DispatchQueue.global().async {
        // 在同步上下文中一般通过 Task 开启一个异步上下文
        Task{
          await callback()
        }
      }
    }
  }
}

extension BankAccount {
  func changeBalances(newValue: Double) {
    balances[1] = newValue
  }

  func test() {
    let user = User.init(name: "Tom", age: 18)

    user.test { [weak self] in
      guard let self = self else { return }

      let b = await self.balances[1] ?? 0.0
      // 对 Actor-isolated property 的修改需提取到单独的方法里
      // 不能直接在 @Sendable 闭包修改
      await self.changeBalances(newValue: b + 1)
      print("i = \(0), \(Thread.current), balance = \(String(describing: await self.balances[1]))")
    }
  }
}
```

# Future Improvement

---

Apple 在 [Protect mutable state with Swift actors - WWDC21](https://developer.apple.com/videos/play/wwdc2021/10133/) 上提到将来 Swift 编译器会禁止共享 (传递) 非 Sendable 类型的实例。

![Compiler error when try to share nonSendable types from actors in Swift](https://i1.wp.com/swiftsenpai.com/wp-content/uploads/2021/10/sendable-data-races-non-sendable-share-error-1024x653.png?resize=1024%2C653&ssl=1)

那么，本文提到的所有 Warning 都将变成 Error！

好了，关于 Sendable 就聊这么多！

# 小结

- `Sendable` 本身是一个 Marker Protocol，用于编译期的合规检查；

- 所有值语义类型都自动遵守 `Sendable` 协议；

- 所有遵守 `Sendable` 协议的类型都可以跨 actor 传递；

- `@Sendable` 用于修饰方法、闭包；

- 对于会在并发环境下执行的闭包都应用 `@Sendable` 修饰。

# 参考资料

[swift-evolution/0296-async-await.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md)

[swift-evolution/0302-concurrent-value-and-concurrent-closures.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)

[swift-evolution/0337-support-incremental-migration-to-concurrency-checking.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0337-support-incremental-migration-to-concurrency-checking.md)

[swift-evolution/0304-structured-concurrency.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#jobs)

[swift-evolution/0306-actors.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)

[swift-evolution/0337-support-incremental-migration-to-concurrency-checking.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0337-support-incremental-migration-to-concurrency-checking.md)

[Understanding async/await in Swift • Andy Ibanez](https://www.andyibanez.com/posts/understanding-async-await-in-swift/)

[Concurrency — The Swift Programming Language (Swift 5.6)](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html#)

[Connecting async/await to other Swift code | Swift by Sundell](https://www.swiftbysundell.com/articles/connecting-async-await-with-other-swift-code/)