---
layout: post
title: Swift 新并发框架之 Task
date: 2022-04-03 22:16:43
tags:
- iOS
- Swift
---

本文是 『 Swift 新并发框架 』系列文章的第四篇，主要介绍基于 Task 的结构化并发 (Structured concurrency) 和 非结构化并发任务 (Unstructured tasks)。

<!--more-->

©原创文章，转载请注明出处！

本系列文章对 Swift 新并发框架中涉及的内容逐个进行介绍，内容如下：

- [Swift 新并发框架之 async/await](https://zxfcumtcs.github.io/2022/03/18/SwiftModernConcurrency-asyncawait/)

- [Swift 新并发框架之 actor](https://zxfcumtcs.github.io/2022/03/19/SwiftModernConcurrency-actor/)

- [Swift 新并发框架之 Sendable](https://zxfcumtcs.github.io/2022/03/19/SwiftModernConcurrency-sendable/)

- Swift 新并发框架之 Task

# Overview

____

前三篇文章分别介绍了用于将异步代码同步化的 [async/await](https://zxfcumtcs.github.io/2022/03/18/SwiftModernConcurrency-asyncawait/)、并发安全模型 [actor](https://zxfcumtcs.github.io/2022/03/19/SwiftModernConcurrency-actor/) 以及用于约束在并发环境下可以安全传值的 [Sendable](https://zxfcumtcs.github.io/2022/03/19/SwiftModernConcurrency-sendable/)。

严格意义上说，它们并不具备提供「并发」的能力，而是为并发提供若干基础辅助功能。

本文主角 Task 则能提供「并发执行」的能力。

# Task

____

 几个关键点：

+ 并发环境中执行任务的基本单元 (「代码块」)；

+ 所有的异步函数 (async) 都运行在 Task 内；

+ Task 属于线程之上的更高级抽象，由系统负责在合适的线程上调度执行 Task。

Task 有 3 种状态：

+ **暂停 (suspended)** — 有 2 种情况会导致 Task 处于暂停状态：
  
  + Task 已准备就绪等待系统分配执行线程；
  
  + 等待外部事件，如 Task 遇到 suspension point 后可能会进入暂停状态并等待外部事件来唤醒。
  
  > ps. 需要注意的是，异步函数 (`A`) 调用另一个异步函数 (`B`)时，调用方会暂停，并不意味着整个 Task 会暂停。
  > 
  > 从函数 `A` 的视角看，其会暂停等待函数 `B` 返回；
  > 
  > 但从 Task 视角看，其不一定会暂停，可能会继续在其上执行被调用的函数 `B`；
  > 
  > 当然，Task 也可能会被暂停，如果被调用的函数要在不同的并发上下文中执行。

+ **运行中 (running)** — Task 当前正在某个线程上运行，直至完成，或遇到 suspension point 而进入暂停状态；

+ **已完成 (completed)** — Task 所有工作都已完成。

总之，**Task 是线程的高级抽象，用于执行一项任务。** 

Task 提供了一些高级抽象能力：

+ Task 可以携带调度信息，如：任务优先级；

+ Task 作为正在执行的任务的句柄 (Handle)，可以用于 cancel 等；

+ Task 可以携带用户提供的 task-local data。

# Structured concurrency

____

Structured concurrency，结构化并发，听起来挺玄乎。

说白了，就是在 Task 间可以有父子关系，并形成一颗「Task tree」：

![](/img/Tasktree.png)

通过 Task 间的父子关系可以更好地对一组 Task 进行管理：

+ 子 Task 的生命周期不会超出父 Task 的范围 (这点非常重要)；

+ cancel 更便捷 (cancel 某个 Task 时，其所有子 Task 也会被 cancel)；

+ 错误处理更方便了，未处理的 error 会自动从子 Task 传播到父 Task；

+ 子 Task 默认会继承父 Task 的优先级；

+ 父子 Task 间会共享 Task-local data；

+ 父 Task 可以很容易收集子 Task 的结果。

以上就是结构化并发的全部！

下面，就其中的细节逐一展开讨论。

目前，实现结构化并发有 2 种方式：

+ `async let`；

+ Task group。

## `async let`

```swift
// given: 
//   func chopVegetables() async throws -> [Vegetables]
//   func marinateMeat() async -> Meat
//   func preheatOven(temperature: Int) async -> Oven
//
func makeDinner() async throws -> Meal {
  async let veggies = chopVegetables()
  async let meat = marinateMeat()
  async let oven = preheatOven(temperature: 350)

  let dish = Dish(ingredients: await [try veggies, meat])
  return try await oven.cook(dish, duration: .hours(3))
}
```

先通过一个例子感受一下，几个关键点：

+ 对异步函数的调用不用 `await`，而是在赋值表达式的最左边加上 `async let` (第 `7~8` 行)，称之为 `async let binding`；

+ 在需要使用 `async let` 表达式的结果时要用 `await`，如结果可能会抛出错误，还需要处理错误 (第 `11~12` 行)；

+ `async let` 只能出现在异步上下文中 (Task closure、async function 以及 async closure)。

> 上述例子来自：[swift-evolution/0317-async-let.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0317-async-let.md)

以上是我们的直观感受，其背后的实现机制是：

+ **系统为每个 `async let` 创建一个并发的子任务；**

+ 子任务创建后立马开始执行；

+ 子任务会继续父任务的优先级以及 task-local datas。

因此，如上例，会创建 3 个并发子任务分别执行 `chopVegetables`、`marinateMeat` 以及 `preheatOven`。

### Implicit `async let` awaiting

有个问题：正常流程下，对 `async let` 需要执行 `await` 操作，如果不执行 `await` 会怎样呢？

会导致子任务溢出吗？(超出父任务的生命周期？)

答案是否定的。

```swift
func makeDinner() async throws -> Meal {
  async let veggies = chopVegetables()
  async let meat = marinateMeat()
  async let oven = preheatOven(temperature: 350)
}
```

如上代码，系统会添加隐式 cancel、await：

```swift
func makeDinner() async throws -> Meal {
  async let veggies = chopVegetables()
  async let meat = marinateMeat()
  async let oven = preheatOven(temperature: 350)
  // implicitly: cancel veggies
  // implicitly: cancel meat
  // implicitly: cancel oven
  // implicitly: await veggies
  // implicitly: await meat
  // implicitly: await oven
}
```

我们通过一个简单的例子验证一下上述结论：

```swift
func noAwaitAsynclet() async {
  print("begin noAwaitAsynclet")
  try? await Task.sleep(nanoseconds: 1_000_000_000)
  Task.isCancelled ? print("noAwaitAsynclet is cancelled") : print("end noAwaitAsynclet")
}
  
func testAsynclet() async {
  let parentTask =
  Task {
    async let test = noAwaitAsynclet()
  }
    
  await parentTask.value
  print("parentTask finished!")
}
```

调用 `testAsynclet` 方法的输出：

```
begin noAwaitAsynclet
noAwaitAsynclet is cancelled
parentTask finished!
```

### cancel

正如前文所述，**在结构化并发中 cancel 操作会从父任务传递给所有子任务**。

```swift
func noAwaitAsynclet() async {
  print("begin noAwaitAsynclet")
  try? await Task.sleep(nanoseconds: 1_000_000_000)
  Task.isCancelled ? print("noAwaitAsynclet is cancelled") : print("end noAwaitAsynclet")
}
  
func testAsynclet() async {
  let parentTask =
  Task {
    async let test = noAwaitAsynclet()
    await test
  }
    
  parentTask.cancel()
  await parentTask.value
  print("parentTask finished!")
}
```

对前面那个例子简单改动一下：

+ 第 `11` 行添加对 `test` 的 `await`；

+ 第 `14` 行对 `parentTask` 执行 `cancel`。

其输出：

```
begin noAwaitAsynclet
noAwaitAsynclet is cancelled
parentTask finished!
```

可以看到，对父任务的 `cancel` 操作传递到了 `async let` 子任务。

## Task group

用 Task group 重写 `makeDinner` 来直观感受一下 Task group：

```swift
func makeDinner() async throws -> Meal {
  // Prepare some variables to receive results from our concurrent child tasks
  var veggies: [Vegetable]?
  var meat: Meat?
  var oven: Oven?

  enum CookingStep { 
    case veggies([Vegetable])
    case meat(Meat)
    case oven(Oven)
  }

  // Create a task group to scope the lifetime of our three child tasks
  try await withThrowingTaskGroup(of: CookingStep.self) { group in
    group.addTask {
      try await .veggies(chopVegetables())
    }
    group.addTask {
      await .meat(marinateMeat())
    }
    group.addTask {
      try await .oven(preheatOven(temperature: 350))
    }

    for try await finishedStep in group {
      switch finishedStep {
        case .veggies(let v): veggies = v
        case .meat(let m): meat = m
        case .oven(let o): oven = o
      }
    }
  }

  let dish = Dish(ingredients: [veggies!, meat!])
  return try await oven!.cook(dish, duration: .hours(3))
}
```

几个关键点：

+ Task group 没有公开的 `init` 方法，只能通过 `withTaskGroup` 或 `withThrowingTaskGroup` 方法来获得 Task group 实例；

+ 通过 Task group 的 `addTask` 方法可以创建并发执行的子任务，且子任务的数量可以是动态的；

+ 同一 group 中所有子任务的结果类型必须相同；
  
  > 上例是通过 enum (`CookingStep`)封装关联值的方式使得所有子任务结果类型相同的。

+ 子任务的生命周期不会超出 group 生命周期；
  
  > 因此当 group(`withTaskGroup`、`withThrowingTaskGroup`) 方法返回时就意味着所有子任务都已完成或 cancel；

+ 通过 `for await ... in` 可以遍历所有子任务的运行结果；
  
  > 需要注意的是遍历的顺序是子任务完成的顺序，而非子任务添加的顺序；

+ 当 group 内部抛出错误时 (如某个子任务抛出异常)，所有未完成的子任务都将被 cancel。

如下，如果在 group 内不显式地等待所有子任务完成，会如何？

```swift
try await withThrowingTaskGroup(of: CookingStep.self) { group in
  group.addTask {
    try await .veggies(chopVegetables())
  }
  group.addTask {
    await .meat(marinateMeat())
  }
  group.addTask {
    try await .oven(preheatOven(temperature: 350))
  }
}
```

**group 还是会隐式的等待所有子任务完成才返回**。

> 注意此处与 `async let` 的区别，如上文所述，`async let` 子任务会先被 cancel，再 await。

## `async let` vs. Task group

`async let` 与 Task group 同属结构化并发范畴，在日常开发中如何选择？

基本原则：**能用 `async let` 就不用 Task group。**

由两个版本的 `makeDinner` 方法可以看出：

+ `async let` 更轻量、更直观；

+ Task group 要求所有子任务的计算结果类型相同，往往需要多一层封装，如 `makeDinner` 中的 `CookingStep`枚举。同时，Task group 接口是基于 closure 的，也进一步导致代码变复杂。

那有什么是 Task group 可以做，而 `async let` 无法做到的？

主要有 2 点：

+ **`async let` 创建子任务的数量是静态的，而 Task group 可以动态创建子任务；**
  
  如下，`loadImages` 方法为每个 url 创建一个下载图片的子任务，其数量由参数 `urls` 动态决定：
  
  ```swift
  func loadImages(urls: [String]) async -> [Image] {
    await withTaskGroup(of: Image.self, body: { group in
      for url in urls {
        group.addTask {
          return await downloadImage(url: url)
        }
      }
  
      var images: [Image] = []
      for await image in group {
        images.append(image)
      }
  
      return images
    })
  }
  ```

+ **`async let` 等待子任务完成的顺序是固定，无法做到按子任务完成顺序取结果。**
  
  如下，无论 3 个子任务哪个先完成，我们一定是先获得 `veggiesValue`，再获得 `meatValue`，最后获取 `ovenValue`。
  
  ```swift
  func makeDinner() async throws -> Meal {
    async let veggies = chopVegetables()
    async let meat = marinateMeat()
    async let oven = preheatOven(temperature: 350)
    let veggiesValue = await veggies
    let meatValue = await meat
    let ovenValue = await oven
  }
  ```
  
  **而 Task group 是以子任务完成的顺序拿到结果的。**
  
  这有什么用吗？
  
  ```swift
  func fastestResponse() async -> Int {
    await withTaskGroup(of: Int.self, body: { group in
      group.addTask {
        let _ = await requestFromServer1()
        return 1
      }
  
      group.addTask {
        let _ = await requestFromServer2()
        return 2
      }
  
      return await group.next()!
    })
  }
  ```
  
  如上，有两台布署了相同服务的服务器，需要确定当前哪台服务器响应速度更快。
  
  通过 Task group 按子任务完成顺序返回的特性很容易就能实现。



## 小结

通过上文讨论，我们知道结构化并发有很多优势。

其中，最重要的一条是：**子任务的生命周期不会超出父任务。**

其使得我们可以很容易做到：

+ 控制一组任务，如 cancel，只要对父任务执行 cancel，其中的所有子任务都会被 cancel；
  
  > 如果子任务的生命周期比父任务长，就很难做到这一点。因为在需要执行 cancel 时，父任务可能已经结束了。

+ 等待一组任务完成，只要等待父任务完成即可，因为父任务完成就意味着所有子任务都已完成；

+ 配合 `async/await` 可以很容易地实现多组任务间的依赖。

要在传统并发模型中实现以上需求往往需大费周章。

# Unstructured tasks

---

非结构化任务，简单讲，就是任务间没有父子关系，不存在 「 Task tree 」。

通过上文我们知道，**结构化并发最重要的特性就是子任务的生命周期不会超出父任务。**

而非结构化任务就不存在这个约束。

有时只需要创建一个并发任务，或在同步上下文中为了调用异步方法而创建异步环境。

以上是非结构化任务的 2 个主要应用场景。

创建非结构化任务有 2 种方式：

+ `Task.init`

+ `Task.detached`

## `Task.init`

```swift
@frozen public struct Task<Success, Failure> : Sendable where Success : Sendable, Failure : Error {}

extension Task where Failure == Error {
  public init(priority: TaskPriority? = nil, operation: @escaping @Sendable () async throws -> Success)
}
```

```swift
let dinnerHandle = Task {
  try await makeDinner()
}

await dinnerHandle.value
dinnerHandle.cancel()
```

如上，`Task.init` 返回一个 task 句柄 (`dinnerHandle`)，通过该句柄可以获取任务执行的结果，也可以取消任务。

### Context inheritance

通过 `Task.init` 创建的任务会从当前上下文中继承重要的元信息，如：

+ 任务优先级；

+ task-local data；

+ actor isolation。

如果 `Task.init` 是在异步上下文中调用的 (意味着调用链上存在 Task)：

+ 新创建的任务会继承当前任务的优先级；

+ 通过拷贝的方式继承当前任务的所有 task-local data；

+ 如果是在 actor 方法中调用 `Task.init` 的，则 Task closure 将成为 actor-isolated。
  
  从上面 `Task.init` 定义可以知道，Task closure 是用 `Sendable` 修饰的。
  
  在「[Swift 新并发框架之 Sendable](https://zxfcumtcs.github.io/2022/03/19/SwiftModernConcurrency-sendable/)」中介绍过，`Sendable closure` 是不能捕获 actor-isolated 属性，否则报错: Actor-isolated property 'x' can not be referenced from a Sendable closure。
  
  但 Task closure 是个例外，因为它本身也是 actor-isolated，所以下面的代码不会报错：
  
  ```swift
  public actor TestActor {
    var value: Int = 0
      
    func testTask() {
      Task {
        value = 1
      }
   }
  }
  ```

如果 `Task.init` 是在同步上下文中调用的 (调用链上没有 Task)：

+ 运行时推断合理的优先级；

## `Task.detached`

```swift
extension Task where Failure == Never {
  public static func detached(priority: TaskPriority? = nil, operation: @escaping @Sendable () async -> Success) -> Task<Success, Failure>
}
```

```swift
let dinnerHandle = Task.detached {
  try await makeDinner()
}
```

通过 `Task.detached` 创建的任务完全独立于当前上下文，也就是不会继承当前上下文的优先级、task-local data 以及 actor isolation。

# 小结

---

至此，基于 Task 创建任务的四种形态全部介绍完了。

在 [Explore structured concurrency in Swift - WWDC21](https://developer.apple.com/videos/play/wwdc2021/10134) 中对它们有一个总结：

![](/img/Flavorsoftasks.png)



结构化并发可以说是一次重大进步，今后编码并发相关的代码会更加容易！



# 参考资料

[swift-evolution/0296-async-await.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md)

[swift-evolution/0317-async-let.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0317-async-let.md)

[swift-evolution/0302-concurrent-value-and-concurrent-closures.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)

[swift-evolution/0337-support-incremental-migration-to-concurrency-checking.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0337-support-incremental-migration-to-concurrency-checking.md)

[swift-evolution/0304-structured-concurrency.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)

[swift-evolution/0306-actors.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)

[swift-evolution/0337-support-incremental-migration-to-concurrency-checking.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0337-support-incremental-migration-to-concurrency-checking.md)

[Understanding async/await in Swift • Andy Ibanez](https://www.andyibanez.com/posts/understanding-async-await-in-swift/)

[Concurrency — The Swift Programming Language (Swift 5.6)](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html)

[Connecting async/await to other Swift code | Swift by Sundell](https://www.swiftbysundell.com/articles/connecting-async-await-with-other-swift-code/)

[Explore structured concurrency in Swift - WWDC21 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2021/10134/)

[Swift concurrency: Behind the scenes - WWDC21 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2021/10254)
