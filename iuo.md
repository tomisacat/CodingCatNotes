# 重新实现隐式解包可选值（IUO）

> Implicitly Unwrapped Optionals，简称 IUO

最主要的改变：调试的时候，一个声明为 T! 的值将会打印为 T? 而不是 T!

### IUO 是变量声明的一部分

许多人认为 IUO 是与一般的可选类型不同的类型。在 Swift 3 里，它确实是像人们所认为的那样工作的：声明为 `var a: Int?` 表示 a 是一个 `Optional<Int>` 变量，声明为 `var b: String!` 表示 b 是一个 `ImplicitlyUnwrappedOptional<String>` 变量。

而现在重新实现的 IUO 与上面不同，可以认为 `!` 是 `?` 的**同义词**，只是在声明的时候添加了一个标记，告诉编译器这个值可以被隐式解包。也就是说，你可以将 `String!` 当成 `Optional<String>` 类型，但同时它也带有额外信息表明在需要的时候它可以被隐式解包。

### 源码兼容性

在 SE-0054 里，`as T!` 这种用法已经不被允许了（因为 IUO 已经不是一种类型，而只是一个标记）。如果你使用 `x as T!` 这种代码，编译器首先会尝试检查它的类型是否可以看成 `x as T?`，如果检查失败，编译器会尝试将它看成 `(x as T?)!` 来强制解包。但是这种临时的处理方法可能在未来的版本中被移除。

上面这种用法属于一种更加常见问题的特例：将 `!` 作为类型的一部分。以下三种将 `!` 作为类型的一部分是允许的（更多参考 [SE-0054](https://github.com/apple/swift-evolution/blob/master/proposals/0054-abolish-iuo.md)）：

1. 属性和变量声明
2. 函数参数（可变参数除外）
3. 函数返回值
4. 初始化函数
5. 下标函数声明

看一下以前的一段代码：

```swift
class C {}
let values: [Any]! = [C()]
let transformed = values.map { $0 as! C }
```

它将对 `values` 强制解包并调用 `map(_:)` 方法，这个规则即使在你给 `ImplicitlyUnwrappedOptional` 自定义了一个 `map` 方法的情况下也是成立的，因为 IUO 的成员查询并不像我们所期待的那样工作。

在新的实现种，由于 `!` 变成了 `?` 的同义词，编译器会尝试调用 `Optional<T>` 的 `map(_:)` 方法：

```swift
let transformed = values.map { $0 as! C } // calls Optional.map; $0 has type [Any]
```

因此会产生这样的警告：`warning: cast from '[Any]' to unrelated type 'C' always fails`，你可以使用 optional chaining 的方式生成一个可选的数组来绕过：

```swift
let transformed = values?.map { $0 as! C } // transformed has type Optional<[C]>
```

或者直接强制解包 `values`：

```swift
let transformed = values!.map { $0 as! C } // transformed has type [C]
```

但多数情况下代码的行为并没有发生改编，例如：

```swift
let values: [Int]! = [1]
let transformed = values.map { $0 + 1 }
```

这段代码还像之前一样工作，因为它无法找到一种方式使得在 `Optional` 上类型检查成功，因此，我们直接强制对 `values` 强制解包并对生成的数组调用 `map(_:)` 方法。

> 上面这段不是特别懂，我能想到的解释就是：假设 `values.map` 是对 `Optional` 调用 `map(_:)` 方法，由于 `map(_:)` 的闭包参数是不可变的，因此 $0 相当于一个不可变的数组，那么 $0 + 1 显然是不合法的，因此对 `Optional` 的类型检查会失败，编译器于是进行第二种尝试：强制解包 `values`。

由于 IUO 不再是一个类型，因此它将不再能够被推导为一个类型或者一个类型的一部分。下面的例子中，尽管赋值语句的右边是一个声明为 IUO 的值，但左边的变量还是被推导为一个可选类型：

```swift
var x: Int!
let y = x   // y has type Int?

func forcedResult() -> Int! { ... }
let getValue = forcedResult    // getValue has type () -> Int?

func id<T>(_ value: T) -> T { return value }
let z = id(x)   // z has type Int?

func apply<T>(_ fn: () -> T) -> T { return fn() }
let w: Int = apply(forcedResult)    // fails, because apply() returns unforced Int?
```

假设一个类型有一个声明为 `UILabel!` 的 `property`：

```swift
func getLabel(object: AnyObject) -> UILabel {
  return object.property // forces both optionals, resulting in a UILabel
}
```

上面的代码里由于对 `Optional` 的类型检查会失败，因此编译器会强制解包可选值。

`if let` 和 `guard let` 只解包一层可选项，因此，如果对一个返回为 IUO 的尝试解包：

```swift
// label is inferred to be UILabel?
if let label = object.property { 
   // Error due to passing a UILabel? where a UILabel is expected
  functionTakingLabel(label)
}
```

可以通过明确类型来迫使编译器强制解包：

```swift
// Implicitly unwrap object.property due to explicit type.
if let label: UILabel = object.property {
  functionTakingLabel(label) // okay
}
```

类似地，`try?` 也会给调用结果增加一层可选层，因此用 `try?` 调用一个返回值类型为 IUO 的函数时，你需要修改代码来显示解包第二个层级的可选项：

```swift
func test() throws -> Int! { ... }

if let x = try? test() {
  let y: Int = x    // error: x is an Int?
}

if let x: Int = try? test() { // explicitly typed as Int
  let y: Int = x    // okay, x is an Int
}

if let x = try? test(), let y = x { // okay, x is Int?, y is Int
 ...
}
```

Swift 4.1 允许下面的代码成立：

```swift
func switchExample(input: String!) -> String {
  switch input {
  case "okay":
    return "fine"
  case let output:
    return output  // implicitly unwrap the optional, producing a String
  }
}
```

但是如果写成下面这样的话则会编译失败：

```swift
func switchExample(input: String!) -> String {
  let output = input  // output is inferred to be String?
  switch input {
  case "okay":
    return "fine"
  default:
    return output  // error: value of optional type 'String?' not unwrapped;
                   // did you mean to use '!' or '?'?
  }
}
```

现在，新的实现下第一个例子里的 `output` 会被推导为 `String?`，因此也会编译失败。

除了强制解包之外我们还可以用模式匹配来明确区分空和非空情况：

```swift
func switchExample(input: String!) -> String {
  switch input {
  case "okay":
    return "fine"
  case let output?: // non-nil case
    return output   // okay; output is a String
  case nil:
    return "<empty>"
  }
}
```

Swift 4.1 对于参数标记为 `inout` 的函数废弃了下面这种方式的重载：

```swift
func someKindOfOptional(_: inout Int?) { ... }

// Warning in Swift 4.1.  Error in new implementation.
func someKindOfOptional(_: inout Int!) { ... }
```

但 4.1 也同时允许参数标记为 `inout` 且为可选类型的函数接收 IUO 类型的参数（也就是参数标记为 inout T? 的函数可以接收 inout T!），反之也成立。因此对于上面的情况我们可以直接删掉第二个重载的函数：

```swift
func someKindOfOptional(_: inout Int?) { ... }

var i: Int! = 1
someKindOfOptional(&i)   // okay! i has type Optional<Int>
```

当桥接 `nil` 值的时候，为了避免运行时错误，`nil` 将被桥接为 `NSNull`：

```swift
import Foundation

class C: NSObject {}

let iuoElement: C! = nil
let array: [Any] = [iuoElement as Any]
let ns = array as NSArray
let element = ns[0] // Swift 4.1: Fatal error: Attempt to bridge
                    // an implicitly unwrapped optional containing nil

if let value = element as? NSNull, value == NSNull() {
  print("pass")     // We reach this statement with the new implementation
} else {
  print("fail")
}
```

