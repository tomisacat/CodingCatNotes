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


