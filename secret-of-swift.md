### Unamanged

这个结构体类似于中间人，用于在 `C Opaque Pointer`（在 Swift 里为 `Unsafe|Mutable|RawPointer`） 和 `CoreFoundation` 类之间转换:

```
CF Object <- Unmanaged -> Unsafe|Mutable|RawPointer
```

具体来说有以下使用方式：

1. CF Object -> Unmanaged -> UnsafeMutableRawPointer
    
    ```swift
    let cfStr = "This is a string" as CFString
    let bits = Unmanaged.passUnretained(cfStr) // CF Object -> Unmanaged
    let ptr = bits.toOpaque() // Unmanaged -> UnsafeMutableRawPointer
    ```
    
    其中，`passUnretained` 方法将一个 `CF Object` 对象转换为一个 `Unmanaged` 结构体，并且不增加它的引用计数。类似的，也有 `passRetained` 方法实现类似的功能，但会增加这个对象的引用计数。
    
2. UnsafeRawPointer -> Unamaged -> CF Object

    ```swift
    // 假设你有一个 UnsafeRawPointer 对象 ptr
    let unmanagedStr = Unmanaged.fromOpaque(ptr) // UnsafeRawPointer -> Unmanaged
    let str: CFString = unmanagedStr.takeUnretainedValue() // Unmanaged -> CF Object
    ```
    
    其中，`fromOpaque` 方法将一个 `UnsafeRawPointer` 对象转换为 `Unmanaged` 结构体，`takeUnretainedValue` 将这个结构体转换为一个 `CF Object`，并且不增加这个对象的引用计数。类似的，`takeRetainedValue` 会增加这个对象的引用计数。
    
容易看出来上面的转换方法可以自由选择是否增加转换对象的引用计数，Swift 还提供了下面几个方法来增加或减少转换过程中这个中间对象的引用计数：

```swift
public func retain() -> Unmanaged<Instance>
public func release()
public func autorelease() -> Unmanaged<Instance>
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


