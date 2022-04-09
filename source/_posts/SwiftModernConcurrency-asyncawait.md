---
layout: post
title: Swift 新并发框架之 async/await
date: 2022-03-18 22:00:56
tags:
- iOS
- Swift
---

即使对于经验丰富的开发者来说，写出健壮性、可维护性高的并发代码也是一项具有挑战性的任务，其挑战主要体现在两个方面：

- 传统并发模型是基于异步模式，代码维护性不够友好；

- 并发往往意味着 Data Races，这是一类难复现、难排查的常见问题。

Swift 在 5.5 开始引入的新并发框架主要着力解决这 2 个问题。

本文是 『 Swift 新并发框架 』系列文章的第一篇，主要介绍 Swift 5.5 引入的 async/await。

<!--more-->

©原创文章，转载请注明出处！

本系列文章对 Swift 新并发框架中涉及的内容逐个进行介绍，内容如下：

- Swift 新并发框架之 async/await

- [Swift 新并发框架之 actor](https://zxfcumtcs.github.io/2022/03/19/SwiftModernConcurrency-actor/)

- [Swift 新并发框架之 Sendable](https://zxfcumtcs.github.io/2022/03/19/SwiftModernConcurrency-sendable/)

- [Swift 新并发框架之 Task](https://zxfcumtcs.github.io/2022/04/03/SwiftModernConcurrency-task/)

# Overview

---

在正式开始前，简单回顾一下同步/异步、串行/并行的概念：

- 同步(Synchronous)、异步(Asynchronous) 通常指方法(/函数)，同步方法表示直到任务完成才返回，异步方法则是将任务抛出去，在任务完成前就返回；
  
  这也就意味着需要通过某种方式获得异步任务的结果，如：Delegate、Closure 等。

- 串行(Serial)、并行(Concurrent) 通常指 App 执行一组任务的模式，串行表示一次只能执行一个任务，只有当前任务完成后才启动下一个任务，而并行指可以同时执行多个任务。最常见的莫过于 GCD 中的串行、并行队列；
  
   ps. 在此我们不严格区分并发、并行的区别。

- 传统的并发模型都是基于异步模式的，即异步获取并发任务的结果。

同步代码是线性的 (straight-line)，非常适合人脑处理。

而异步代码是非线性的、跳跃式的 (类似于 `goto` 语句)，对于单核的人脑来说是一大挑战。

除了在阅读上对人脑思维模式构成较大挑战外，异步代码在具体实现上常伴有以下问题：

- 回调地狱 (Callback Hell)；

- 错误处理 (Error Handling)；

- 容易出错。

# 初探

---

我们先通过一个简单的例子对比一下传统并发模型与新的并发模型间的区别。

该例子通过 token 获取头像，其步骤有：

- 通过 token 获取头像 URL；

- 通过 URL 下载头像数据(加密)；

- 对头像数据解密；

- 图片解码。

```swift
class AvatarLoader {
  func loadAvatar(token: String, completion: (Image) -> Void) {
    fetchAvatarURL(token: token) { url in
      fetchAvatar(url: url) { data in
        decryptAvatar(data: data) { data in
          decodeImage(data: data) { image in
            completion(image)
          }
        }
      }
    }
  }

  func fetchAvatarURL(token: String, completion: (String) -> Void) {
    // fetch url from net...
    completion(avatarURL)
  }

  func fetchAvatar(url: String, completion: (Data) -> Void) {
    // download avatar data...
    completion(avatarData)
  }

  func decryptAvatar(data: Data, completion: (Data) -> Void) {
    // decrypt...
    completion(decryptedData)
  }

  func decodeImage(data: Data, completion: (Image) -> Void) {
    // decode...
    completion(avatar)
  }
}
```

`loadAvatar` 方法中回调层级之深不言而喻。

上述代码还遗漏了一个重要问题：错误处理，其中的网络请求、解密、解码都有可能出错。

> 优雅地处理错误是一项非常考验基本功的任务。

一般地，错误处理分为 2 种情况：

- 同步方法：优先考虑通过 `throw` 抛出`error`，这样调用方就不得不处理错误，因此带有一定的强制性；

- 异步方法：在回调中传递 error，这种情况下调用方通常会有意无意地忽略错误，使健壮性大打折扣。

为了处理错误，对上述代码进行升级：

```swift
class AvatarLoader {
  func loadAvatar(token: String, completion: (Image?, Error?) -> Void) {
    fetchAvatarURL(token: token) { url, error  in
      guard let url = url else {
        // 在这个路径，经常容易漏掉执行 completion 或者 return 语句
        completion(nil, error)
        return
      }

      fetchAvatar(url: url) { data, error in
        guard let data = data else {
          completion(nil, error)
          return
        }

        decryptAvatar(data: data) { data, error in
          guard let data = data else {
            completion(nil, error)
            return
          }

          decodeImage(data: data) { image, error in
            completion(image, error)
          }
        }
      }
    }
  }

  func fetchAvatarURL(token: String, completion: (String?, Error?) -> Void) {
    // fetch url from net...
    completion(avatarURL, error)
  }

  func fetchAvatar(url: String, completion: (Data?, Error?) -> Void) {
    // download avatar data...
    completion(avatarData, error)
  }

  func decryptAvatar(data: Data, completion: (Data?, Error?) -> Void) {
    // decrypt...
    completion(decryptedData, error)
  }

  func decodeImage(data: Data, completion: (Image?, Error?) -> Void) {
    // decode...
    completion(avatar, error)
  }
}
```

可以看到，为了处理错误，在 `completion` 中增加了 `error` 参数，同时需要将 2 个参数都定义成 `Optional`。

同时，在 `loadAvatar` 中添加了大量的 `guard`，这样的代码无疑非常丑陋。

> Optional 无形中增加了代码成本。

为此，Swift 5 引入了 `Result` 用于优化上述错误处理场景：

```swift
class AvatarLoader {
  func loadAvatar(token: String, completion: (Result<Image, Error>) -> Void) {
    fetchAvatarURL(token: token) { result in
      switch result {
      case let .success(url):
        fetchAvatar(url: url) { result in
          switch result {
          case let .success(decryptData):
            decryptAvatar(data: decryptData) { result in
              switch result {
              case let .success(avaratData):
                decodeImage(data: avaratData) { result in
                  completion(result)
                }

              case let .failure(error):
                completion(.failure(error))
              }
            }
          case let .failure(error):
            completion(.failure(error))
          }
        }
      case let .failure(error):
        completion(.failure(error))
      }
    }
  }

  func fetchAvatarURL(token: String, completion: (Result<String, Error>) -> Void) {
    // fetch url from net...
    completion(.success(avatarURL))
  }

  func fetchAvatar(url: String, completion: (Result<Data, Error>) -> Void) {
    // download avatar data...
    completion(.success(avatarData))
  }

  func decryptAvatar(data: Data, completion: (Result<Data, Error>) -> Void) {
    // decrypt...
    completion(.success(decryptData))
  }

  func decodeImage(data: Data, completion: (Result<Image, Error>) -> Void) {
    // decode...
    completion(.success(avatar))
  }
}
```

> Result 是 enum 类型，含有 `success`、`failure` 2 个 case。

可以看到，通过使用 `Result`，参数不必是 `Optional`，另外可以通过 `switch/case` 来处理结果，在一定程度保证了调用方对错误的处理。

> 在上面这个 Callback Hell 中，直观上， Result 不但没有使代码简洁，反而更加复杂了。
> 
> 主要是没有把代码抽离开来，不要对 Result 有什么误解^__^。

通过这个简单的例子，可以看到基于 Callback 的异步模型问题不少。

因此，将异步代码同步化一直是业界努力的方向。

如：[Promise](https://www.promisejs.org/)，不过其同步也是建立在 callback 基础上的。

Swift 5.5 引入了 `async/await` 用于将异步代码同步化。

> 很多语言都已支持 `async/await`，如： JavaScript、Dart 等

先直观感受一下 `async/await`：

```swift
class AvatarLoader {
  func loadAvatar(token: String) async throws -> Image {
    let url = try await fetchAvatarURL(token: token)
    let encryptedData = try await fetchAvatar(url: url)
    let decryptedData = try await decryptAvatar(data: encryptedData)
    return try await decodeImage(data: decryptedData)
  }

  func fetchAvatarURL(token: String) async throws -> String {
    // fetch url from net...
    return avatarURL
  }

  func fetchAvatar(url: String) async throws -> Data {
    // download avatar data...
    return avatarData
  }

  func decryptAvatar(data: Data) async throws -> Data {
    // decrypt...
    return decryptData
  }

  func decodeImage(data: Data) async throws -> Image {
    // decode...
    return avatar
  }
}
```

相比基于 Callback 的异步版本，基于 `async/await` 的版本是不是清晰多了。

尤其是 `loadAvatar` 方法从感观上就是一个同步方法，阅读起来无比顺畅。

其错误处理也使用了同步式的 throws。

至此，通过对比，对 `async/await` 有了一个较直观的认识，下面简单探讨一下其实现机制。

# 深究

---

首先，还是有必要对 `async/await` 作一个正式的介绍：

- `async` — 用于修饰方法，被修饰的方法则被称为异步方法 (*asynchronous method*)，异步方法意味着其在执行过程中可能会被暂停 (挂起)；

- `await` — 对 *asynchronous method* 的调用需加上 `await`。同时，`await`只能出现在异步上下文中 (*asynchronous context*)；
  
  `await` 则表示一个潜在暂停点 (*potential suspension points*)。

什么是 *asynchronous context* ？其存在于 2 种环境下：

- asynchronous method body — 异步方法体属于异步上下文的范畴；

- Task closure — Task 任务闭包也属于 asynchronous context。
  
  > Task 是在 Swift 5.5 中引入的，主要用于创建、执行异步任务，后续文章会介绍。

因此，**只能在异步方法或 Task 闭包中通过 `await` 调用异步方法。**

异步方法执行过程中可能会暂停？

potential suspension points？

怎么暂停？

刚开始接触 `async/await` 时，下意识地可能会有这些疑问。

2 个关键点：

- 暂停的是方法，而不是执行方法的线程；

- 暂停点前后可能会发生线程切换。

在 Swift 新并发模型中进一步弱化了『 线程 』，理想情况下整个 App 的线程数应与内核数一致，线程的创建、管理完全交由并发框架负责。

Swift 对异步方法 (asynchronous method) 的处理就遵守了上述思想：

- 异步方法被暂停点 (suspension points) 分割为若干个 `Job`；

- 在并发框架中 `Job` 是任务调度的基本单元；

- 并发框架根据实时情况动态决定某个 `Job` 的执行线程；

- 也就是同一个异步方法中的不同 `Job` 可能运行在不同线程上。

正是由于异步方法在其暂停点前后可能会变换执行线程，因此在异步方法中要慎用锁、信号量等同步操作。

```swift
let lock = NSLock.init()
func test() async {
  lock.lock()
  try? await Task.sleep(nanoseconds: 1_000_000_000)
  lock.unlock()
}

for i in 0..<10 {
  Task {
    await test()
  }
}
```

像上面这样的代码在 `lock.lock()` 处会产生死锁，换成信号量也是一样。

> await 之所以称为『 潜在 』暂停点，而不是暂停点，是因为并不是所有的 await 都会暂停，只有遇到类似 IO、手动起子线程等情况时才会暂停当前调用栈的运行。

总之，对于异步方法如何切分 `Job` 等细节可以不关心，**但 `await` 可能会暂停当前方法的运行，并在时机成熟后在其他线程恢复运行是我们需要明确了解的**。

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
