## Metal Performance Shader (MPS)

优点：

* 底层、高性能
* 为不同的 Metal GPU 簇优化了图形和计算内核函数(`kernel`)的性能
* 可以与 app 里已经申请使用的 Metal 资源（例如：`MTLCommandBuffer`/`MTLTexture`/`MTLBuffer`）或着色器（`shader`）共同使用

功能：

* iOS/tvOS 9 的时候引入了用于图像处理的 kernel 函数对象（处理 `MTLTexture`）
* iOS/tvOS 10 的时候：
    * 引入了卷积神经网络（CNN）用于机器学习
    * 图像颜色转换
    * 矩阵乘法

### MPSKernel

框架 kernel 函数基类，定义了：

* 所有 kernel 函数对象的基础行为（baseline behaviors）
* 声明 kernel 函数对象将要运行到的 Metal 设备（也就是 GPU）
* 调试选项（debugging options）
* 一个便于开发人员调试用的标签（user-friendly label）

此外，对于图像处理，还有两个继承自 MPSKernel 的子类：MPSUnaryImageKernel 和 MPSBinaryImageKernel。他们定义了绝大多数图像处理会用到的一些属性和行为（edging modes, clipping, and tiling support for image operations that consume one or two source textures）。

这三个类不是拿来直接使用的，它们只是定义了抽象的接口。基于它们的子类主要需提供以下定义：

* 特定的初始化方法，用于初始化处理过程中一些需要用到的特定的属性，例如 device（GPU），size（width/height）
* encoding 方法，用来把处理原语（processing primitive）编码到 command buffer 里
* 定义额外的配置属性

### 使用步骤

1. 检测框架是否支持当前设备（GPU）：`MPSSupportsMTLDevice(_:)` 
2. 申请分配计算管线需要用到的 Metal 资源：`MTLDevice`, `MTLCommandQueue`, 以及`MTLCommandBuffer`。
3. 创建要使用的 kernel 函数对象，例如 `MPSImageGaussianBlur`。通常这些 kernel 函数对象都很轻量便于重用，但是不支持多线程并发使用。所以你应该复制这些函数（MPSKernel 遵循 NSCopying 协议）
4. 调用 kernel 函数对象的 encoding 方法。这个对象会创建 command encoder (MTLCommandEncoder，通过 MTLCommandBuffer 创建)，将 kernel 函数写入到 command buffer（MTLCommandBuffer）里
5. 如果需要 encode 更多的指令到 command buffer 里，你需要创建一个新的 MTLCommandEncoder 来完成这件事。
6. 当所有指令都 encoding 后，调用 MTLCommandBuffer 的 `commit` 方法来让 GPU 开始执行这些指令。可以使用 `waitUntilCompleted()` 或 `addCompletedHandler` 方法接收指令执行完成的通知。

每个 kernel 都只与一个 device 绑定，因为在 `init(device:)` 方法中经常需要分配 buffer 或 texture 对象用来存储处理过程中的数据。不过 kernel 同时也提供了 `copy(with:device:)` 方法来为新设备复制一个同样的 kernel 函数对象。

**kernel 对象不是线程安全的。**

### 调优建议

1. 不要等到上一个 command buffer 完成才 enqueue 下一个，也就是尽量不要调用 `waitUntilCompleted` 方法。这样的话当上一个执行完，已经 enqueue 的指令可以立即得到 GPU 资源并开始执行。
2. buffer 和 texture 的分配是很耗时的，应该使用预分配
3. 尽量避免 command encoder 和 command buffer 的切换（不是特别理解这句，跟 Metal best practices 说的有点不一样）
4. 图片链式处理可以用分块（tiling）的方式来加速。


