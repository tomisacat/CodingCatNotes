### 与 Objective-C APIs 交互

##### 动态派发（dynamic dispatch）

当 Objective-C 运行时被导入到 Swift 中时，系统并不保证使用动态派发的方式访问 属性(property)/方法/下标操作/初始化方法，Swift 可能使用虚表或内联的方式来优化性能。

一般情况下你并不一定要使用动态派发，但是如果和 `KVO` 或 `method_exchangeImplementations` 之类的方法打交道，那么你就得用动态派发的方式了，这时候你可以使用 `dynamic` 修饰符来显示定义，并且通常还需要和 `@objc` 一起使用，除非声明的上下文(declaration context)已经隐式的标明了 `@objc` 修饰。

###### Selectors

Swift 用 `Selector` 结构体表示 Objective-C 里的 `selector`，并使用 `#selector` 表达式创建：

* 对于方法，使用方法名：`#selector(MyViewController.tappedButton(sender:))`
* 对于属性(property)，需要添加 `getter` 或 `setter` 前缀：`#selector(getter: MyViewController.myButton)`

并且，对于方法来说，要使用 `@objc` 修饰：

```swift
import UIKit
class MyViewController: UIViewController {
    let myButton = UIButton(frame: CGRect(x: 0, y: 0, width: 100, height: 50))
    
    override init(nibName nibNameOrNil: NSNib.Name?, bundle nibBundleOrNil: Bundle?) {
        super.init(nibName: nibNameOrNil, bundle: nibBundleOrNil)
        let action = #selector(MyViewController.tappedButton)
        myButton.addTarget(self, action: action, forControlEvents: .touchUpInside)
    }
    
    @objc func tappedButton(sender: UIButton?) {
        print("tapped button")
    }
    
    required init?(coder: NSCoder) {
        super.init(coder: coder)
    }
}
```

注意：由于 `#selector` 表达式没有类型信息，因此如果一个方法有类似的重载方法，则需要使用 `as` 操作符：`#selector(((UIView.insert(subview:at:)) as (UIView) -> (UIView, Int) -> Void))`。

##### KVC/KVO

现在可以使用 `\` 表达式，称为 `key-path expression`。比如：

```swift
class Animal: NSObject {
    @objc var name: String
    
    init(name: String) {
        self.name = name
    }
}
 
let llama = Animal(name: "Llama")
let nameAccessor = \Animal.name
let nameCountAccessor = \Animal.name.count
 
llama[keyPath: nameAccessor]
// "Llama"
llama[keyPath: nameCountAccessor]
// "5"
```

它与以前的 `#keyPath`，也就是 `key-path string expression` 的不同之处在于：`#keyPath` 不传递类型信息。

### 与 C APIs 交互

##### 原始类型（简单类型）

对于原始类型来说（比如 `int`），Swift 并不支持与 C 的隐式转换，对于 C 中的每一种原始类型，Swift 都提供了想对应的类型，如下表所示：

| C Type | Swift Type |
| :-: | :-: |
| bool | CBool |
| char, signed char | CChar |
| unsigned char | CUnsignedChar |
| int | CInt |
| wchar_t | CWideChar |
| char16_t | CChar16 |

不过，如果没有明确的需求，最好统一使用 `Int`。

##### 全局常量

C/Objective-C 源码里定义的全局常量在导入到 Swift 项目时会自动转换为遵循 `RawRepresentable` 协议的结构体类型。

* 使用 `NS_STRING_ENUM` 表示无法再添加自定义的这个类型的常量
* 使用 `NS_EXTENSIBLE_STRING_ENUM` 表示可以再添加自定义的这个类型的变量

##### 函数

C 定义的函数，若参数为指针类型，则导入到 Swift 时一般会被转换为 `UnsafeMutablePointer`，如果原定义使用 `const` 属性则为 `UnsafePointer`。其他参数类型转换为 Swift 中对应的类型。

对于可变参数函数来说，可以使用 `getVaList(_:)` 或 `withVaList(_:_:)` 转换为 `CVaListPointer` 类型。如：

```swift
func swiftprintf(format: String, arguments: CVarArg...) -> String? {
    return withVaList(arguments) { va_list in
        var buffer: UnsafeMutablePointer<Int8>? = nil
        return format.withCString { cString in
            guard vasprintf(&buffer, cString, va_list) != 0 else {
                return nil
            }
            
            return String(validatingUTF8: buffer!)
        }
    }
}
print(swiftprintf(format: "√2 ≅ %g", arguments: sqrt(2.0))!)
// Prints "√2 ≅ 1.41421"
```

##### 联合体

Swift 不支持联合体，当定义在 C 源文件中的联合体类型被导入到 Swift 中时，它将被自动转换为结构体，同时生成多个初始化方法，每个初始化方法仅有一个参数，对应原来联合体里的属性。

##### 位域

C 结构体类型里的属性在定义的时候可以设置位域，当导入到 Swift 中时，这些属性就会被转换为计算属性，访问的时候 Swift 会自动将其转换为与 Swift 兼容的类型。如：

```swift
var a: Decimal = Decimal()
a._isNegative = 1
```

##### 匿名结构体/联合体

如果一个 C 结构体定义如下：

```c
struct Cake {
    union {
        int layers;
        double height;
    };
 
    struct {
        bool icing;
        bool sprinkles;
    } toppings;
};
```

当导入到 Swift 会自动生成按位初始化方法：

```swift
let cake = Cake(
    .init(layers: 2),
    toppings: .init(icing: true, sprinkles: false)
)
 
print("The cake has \(cake.layers) layers.")
// Prints "The cake has 2 layers."
print("Does it have sprinkles?", cake.toppings.sprinkles ? "Yes." : "No.")
// Prints "Does it have sprinkles? No."
```

##### 指针

1. 对于 返回值/变量/参数：
    
    
| C Syntax | Swift Syntax |
| :-: | :-: |
| const Type * | UnsafePointer<Type> |
| Type * | UnsafeMutablePointer<Type> |

2. 对于 class 类型：

| C Syntax | Swift Syntax |
| :-: | :-: |
| Type * const * | UnsafePointer<Type> |
| Type * __strong * | UnsafeMutablePointer<Type> |
| Type ** | AutoreleasingUnsafeMutablePointer<Type> |

3. 对于 无类型/原始类型：

| C Syntax | Swift Syntax |
| :-: | :-: |
| const void * | UnsafeRawPointer |
| void * | UnsafeMutableRawPointer |

###### 不可变指针

如果一个函数声明接收一个 `UnsafePointer<Type>` 或 `UnsafeRawPointer` 类型的指针，那么调用的时候它可以接收如下指针：

* `UnsafePointer<Type>`, `UnsafeMutablePointer<Type>`, or `AutoreleasingUnsafeMutablePointer<Type>`。它们会自动转换为 `UnsafePointer<Type>`
* `String`。如果 `Type` 是 `Int8` 或 `UInt8`
* ` Type 类型的 in-out 表达式`。也就是 `&v`，其中 `v` 是 `Type` 类型 
* `[Type]`。这个数组的第一个元素将被作为指针传递给函数

比如：

```swift
func takesAPointer(_ p: UnsafePointer<Float>) {
    // ...
}

var x: Float = 0.0
takesAPointer(&x)  // in-out expression
takesAPointer([1.0, 2.0, 3.0])  // [Type]
```

###### 可变指针

如果一个函数声明接收一个 `UnsafeMutablePointer<Type>` 或 `UnsafeMutableRawPointer` 类型的指针，那么调用的时候它可以接收如下指针：

* `UnsafeMutablePointer<Type>`
* `Type 类型的 in-out 表达式`。也就是 `&v`，其中 `v` 是 `Type` 类型
* `[Type] 类型的 in-out 表达式`。也就是 `&[Type]`

比如：

```swift
func takesAMutablePointer(_ p: UnsafeMutablePointer<Float>) {
    // ...
}

var x: Float = 0.0
var a: [Float] = [1.0, 2.0, 3.0]
takesAMutablePointer(&x)  // in-out
takesAMutablePointer(&a)  // &[Type]
```

###### Autoreleasing 指针

如果一个函数声明接收一个 `AutoreleasingUnsafeMutablePointer<Type>` 类型的指针，那么调用的时候它可以接收如下指针：

* `AutoreleasingUnsafeMutablePointer<Type>`
* `Type 类型的 in-out 表达式`

比如：

```swift

    // ...
}

var x: NSDate? = nil
takesAnAutoreleasingPointer(&x)
```

注意：这种类型的指针不会自动与 Objective-C 类型指针进行桥接。比如 `NSString **` 转换到 Swift 中是 `AutoreleasingUnsafeMutablePointer<NSString?>` 而不是 `AutoreleasingUnsafeMutablePointer<String?>`。

###### 函数指针

C 源文件中的函数指针导入到 Swift 时会被转换成 `@convention(c)` 的闭包。比如 `int (*)(void)` 转换到 Swift 时变为：`@convention(c) () -> Int32`。

举个具体的例子：

```swift
func customCopyDescription(_ p: UnsafeRawPointer?) -> Unmanaged<CFString>? {
    // return an Unmanaged<CFString>? value
}
 
var callbacks = CFArrayCallBacks(
    version: 0,
    retain: nil,
    release: nil,
    copyDescription: customCopyDescription,
    equal: { (p1, p2) -> DarwinBoolean in
        // return Bool value
}
)
 
var mutableArray = CFArrayCreateMutable(nil, 0, &callbacks)
```

注意：在 Swift 中，只有标记为 `@convention(c)` 的方法才能用作一个函数(或方法)的函数指针参数。

###### Buffer 指针

Buffer 指针 指向内存的一块区域。Swift 有如下 Buffer 指针类型：

* `UnsafeBufferPointer`
* `UnsafeMutableBufferPointer`
* `UnsafeRawBufferPointer`
* `UnsafeMutableRawBufferPointer`

其中，`UnsafeBufferPointer` 和 `UnsafeMutableBufferPointer` 称为有类型的 Buffer 指针，它允许你以集合的方式访问一块连续的内存空间，这个集合里的每一个元素都是 `Item` 类型。`UnsafeRawBufferPointer` 和 `UnsafeMutableRawBufferPointer` 与之类似，只不过这个集合的元素类型不是泛型的 `Item` 类型，而是 `UInt8`。

###### 空指针

| Objective-C Syntax | Swift Syntax |
| :-: | :-: |
| const Type * _Nonnull | UnsafePointer<Type> |
| const Type * _Nullable | UnsafePointer<Type>? |
| const Type * _Null_unspecified | UnsafePointer<Type>! |

###### 指针运算

```swift
let pointer: UnsafePointer<Int8>
let offsetPointer = pointer + 24
// offsetPointer is 24 strides ahead of pointer
```

注意：上面的运算是步进了 24 个步长的指针，而不是 24 个字节。

##### 一次初始化

Swift 原生保证全局变量（常量）和 类型计算属性仅初始化一次（即使多线程访问），因此 `pthread_once()`/`dispatch_once()`/`dispatch_once_f()` 函数并没有导入到 Swift 中。

##### 预处理指令

Swift 编译器没有预处理器，通过编译时属性、条件编译块以及语言原生功能可以实现同样的功能。

###### 宏

简单宏，比如一些原始类型变量的定义在导入 Swift 后会转换为全局变量。而复杂宏，比如功能与函数类似的宏，则没有导入 Swift。

###### 条件编译

与 Objective-C 不一样，Swift 使用条件编译块。比如：

```swift
#if DEBUG_LOGGING
print("Flag enabled.")
#endif
```

此外还有：

| Function | Valid arguments |
| :-: | :-: |
| os() | macOS, iOS, watchOS, tvOS, Linux |
| arch() | x86_64, arm, arm64, i386 |
| swift() | >= followed by a version number |

如果有多个条件，可以使用 `&&` 和 `||` 合并，或者使用 `!` 。




