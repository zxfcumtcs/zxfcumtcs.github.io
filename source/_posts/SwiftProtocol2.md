---
layout: post
title: Swift Protocol 背后的故事(理论)
date: 2022-02-04 22:16:24
tags:

- iOS
- Swift
- Protocol
---

本系列文章将从实践技巧、实现原理以及追踪语言更新等方面对 Swift Protocol 展开深入讨论。主要内容有：

[Swift Protocol 背后的故事(实践)](https://zxfcumtcs.github.io/2022/02/01/SwiftProtocol1/)
Swift Protocol 背后的故事(理论)
[Swift Protocol 背后的故事(Swift 5.6/5.7)](https://zxfcumtcs.github.io/2022/06/30/SwiftProtocol3/)
...

本文是系列文章第二篇，主要讨论 Swift Protocol 实现机制。

内容涉及 Type Metadata、Protocol 内存模型 Existential Container、Generics 的实现原理以及泛型特化等。

<!--more-->

©原创文章，转载请注明出处！

# Type Metadata

____

在 Objective-C 中，通过 MetaClass 模型来表达类的元信息，并通过实例的 `isa` 指针来引用 MetaClass。这是整个 Objective-C runtime 的核心机制。

那在 Swift 中类型信息 (Type Metadata) 是如何表达的呢？

Swift runtime 为每种类型 (Class、Struct、Enum、Protocol、Tuple、Function 等等) 生成了一份元信息记录 (Metadata Record)，几个关键点：

- 对于常规类型 (nominal types，如：类、结构体、枚举)，其对应的 Metadata Record 在编译期由编译器静态生成；

- 对于内在类型 (intrinsic types) 以及泛型实例，对应的 Metadata Record 则在运行时动态生成，如：元组、函数、协议等；

- 每个类型的 Metadata Record 都是唯一的，相同类型的 Metadata Record 是同一个

不同类型的 Metadata 所包含的信息也不一样 (Metadata Layout)，但它们有一个共同的头部，其包含：

- VWT (Value Witness Table) Poniter — 指向 VWT (虚函数表) 的指针，在 VWT 中包含了该类型的实例如何分配内存 (allocating)、copy (copying)、销毁 (destroying) 等基础操作 (函数指针)；
  
  > 在 VWT 中除了包含上述函数指针外，还有该类型实例的 size、alignment、stride 等基础信息。

- Kind — 标记 Metadata 的类型，如：0 -- Class，1 -- Struct，2 -- Enum，12 -- Protocol 等等。

![](/img/MetadataType.png)

与本文讨论相关的是 VWT，关于 Metadata 的更多信息请参考：[swift/TypeMetadata.rst at main · apple/swift · GitHub](https://github.com/apple/swift/blob/main/docs/ABI/TypeMetadata.rst#protocol-metadata)。

# Inside Protocol

---

## Existential Container

Class、Struct 以及 Enum 对应的实例都有确定的『模型』用于指导其内存布局。

> 『模型』就是 Class、Struct 以及 Enum 本身的定义，它们包含的成员。

而 Protocol 并没有确定的『模型』，因为其背后的真实类型可能千奇百怪，那么 Protocol 类型的变量按什么进行内存布局？

Swift 用了一种称之为 **Existential Container** 的模型来指导 Protocol 变量布局内存。

Existential Container 又分为两类：

- *Opaque Existential Container* — 用于没有**类**约束的 Protocol (no class constraint on protocol)，也就是说这种协议背后的真实类型可能是类、结构体以及枚举等。因此其存储就非常复杂。
  
  ```swift
  struct OpaqueExistentialContainer {
    void *fixedSizeBuffer[3];
    Metadata *type;
    WitnessTable *witnessTables[NUM_WITNESS_TABLES];
  };
  ```
  如上，`OpaqueExistentialContainer` 包含3个成员：
  
  - `fixedSizeBuffer` — 3 个指针大小的 buffer 空间，当真实类型的 size (内存对齐后的大小) 小于 3 个字时则其内容直接存储在 fixedSizeBuffer 中，否则在 heap 上另辟空间存储，并将指针存储在 fixedSizeBuffer 中；
  
  - `type` — 指向真实类型的 Metadata，最重要的就是引用其中的 VWT 用于完成内存的各种操作；
  
  - `witnessTables` — 指向协议函数表 (Protocol Witness Table, PWT)，协议函数表中存储的是真实类型中对应函数的地址。

- *Class Existential Container* — 用于有**类**约束的 Protocol，该协议背后真实的类型只能是类，而类的实例都是在 Heap 上分配内存的。
  
  因此，在 Existential Container 中只需要一个指向堆内存的指针即可。
  
  同时，由于类的实例含有指向其 Metadata 的指针，故在 Existential Container 中也就没必要再存一份 Metadata 的指针了：
  
  ```swift
  struct ClassExistentialContainer {
    HeapObject *value;
    WitnessTable *witnessTables[NUM_WITNESS_TABLES];
  };
  ```
  如上，`ClassExistentialContainer` 只包含2个成员：
  
  - `value` — 指向堆内存的指针；
  
  - `witnessTables` — PWT 指针。

下面我们来看一个例子：

![](/img/ExistentialContainer.png)

如图，由于  `protocol Drawable` 没有 class constraint，故其对应的 Existential Container 是 `OpaqueExistentialContainer`：

- 由于 `Point` 实例占用 2 个字的内存空间 (小于 3)，故对于 `Drawable` 协议类型的变量 `point` 直接使用 `OpaqueExistentialContainer#buffer` 来存储其内容(`x`、`y`)；

- 而 `Line` 的实例要占用 4 个字的内存空间，故 `line` 需要在 heap 上分配内存用于存储其内容(`x0`、`y0`、`x1`、`y1`)；

- Existential Container 中的 `type` 分别指向了其背后真实类型的 Metadata；

- PWT 中的函数指针则指向真实类型中的函数。

对应编译器生成的(伪)代码如下：

```swift
let point: Drawable = Point(x: 0, y: 0)
point.draw()

// 编译器生成的(伪)代码
let _point: Point = Point(x: 0, y: 0)
var point: OpaqueExistentialContainer = OpaqueExistentialContainer()
let metadata = _point.type
let vwt = metadata.vwt
vwt.copy(&(point.fixedSizeBuffer), _point)
point.type = metadata
point.witnessTables = PWT(Point.draw)
point.pwt.draw()
vwt.dealloc(point)
```

```swift
let line: Drawable = Line(x0: 0, y0: 0, x1: 1, y1: 1)
line.draw()

// 编译器生成的(伪)代码
let _line: Line = Line(x0: 0, y0: 0, x1: 1, y1: 1)
var line：OpaqueExistentialContainer = OpaqueExistentialContainer()
let metadata = _line.type
let vwt = metadata.vwt
line.fixedSizeBuffer[0] = vwt.allocate()
vwt.copy(line.fixedSizeBuffer, _line)
line.type = metadata
line.witnessTables = PWT(Line.draw)
line.pwt.draw()
vwt.dealloc(line)
```

关于更多 Type Layout 的信息请参考: [swift/TypeLayout.rst at main · apple/swift · GitHub](https://github.com/apple/swift/blob/main/docs/ABI/TypeLayout.rst)

从上面的伪代码可以看到对于协议类型的变量，编译器在背后做了大量的工作。也有一定的性能损耗。

## Protocol Type Stored Properties

从上一小节可知，协议类型的变量其实质类型是 Existential Container (`OpaqueExistentialContainer` / `ClassExistentialContainer`)。

因此，当协议类型变量作为存储属性时，其在寄主实例中的内存占用就是一个 Existential Container 实例 (下图来自: [Understanding Swift Performance · WWDC2016](https://developer.apple.com/videos/play/wwdc2016/416/))：

![](/img/ProtocolTypeStoredProperties.png)

## 小结

Protocol 不同于一般类型 (Class、Struct、Enum)，具有以下特点：

- 使用 Existential Container (`OpaqueExistentialContainer` / `ClassExistentialContainer`) 作为内存模型；

- 内存占用 <= 3 的实例，直接存储在 Existential Container buffer 中，否则在 heap 上另行分配内存，并将指针存储在 Existential Container buffer[0] 中；

- 内存管理 (allocating、copying、destroying) 相关的方法保存在 VWT 中；

- 通过 PWT 实现方法动态派发 (Dynamic dispatch)。

# Generics

____

泛型作为提升代码灵活性、可复用性的重要手段被大多数语言所支持，如：Swift、Java、C++ (模板)。

Swift 结合 Protocol 赋以泛型更多的灵活性。

下面我们简单探讨一下泛型在 Swift 中是如何实现的。

Swift 中泛型可以添加类型约束 (Type Constraints)，约束可以是类，也可以是协议。

因此，Swift 泛型根据类型约束可以分为 3 类：

- No Constraints

- Class Constraints

- Protocol Constraints

下面，分别对这 3 类情况进行简要分析。

> 通过 SIL ([swift/SIL.rst at main · apple/swift · GitHub](https://github.com/apple/swift/blob/main/docs/SIL.rst)) 可以大致了解 Swift 背后的实现原理。
> 
> `swiftc demo.swift -O -emit-sil -o demo-sil.s` 
> 
> 如上，通过 swiftc 命令可以生成 SIL。
> 
> 其中的 `-O` 是对生成的 SIL 代码进行编译优化，使 SIL 更简洁高效。
> 
> 后面要讲到的泛型特化 (Specialization of Generics) 也只有在 `-O` 优化下会发生。

## No Constraints

其实，这类泛型能执行的操作非常少。无法在 heap 上实例化 (创建) 对象，也不能执行任何方法。

```swift
@inline(never)  // 禁止编译器做 inline 优化
func swapTwoValues<T>(_ a: inout T, _ b: inout T) {
    let temp = a
    a = b
    b = temp
}
```

如上，`swapTwoValues` 用于交换 2 个变量的值，其泛型参数 `T` 没有任何约束。

其对应的 SIL 如下，关键点：

+ 第`8`行，通过 `alloc_stack` 在栈上为类型 `T` 分配内存 (`temp`);

+ 第`9`行，通过 `copy_addr` 进行内存级拷贝 (不会执行任何 `init` 方法)；

+ 第`12`行，通过 `dealloc_stack` 销毁上述开辟的内存；

+ 其他就是做一些内存拷贝的操作。

> 需要注意的是，对于引用类型，`$T`是指针，即在栈上开辟的是存储指针的内存，而非引用类型本身。

```swift
1   // swapTwoValues<A>(_:_:)
2   sil hidden [noinline] @$s4main13swapTwoValuesyyxz_xztlF : $@convention(thin) <T> (@inout T, @inout T) -> () {
3   // %0 "a"                                         // users: %5, %6, %2
4   // %1 "b"                                         // users: %7, %6, %3
5   bb0(%0 : $*T, %1 : $*T):
6    debug_value_addr %0 : $*T, var, name "a", argno 1 // id: %2
7    debug_value_addr %1 : $*T, var, name "b", argno 2 // id: %3
8    %4 = alloc_stack $T, let, name "temp"           // users: %8, %7, %5
9    copy_addr %0 to [initialization] %4 : $*T       // id: %5
10   copy_addr [take] %1 to %0 : $*T                 // id: %6
11   copy_addr [take] %4 to [initialization] %1 : $*T // id: %7
12   dealloc_stack %4 : $*T                          // id: %8
13   %9 = tuple ()                                   // user: %10
14   return %9 : $()                                 // id: %10
15 } // end sil function '$s4main13swapTwoValuesyyxz_xztlF'
```

## Class Constraints

```swift
class Shape {
  required
  init() {}

  func draw() -> Bool {
    return true
  }
}

class Triangle: Shape {
  required
  init() {}

  override
  func draw() -> Bool {
    return true
  }
}

@inline(never)
func drawShape<T: Shape>(_ s: T) {
  let s0 = T()
  s0.draw()
}
```

如例：

- 将泛型类型 upcast 到约束类型 (第 `5~6`行)；

- 通过 `class_method` 指令找到要执行的方法 (`init`、`draw` 方法 [第 `7~10` 行]，在 vtable 中找)；

- 可以在 Heap 上创建泛型类型的实例 (第 `8` 行)；

- 其实质就是通过虚函数表实现的多态。

总之，由于有基础类作为类型约束，通过虚函数表就可以执行所有基础类公开的方法。

```swift
1  // drawShape<A>(_:)
2  sil hidden [noinline] @$s4main9drawShapeyyxAA0C0CRbzlF : $@convention(thin) <T where T : Shape> (@guaranteed T) -> () {
3  // %0 "s"
4  bb0(%0 : $T):
5    %1 = metatype $@thick T.Type                    // user: %2
6    %2 = upcast %1 : $@thick T.Type to $@thick Shape.Type // users: %4, %3
7    %3 = class_method %2 : $@thick Shape.Type, #Shape.init!allocator : (Shape.Type) -> () -> Shape, $@convention(method) (@thick Shape.Type) -> @owned Shape // user: %4
8    %4 = apply %3(%2) : $@convention(method) (@thick Shape.Type) -> @owned Shape // users: %7, %5, %6
9    %5 = class_method %4 : $Shape, #Shape.draw : (Shape) -> () -> Bool, $@convention(method) (@guaranteed Shape) -> Bool // user: %6
10   %6 = apply %5(%4) : $@convention(method) (@guaranteed Shape) -> Bool
11   strong_release %4 : $Shape                      // id: %7
12   %8 = tuple ()                                   // user: %9
13   return %8 : $()                                 // id: %9
14 } // end sil function '$s4main9drawShapeyyxAA0C0CRbzlF'
15
16 sil_vtable Shape {
17   #Shape.init!allocator: (Shape.Type) -> () -> Shape : @$s4main5ShapeCACycfC    // Shape.__allocating_init()
18   #Shape.draw: (Shape) -> () -> Bool : @$s4main5ShapeC4drawSbyF    // Shape.draw()
19   #Shape.deinit!deallocator: @$s4main5ShapeCfD    // Shape.__deallocating_deinit
20 }
21
22 sil_vtable Triangle {
23   #Shape.init!allocator: (Shape.Type) -> () -> Shape : @$s4main8TriangleCACycfC [override]    // Triangle.__allocating_init()
24   #Shape.draw: (Shape) -> () -> Bool : @$s4main8TriangleC4drawSbyF [override]    // Triangle.draw()
25   #Triangle.deinit!deallocator: @$s4main8TriangleCfD    // Triangle.__deallocating_deinit
26 }
```

## Protocol Constraints

> 这里讨论的 Protocol 是没有 class constraint 的，对于只能由类实现的协议作为泛型约束时，其效果同上节讨论的 Class Constraints。

```swift
@inline(never)
func equal<T: Equatable>(_ a: T, _ b: T) -> Bool {
  let a0 = a
  let b0 = b
  return a0 == b
}
```

从下列 SIL 可以看到：

- 通过 `alloc_stack` 可以为泛型类型在 Stack 上分配内存 (无论泛型类型是值类型还是引用类型)；

- 同样通过 `copy_addr` 执行内存拷贝；

- 通过 `witness_method` 指令在泛型类型上查找 Protocol 指定的方法 (查 PWT 表)。

```swift
// equal<A>(_:_:)
sil hidden [noinline] @$s4main5equalySbx_xtSQRzlF : $@convention(thin) <T where T : Equatable> (@in_guaranteed T, @in_guaranteed T) -> Bool {
// %0 "a"                                         // users: %5, %2
// %1 "b"                                         // users: %8, %3
bb0(%0 : $*T, %1 : $*T):
  debug_value_addr %0 : $*T, let, name "a", argno 1 // id: %2
  debug_value_addr %1 : $*T, let, name "b", argno 2 // id: %3
  %4 = alloc_stack $T, let, name "a0"             // users: %9, %10, %8, %5
  copy_addr %0 to [initialization] %4 : $*T       // id: %5
  %6 = metatype $@thick T.Type                    // user: %8
  %7 = witness_method $T, #Equatable."==" : <Self where Self : Equatable> (Self.Type) -> (Self, Self) -> Bool : $@convention(witness_method: Equatable) <τ_0_0 where τ_0_0 : Equatable> (@in_guaranteed τ_0_0, @in_guaranteed τ_0_0, @thick τ_0_0.Type) -> Bool // user: %8
  %8 = apply %7<T>(%4, %1, %6) : $@convention(witness_method: Equatable) <τ_0_0 where τ_0_0 : Equatable> (@in_guaranteed τ_0_0, @in_guaranteed τ_0_0, @thick τ_0_0.Type) -> Bool // user: %11
  destroy_addr %4 : $*T                           // id: %9
  dealloc_stack %4 : $*T                          // id: %10
  return %8 : $Bool                               // id: %11
} // end sil function '$s4main5equalySbx_xtSQRzlF'
```

再看一个例子：

```swift
protocol Drawable {
  init()
  func draw() -> Bool
}

@inline(never)
func drawShape<T: Drawable>(_ s: T) -> T {
  var s0 = T()
  s0.draw()
  return s0
}
```

从下列 SIL 可以看出：

- 可以在 Heap 上创建泛型类型实例 (无论是值类型还是引用类型)；

```swift
// drawShape<A>(_:)
sil hidden [noinline] @$s4main9drawShapeyxxAA8DrawableRzlF : $@convention(thin) <T where T : Drawable> (@in_guaranteed T) -> @out T {
// %0 "$return_value"                             // users: %4, %6
// %1 "s"
bb0(%0 : $*T, %1 : $*T):
  %2 = metatype $@thick T.Type                    // user: %4
  %3 = witness_method $T, #Drawable.init!allocator : <Self where Self : Drawable> (Self.Type) -> () -> Self : $@convention(witness_method: Drawable) <τ_0_0 where τ_0_0 : Drawable> (@thick τ_0_0.Type) -> @out τ_0_0 // user: %4
  %4 = apply %3<T>(%0, %2) : $@convention(witness_method: Drawable) <τ_0_0 where τ_0_0 : Drawable> (@thick τ_0_0.Type) -> @out τ_0_0
  %5 = witness_method $T, #Drawable.draw : <Self where Self : Drawable> (Self) -> () -> Bool : $@convention(witness_method: Drawable) <τ_0_0 where τ_0_0 : Drawable> (@in_guaranteed τ_0_0) -> Bool // user: %6
  %6 = apply %5<T>(%0) : $@convention(witness_method: Drawable) <τ_0_0 where τ_0_0 : Drawable> (@in_guaranteed τ_0_0) -> Bool
  %7 = tuple ()                                   // user: %8
  return %7 : $()                                 // id: %8
} // end sil function '$s4main9drawShapeyxxAA8DrawableRzlF'


sil_witness_table hidden Shape: Drawable module main {
  method #Drawable.init!allocator: <Self where Self : Drawable> (Self.Type) -> () -> Self : @$s4main5ShapeCAA8DrawableA2aDPxycfCTW    // protocol witness for Drawable.init() in conformance Shape
  method #Drawable.draw: <Self where Self : Drawable> (Self) -> () -> Bool : @$s4main5ShapeCAA8DrawableA2aDP4drawSbyFTW    // protocol witness for Drawable.draw() in conformance Shape
}
```

通过上面简单的分析可以看出 No Constraints、Class Constraints 以及 Protocol Constraints 的泛型类型在实现上的区别：

- No Constraints 泛型能做的事很少，不能执行任何方法，只能 Stack 上为泛型类型分配内存并执行内存拷贝；

- Class Constraints 泛型可以在 Heap 上创建新实例，方法调用通过虚函数表 (vtable) 实现；

- Protocol Constraints 泛型可以在 Stack 上也可以在 Heap 上按需创建泛型类型实例，无论泛型是值类型还是引用类型；

- Protocol Constraints 泛型通过 PWT 实现方法调用；

- 也就是说泛型中的方法调用都是动态派发 (Dynamic dispatch)，通过 vtable 或者 PWT。

## Specialization of Generics

从上一小节可知，泛型方法调用都是动态派发 (通过 vtable 或 PWT)，有一定的性能损耗。

为了优化此类损耗，Swift 编译器会对泛型进行特化 (Specialization of Generics)。

所谓特化就是为具体类型生成相应版本的函数，从而将泛型转成非泛型，实现方法调用的静态派发。

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

如例，通过 `Int` 型参数调用 `swapTwoValues` 时，编译器就会生成该方法的 `Int` 版本：

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

另外在编译时若开启了 Whole-Module Optimization ，同一模块内部的泛型调用也可以被特化。

> 关于全模块优化请参考[Swift.org - Whole-Module Optimization in Swift 3](https://www.swift.org/blog/whole-module-optimizations/)，在此不再赘述。

# 小结

- Swift 为每种类型生成了一份 Metadata Record，其中包含了 VWT (Value Witness Table)；

- Protocol 使用 Existential Container 作为其内存模型，所有 Protocol 类型的变量都是 Existential Container 的实例；

- Protocol 通过 PWT 实现方法动态派发 (Dynamic dispatch)；

- 泛型调用在满足一定条件时会进行特化，以提升性能。

# 参考资料

[swift-evolution · Opaque Result Types](https://github.com/apple/swift-evolution/blob/master/proposals/0244-opaque-result-types.md)

[OpaqueTypes](https://docs.swift.org/swift-book/LanguageGuide/OpaqueTypes.html)

[Different flavors of type erasure in Swift](https://www.swiftbysundell.com/articles/different-flavors-of-type-erasure-in-swift/#closures-to-the-rescue)

[Opaque Return Types and Type Erasure](https://www.raywenderlich.com/24942207-opaque-return-types-and-type-erasure)

[Phantom types in Swift](https://www.swiftbysundell.com/articles/phantom-types-in-swift/)

[How to use phantom types in Swift](https://www.hackingwithswift.com/plus/advanced-swift/how-to-use-phantom-types-in-swift)

[swift/TypeMetadata.rst at main · apple/swift · GitHub](https://github.com/apple/swift/blob/main/docs/ABI/TypeMetadata.rst#protocol-metadata)

[swift/TypeLayout.rst at main · apple/swift · GitHub](https://github.com/apple/swift/blob/main/docs/ABI/TypeLayout.rst)

[Swift Type Metadata](https://speakerdeck.com/kateinoigakukun/swift-type-metadata?slide=48)

[Understanding Swift Performance · WWDC2016](https://developer.apple.com/videos/play/wwdc2016/416/)

[Swift.org - Whole-Module Optimization in Swift 3](https://www.swift.org/blog/whole-module-optimizations/)
