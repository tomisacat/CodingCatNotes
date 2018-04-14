SwiftNIO 主要有 6 个类型：

* EventLoopGroup
* EventLoop
* Channel
* ChannelHandler
* Bootstrap
* ByteBuffer
* EventLoopPromise 和 EventLoopFuture

### EventLoop/EventLoopGroup

SwiftNIO 的基本 I/O 原语就是 EventLoop. 它的功能就是等待事件的发生（通常是 I/O 事件，比如说收到数据），然后执行事件对应的回调。大多数 SwiftNIO 应用只需要很少的 EventLoop 实例，通常一个 CPU 核有一到两个就行了。

EventLoop 由 EventLoopGroup 组织管理，它根据一定的机制来分配任务到不同的 EventLoop 上。例如，监听接入连接时，连接建立的 socket 会被注册到一个 EventLoop 上，但是很显然我们不会把所有的连接都注册到同一个 EventLoop 上，否则就会造成一个 EventLoop 负载很大而其他 EventLoop 处于空闲状态的情况发生。

目前，SwiftNIO 有一个 EventLoopGroup 的实现（EventLoopGroup 本身只是一个协议）：MultiThreadedEventLoop，和两个 EventLoop 的实现：SelectableEventLoop/EmbeddedEventLoop。每个 EventLoopGroup 都会通过 `pthread` 创建一系列的线程并分配给 SelectableEventLoop，而 SelectableEventLoop 使用 selector（根据系统的不同，可能是 `kqueue` 或者 `epoll`）来处理 I/O 事件。EmbeddedEventLoop 是个假的 EventLoop，主要是用于测试。

对于一个 SwiftNIO 应用来说，几乎所有的事情都是通过 EventLoop 来完成的，因此，为了确保线程安全，所有的工作都需要通过 EventLoop 来分发。

### Channel/ChannelHandler/ChannelPipeline/ChannelContext

尽管 EventLoop 是 SwiftNIO 最重要的组成部分，但实际上大多数用户除了向 EventLoop 请求创建 EventLoopPromise 外并不会与 EventLoop 打交道。大多数情况下我们主要跟 Channel 和 ChannelHandler 打交道。

在 SwiftNIO 里，几乎所有的文件操作符（fd）都与一个 Channel 一一对应。Channel 持有这个 fd，当 EventLoop 产生了与某个 fd 有关的事件时，它就会向持有这个 fd 的 Channel 发送通知。

Channel 本身不做实际的处理工作，但是它的另一个部分 ChannelPipeline 则非常重要。ChannelPipeline 是一个 ChannelHandler 序列，它们才是实际处理数据的对象。所有的 ChannelHandler 都是 Inbound 或者 Outbound handler（或者兼而有之）：

    * Inbound: 由远端主动建立连接产生的事件
    * Outbound：由本地产生的事件，包括尝试与远端建立连接或者写数据

ChannelHandler 通过 ChannelHandlerContext 来追踪他们在一个 ChannelPipeline 中的位置，它内部保存了 pipeline 里上一个和下一个 ChannelHandler 的引用，这可以确保即使 ChannelHandler 身处一个 pipeline 中依然可以产生事件。SwiftNIO 默认内建了许多 ChannelHandler 实现，例如 HTTP 解析。此外，SwiftNIO 也内建了许多 Channel 的实现，例如：

    * ServerSocketChannel：用来处理 Inbound 的 socket 连接
    * SocketChannel：用来处理 TCP 连接
    * DatagramChannel：处理 UDP 连接
    * EmbeddedChannel：用于测试

需要注意的是，ChannelPipeline **并不是线程安全的**。

### Bootstrap

Bootstrap 的用途就是用来批量创建 Channel 对象。当前 SwiftNIO 提供了三个 Bootstrap 对象：

    * ServerBootstrap：用于创建监听 Channel
    * ClientBootstrap：用于创建客户端 TCP Channel
    * DatagramBootstrap：UDP Channel
    
### ByteBuffer

一个写时复制（copy-on-write）的数据结构。它默认关闭边界检查来提高性能，当然，这也带来潜在的内存错误的问题。

### Promise/Future

EventLoopFuture<T> 是用于存放返回值的容器，每个 EventLoopFuture<T> 都有一个对应的 EventLoopPromise<T>，请求的结果将写入到这个 promise 里，完成后填充到 future 里。



