---
layout: post
title: Swift 最佳实践之 Generics
date: 2023-04-05 11:12:35
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

- Generics

- Property Wrapper

- Structured Concurrent

- Result builder

- Error Handle

- Advanced Collections (Asyncsequeue/OptionSet/Lazy)

- Expressible by Literal

- Pattern Matching

- Metatypes(.self/.Type/.Protocol)

> ps. 本系列不是入门级语法教程，需要有一定的 Swift 基础



本文是系列文章的第五篇，介绍  Generics，通过泛型可以写出更灵活、通用性更好的代码。

> Write code that works for multiple types and **specify requirements for those types.** -- [Swift Docs · Generics](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/generics/)



Swift 通过 Type Constraints 赋以 Generics 更强大的能力，可以更加灵活的控制 Generics 具备的能力和使用场景👍。

于此同时，因为 Generics 需要 Boxing 以及方法调用都是动态派发有一定的性能损耗。

为此，Swift 在编译时会做特化处理 (Specialization) 以优化 Generics 的性能。

Phantom Types 在 Swift 现有类型安全基础之上还可以进一步强化类型。



# 邂逅 Generics

---

在 Swift 中，可以定义泛型类型 (Generic class/struce/enum)，也可以定义泛型方法。

下面我们通过一个~~做作的~~典型的例子来逐步介绍 Swift Generics 的特性。

## Generic Types

实现一个自定义的 Array (`BetterArray`)：

```swift
public struct BetterArray {
  var storages: [Any] = []    // 🫢😵‍💫

  mutating func append(_ newEelement: Any) {
    stroages.append(newEelement)
  }
}
```

看起来还不错？

but，元素类型怎么是 `Any`❓

很不 "Swift"！

元素类型又不能写死，那该怎么办？🤔

这时就轮到泛型登场了：

```swift
//                       👇
public struct BetterArray<T> {
  var storages: [T] = []
  
  mutating func append(_ newElement: T) {
    storages.append(newElement)
  }
}
```

如上，为 `BetterArray` 添加了泛型 (`T`)

对泛型名 `T` 不是很满意，可以给它起个更有意义的名字：

```swift
//                          👇
public struct BetterArray<Element> {
  var storages: [Element] = []
  
  mutating func append(_ newElement: Element) {
    storages.append(newElement)
  }
}
```

初始化 `BetterArray` 时需指定泛型的具体类型，如：

```swift
//                            👇
var betterArray = BetterArray<Int>()
```



目前 `BetterArray` 的功能有点简单，给它添加一个 `index(of:)` 的能力，即检索某个元素的 index：

```swift
func index(of element: Element) -> Int? {
  storages.firstIndex(of: element)
}
```

很遗憾，编译报错：

```swift
Referencing instance method 'firstIndex(of:)' on 'Collection' requires that 'Element' conform to 'Equatable'
```

简单讲，就是要求 `BetterArray` 中的元素实现 `Equatable` 协议。

问题不大，Generics 可以添加类型约束 (Type Constraints)。

## Type Constraints

```swift
// 也可以用 where clause: 
// public struct BetterArray<Element> where Element: Equatable
//                                    👇
public struct BetterArray<Element: Equatable> {
  var storages: [Element] = []
  
  mutating func append(_ newElement: Element) {
    storages.append(newElement)
  }
  
  func index(of element: Element) -> Int? {
    storages.firstIndex(of: element)
  }
}
```

完美！👍

but，可能会接到投诉，我只是要用 `BetterArray` 做些存储，并不需要调用 `index(of:)` 方法，凭啥要实现 `Equatable` ？！

问题大不，Type Constraints 不仅可以加在 Generics 类型定义时，也可以通过「 where clause 」加在具体方法上：

```swift
public struct BetterArray<Element> {
  // ...
  
  //                                                        👇
  func index(of element: Element) -> Int? where Element: Equatable {
    storages.firstIndex(of: element)
  }
}
```

这时，只要不调用 `index(of:)` 方法，任何类型都可以用 `BetterArray`：

```swift
struct DemoElement {}

var betterArray = BetterArray<DemoElement>()    // ✅
betterArray.append(DemoElement())               // ✅

// ❌ Instance method 'index(of:)' requires that 'DemoElement' conform to 'Equatable'
betterArray.index(of: DemoElement())
```

`BetterArray` 还需要个 `remove` 功能 🤨：

```swift
public struct BetterArray<Element> {
  // ...
  
  //                                                        👇
  func index(of element: Element) -> Int? where Element: Equatable {
    storages.firstIndex(of: element)
  }

  //                                                              👇
  mutating func remove(_ nouseElement: Element) where Element: Equatable {
    storages.removeAll { $0 == nouseElement }
  }
}
```

如上，`index(of)`、`remove` 两个方法都要求 `Element` 实现 `Equatable`， 此时可以为 `BetterArray` 增加一个分类，并将 Type Constraints 统一放在分类上：

```swift
public struct BetterArray<Element> { /* ... */ }

//                                      👇
extension BetterArray where Element: Equatable {
  func index(of element: Element) -> Int? {
    storages.firstIndex(of: element)
  }
  
  mutating func remove(_ nouseElement: Element) {
    storages.removeAll { $0 == nouseElement }
  }
}
```



关于 Generic Type Constraints，有三种情况：

- **Protocol Constraints**：如上所示，要求类型实现某个协议 (`where Element: Equatable`)；

- **Class Constraints**：要求类型是某个类的子类 (where Element: UIView)，如：
  
  ```swift
  extension BetterArray where Element: UIView {
    func subviews(at index: Int) -> [UIView]? {
      guard index < storages.count else {
        return nil
      }
      
      return storages[index].subviews
    }
  }
  ```

- **Same-type Constraints**：要求类型是某个具体的类型值 (`where Element == String`)，如：
  
  ```swift
  extension BetterArray where Element == String {
    func splice() -> Element {
      storages.reduce("") { partialResult, element in
        partialResult + element
      }
    }
  }
  ```
  
  Same-type Constraints 一般只出现在 extension 或 具体某个方法上，若出现在类型定义上就没有意义了，如：
  
  ```swift
  // ⚠️ Same-type requirement makes generic parameter 'T' non-generic; this is an error in Swift 6
  struct BadArray<T> where T == String {}
  ```

> Protocol associatedtype Constraints 也是上面 3 种情况。



总之，Type Constraints 赋以 Generics 更大的操作空间。

> 不加 Type Constraints 的泛型除了存储，其他基本上什么也做不了！
> 
> 连实例化都做不了，因为没有`init`方法！



## Generic Functions

`BetterArray` 怎么能少了「函数式」的能力呢，加个 `map`：

```swift
public struct BetterArray<Element> {
  //      👇
  func map<T>(_ transform: (Element) throws -> T) rethrows -> [T] {
    try storages.map(transform)
  }
}
```

如上，不仅可以给类型 (Class、Struct、Enum) 加上 Generics，还可以给方法添加 Generics。

泛型方法的调用不需要显式指定对应的具体类型：

```swift
// 通过 Inferring Type，可知具体类型为 String，不需要手动指定
//
let result = betterArray.map { _ in "" }
```

> 正如在 [Swift 最佳实践之 Protocol](https://juejin.cn/post/7217360688263036985) 中介绍的，从 Swift 5.7 起，对于有 **Protocol Constraints** 的泛型方法可以用 `some` 关键字改写，更简洁：
> 
> ```swift
> func someDemo<P: Equatable>(_ other: P) -> Bool {
>   // ...
> }
> 
> // Equivalent to 👇
> 
> func someDeom(_ other: some Equatable) -> Bool {
>   // ...
> }
> ```

# "深入" Generics

---

编译器是如何处理 Generics 的？🤔

根据 [Swift 最佳实践之 Protocol](https://juejin.cn/post/7217360688263036985) 中相关经验看，应该不简单🧐

总的来说，Swift 对 Generics 的处理分 2 种情况：

- **运行时**，对 Generics 做装箱处理 (Boxing)

- **编译时**，对 Generics 做特化处理 (Specialization)



## Boxing

所谓 Boxing，与用于处理 Protocol 作为类型 (*Existential Type*) 时的 **Existential Container** 非常类似。

简单来说，就是要对 Generics 做一次封装转换，Generics 在使用是真实类型可能千差万别，但 Generics 定义是需要有「固定的对象模型」。

> 所谓对象模型 (Object Model)，主要有几个职责：
> 
> - 指导对象实例化时属性如何存储
> 
> - 指导对象如何执行 allocate、copy、destroy 等基础内存操作以及获取 size、alignment 等内存信息
> 
> - 指导如何查找实例方法的入口地址



如上节所述，根据 Generic Type Constraints 的不同，可以分为三种情况：

- **No Constraints**，这类泛型能做的事非常少，Boxing 只需关心 allocate、copy、destroy 等基本操作如何执行即可

- **Class Constraints**，有基础类作为约束，除了 allocate、copy、destroy 以外，还需要通过 *VWT (Value Witness Table)* 存储约束类中定义的方法，以便通过 generic-types 可以调用到它们

- **Protocol Constraints**，除了 allocate、copy、destroy 以外，还需要通过 *PWT (Protocol Witness Table)* 存储协议中指定的方法，以便通过 generic-types 可以调用它们
  
  > 这里讨论的 Protocol 是没有 class constraint 的，对于只能由类实现的协议作为泛型约束时，其效果同上面讨论的 Class Constraints。



> 通过 [SIL](https://github.com/apple/swift/blob/main/docs/SIL.rst) (Swift Intermediate Language) 可以大致了解 Swift 背后的实现原理。
> 
> `swiftc demo.swift -O -emit-sil -o demo-sil.s`
> 
> 如上，通过 swiftc 命令可以生成 SIL。
> 
> 其中的 `-O` 是对生成的 SIL 代码进行编译优化，使 SIL 更简洁高效。
> 
> 后面要讲到的泛型特化 (Specialization of Generics) 也只有在 `-O` 优化下会发生。



总之，Generics 对性能有影响，主要体现在 2 个方面：

- Boxing 处理

- 通过 Generics 调用的方法都是动态派发 (通过 VWT 或 PWT)



## Specialization

Generics 带来的性能影响可以通过特化 (Specialization of Generics) 来优化。

所谓特化就是生成泛型的特定版本，将泛型转换为非泛型，如：

```swift
@inline(never)
func swapTwoValues<T>(_ a: inout T, _ b: inout T) {
  let temp = a
  a = b
  b = temp
}

var a = 1
var b = 2
swapTwoValues(&a, &b)
```

如上，通过 `Int` 型参数调用 `swapTwoValues` 时，编译器就会生成该方法的 `Int` 版本：

```swift
// specialized swapTwoValues<A>(_:_:)
sil shared [noinline] @$s4main13swapTwoValuesyyxz_xztlFSi_Tg5 : $@convention(thin) (@inout Int, @inout Int) -> () {
// %0 "a"                                         // users: %6, %4, %2
// %1 "b"                                         // users: %7, %5, %3
bb0(%0 : $*Int, %1 : $*Int):
  debug_value_addr %0 : $*Int, var, name "a", argno 1 // id: %2
  debug_value_addr %1 : $*Int, var, name "b", argno 2 // id: %3
  %4 = load %0 : $*Int                            // user: %7
  %5 = load %1 : $*Int                            // user: %6
  store %5 to %0 : $*Int                          // id: %6
  store %4 to %1 : $*Int                          // id: %7
  %8 = tuple ()                                   // user: %9
  return %8 : $()                                 // id: %9
} // end sil function '$s4main13swapTwoValuesyyxz_xztlFSi_Tg5'
```

那么，什么时候会进行泛型特化呢？

总的原则是在编译泛型方法时知道有哪些调用方，同时调用方的类型是可推演的。

最简单的情况就是泛型方法与调用方在同一个源文件里，一起进行编译。

另外，在编译时若开启了 [Whole-Module Optimization](https://www.swift.org/blog/whole-module-optimizations/)，同一模块内部的泛型调用也可以被特化。



# Phantom Types

---

Phantom Types 并非 Swift 特有的，属于一种通用编码技巧。

Phantom Types 没有严格的定义，一般表述是：**出现在泛型参数中，但没有被真正使用**。

如下代码中的 `Role` (例子来自 [How to use phantom types in Swift](https://www.hackingwithswift.com/plus/advanced-swift/how-to-use-phantom-types-in-swift))，它只出现在泛型参数中，在 `Employee` 实现中并未使用：

```swift
struct Employee<Role>: Equatable {
    var name: String
}
```

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

```swift
Cannot convert value of type 'Employee' to expected argument type 'Employee'
```

> 将 Phantom Types 定义成空 enum，使其无法被实例化，从而真正满足 Phantom Types 语义。



# 小结

本文对 Swift Generics 进行了简要介绍，通过 Generics + Type Constraints 可以写出非常灵活实用的代码。

 Generics 也会带来一定的性能损耗，通过泛型特化 (Specialization) 可以优化 Generics 性能。

Phantom Types 作为一种通用编码技巧，在 Swift 中同样可以用来实现类型增加。



# 参考资料

[Embrace Swift generics - WWDC22 - Videos](https://developer.apple.com/videos/play/wwdc2022/110352/)

[Swift Generics (Expanded) - WWDC18 - Videos](https://developer.apple.com/videos/play/wwdc2018/406/)

[Swift Docs · Generics](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/generics/)

[swift/OptimizationTips.rst at main · apple/swift · GitHub](https://github.com/apple/swift/blob/main/docs/OptimizationTips.rst)

[Whats behind swift generic system?](https://nekitosss.github.io/programming/2019-05-12-swift-generics/)

[Swift.org - Whole-Module Optimization in Swift 3](https://www.swift.org/blog/whole-module-optimizations/)

[swift/SIL.rst at main · apple/swift · GitHub](https://github.com/apple/swift/blob/main/docs/SIL.rst)

[How to use phantom types in Swift](https://www.hackingwithswift.com/plus/advanced-swift/how-to-use-phantom-types-in-swift)

[Measurements and Units with Phantom Types](https://oleb.net/blog/2016/08/measurements-and-units-with-phantom-types/)

[Phantom types in Swift](https://www.swiftbysundell.com/articles/phantom-types-in-swift/)

[Building type-safe networking in Swift](https://swiftwithmajid.com/2021/02/10/building-type-safe-networking-in-swift/)

[Type-Safe File Paths with Phantom Types - Swift Talk](https://talk.objc.io/episodes/S01E71-type-safe-file-paths-with-phantom-types)
