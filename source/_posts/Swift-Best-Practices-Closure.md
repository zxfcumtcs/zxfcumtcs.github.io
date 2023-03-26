---
layout: post
title: Swift 最佳实践之 Closure
date: 2023-03-23 23:21:38
tags:
- iOS
- Swift
---

Swift 作为现代、高效、安全的编程语言，其背后有很多高级特性为之支撑。

『 Swift 最佳实践 』系列对常用的语言特性逐个进行介绍，助力写出更简洁、更优雅的 Swift 代码，快速实现从 OC 到 Swift 的转变。

该系列内容主要包括：

- [Optional](https://juejin.cn/post/7212101192746696760)

- [Enum](https://juejin.cn/post/7212143399036813349)

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

本文是系列文章的第三篇，介绍 Closure，内容主要包括如何利用 Inferring Type 简化闭包的使用、escaping-closure 与 nonescaping-closure 的区别、Capture List 注意事项、Trailing Closures 以及 Auto Closures 等。

# Overview

---

Swift Closure 与 Objective-C Block 有很多相似之处，都属于匿名函数/lambdas-expressions 的范畴。

相比之下，Swift Closure 更安全、更简洁。

首先，简要回顾一下闭包的基本语法，Closure 完整定义如下，几个关键组成：

- 参数列表

- 返回值类型

- 关键字 `in`

- closure body statements

```swift
{ (parameters) -> type in
   statements
}
```

声明 Closure 变量：

```swift
let closure: (parameters) -> type
```

看个简单的例子，如下，为数组排序方法 `sorted` 传入了用于排序操作的 Closure，其类型为：`(String, String) -> Bool`，即有 2 个 `String` 类型的参数，返回值为 `Bool`：

```swift
let names = ["Chris", "Alex", "Ewa", "Barry", "Daniella"]
let reversedNames = names.sorted(by: { (s1: String, s2: String) -> Bool in
    return s1 > s2
})
```

由于 Closure body 只有一行代码，故可以直接将其放在 `in` 后面：

```swift
names.sorted(by: { (s1: String, s2: String) -> Bool in return s1 > s2 })
```

# Inferring Type

---

得益于 Swift 强大的类型推演能力，上述排序 Closure 可以简化为：

```swift
reversedNames = names.sorted(by: { s1, s2 in return s1 > s2 } )
```

即不用显式写明参数、返回值的类型，编译器根据上下文完全可以推演出来

从 Swift 5.1 起，对于只有一个表达式 (Single-expression) 的方法/闭包，会隐式返回该表达式的值 [swift-evolution/0255-omit-return · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0255-omit-return.md)

简单讲，就是对于 Single-expression 的方法/闭包可以省略 `return` 关键字：

```swift
reversedNames = names.sorted(by: { s1, s2 in s1 > s2 } )
```

Swift 为 Closure 参数提供了一种简写方式，即分别用 `$0`、`$1` 来表示参数列表中的参数，此时可以忽略闭包参数列表，如下：

```swift
reversedNames = names.sorted(by: { $0 > $1 } )

// 如后文所述还有更简洁的版本
// reversedNames = names.sorted(by: >)
```

> 对于只有一个参数的闭包可以使用参数的简写形式 `$0`
> 
> 对于有 2 个及以上参数的情况慎用简写形式，可能会影响代码可读性

# Escaping Closures

---

escaping-closure、nonescaping-closure 是 Swift Closure 相较 OC Block 出现的一个新概念。

**escaping**、**nonescaping** 描述的是 Closure 作为**方法参数**时的分类：

- 当作为参数的 Closure，其**生命周期不会逃逸出所在方法**时，称为 nonescaping-closure，(意味着该闭包在方法调用链上会被执行)，如：
  
  ```swift
  func foo(_ closure: () -> Void) {
    // closure 没有逃逸出 foo
    closure()
  }
  ```
  
  如下，虽在方法 `bar` 中没有直接执行 `closure`，但在其调用链上的 `foo` 会执行 `closure`，`closure` 并没有逃逸出 `bar`：
  
  ```swift
  func bar(_ closure: () -> Void) {
    foo(closure)
  }
  ```

- 当 Closure 的生命周期逃逸出所在方法时，称为 escaping-closure。
  
  如下，`foo` 将参数 `escapingClosure` 存储在属性 `closure` 中，使其逃逸出 `foo`，即 `foo` 返回后 `escapingClosure` 还存在：
  
  ```swift
  public class EscapingClosureDemo {
    var closure: (() -> Void)?
  
    func foo(_ escapingClosure: @escaping () -> Void) {
      closure = escapingClosure
    }
  }
  ```
  
  此时，参数 `escapingClosure` 的定义须加上关键字 `@escaping`，显式表明定义的是 escaping-closure，否则编译报错：
  
  ![](/img/escaping-nonescaping-error.png)
  
  > OC 中相当于所有 block 默认都是 escaping
  
  > 在 Swift 3 以前， 闭包类型的参数默认是 escaping，并提供了 `@noescape` 关键字用于声明 nonescaping Closure
  > 
  > 但从 Swift 3 开始，闭包参数默认是 nonescaping，对于 escaping closure 须显式声明 `@escaping`，并废弃了 `@noescape`
  > 
  > [swift-evolution/0103-make-noescape-default · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0103-make-noescape-default.md)

**Why❓**

为什么要区分 escaping、nonescaping，并在 Swift 3 中将默认值从 escaping 改成 nonescaping？

主要原因有两个：

- **编译器优化** 👍，对于 escaping closure 需要更复杂的内存管理，而 nonescaping closure 编译器可以做优化

- **显式提醒开发人员** ⚡️：「你正在危险的边缘试探——正在定义/调用的是 escaping closure！」

escaping-closure 为何就危险了❓

**原因在于 escaping-closure 可能会产生循环引用 (Strong Reference Cycles)，而 nonescaping closure 一定是不会有循环引用的。**

因此，在 escaping-closure 中不允许隐式捕获 `self`，以免在「不经意间 😴」引起循环引用：

![](/img/explicitly-reference-self.png)

# Capturing Values

---

我们从一个简单的问题开始：[Can You Answer This Simple Swift Question Correctly?](https://swiftsenpai.com/swift/can-you-answer-this-simple-swift-question-correctly)

![](/img/closure-quiz.png)

> 在 1334 位回答者中只有 44% 回答正确 🙈
> 
> 正确答案：
> 
> - 1 -- Objc
> 
> - 2 -- Swift

1 和 2 的区别在于：1 用了捕获列表 (Capture List)，而 2 没有。

Capture List：

- 用 `[]` 声明的表达式列表，表达式间用 `,` 分隔，放在参数列表前 (如有)：
  
  ```swift
  func bar() {
    var age = 10
    var name = "Jim"
  
    let closure = { [age, name] in  // 等价于 [age = age, name = name]
      // 此处的 age、name 与闭包外的已没任何关系，仅名字相同而以
      print("\(name) is \(age) years old!")
    }
  
    age = 11
    name = "Tom"
    closure()    // Jim is 10 years old!
  }
  ```

- capture list 在闭包定义时完成初始化赋值 (`[age = age, name = name]`)
  
  所以上面输出的是 `Jim is 10 years old!`，而不是 `Tom is 11 years old!`
  
  > 如果不用 capture list，而是直接引用，则直到 closure 执行时才去取值：
  > 
  > ![](/img/no-capture-list.png)
  
  but，如果捕获的是引用类型 (reference types)，那情况就不一样了：
  
  > String 在 Swift 中是值类型，而非引用类型
  
  ```swift
  class Foo {
    var age: Int
    var name: String
  
    init(age: Int, name: String) {
      self.age = age
      self.name = name
    }
  }
  
  func bar() {
    var foo = Foo(age: 10, name: "Jim")
    let closure = { [foo] in    // 捕获引用类型 (foo)
      print("\(foo.name) is \(foo.age) years old!")
    }
  
    foo.age = 11
    foo.name = "Tom"
  
    closure()    // Tom is 11 years old!
  }
  ```
  
  虽然，捕获引用类型时，其输出有所不同，但「***capture list 在闭包定义时完成初始化赋值***」的特性并没有变，只不过此时赋值的是「指针」
  
  如下，若给 `foo` 赋一个新值，则与 closure 内捕获的就没任何关系了：
  
  ```swift
  func bar() {
    var foo = Foo(age: 10, name: "Jim")
    let closure = { [foo] in
      print("\(foo.name) is \(foo.age) years old!")
    }
  
    foo = Foo(age: 11, name: "Tom")  // 此时，foo 指向新实例
  
    closure()    // Jim is 10 years old!
  }
  ```
  
  ![](/img/capture-list.png)

- 通过 capture list 还可以指定捕获类型：`strong` (默认)、`weak` 以及 `unowned`，用于避免循环引用：
  
  ```swift
  { print(self.name) }                    // implicit strong capture
  { [self] in print(self.name) }          // explicit strong capture
  { [weak self] in print(self?.name) }    // weak capture
  { [unowned self] in print(self.name) }  // unowned capture
  
  // 还可以在 capture list 用表达式赋值
  { [name = self.name] in print(name)}    // strong capture name
  ```

总之，capture list 在闭包定义时完成初始化赋值 (而非执行时)：

- 对于值类型 (value types)，capture value 初始化实质上就是 copy，从此以后闭包内外的值就没任何关系了 (只是名字相同而以)

- 对于引用类型 (reference types)，capture value 初始化 copy 的实质上是个指针，闭包内外引用的还是同一个实例

# Trailing Closures

---

如果方法的最后一个参数是 closure，那么在调用时可以简化：

```swift
func foo() {
  // 正常调用
  bar(doSomething: {
    print("normal call!")
  })

  // trailing closure call
  // 省略()、label
  bar {
    print("trailing closure call!")
  }
}

func bar(doSomething: () -> Void) {
  doSomething()
}
```

对于有多个 closure 参数的情况，建议只对最后一个参数用 trailing closure call，以免影响可读性：

```swift
func foo() {
  // ❎ 不建议
  bar {
    print("1")
  } doSomething1: {
    print("2")
  }

  // ✅
  bar(doSomething: {
    print("1")
  }) {
    print("2")
  }
}

func bar(doSomething: () -> Void, doSomething1: () -> Void) {
  doSomething()
  doSomething1()
}
```

# Auto closures

---

如下，给 closure 加上 `@autoclosure` 后，在调用时可以直接用表达式，传入的表达式会自动封装成 closure，而无需显式的写成闭包的形式：

```swift
func foo() {
  bar(doSomething: print("a"))    // ❌ bar(doSomething: { print("a") })
  baz(value: 1)                   // ❌ baz(value: { 1 })
}

func bar(doSomething: @autoclosure () -> Void) {
  print("b")
  doSomething()
}

func baz(value: @autoclosure () -> Int) {
  print(value())
}

// 输出：
// b
// a
// 1
```

- 对 closure 加 `@autoclosure` 的前提是其没有参数

- 表达式是 lazy，只有到闭包执行时才执行表达式，而非方法调用时

正是由于 `@autoclosure` 的 lazy 特性，其常被用于那些期望懒加载的场景，如：Swift 标准库提供的 `assert` 函数：

```swift
public func assert(
  _ condition: @autoclosure () -> Bool,
  _ message: @autoclosure () -> String = String(),
  file: StaticString = #file, line: UInt = #line
) {
  // Only assert in debug mode.
  if _isDebugAssertConfiguration() {
    if !_fastPath(condition()) {
      _assertionFailure("Assertion failed", message(), file: file, line: line, flags: _fatalErrorFlags())
    }
  }
}
```

由于 `condition`、`message` 是 auto closure，故可以简化 `assert` 调用：

`assert(someCondition(), "Failed!")`

如果它们是常规闭包，则调用时有点麻烦：

`assert({ someCondition() }, { "Failed!" })`

类似的，如 [Alamofire](https://github.com/Alamofire/Alamofire) 中对 Error 的扩展：

![](/img/alamofire-error-autoclosure.png)

在项目中经常会对 `Dictionary` 的取值做些保护并提供默认值，如：

```swift
extension Dictionary {
  func value<T>(forKey key: Key, defaultValue: @autoclosure () -> T) -> T {
    self[key].flatMap { $0 as? T } ?? defaultValue()
  }
}
```

# First-class functions first, closures second

---

这一小节有点挖墙脚的意思：「**优先考虑一等函数，其次才是闭包**」，前提是闭包参数与函数参数匹配，如：

```swift
let names = ["Chris", "Alex", "Ewa", "Barry", "Daniella"]

// closure
let reversedNames = names.sorted(by: { $0 > $1 } )

// first-class function
let reversedNames = names.sorted(by: >)
```

```swift
let domain: String? = "docs.swift.org"

// if let
var url: URL?
if let domain {
  url = URL(string: domain)
}

// closure
let url = domain.flatMap { URL(string: $0) }

// first-class function
let url = domain.flatMap(URL.init)  // 等价于 domain.flatMap(URL.init(string:))
```

```swift
let subviews: [UIView] = [...]

// closure
subviews.forEach { addSubview($0) }

// first-class functions
subviews.forEach(addSubview)
```

```swift
let ages = [1, 2, 3]

// closure
ages.forEach { baz(value: $0) }

// first-class functions
ages.forEach(baz(value:))

func baz(value: Int) {
  print(value)
}
```

# 小结

本文对 Closure 的主要特性以及最佳实践进行了简要分析介绍。

如，利用 Inferring Type 可以简化闭包的使用，Capture List 对于值类型和引用类型的区别，Trailing Closures 以及 Auto closures 的使用等。

最后，提到某些场景下利用函数一等公民身份相比闭包可以简化代码。



# 参考资料

[Apple-Documentation-closures](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/closures/)

[Swift clip: First class functions](https://www.swiftbysundell.com/clips/1/)

[Swift’s closure capturing mechanics](https://www.swiftbysundell.com/articles/swifts-closure-capturing-mechanics/)

[Using @autoclosure when designing Swift APIs](https://www.swiftbysundell.com/articles/using-autoclosure-when-designing-swift-apis/)

[swift-evolution/0255-omit-return · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0255-omit-return.md)

[swift-evolution/0103-make-noescape-default · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0103-make-noescape-default.md)

[swift/Assert.swift at main · apple/swift · GitHub](https://github.com/apple/swift/blob/main/stdlib/public/core/Assert.swift)
