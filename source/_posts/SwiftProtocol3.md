---
layout: post
title: Swift Protocol 背后的故事(Swift 5.6/5.7)
date: 2022-06-30 22:21:10
tags:

- iOS
- Swift
- Protocol
---

本系列文章将从实践技巧、实现原理以及追踪语言更新等方面对 Swift Protocol 展开深入讨论。主要内容有：

[Swift Protocol 背后的故事(实践)](https://zxfcumtcs.github.io/2022/02/01/SwiftProtocol1/)
[Swift Protocol 背后的故事(理论)](https://zxfcumtcs.github.io/2022/02/04/SwiftProtocol2/)
Swift Protocol 背后的故事(Swift 5.6/5.7)
...

本文是系列文章第三篇，简要介绍 Swift 5.6/5.7 在 Protocol 上的相关扩展和优化，主要包括：`any`、Opaque Parameter、Unlock existentials for all protocols 以及 Primary Associated Types。

<!--more-->

©原创文章，转载请注明出处！

# Overview

____

含有 `Self requirements` 、`Associated Type` 的协议不能作为类型。

> Protocol 'Equatable' can only be used as a generic constraint because it has Self or associated type requirements.

在『 [Swift Protocol 背后的故事(实践)](https://zxfcumtcs.github.io/2022/02/01/SwiftProtocol1/) 』中介绍了相关的解决方案：
+ Type Erasure；
+ Opaque Types，并由此引入了 `some` 关键字。

在『[Swift Protocol 背后的故事(理论)](https://zxfcumtcs.github.io/2022/02/04/SwiftProtocol2/) 』中详细分析了直接将协议作为类型使用时(Existential Types)，编译器是如何处理的，并得出结论：**对性能有较大影响。**

在 Swift 5.6、5.7 中对上述问题做了不少改进和优化，本文主要对这些改进优化做一个简要介绍：

+ `any`

+ Opaque Parameter

+ Unlock existentials for all protocols

+ Primary Associated Types

# `any`

____

通过前两篇文章的介绍，我们知道对于没有 `Self requirements` 和关联类型的协议，可以单独作为类型使用，并称之为『 Existential Type 』。

应该说，『 Existential Type 』用起来还是挺香的~

```swift
protocol Animal {
  func eat()
}

struct Farm {
  func genericsFeed<T>(_ animal: T) where T: Animal {
    animal.eat()
  }

  func existentialFeed(_ animal: Animal) {
    animal.eat()
  }
}
```

如上，从『 功能使用 』角度看，`genericsFeed`、`existentialFeed`没有任何区别，都能达到泛型效果。

但从『 代码实现 』角度看，显然 `existentialFeed` 更简洁明了。

作为开发人员，无意识中更容易写出 `existentialFeed`，毕竟简单。

这显然不是好现象，毕竟『 Existential Type 』有性能问题。

Swift 团队也意识到这个问题，故在 Swift 5.6 引入了 `any` 关键字 ([swift-evolution/0335-existential-any](https://github.com/apple/swift-evolution/blob/main/proposals/0335-existential-any.md))：

+ `any` 本身没有任何功能，只是一个标记；

+ 当把协议作类型用时，在其前添加 `any` 来显式表明开发人员要使用『 Existential Type 』的意图；

+ 目前 `any` 还不是强制的，但从 Swift 6.0 开始将强制使用 `any`，否则编译报错。因此，尽早用上 `any`，以免后期升级成本过高。

```swift
struct Farm {
  var animals: [any Animal]

  func existentialFeed(_ animal: any Animal) {
    animal.eat()
  }

  func firstAnimal() -> (any Animal)? {
    animals.first
  }
}
```

如上，在协议用作类型的地方加上 `any` 即可。

相当于显式声明这是个『 Existential Type 』！

这是一个悲壮的故事，意味着性能上损耗！

那么，有没有办法做到：*既有泛型的性能又有『 Existential Types 』的简洁？*

鱼和熊掌兼得？！

答案是肯定的！请往下看~

# Opaque Parameter

____

还记得在『 [Swift Protocol 背后的故事(实践)](https://zxfcumtcs.github.io/2022/02/01/SwiftProtocol1/) 』中介绍的 `some` 吗？

Swift 5.1 引入了 `some` 关键字，用于 [Opaque Result Type](https://github.com/apple/swift-evolution/blob/main/proposals/0244-opaque-result-types.md)。

Swift 5.7 对 `some` 的功能做了扩展，可用于声明方法参数，即：Opaque Parameter。

```swift
struct Farm {  
  func genericsFeed<T>(_ animal: T) where T: Animal {
    animal.eat()
  }

  // some Animal 可以理解为一个匿名的具体类型
  // 并且该类型实现了 Animal 协议
  //
  func someFeed(_ animal: some Animal) {
    animal.eat()
  }
}
```

如上，`someFeed` 在性能上与 `genericsFeed` 无任何差别，但更简洁、可读性更好。

**Opaque Parameter 可以理解为泛型的简化版本。**

关于 Opaque Parameter 有 2 点需要注意：

+ 在代替泛型时，仅适用于泛型类型只使用一次的场景，因为 Opaque Parameter 是匿名的，无法复用：
  
  ```swift
  // t1 与 t2 具有相同的类型
  //
  func isEqualGenerics<T>(t1: T, t2: T) -> Bool where T: Equatable {
    t1 == t2
  }
  
  // 并不要求 t1、t2 的类型完全相同
  // 只要它们都实现了 Equatable 即可
  //
  func isEqualSome(t1: some Equatable, t2: some Equatable) -> Bool {
    t1 == t2
  }
  ```
  如上，`isEqualGenerics` 中的泛型类型 `T` 使用了2 次，分别用于定义`t1`、`t2`。
  因此，`isEqualGenerics` 与 `isEqualSome` 并不等价，并且 `isEqualSome` 会报错：
  *Cannot convert value of type (generic parameter of global function isEqualSome(t1:t2:)) to expected argument type (generic parameter of global function isEqualSome(t1:t2:))*

+ 与 Opaque Result Type 不同，可以使用不能类型的实例调用具有 Opaque Parameter 的方法：
  
  ```swift
  struct Cat: Animal {
    func eat() {
      print("Eat fish!")
    }
  }
  
  struct Panda: Animal {
    func eat() {
      print("Eat bamboo!")
    }
  }
  
  struct Farm {
    func feedAnimals() {
      let cat = Cat()
      let panda = Panda()
  
      someFeed(cat)      // ✅
      someFeed(panda)    // ✅
    }
  
    func someFeed(_ animal: some Animal) {
      animal.eat()
    }
  }
  ```
  如上，用 `cat`、`panda` 都可以调用 `someFeed` 方法。
  因为，`someFeed` 本质上是一个泛型方法。

# `any` VS. `some`

____

`any`、`some` 都可以配合协议作类型用。

它们的区别是啥？

我们该如何选择？

其实，通过前两篇文章对『 Opqaue type 』与『 Existential type 』的介绍，它们的区别已经比较清楚了。

下面，我们再来直观感受一下：

![](/img/some-existentialtype.png)

如上：

+ 通过 `some` 定义的变量是『 Opqaue type 』；

+ 通过 `any` 定义的变量是 『 Existential type 』；

+ 『 Opqaue type 』是『 匿名 』的、但『真实存在 』的类型，如 `someCat` 其背后的类型就是 `Cat`，在方法派发上有较好的性能；
  
  正是因为『 Opqaue type 』是『 匿名 』的，像下面这些写法都是不允许的：
  
  ```swift
  var cat: some Animal = Cat()    // ✅
  cat = Cat()   // ❌  Cannot assign value of type 'Cat' to type 'some Animal'
  
  var copyCat: some Animal = cat    // ✅
  copyCat = cat   // ❌  Cannot assign value of type 'some Animal' (type of 'cat') to type 'some Animal' (type of 'copyCat')
  ```
  『 Opqaue type 』类型的变量一旦初始化完成，就不能修改，因此像下面这种写法也是不允许的：
  ```swift
  var animals: [some Animal] = [
    Cat()   // ✅
  ]
  
  var cat1: some Animal = Cat()
  animals.append(cat1)   // ❌  No exact matches in call to instance method 'append'
  ```

+ 『 Existential type 』是二次封装后的间接类型，在变量赋值时有一个装箱 (Boxing) 的过程，方法派发一律是动态派发，故性能较差。有失必有得，其灵活性比较好：
  
  ```swift
  var cat: any Animal = Cat()
  cat = Cat()   // ✅
  
  var copyCat: any Animal = cat    // ✅
  copyCat = cat   // ✅
  ```
  ```swift
  var animals: [any Animal] = [
    Cat()   // ✅
  ]
  
  var cat1: any Animal = Cat()
  animals.append(cat1)   // ✅
  ```
  ```swift
  // ❌  Type of expression is ambiguous without more context
  //
  let someAnimals: [some Animal] = [
    Cat(),
    Panda()
  ]
  
  // ✅
  //
  let anyAnimals: [any Animal] = [
    Cat(),
    Panda()
  ]
  ```

Blabla，说这么多，具体开发时该怎么选？

**Write `some` by default.**

只有当 `some` 不能满足需要时 (也就是当你遇到无法解决的编译错误时)，才考虑用 `any`。

对了，这是 Apple 的建议。



**通过 Opaque Parameter，可以兼得泛型的性能和『 Existential type 』的简洁！**

心情是不是美美哒^__^

别急，还有更美的事呢！

# Unlock existentials for all protocols

____

从 Swift 5.7 开始，不再会有类似下面的编译错误：

*Protocol 'Equatable' can only be used as a generic constraint because it has Self or associated type requirements.*

也就是说，**所有的协议都可以作为类型用！(Unlock all protocols)**

*无论是否有 `Self requirements` 、是否有`Associated Type`！*

喜！

惊？

在『 [Swift Protocol 背后的故事(实践)](https://zxfcumtcs.github.io/2022/02/01/SwiftProtocol1/) 』中对上面那个编译错误的解释是：

**归根结底是因为 Protocol 是运行时特性，而其附带的 `Self requirements` / `Associated Type` 却需要在编译时保证。**

难道现在这个解释不成立了？

非也！

Apple 解锁所有 Protocols 的出发点是：

协议中除了有跟 `Self requirements`、`Associated Type` 相关的接口，还有不相关的接口。

而那些不相关的接口应该可以被正常使用，如下：

```swift
protocol Animal {
  associatedtype FeedType

  func feed(_ feed: FeedType)
  func drink()
}

struct Farm {
  // 注意，对于含有 `Self requirements`、`Associated Type` 的协议，必须在前面加上 any
  // 否则报错：Use of protocol 'Animal' as a type must be written 'any Animal'
  //
  func drinkAnimal(_ animal: any Animal) {
    animal.drink()
  }
}
```

虽然 `Animal` 含有关联类型 (`FeedType`)，但其中的 `drink` 接口与关联类型无关。

因此，`drink` 理应可以正常调用。

Swift 5.7 就是这么做的，故上面的代码在 Swift 5.7 上可以正常编译 ✅ 。

当然了，与 `Self requirements`、 `Associated Type` 有关的接口还是不能调用，如 `func feed(_ feed: FeedType)`：

```swift
struct Farm {
  func drinkAnimal(_ animal: any Animal) {
    // ✅
    //
    animal.drink()
  }
  
  func feedAnimal(_ animal: any Animal) {
    // ❌ Member 'feed' cannot be used on value of type 'any Animal'; consider using a generic constraint instead
    //
    animal.feed(???)
  }
}
```

Unlock protocols 虽好，但还不够好。

毕竟，使用上还有限制。

能不能做的更好？！

答案是肯定的！请往下看~

# Primary Associated Types

____

除了上面提到的 Unlock protocols 不能放开了用，还有一类问题也值得我们关注：

```swift
struct Farm { 
  func drinkAnimals(_ animals: any Sequence) {
    // ❌  Value of type 'Any' has no member 'drink'
    //
    animals.forEach { $0.drink() }
  }
}
```

如上，我们期望 `drinkAnimals` 能批量的对 Animals 调用 `drink`方法。

但事与愿违，因为集合 `animals` 的元素类型未知，编译器看到的都是 `Any`。

传统地，得像下面这样改造：

```swift
struct Farm {
  func drinkAnimals<T: Sequence>(_ animals: T) where T.Element: Animal {
    animals.forEach { $0.drink() }
  }
}
```

又回到泛型的老路上了！

上述这两个问题，归根结底都是关联类型惹的祸！

此时，就轮到 Primary Associated Types 登场了~



『 Primary Associated Types 』是啥？

不防先来看个例子，直观感受一下：

这是在 Swift <= 5.6 版本上，`Sequence` 的定义：

```swift
public protocol Sequence {
  /// A type representing the sequence's elements.
  associatedtype Element where Self.Element == Self.Iterator.Element
  ...
}
```

在 Swift 5.7 上的定义：

```swift
public protocol Sequence<Element> {
  /// A type representing the sequence's elements.
  associatedtype Element where Self.Element == Self.Iterator.Element
  ...
}
```

因此，可以对 `drinkAnimals` 作如下改造：

```swift
struct Farm {
  // 指定 animals 中的元素需实现 Animal 协议
  //
  func drinkAnimals(_ animals: any Sequence<Animal>) {
    animals.forEach { $0.drink() }  // ✅
  }
}
```

简单讲，『 Primary Associated Types 』：

+ 可以通过 『 泛型约束 』的方式将协议中的关联类型曝露出来，如：
  
  ```swift
  // Element 是 Sequence 中定义的关联类型
  //
  public protocol Sequence<Element>
  ```

+ 在使用时可以指定对关联类型的约束，如：
  
  ```swift
  var animals: any Sequence<Animal>
  var animals: some Sequence<Animal>
  ```

+ 需要注意的是，『 Primary Associated Types 』与『 泛型约束 』不是一回事，在使用时可以不指定，如在 Swift 5.7 上下面的写法也是合法的：
  
  ```swift
  var animals: any Sequence    // ✅
  ```

+ 还可以在 extension 上使用『 Primary Associated Types 』，更简洁了，如：
  
  ```swift
  extension Sequence<String> { }
  
  // Equivalent to:
  
  extension Sequence where Element == String { }
  ```

针对上一小节提到的错误：

![](/img/animals-error.png)

利用『 Primary Associated Types 』可以做如下改造：

```swift
// 将关联类型 FeedType 设定为 Primary Associated Types
//
protocol Animal<FeedType> {
  associatedtype FeedType

  func feed(_ feed: FeedType)
}

struct FeedTypeIMP {}

struct Farm {
  // 指定关联类型为 FeedTypeIMP
  //
  func feedAnimal(_ animal: any Animal<FeedTypeIMP>) {
    animal.feed(FeedTypeIMP())
  }
}
```

可以看到，指定关联类型约束时，可以是协议，如：

```swift
var animals: any Sequence<Animal>  // Animal 是协议
```

也可以是具体类型，如：

```swift
var animal: any Animal<FeedTypeIMP>  // FeedTypeIMP 是 struct
```

总之，通过『 Primary Associated Types 』进一步解放了生产力！



# Structural opaque result types

____

在 Swift 5.7  中 Opaque Result Types 样式更加丰富了：

```swift
func f0() -> [some P] { /* ... */ }           // ✅ on Swift 5.7

func f1() -> (some P, some Q) { /* ... */ }   // ✅ on Swift 5.7

func f2() -> () -> some P { /* ... */ }       // ✅ on Swift 5.7

func f3() -> S<some P> { /* ... */ }          // ✅ on Swift 5.7
```

在 Swift 5.7 之前，上面这些写法都是不允许的。



# 小结

+ Swift 5.6 引入 `any` 用于显式声明『 Existential type 』，避免无心滥用；

+ Swift 5.7 将 `some` 扩展到方法参数，从而兼得泛型的性能和『 Existential type 』的简洁；

+ Swift 5.7 解锁了所有协议均可作为类型；

+ 关联类型在作用上有不少障碍，Swift 5.7 引入了『 Primary Associated Types 』，使用方可以更方便地设定关联类型的信息；

+ Swift 5.7 对 Opaque Result Types 支持的样式进行了扩展。



经过 Swift 5.6/5.7 的扩展和优化，Swift 在 Protocol 的使用上效率更高了！



# 参考资料

[swift-evolution/0346-light-weight-same-type-syntax.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0346-light-weight-same-type-syntax.md)

[swift-evolution/0341-opaque-parameters.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0341-opaque-parameters.md)

[swift-evolution/0328-structural-opaque-result-types.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0328-structural-opaque-result-types.md)

[swift-evolution/0335-existential-any.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0335-existential-any.md)

[swift-evolution/0309-unlock-existential-types-for-all-protocols.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md)

[swift-evolution/0352-implicit-open-existentials.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0352-implicit-open-existentials.md)

[Embrace Swift generics - WWDC22 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2022/110352/)

[Design protocol interfaces in Swift - WWDC22 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2022/110353)

[Improving the UI of generics - Discussion - Swift Forums](https://forums.swift.org/t/improving-the-ui-of-generics/22814)

[What is the any keyword in Swift?](https://www.donnywals.com/what-is-the-any-keyword-in-swift/)

[What is the some keyword in Swift?](https://www.donnywals.com/what-is-the-some-keyword-in-swift/)

[What are primary associated types in Swift 5.7?](https://www.donnywals.com/what-are-primary-associated-types-in-swift-5-7/)

[What’s the difference between any and some in Swift 5.7?](https://www.donnywals.com/whats-the-difference-between-any-and-some-in-swift-5-7/)

[Using the ‘some’ and ‘any’ keywords to reference generic protocols in Swift 5.7 | Swift by Sundell](https://www.swiftbysundell.com/articles/referencing-generic-protocols-with-some-and-any-keywords/)

https://betterprogramming.pub/understanding-the-some-and-any-keywords-in-swift-5-7-19d2cb52eae2

https://swiftsenpai.com/swift/understanding-some-and-any/
