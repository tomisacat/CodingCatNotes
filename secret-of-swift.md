### Unamanged

这个结构体类似于中间人，用于在 `C Opaque Pointer`（在 Swift 里为 `Unsafe|Mutable|RawPointer`） 和 `AnyObject` 对象之间转换:

```
AnyObject <- Unmanaged -> Unsafe|Mutable|RawPointer
```

具体来说有以下使用方式：

1. AnyObject -> Unmanaged -> UnsafeMutableRawPointer
    
    ```swift
    let cfStr = "This is a string" as CFString
    let bits = Unmanaged.passUnretained(cfStr) // CF Object -> Unmanaged
    let ptr = bits.toOpaque() // Unmanaged -> UnsafeMutableRawPointer
    ```
    
    其中，`passUnretained` 方法将一个 `AnyObject` 对象转换为一个 `Unmanaged` 结构体，并且不增加它的引用计数。类似的，也有 `passRetained` 方法实现类似的功能，但会增加这个对象的引用计数。
    
2. UnsafeRawPointer -> Unamaged -> AnyObject

    ```swift
    // 假设你有一个 UnsafeRawPointer 对象 ptr
    let unmanagedStr = Unmanaged.fromOpaque(ptr) // UnsafeRawPointer -> Unmanaged
    let str: CFString = unmanagedStr.takeUnretainedValue() // Unmanaged -> CF Object
    ```
    
    其中，`fromOpaque` 方法将一个 `UnsafeRawPointer` 对象转换为 `Unmanaged` 结构体，`takeUnretainedValue` 将这个结构体转换为一个 `AnyObject`，并且不增加这个对象的引用计数。类似的，`takeRetainedValue` 会增加这个对象的引用计数。
    
容易看出来上面的转换方法可以自由选择是否增加转换对象的引用计数，Swift 还提供了下面几个方法来增加或减少转换过程中这个中间对象的引用计数：

```swift
public func retain() -> Unmanaged<Instance>
public func release()
public func autorelease() -> Unmanaged<Instance>
```

注意到 `Unmanaged` 仅适用于 `AnyObject` 对象，也就是 `class` 类型的对象，那么对于 `String` 之类的 `struct` 类型的对象来说，可以使用下面的方法：

```swift
public func withUnsafeMutablePointer<T, Result>(to arg: inout T, _ body: (UnsafeMutablePointer<T>) throws -> Result) rethrows -> Result

public func withUnsafePointer<T, Result>(to arg: inout T, _ body: (UnsafePointer<T>) throws -> Result) rethrows -> Result
```

比如：

```swift
var header: String = "\0x00\0x00\0x00\0x01"
withUnsafePointer(to: &header) { (bytePointer) in
     // handle bytePointer
}
```

### unsafeBitCast

除了 `Unmanaged` 之外还有一个方法也经常被用来做对象之间的强制转换：

```swift
public func unsafeBitCast<T, U>(_ x: T, to type: U.Type) -> U
```

这个方法的用途是在其他转换方法都不合适的情况下在两个内存布局兼容的类型之间做转换，那么在迫不得已使用这个方法之前我们有哪些可以使用的方法？Swift 标准库支持以下这些转换：

    1. 整数类型互转：使用目标类型的初始化方法或 `numericCast(_:)` 函数
    2. 如果 1 中前后两个整数类型长度不同，可以使用 `init(truncatingIfNeeded:)` 或 `init(bitPattern:)` 按位转换
    3. 指针转整数：使用目标类型的 `init(bitPattern:)` 方法，指针内存地址的位模式作为参数值（这句存疑，不是很理解注释的说法，也许是我翻译错误）
    4. 对于引用类型，使用 `as`/`as!`/`as?` 或 `unsafeDowncast(_:to:)` 函数。不要使用 `unsafeBitCast(_:to:)` 来转换 class 类型和指针类型，这种行为是未定义的。

关于第 4 点，很多人往往会忽略后面那句话，比如 `StackOverflow` 上[这个问题](https://stackoverflow.com/questions/40780419/how-to-use-cfdictionarysetvalue-in-swift)，作者使用 `unsafeBitCast(_:to:)` 方法来转换，通过答案可以知道稍加修改它的代码是可以顺利运行的，但是，一种更好的方法就是使用上面的 `Unmanaged` 结构体来做这种转换。

```swift
let rawDic: UnsafeRawPointer = CFArrayGetValueAtIndex(attachments, 0)
let dic: CFDictionary = Unmanaged.fromOpaque(rawDic).takeUnretainedValue()
CFDictionarySetValue(dic, Unmanaged.passUnretained(kCMSampleAttachmentKey_DisplayImmediately).toOpaque(), unsafeBitCast(kCFBooleanTrue, to: UnsafeRawPointer.self))
```

`StackOverflow` 上另一题[这个答案](https://stackoverflow.com/a/33310021)也比较详细。

### error handling

对一个可能抛出异常的方法调用，可以使用 `try`、`try!` 和 `try?` 这三种方式：

1. try
    
    需要在一个 `do-catch` 里使用，或者在这个 `do-catch` 的 surrounding scope 里处理可能抛出的异常，这意味着你也可以不使用 `do-catch`，但这个调用语句的 surrounding scope 要能处理这个异常，否则就会 crash。
    
    ```swift
    var vendingMachine = VendingMachine()
    vendingMachine.coinsDeposited = 8
    do {
        try buyFavoriteSnack(person: "Alice", vendingMachine: vendingMachine)
    } catch VendingMachineError.invalidSelection {
        print("Invalid Selection.")
    } catch VendingMachineError.outOfStock {
        print("Out of Stock.")
    } catch VendingMachineError.insufficientFunds(let coinsNeeded) {
        print("Insufficient funds. Please insert an additional \(coinsNeeded) coins.")
    }
    ```
    
    这个例子里，`do-catch` 结构并没有处理所有的错误，因此如果有其他错误出现，这个错误会被抛给这段代码的 surrounding scope 来处理。
    
2. try!

    有的方法调用会抛出异常，比如说根据 URL 加载一张图片，但是如果这张图片是打包在 app bundle 里的，你知道这肯定不会抛出异常，所以这时候你就可以使用 `try!` 来调用这个方法：
    
    ```swift
    let photo = try! loadImage(atPath: "./Resources/John Appleseed.jpg")
    ```
    
    当然，如果真的有异常发生，那就会 crash 了。
    
3. try?

    类似 `try!`，如果你不想处理异常，但又不想让可能的 crash 发生，就可以使用 `try?`：
    
    ```swift
    func someThrowingFunction() throws -> Int {
    // ...
    }
    
    // 1.
    let x = try? someThrowingFunction()
    
    // 2.
    let y: Int?
    do {
        y = try someThrowingFunction()
    } catch {
        y = nil
    }
    ```
    
    上面的例子中，1 和 2 是等价的，当你使用 `try?` 的时候，可能的异常抛出会被转化为一个 nil 值。

### UnsafeMutablePointer/UnsafeMutableBufferPointer/UnsafeMutableRawBufferPointer

### Collection

### Hashable

### RandomAccessCollection

### weak/unowned

对于属性来说：

1. unowned: 当指向的 class 实例的生命周期 >= 自己时
2. weak: 当指向的 class 实例的生命周期 < 自己时

对于 block 来说：

1. unowned: 当 block 和 class 实例总是互相指向，并且同时 deallocate 时
2. weak: 当 block capture 的 class 实例可能在某个时刻变成 nil 时

### Any/AnyObject

与 OC/Cocoa 交互时，Swift 编译器会自动将 Swift 类型（主要是值类型）转化为 OC 类型（Int -> NSNumber），或者将 OC 类型转化为 Swift 类型（NSString -> String）。这些可以在 OC 和 Swift 之间相互转化的类型被称为 `bridged types`。

因此，在任何可以使用 OC 可桥接类型（bridged types）的地方你都可以用 Swift 里对应的值类型代替。

从 Swift 3 开始，OC 里的 `id` 类型会被映射成 `Any` 类型（而不是 AnyObject），相比映射成 `AnyObject` 类型，这减少了你手动`装箱/拆箱`的操作。

对于集合类型，这个映射规则也同样适用。例如 OC 里的 NSArray/NSDictionary/NSSet 等，之前你只能存放 AnyObject 类型的元素，现在你可以存放 Any 类型的元素。

既然如此，那么是否就应该只用 Any 而不用 AnyObject 呢？当然不是。首先，它仅限于 `bridged types` 类型，所以如果有自定义的 class 类型，那么你只能使用 AnyObject；其次，对于 Swift 这个强类型语言来说，类型越明确，编译器能做的检查更多，可以帮你写出更健壮的代码。

> 参考：[Any vs. AnyObject in Swift 3.0](https://medium.com/@mimicatcodes/any-vs-anyobject-in-swift-3-b1a8d3a02e00)


