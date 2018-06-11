## WWDC 2018: What's New in Swift

### 编译优化：

* 编译模式（Compilation Mode）
    * Debug：Incremental，增量编译，可以加快编译速度
    * Release：Whole Module，优化生成的 SIL，有利于后续生成二进制代码的优化
* 优化等级（Optimization Level）
    * Debug：-Onone，保留调试信息
    * Release：-O，最大化地优化执行速度

### 运行时优化

* 对于引用类型，将原先的采用的 “Owned” 调用惯例修改为 “Guaranteed”，避免不必要的 retain 和 release 操作。这个优化既提高了性能，也减少了代码体积（因为去掉了 retain 和 release 调用）
* 现在 String 使用 16 个字节（之前是 24 个字节）表示，并且最后一个字节用来当作一个“哨兵”，它表示这个字符串是否长度不超过 15 个字节，这样可以将小字符串直接存储在这个 String 实例里，避免不必要的内存分配。

### 新的语言特性

首先，目前 Swift 演进的流程是这样的：

1. 在 forums.swift.org 上提出痛点
2. 编写改进草案
3. 编写代码实现
4. 讨论评估
5. 由核心团队做出决定是否吸收这个提案

* CaseIterable 协议
* Conditional Conformance
    * 标准库默认为 Optional/Array/Dictionary 添加了对 Equatable/Hashable/Codable/Decodable 等协议的支持
* 自动合成 Equatable 和 Hashable 代码
* Hashable 改进
    
    ```swift
    // Swift 4.1
    protocol Hashable {
        var hashValue: Int { get }
    }
    ```
    
    改进为：
    
    ```swift
    // Swift 4.2
    protocol Hashable {
        func hash(into hasher: inout Hasher)
    }
    ```
    
    并且为了提高安全性，hash 种子是在 app 启动时随机生成的，这确保了每次启动 app 都可以生成不同的 hash 值。但是如果你想要在调试时得到一个确定的 hash 值，可以修改 scheme，添加一个名为 **SWIFT_DETERMINISTIC_HASHING** 的环境变量，并设置值为 **1**.
    
* 统一的随机数生成 API
    * random 函数
    * RandomNumberGenerator 协议
* canImport
* hasTargetEnvironment
* Implicitly Unwrapped Optionals(IUO)
    * 现在，使用 IUO 时，要么把它当成一个可选值（as T?），要么需要显式强制解包
    * 不允许作为一个类型的一部分，例如 [Int!]


